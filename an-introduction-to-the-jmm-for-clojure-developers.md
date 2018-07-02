# An introduction to the JMM for clojure developers

This article aims to provide a practical guide to the semantics described in the chapter 17 of the [Java Language Specification](https://docs.oracle.com/javase/specs/) and its implications for the average clojure developer.

At the very least, I recommend reading the documentation of the java.util.concurrent package, especially [the section related to memory visibility](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/package-summary.html#MemoryVisibility).

A *variable* is an identifier for a place in shared memory. On the JVM, shared memory lives in the heap so the name *variable* will refer to any object field or array element.

## Unsynchronized
Unsynchronized is the default behavior for variables in the java language. It's actually so common it doesn't even have a name (AFAICT *unsynchronized* was coined by clojure, and *non-volatile* is used sporadically as a synonym in java documentation). Unsynchronized variables are optimized for sequential access and require explicit synchronization to be safely shared across threads. This property plays a major role in the runtime optimization process, because a given thread can manipulate its own private copy of each variable between synchronization points, allowing aggressive reordering strategies.

If you have any experience with clojure, you already know that introducing unsynchronized variables at the language level is a hard endeavour. If you're wondering why, you'll be pleased to learn that [the reason](https://groups.google.com/forum/#!topic/clojure/PCKzXweeDeY) is *not* technical (TLDR: if it were easy, users would do it).

I'm still to be conviced of the rightness of making it deliberately hard to achieve, but I'm fine with this as long as it *can* be made easy with third-party tooling. What I'm not fine with is people arguing that it should be impossible for users to use dangerous features. Dangerous features allow speed and expressivity.

So right now if you need to explicitly represent unsynchronized variables, the core library provides the following tools :
* interop with java collections, including plain arrays
* deftype fields tagged with `^:unsynchronized-mutable`

However, introducing unsynchronized variables at the bytecode level is surprisingly easy. For instance, each time you close over local bindings in clojure, the compiler will generate a class and store captured values in unsynchronized fields. It is a well-known fact that closures can be safely shared across threads in clojure, so let's see what makes this possible.


### Futures
```clojure
(let [i 41] @(future (inc i)))       ;; => 42
```

The evaluation of this snippet involves two threads : thread A, running the top-level form, and thread B, running the future body.

`future` closes over `i`, so a class will be generated with an unsynchronized field holding the runtime value of `i` and a method holding the code to execute.
This field is assigned in thread A and dereferenced in thread B.

`@` blocks thread A until the result is made available by thread B. This behavior is delegated to `java.util.concurrent.FutureTask/get`, which also uses an unsynchronized field to store the outcome of the computation. This field is assigned in thread B and dereferenced in thread A.

To convince ourselves that the correct value of `i` will be visible by thread B, and the correct result of `inc` visible by thread A, let's look at the documentation of the concurrency primitives involved, and see what guarantees they provide in terms of happen-before. The part of interest to us lies in the javadoc of `java.util.concurrent.ExecutorService` :
> Memory consistency effects: Actions in a thread prior to the submission of a Runnable or Callable task to an ExecutorService happen-before any actions taken by that task, which in turn happen-before the result is retrieved via Future.get().

This gives us the following HB ordering :
* (A) create a closure object, assign 41 to its field i and submit it to Agent.soloExecutor
* (B) dereference field i of closure object, call increment on value and assign result to FutureTask
* (A) return result from Future.get()

And this is sufficient to prove that 41 will be visible to B and that 42 will be visible to A.

### Agents
An agent is concurrency primitive ensuring to perform each step at most once, making it a good choice for side-effecting updates (as opposite to atoms and refs, which are retry-based). This property allows usage of mutable (and performant) datastructures to keep track of state.

For instance, transients are safe to use with agents :
```clojure
(def a (agent (transient #{})))
(dotimes [i 20] (send a conj! i))
```
Of course, any deref on the agent may return an inconsistent state because a subsequent message could have updated the transient in-place. Documentation on transients is clear about it, we have been warned.

Transients store intermediate values in plain arrays, so let's make sure two successive updates to the same agent (which may happen on two different threads) are memory consistent. Formally, we need to prove that write actions performed in step N happen before read actions performed in step N+1.

An agent is associated with two variables :
* the current queue of pending messages (an AtomicReference holding a PersistentQueue)
* the current state (a volatile field to allow polling, but the proof doesn't require it)

A message is a tuple [function, arguments, executor]

A message send is a concurrent operation performing the following steps :
* enqueue message
* if queue was empty before enqueuing, submit processing of message to its executor

A message is processed with the following steps :
* pick first message in queue
* apply message function to current state along with message arguments and update current state with returned value
* dequeue message
* if queue has still at least one message after dequeuing, submit processing of this message to its executor


### Channels
```clojure
(def c (a/chan 1 (partition-all 2)))
(a/onto-chan c [:a 1 :b 2 :c 3])
(a/<!! (a/into {} c))
```


## Volatile
A volatile variable is a special kind of variable optimized for concurrent reads. Marking a variable volatile informs the JVM that it can be modified by any thread, at any time. This is a lightweight form of synchronization providing the same visibility guarantees as monitors, without the mutex overhead.

In clojure, every reference type (vars, atoms, refs, agents) stores its current value in a volatile variable. In general, it's a good idea to make a variable volatile if it's meant to be exposed to many readers, so that any thread can efficiently poll it at any time without extra synchronization care (you still need to care about concurrent writes, of course).

A common misconception about volatile variables is to think they're required to make mutable state visible across threads, typically in the design of stateful sequential processes. In this situation, a volatile doesn't provide any useful information to the JVM and introduces unnecessary overhead. If state is not meant to be exposed to an arbitrary number of concurrent reads, the correct way to handle it is to store it unsynchronized and ensure each step happens-before the next, so that writes performed by the thread doing step n are made visible to the thread doing step n+1.

### Transducers

```clojure
(deftype Dedupe [rf ^:unsynchronized-mutable pv]
  clojure.lang.IFn
  (invoke [_] (rf))
  (invoke [_ result] (rf result))
  (invoke [_ result input]
    (let [prior pv]
      (set! pv input)
      (if (= prior input)
        result
        (rf result input)))))

(def unsynchronized-dedupe #(->Dedupe % :clojure.core/none))
(def volatile-dedupe (dedupe))

(def input (shuffle (for [i (range 200) j (range i) k (range j)] k)))

(require '[criterium.core :refer [quick-bench]])
(quick-bench (into [] unsynchronized-dedupe input))
(quick-bench (into [] volatile-dedupe input))
```

The volatile version is consistently between 25% and 50% slower with default settings.

### Transients
TODO perf test with transients

### Go blocks
TODO perf test with go blocks


## Final
Final provides additional guarantees in face of data races : after an object is constructed, its final fields are guaranteed to be assigned.

It has been [said](https://twitter.com/alexelcu/status/991080080104357889) that clojure doesn't support java's final semantics, which is partially correct. Fields of closures holding captured local bindings are non-final, but fields of `deftype` not tagged as mutable are final. However, if the memory consistency of your program relies on the `final` semantics, that means a data race is used somewhere to pass an object across threads, which is bad design.

```clojure
(defn final-test []
  (let [a (object-array 1)]
    ((fn f [] (aset a 0 #(f))))
    (dotimes [i 20]
      (.start (Thread. #(loop [n 0]
                          (when (zero? n) (prn i))
                          ((aget a 0)) (recur (mod (inc n) 10000000))))))))
```
