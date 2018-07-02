# Dynamic vars are not thread local

Dynamic vars may be one of the most misunderstood features of Clojure. As a beginner, I used to consider them as a clojure-flavored version of Java's `ThreadLocal`, and this false belief eventually bite me when I tried to use them this way. The myth is kept alive by many clojure programmers (including skilled ones) and most of related documentation (including [the official one](https://clojure.org/reference/vars)). I think we should stop conflating these two separate concepts and I'll try to explain why.

## What thread locality is, and when is it useful
A thread local variable is a place in memory existing in as many occurences as there are threads to use it. This property implies that every access to a thread local variable can be free of synchronization, because the only thread that will ever interact with a given instance of it is the thread *owning* it.

On the JVM, thread local variables are stored in a map attached to every `Thread` instance, and `ThreadLocal` instances are used as keys to look up the map of the current thread. Accessing a thread local variable is a very fast operation.

At first glance, a mechanism providing a strong guarantee to never leak any data outside of a given thread seems of little interest in our modern, scalable, concurrent programming world. That's why I'll start by enumerating the legitimate use cases I encountered so far. This is not meant to be exhaustive.

### Temporary state
Suppose you wrote a pure function implementing an algorithm that needs a temporary buffer to perform its computation. For instance, let's isolate [the extended arity of `clojure.core/str`](https://github.com/clojure/clojure/blob/131c5f71b8d65169d233b03d39f7582a1a5d926e/src/clj/clojure/core.clj#L554-L559).
```clojure
(defn str [x & ys]
  ((fn [^StringBuilder sb more]
     (if more
       (recur (. sb  (append (str (first more)))) (next more))
              (str sb)))
   (new StringBuilder (str x)) ys))
```
A naive way to implement this function would be to reduce the sequence over string concatenation. This strategy has the benefit of beeing beautifully declarative and the drawback of needlessly producing a significant amount of intermediate strings, increasing pressure on garbage collector. Instead, this algorithm uses `StringBuilder`, a mutable buffer allocating its space on-demand by chunks, reducing the amount of single allocations. The code is more imperative, but it's not really a problem as the function is still pure from the outside.

However, we still allocate and grow a buffer each time we need to perform a new traversal, and it's released for GC as soon as we return. Let's improve this part with this new version.
```clojure
(let [tl (ThreadLocal/withInitial
           (reify java.util.function.Supplier
             (get [_] (StringBuilder.))))]
  (defn str [x & ys]
    (let [^StringBuilder sb (.get tl)]
      (try
        (loop [x x ys ys]
          (. sb append x)
          (if ys
            (recur (first ys) (next ys))
            (. sb toString)))
        (finally (. sb setLength 0))))))
```

This time, the buffer is allocated once per thread, which can be a very small amount in a well-designed application where cpu-bound operations like this are run on a fixed-size thread pool. This technique is applicable each time a temporary buffer is needed, and thread-local buffers can even be shared by several functions (as long as they don't call each other).

It should also be noted that this pattern can be leveraged to implement imperative algorithms, where the state of the computation needs to be tracked along a possibly large lexical scope. In some cases, thread-local state mutation may express intent better than purely functional techniques such as a state monad. The clojurescript compiler is a notable example of a nontrivial program implemented in this programming style.


### Dynamic extent
The extent of a given resource is its temporal span, that is, the period of time during which it is allowed to be used. In clojure, most of the resources we are dealing with have indefinite extent, but in some rare cases we have to manage resources requiring setup and teardown. Such resources are said to have dynamic extent.

A good example of a resource with dynamic extent in the clojure language would be a STM transaction. Because the lifecycle of a STM transaction must be fully managed by the underlying engine, we are not allowed to perform arbitrary updates to refs. Instead, we have to wrap the set of actions to be performed in a `dosync` block, and let the engine take care of running it within an appropriate transactional context. Transactions are never actually exposed, they're just implicitly spanning the synchronous execution of the expression block.

```clojure
(def names (ref []))
(dosync
  (alter names conj "zack")                  ;; ok
  @(future (alter names conj "shelley")))    ;; error, not in transaction
```

What allows the transaction object to never be exposed to the user is that it's stored in a `ThreadLocal` [variable](https://github.com/clojure/clojure/blob/131c5f71b8d65169d233b03d39f7582a1a5d926e/src/jvm/clojure/lang/LockingTransaction.java#L36) during the execution of user code, and cleaned up afterwards. Any STM action performs a lookup to this variable to get the current transaction and updates it to keep track of intents. The variable is invisible to other threads so any attempt to escape synchronous scope of execution is doomed to fail.


### Dynamic scoping
[Dynamic scope](https://stuartsierra.com/2013/03/29/perils-of-dynamic-scope) is the combination of dynamic extent and indefinite scope, that is, the usage of global bindings with finite temporal span. This can be used to convey implicit context along the evaluation of an expression.

```clojure
(defmacro deflocal [name & body]
  `(def ~name
     (ThreadLocal/withInitial
       (reify java.util.function.Supplier
         (get [_] ~@body)))))

(defmacro locally [bindings & body]
  (let [bindings (partition 2 bindings)
        previous (repeatedly gensym)]
    `(let [~@(interleave previous (map first bindings))]
       ~@(map (partial cons '.set) bindings)
       (try ~@body (finally ~@(map (partial list '.set) (map first bindings) previous))))))

(deflocal vat 0.2)

(defn with-taxes [amount]
  (+ amount (* amount (.get vat))))

(with-taxes 50)       ;; => 60.0
(locally [vat 0.1]
  (with-taxes 50))    ;; => 55.0
```

With this example, it should be very clear to you that we crossed the red line of referential transparency and our code is not functional anymore. Don't do that.

However, you still have to be aware of this pattern because it's used in the standard library and some third-party (although it seems to have gone out of fashion, which is a good thing).

In general, you should avoid implicitness at any cost, because it adds a significant mental overhead to reason about programs. When you will have to dig over a hairy problem in an obscure code base, the last thing you'll want to hear about is implicit context. The only benefit of dynamic scoping over explicit arguments is to save a bunch of character strokes when some piece of context is not meant to change frequently, so compare this to the cost of breaking referential transparency and choose wisely.

Good candidates for this pattern are technical data. Examples of technical information include :
* a thread pool manager
* a caching strategy
* a maximum buffer size
All these items only impact the non-deterministic parts of the behavior of your program, so you may want to avoid cluttering business logic with these details and leave them implicit (and no, your database connection does *not* fall in this category).

## What dynamic vars are, and when are they useful
[This post](http://blog.cognitect.com/blog/2016/9/15/works-on-my-machine-understanding-var-bindings-and-roots) does a pretty good job at explaining how dynamic vars work internally. After reading this, if you're still enthusiast at the idea of using them in your production application, I recommend to read the code as well.

Now you should understand why dynamic vars are not thread local, so let's check this.
```clojure
(def ^:dynamic *dyn*)
(binding [*dyn* (reify)]
  (identical? *dyn* @(future *dyn*))) ;; => true
```

Binding conveyance


Implicit binding conveyance = bad idea
Better delegate this task to a custom executor, because that's what manages threads.

## Conclusion

Beware of lazy evaluation !
Don't allocate an arbitrary number of ThreadLocal instances
Check plop.core/local
