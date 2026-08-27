# Advanced Java Interview Questions --- First 20 + 2 Additional

## How to use these notes

These answers are written in an **interview-speaking style** for a 4+
year Java / Spring Boot / Microservices developer.

For each question: - Start with the **Interview Answer**. - Use the
**Internal Working** section if the interviewer asks "how internally?" -
Use the **Code / Example** when they ask for implementation. - Use the
**Important Point** to avoid common interview mistakes.

------------------------------------------------------------------------

# 1. How does `CompletableFuture` work internally? How are dependent stages stored and executed?

### Interview Answer

"`CompletableFuture` represents the result of an asynchronous
computation. Internally, it maintains a completion state and a set of
dependent completion stages. When the current stage completes, those
dependent stages are triggered."

### Code

``` java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> 10);

future.thenApply(x -> x * 2)
      .thenAccept(System.out::println);
```

### Internal Working

Conceptually:

``` text
supplyAsync()
     |
     v
CompletableFuture
     |
     | result = 10
     v
dependent stage: thenApply()
     |
     | result = 20
     v
dependent stage: thenAccept()
```

The dependent stages are represented internally using completion
objects. They are attached to the previous stage and are triggered when
that stage completes.

The important point is that Java does not repeatedly poll the previous
future. Completion of one stage triggers the dependent completion
actions.

If `supplyAsync()` is used without an executor, the asynchronous
supplier normally uses `ForkJoinPool.commonPool()`.

### Blocking Point

`CompletableFuture` supports non-blocking composition, but these calls
are blocking:

``` java
future.get();
future.join();
```

### Interview Punchline

> "`CompletableFuture` builds a completion pipeline where dependent
> stages are registered with previous stages and are triggered when
> those stages complete."

------------------------------------------------------------------------

# 2. What is the difference between `thenApply()`, `thenCompose()`, and `thenCombine()` internally?

### Interview Answer

"I use `thenApply()` for transforming a result, `thenCompose()` when the
next operation itself returns a `CompletableFuture`, and `thenCombine()`
when I want to combine two independent asynchronous computations."

### `thenApply()`

``` java
CompletableFuture<Integer> result =
        CompletableFuture.supplyAsync(() -> 10)
                .thenApply(x -> x * 2);
```

Result:

``` text
CompletableFuture<Integer>
        |
        v
Integer
```

### `thenCompose()`

Suppose:

``` java
CompletableFuture<User> getUser();

CompletableFuture<List<Order>> getOrders(User user);
```

Then:

``` java
getUser()
    .thenCompose(user -> getOrders(user));
```

Without `thenCompose()`, I could end up with:

``` text
CompletableFuture<CompletableFuture<List<Order>>>
```

`thenCompose()` flattens it to:

``` text
CompletableFuture<List<Order>>
```

I use it for **dependent asynchronous calls**.

### `thenCombine()`

``` java
CompletableFuture<User> userFuture =
        CompletableFuture.supplyAsync(() -> getUser());

CompletableFuture<List<Order>> orderFuture =
        CompletableFuture.supplyAsync(() -> getOrders());

CompletableFuture<String> result =
        userFuture.thenCombine(
                orderFuture,
                (user, orders) ->
                        user.getName() + " has " + orders.size() + " orders"
        );
```

Conceptually:

``` text
getUser() --------\
                   > thenCombine() -> result
getOrders() ------/
```

### Quick Comparison

  Method            Purpose
  ----------------- ----------------------------------
  `thenApply()`     Transform a result
  `thenCompose()`   Chain dependent async operations
  `thenCombine()`   Combine independent futures

------------------------------------------------------------------------

# 3. How does `CompletableFuture` use `ForkJoinPool`? What happens with `supplyAsync()` without an Executor?

### Interview Answer

"If I call `supplyAsync()` without providing an Executor, the
asynchronous task normally runs using `ForkJoinPool.commonPool()`. If I
provide an Executor, that executor is used for the asynchronous
supplier."

### Code

``` java
CompletableFuture.supplyAsync(() -> callService());
```

Conceptually:

``` text
Main Thread
    |
    v
ForkJoinPool.commonPool()
    |
    +-- Worker
    +-- Worker
    +-- Worker
```

With a custom executor:

``` java
ExecutorService executor =
        Executors.newFixedThreadPool(10);

CompletableFuture.supplyAsync(
        () -> callService(),
        executor
);
```

### Interview Point

"For blocking I/O, I don't blindly put all work on the common pool. I
prefer an appropriately sized dedicated executor when that isolation is
important."

------------------------------------------------------------------------

# 4. How does `HashMap` work internally? Explain hashing, buckets, collisions, treeification, and resizing.

### Interview Answer

"`HashMap` stores key-value entries in an internal table of buckets.
During `put()` or `get()`, Java uses the key's hash to determine the
bucket. If multiple keys map to the same bucket, that is a collision."

### Example

``` java
Map<Integer, String> map = new HashMap<>();

map.put(101, "Java");
map.put(102, "Spring");
```

Conceptually:

``` text
HashMap table

bucket 0 -> null
bucket 1 -> Entry
bucket 2 -> null
bucket 3 -> Entry -> Entry
bucket 4 -> null
```

### `put()` Flow

``` text
key
 |
 v
hashCode()
 |
 v
hash spreading
 |
 v
bucket index
 |
 +-- empty -> insert
 |
 +-- occupied -> compare keys
                 |
                 +-- equals() true -> replace value
                 |
                 +-- false -> collision structure
```

Modern Java can convert a heavily populated bucket from a linked-node
structure into a red-black tree when the implementation's treeification
conditions are met.

### Resizing

HashMap has a capacity and load factor. With the usual default load
factor of `0.75`, crossing the resize threshold causes the table to
grow.

### Important Correction

Do not say:

> `bucket = hashCode % capacity`

as the exact modern implementation.

In current OpenJDK HashMap implementations, the hash is spread and a
power-of-two table uses bit operations to calculate the index.

### Complexity

Average:

``` text
get() -> O(1)
put() -> O(1)
```

Worst-case behavior depends on collisions; treeified bins improve
heavily collided cases toward logarithmic lookup for comparable keys.

------------------------------------------------------------------------

# 5. Why is `HashMap` not thread-safe, and what can go wrong during concurrent modification?

### Interview Answer

"`HashMap` does not provide synchronization or concurrency guarantees.
If multiple threads access and modify the same HashMap concurrently,
operations can interfere with each other and I can get lost updates,
inconsistent observations, or other undefined application-level
behavior."

### Example

``` java
Map<Integer, Integer> map = new HashMap<>();

// Thread 1
map.put(1, 100);

// Thread 2
map.put(2, 200);
```

The problem is not simply that "two threads use it." Concurrent access
is unsafe when there is unsynchronized conflicting access, particularly
writes.

For concurrent map access:

``` java
Map<Integer, Integer> map =
        new ConcurrentHashMap<>();
```

### Important Modern Point

Do not rely on the old Java 7 explanation that concurrent HashMap
resizing necessarily causes an infinite loop. That is historical
implementation behavior. The correct current answer is that HashMap
provides no thread-safety guarantee.

------------------------------------------------------------------------

# 6. How does `ConcurrentHashMap` achieve thread safety without locking the entire map?

### Interview Answer

"`ConcurrentHashMap` is designed for concurrent access. It does not use
one global lock for every operation. Modern implementations use atomic
operations such as CAS and localized synchronization around contended
bins, allowing different parts of the map to be accessed concurrently."

### Example

``` java
ConcurrentHashMap<Integer, String> map =
        new ConcurrentHashMap<>();

map.put(1, "Java");
map.put(2, "Spring");
```

Conceptually:

``` text
Bucket 1 <- Thread A

Bucket 2 <- Thread B

Bucket 3 <- Thread C
```

The implementation coordinates updates at a much more localized level
than a single lock around the entire map.

### Important

`ConcurrentHashMap` does not allow null keys or null values:

``` java
map.put(1, null); // NullPointerException
```

### Interview Punchline

> "ConcurrentHashMap provides concurrent access with much finer-grained
> coordination than synchronizing the whole map."

------------------------------------------------------------------------

# 7. What happens internally when you create a Java object using `new`?

### Interview Answer

"When I execute `new`, Java allocates storage for the object,
initializes its fields to default values, performs the required
initialization of the class if necessary, executes the constructor, and
returns a reference to the object."

### Example

``` java
Employee e = new Employee();
```

Conceptually:

``` text
new Employee()
      |
      v
allocate object storage
      |
      v
default field initialization
      |
      v
constructor / initialization
      |
      v
reference returned
      |
      v
e
```

Objects are generally allocated on the heap, but the JVM can optimize
allocations using techniques such as escape analysis.

### Important

The reference variable and the object are different things:

``` text
e  --------->  Employee object
reference       object
```

------------------------------------------------------------------------

# 8. How does the JVM allocate memory? Explain Stack, Heap, Metaspace, TLAB, and object allocation.

### Interview Answer

"I mainly think about JVM memory as heap memory, per-thread stacks,
Metaspace, and other native memory. Objects are generally allocated on
the heap, while each thread has its own stack for method execution."

### Stack

Each thread has its own stack:

``` text
Thread
 |
 v
Stack
 +-- method frame
 +-- local variables
 +-- operand stack
 +-- references
```

### Heap

Objects and arrays are generally allocated here:

``` text
Heap
 +-- Employee
 +-- String objects
 +-- arrays
```

### Metaspace

Metaspace stores class metadata and is allocated from native memory
rather than the traditional fixed PermGen area.

### TLAB

"HotSpot can provide each thread with a Thread Local Allocation Buffer,
or TLAB. Small allocations can then often be handled efficiently within
the thread's local allocation area without contending on a shared
allocation path."

### Interview Punchline

> "The exact physical allocation is JVM-implementation dependent, but
> conceptually heap is for objects, stacks are per-thread execution
> memory, and Metaspace holds class metadata."

------------------------------------------------------------------------

# 9. How does Garbage Collection actually work in the JVM? Minor GC, Major GC, and Full GC?

### Interview Answer

"Garbage Collection identifies objects that are no longer reachable from
GC roots and reclaims their memory."

Example:

``` java
Employee e = new Employee();

e = null;
```

If there is no other reference to that Employee object, it may become
eligible for collection.

### Concept

``` text
GC Roots
   |
   +--> reachable objects -> live
   |
   +--> unreachable objects -> reclaimable
```

### Minor GC

Traditionally means a collection focused on the young generation.

### Major GC

The term is not defined uniformly across all collectors and can mean
collection work involving the old generation.

### Full GC

Generally means a collection involving the whole heap, although the
exact behavior depends on the garbage collector.

### Important Interview Correction

Do not say:

> "Minor GC always means Eden and Major GC always means Old Generation."

That is too simplistic. Modern collectors such as G1 use region-based
mechanisms and their terminology does not map perfectly to the old
textbook young/old-generation model.

------------------------------------------------------------------------

# 10. How does G1 GC work internally? Why does it divide the heap into regions?

### Interview Answer

"G1, or Garbage-First GC, divides the heap into many regions instead of
treating the heap as one large contiguous young and old area. It tracks
regions and tries to collect regions that give the most reclaimable
space while respecting its pause-time goals."

Conceptually:

``` text
Heap

[Region][Region][Region][Region]
[Region][Region][Region][Region]
[Region][Region][Region][Region]
```

Regions can be used for different roles, such as:

``` text
Young regions
Old regions
Humongous allocations
```

### Why regions?

"Regions allow G1 to select a subset of the heap for collection instead
of requiring every collection to process the entire heap."

### High-Level Flow

``` text
Application
    |
    v
G1 regions
    |
    +--> young collection
    |
    +--> concurrent marking
    |
    +--> mixed collection
```

G1 also uses remembered sets to track references between regions.

### Interview Punchline

> "The region model gives G1 flexibility to choose which areas to
> collect and helps it target predictable pause times."

------------------------------------------------------------------------

# 11. What is the Java Memory Model, and why do we need it?

### Interview Answer

"The Java Memory Model defines the rules for how threads interact
through shared memory. It tells us when a write by one thread is
guaranteed to become visible to another thread and what ordering
guarantees exist."

### Main Concepts

``` text
Visibility
Ordering
Atomicity
Happens-before
```

Example:

``` java
boolean ready = false;
```

One thread:

``` java
ready = true;
```

Another thread:

``` java
while (!ready) {
}
```

Without the appropriate synchronization, the second thread is not
guaranteed to observe the write as required.

Using:

``` java
volatile boolean ready = false;
```

gives volatile visibility and ordering semantics for this flag.

### Interview Punchline

> "JMM is important because multiple threads can otherwise observe
> shared-memory operations in ways that developers might not intuitively
> expect."

------------------------------------------------------------------------

# 12. How does the `volatile` keyword work internally? What does it guarantee and what does it not guarantee?

### Interview Answer

"`volatile` gives visibility guarantees for reads and writes to that
variable and establishes ordering rules around those accesses. It does
not make arbitrary compound operations atomic."

### Example

``` java
volatile boolean running = true;
```

Thread 1:

``` java
running = false;
```

Thread 2:

``` java
while (running) {
    // work
}
```

The volatile write is visible to subsequent volatile reads according to
the Java Memory Model.

### But this is not atomic:

``` java
volatile int count = 0;

count++;
```

Because:

``` text
read
 |
add
 |
write
```

Two threads can interfere.

For atomic increment:

``` java
AtomicInteger count = new AtomicInteger();

count.incrementAndGet();
```

### What volatile gives

-   Visibility
-   Ordering / happens-before guarantees for volatile accesses

### What volatile does NOT give

-   Mutual exclusion
-   General compound-operation atomicity

### Interview Punchline

> "Volatile is appropriate for visibility and certain state flags, but
> not as a replacement for a lock when I need a multi-step operation to
> be atomic."

------------------------------------------------------------------------

# 13. How does `synchronized` work internally? What are object monitors, monitor enter, and monitor exit?

### Interview Answer

"`synchronized` provides mutual exclusion and memory-visibility
guarantees. A synchronized block is associated with an object's monitor.
A thread must acquire that monitor before entering the critical section
and releases it when leaving."

### Code

``` java
synchronized (lock) {
    balance -= 100;
}
```

Conceptually:

``` text
Thread
   |
   v
monitor enter
   |
   v
critical section
   |
   v
monitor exit
```

If another thread tries to acquire the same monitor while it is held:

``` text
Thread A -> owns monitor

Thread B -> waits to acquire monitor
```

### Synchronized Instance Method

``` java
public synchronized void increment() {
    count++;
}
```

The monitor is associated with the instance (`this`).

### Static Synchronized Method

``` java
public static synchronized void test() {
}
```

The monitor is associated with the corresponding `Class` object.

### Interview Punchline

> "`synchronized` gives me mutual exclusion and visibility with
> automatic lock release when the synchronized block or method exits."

------------------------------------------------------------------------

# 14. What are biased locking, lightweight locking, and heavyweight locking?

### Interview Answer

"These are JVM synchronization implementation concepts rather than three
lock APIs that I directly choose in application code."

Historically, HotSpot discussed:

``` text
Biased locking
      |
Lightweight locking
      |
More expensive monitor contention
```

### Biased Locking

Historically optimized the case where the same thread repeatedly
acquired a lock.

**Important:** biased locking was removed from HotSpot starting with JDK
15, so I would not describe it as an active Java 21 lock mechanism.

### Lightweight Locking

The JVM can optimize low-contention synchronization without immediately
requiring expensive blocking.

### Heavy Contention

When contention increases, threads may have to wait and the JVM can use
monitor/parking mechanisms.

### Interview Punchline

> "I would treat biased, lightweight, and heavyweight terminology as JVM
> implementation details. In modern JDKs, especially Java 21, I focus on
> the semantics of synchronized rather than assuming these historical
> states."

------------------------------------------------------------------------

# 15. How does `ReentrantLock` work internally, and how is it different from `synchronized`?

### Interview Answer

"`ReentrantLock` is an explicit lock implementation built using
`AbstractQueuedSynchronizer`, or AQS. It maintains synchronization state
and coordinates threads waiting to acquire the lock."

### Code

``` java
ReentrantLock lock = new ReentrantLock();

lock.lock();

try {
    // critical section
} finally {
    lock.unlock();
}
```

### Reentrant

The same thread can acquire the lock multiple times:

``` text
Thread A
   |
lock()       -> hold count = 1
   |
lock()       -> hold count = 2
```

It must release it the same number of times:

``` text
unlock() -> 1
unlock() -> 0
```

### Advantages over `synchronized`

``` java
lock.tryLock();

lock.lockInterruptibly();
```

I can also configure fairness:

``` java
new ReentrantLock(true);
```

### Comparison

  -----------------------------------------------------------------------
  `synchronized`                      `ReentrantLock`
  ----------------------------------- -----------------------------------
  Language construct                  Explicit lock API

  Automatic release                   Must explicitly unlock

  No `tryLock()`                      Supports `tryLock()`

  No interruptible lock acquisition   Supports `lockInterruptibly()`
  API                                 

  Simpler                             More control
  -----------------------------------------------------------------------

### Interview Punchline

> "I prefer synchronized when simple mutual exclusion is enough. I
> choose ReentrantLock when I need features such as tryLock,
> interruptible acquisition, or fairness."

------------------------------------------------------------------------

# 16. How does `ThreadPoolExecutor` work internally? What happens when a new task is submitted?

### Interview Answer

"When a task is submitted, ThreadPoolExecutor first tries to create a
worker if the current worker count is below the core pool size. Once the
core size is reached, new tasks are normally queued. If the queue is
full, ThreadPoolExecutor can create additional workers up to the maximum
pool size. If it cannot accept the task after that, the rejection policy
is invoked."

### Example

``` java
ThreadPoolExecutor executor =
        new ThreadPoolExecutor(
                2,
                5,
                60,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(10)
        );
```

Configuration:

``` text
core = 2
max  = 5
queue capacity = 10
```

### Submission Flow

``` text
New Task
   |
   v
workers < core?
   | yes
   v
create worker
```

If core workers already exist:

``` text
New Task
   |
   v
put in queue
```

If queue is full:

``` text
queue full
   |
   v
workers < max?
   |
  yes
   |
create additional worker
```

If max workers are already running and the queue is full:

``` text
RejectedExecutionHandler
```

Common policies:

``` text
AbortPolicy
CallerRunsPolicy
DiscardPolicy
DiscardOldestPolicy
```

### Important

The executor does **not** simply create `maxPoolSize` threads
immediately.

------------------------------------------------------------------------

# 17. What is the internal difference between `Runnable` and `Callable` when used with an Executor?

### Interview Answer

"`Runnable` represents a task that doesn't return a result, while
`Callable` can return a result and can throw checked exceptions."

### Runnable

``` java
Runnable task = () -> {
    System.out.println("Processing");
};

executor.submit(task);
```

### Callable

``` java
Callable<Integer> task = () -> {
    return 100;
};

Future<Integer> future =
        executor.submit(task);

Integer result = future.get();
```

### Comparison

  -----------------------------------------------------------------------
  Runnable                            Callable
  ----------------------------------- -----------------------------------
  No result                           Returns a result

  `run()`                             `call()`

  Cannot directly throw checked       Can throw checked exception
  exception                           

  Can be submitted to Executor        Can be submitted to Executor
  -----------------------------------------------------------------------

### Interview Punchline

> "If I only need execution, I use Runnable. If I need a result or
> checked exception propagation, I use Callable."

------------------------------------------------------------------------

# 18. How do Java Streams work internally? Why are intermediate operations lazy?

### Interview Answer

"Java Stream is not a collection or data structure. It is a pipeline for
processing elements from a source. The pipeline contains a source,
intermediate operations, and a terminal operation."

### Code

``` java
List<Integer> numbers =
        List.of(1, 2, 3, 4, 5);

numbers.stream()
       .filter(x -> x % 2 == 0)
       .map(x -> x * 2)
       .forEach(System.out::println);
```

Pipeline:

``` text
Source
  |
filter
  |
map
  |
forEach  <- terminal operation
```

### Why Lazy?

If I write:

``` java
Stream<Integer> stream =
        numbers.stream()
               .filter(x -> {
                   System.out.println("filter " + x);
                   return x > 2;
               });
```

nothing is processed yet.

`filter()` only contributes to the pipeline.

Processing begins when I invoke a terminal operation:

``` java
stream.forEach(System.out::println);
```

### Internal Processing

For a pipeline:

``` java
numbers.stream()
       .filter(x -> x > 2)
       .map(x -> x * 2)
       .forEach(System.out::println);
```

conceptually:

``` text
element 1 -> filter -> rejected

element 2 -> filter -> rejected

element 3 -> filter -> map -> 6 -> forEach

element 4 -> filter -> map -> 8 -> forEach
```

The stream can fuse multiple operations into one traversal rather than
materializing a new collection after every intermediate operation.

### Why Lazy?

"Lazy evaluation allows the stream to avoid unnecessary work and enables
optimizations such as short-circuiting."

Example:

``` java
boolean found =
        numbers.stream()
               .filter(x -> x > 10)
               .findFirst()
               .isPresent();
```

The pipeline can stop once the terminal operation has enough
information.

### Interview Punchline

> "Intermediate operations build the pipeline; the terminal operation
> triggers traversal and actual processing."

------------------------------------------------------------------------

# 19. How does a parallel stream work internally, and why can it sometimes be slower than a normal stream?

### Interview Answer

"A parallel stream divides the source into smaller portions, processes
those portions concurrently, and then combines the results. It relies
heavily on the source's Spliterator and ForkJoinPool infrastructure."

### Example

``` java
long count =
        numbers.parallelStream()
               .filter(x -> x > 100)
               .count();
```

Conceptually:

``` text
                 Source
                   |
                 split
          /        |        \
       Part 1    Part 2    Part 3
          |        |         |
       worker    worker    worker
          \        |         /
                 combine
                   |
                 result
```

### Spliterator

A parallel stream needs to divide work. The source provides a
`Spliterator` capable of splitting data:

``` java
Spliterator<Integer> spliterator =
        numbers.spliterator();

Spliterator<Integer> part =
        spliterator.trySplit();
```

### Why Can Parallel Be Slower?

Parallelism has overhead:

``` text
split data
   +
create/schedule tasks
   +
thread coordination
   +
combine results
```

For small collections or cheap operations, this overhead can exceed the
performance benefit.

### Other Problems

#### Shared mutable state

Bad:

``` java
List<Integer> result = new ArrayList<>();

numbers.parallelStream()
       .forEach(x -> result.add(x));
```

`ArrayList` is not thread-safe.

Prefer:

``` java
List<Integer> result =
        numbers.parallelStream()
               .toList();
```

#### Blocking I/O

Avoid blindly doing:

``` java
numbers.parallelStream()
       .map(x -> callExternalService(x))
       .toList();
```

The common pool can become occupied by blocking calls.

#### Ordering

Operations that require encounter order can reduce the benefit of
parallel execution.

#### Expensive combination

A reduction can have significant merge/combination overhead.

### Interview Punchline

> "I don't assume parallelStream is automatically faster. I consider
> data size, CPU cost per element, splittability, ordering, shared
> state, and combination cost."

------------------------------------------------------------------------

# 20. How does class loading work in Java? Explain Loading, Linking, Initialization, and Parent Delegation.

### Interview Answer

"When the JVM needs a class, class loading involves three broad phases:
Loading, Linking, and Initialization."

``` text
Loading
   |
   v
Linking
   |
   +-- Verification
   +-- Preparation
   +-- Resolution
   |
   v
Initialization
```

## Loading

The class loader finds the class bytecode and creates the JVM
representation of the class.

``` text
Employee.class
     |
     v
Class Loader
     |
     v
JVM class representation
```

## Linking

### Verification

Checks that the bytecode is structurally valid and satisfies JVM
constraints.

### Preparation

Static fields receive their initial default values and the JVM prepares
the required class-level data.

For:

``` java
static int count = 10;
```

during preparation, the field initially has its default value:

``` text
count = 0
```

The assignment to `10` happens during initialization.

### Resolution

Symbolic references can be resolved to runtime references. Resolution
can occur lazily depending on the JVM implementation.

## Initialization

Static initialization occurs:

``` java
class Employee {

    static int count = 10;

    static {
        System.out.println("initialized");
    }
}
```

During initialization:

``` text
count = 10
static block executes
```

------------------------------------------------------------------------

## Parent Delegation Model

Java class loaders normally delegate to their parent first.

Conceptually:

``` text
Application ClassLoader
          |
          v
Platform ClassLoader
          |
          v
Bootstrap ClassLoader
```

For a platform class such as `java.lang.String`, the request is
delegated upward before the application class loader tries to define the
class itself.

### Why?

It prevents application code from replacing core platform classes with
arbitrary versions.

### Interview Punchline

> "Parent delegation protects core platform classes and helps maintain a
> consistent class-loading hierarchy."

### Important Correction

Do not describe this as an absolute sequence where every class always
goes Bootstrap -\> Platform -\> Application. The important rule is
**child delegates to parent first; if the parent cannot load it, the
child attempts to load it.**

------------------------------------------------------------------------

# 21. What are the hidden pitfalls of parallel streams?

### Interview Answer

"Parallel streams can improve CPU-bound workloads, but there are several
hidden pitfalls. I mainly watch for shared mutable state, blocking I/O,
common-pool contention, ordering requirements, poor splitting, excessive
task overhead, and expensive result combination."

## 1. Shared Mutable State

Bad:

``` java
List<Integer> result = new ArrayList<>();

numbers.parallelStream()
       .forEach(x -> result.add(x));
```

Multiple threads modify the same `ArrayList`.

Prefer:

``` java
List<Integer> result =
        numbers.parallelStream()
               .toList();
```

------------------------------------------------------------------------

## 2. Blocking I/O

``` java
numbers.parallelStream()
       .map(x -> callRemoteAPI(x))
       .toList();
```

The common ForkJoinPool may have workers blocked waiting for
network/database responses.

For service applications, I prefer explicit concurrency design and
appropriate executors for blocking workloads.

------------------------------------------------------------------------

## 3. Common Pool Contention

Parallel streams commonly use the common ForkJoinPool.

If unrelated asynchronous tasks are also using that pool, one workload
can affect another.

------------------------------------------------------------------------

## 4. Small Dataset

For:

``` java
List<Integer> numbers = List.of(1, 2, 3, 4);
```

parallel processing can be slower because:

``` text
split + scheduling + coordination > actual work
```

------------------------------------------------------------------------

## 5. Ordering

Operations that depend heavily on encounter order can reduce parallelism
benefits.

------------------------------------------------------------------------

## 6. Poor Splitting

Not every data source splits equally well.

A source that is difficult or expensive to split can reduce the benefit
of parallel execution.

------------------------------------------------------------------------

## 7. Expensive Combination

For example:

``` java
parallelStream().reduce(...)
```

may need to merge partial results.

If merging is expensive, parallelism may not help.

------------------------------------------------------------------------

## 8. Debugging Complexity

Execution order is not necessarily the same as input order, and multiple
threads make debugging harder.

### Interview Punchline

> "Parallel streams are a tool, not a default optimization. I benchmark
> the real workload before using them."

------------------------------------------------------------------------

# 22. What is thread pinning in Java, especially with virtual threads?

### Interview Answer

"Thread pinning occurs when a virtual thread cannot efficiently unmount
from its carrier platform thread while it is blocked. This can happen
particularly when a virtual thread performs certain blocking operations
while executing inside a synchronized block or method, or during
native/foreign-function execution."

### Virtual Thread Concept

Normally:

``` text
Virtual Thread
      |
      v
Carrier Platform Thread
      |
      v
CPU
```

A virtual thread can execute on a carrier thread and, during supported
blocking operations, unmount so the carrier can execute another virtual
thread.

### Pinning

With a problematic synchronized region:

``` java
synchronized (lock) {
    blockingOperation();
}
```

Conceptually:

``` text
Virtual Thread
      |
      v
Carrier Thread
      |
   BLOCKED
      |
carrier may remain occupied
```

That reduces one of the major scalability advantages of virtual threads.

### Why is this important?

Suppose I create thousands of virtual threads:

``` java
Executors.newVirtualThreadPerTaskExecutor();
```

and many of them become pinned while holding monitors during blocking
operations.

The carrier threads can become tied up, reducing scalability.

### What should I do?

"I avoid long blocking operations inside synchronized sections when
using virtual threads. I can use appropriate `java.util.concurrent`
locks and redesign the critical section so that blocking work happens
outside the lock when possible."

Example:

``` java
lock.lock();
try {
    // short critical section
} finally {
    lock.unlock();
}

blockingOperation();
```

The exact design depends on whether the blocking operation needs to be
protected by the lock.

### Important Modern Point

Virtual threads are designed to work well with blocking I/O. The goal is
not "never block"; the JVM can park/unmount virtual threads for many
blocking operations. The problem is **pinning**, where the virtual
thread cannot be unmounted efficiently.

### Interview Punchline

> "Virtual-thread pinning is when a virtual thread remains tied to its
> carrier during a blocking situation, commonly due to monitor-based
> synchronization or native execution. I avoid long blocking operations
> while holding synchronized monitors."

------------------------------------------------------------------------

# Final Interview Revision Sheet

## CompletableFuture

``` text
thenApply   -> transform
thenCompose -> dependent async call / flatten
thenCombine -> combine independent futures
```

## HashMap

``` text
key
 |
hash
 |
bucket
 |
collision handling
 |
treeification if conditions are met
 |
resize when threshold is crossed
```

## JVM

``` text
Object allocation -> generally Heap
Method execution  -> Stack
Class metadata    -> Metaspace
GC                -> reclaims unreachable objects
G1                -> region-based collector
```

## Concurrency

``` text
volatile        -> visibility + ordering, not compound atomicity
synchronized    -> mutual exclusion + visibility
ReentrantLock   -> explicit lock + advanced controls
ThreadPool      -> reuse workers + control concurrency
```

## Streams

``` text
Source
  |
Intermediate operations
  |
Terminal operation
  |
Actual traversal
```

## Parallel Stream

``` text
Source
  |
Spliterator splitting
  |
parallel tasks
  |
ForkJoinPool infrastructure
  |
combine
```

## Class Loading

``` text
Loading
   |
Linking
   +-- Verification
   +-- Preparation
   +-- Resolution
   |
Initialization
```

## Parallel Stream Pitfalls

``` text
Shared state
Blocking I/O
Common-pool contention
Small workloads
Ordering
Poor splitting
Combination overhead
Debugging complexity
```

## Virtual Thread Pinning

``` text
Virtual Thread
      |
      v
Carrier Thread
      |
blocking while pinned
      |
carrier remains occupied
```

------------------------------------------------------------------------

# One Rule for the Interview

For any of these questions, use this structure:

``` text
1. Definition
2. Small code/example
3. Internal working
4. Real-world use case
5. Limitation / pitfall
```

That makes the answer sound like you **understand the feature rather
than memorized a definition**.
