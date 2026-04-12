# Java Concurrency Bug Patterns, JMM, and Code Review Guide

## Table of Contents
1. Java Memory Model (JMM) Fundamentals
2. Synchronization Points / Mechanisms
3. Concurrent Data Structures -- Pitfalls
4. Common Concurrency Bug Patterns
5. Code Review Checklist
6. Tools and Techniques
7. Modern Java: Virtual Threads & Structured Concurrency

---

## 1. Java Memory Model (JMM) Fundamentals

### 1.1 What Guarantees Does the JMM Provide?

The Java Memory Model (JSR-133, formalized in Java 5) defines the rules by which the JVM, compiler, and hardware may reorder, cache, or delay memory operations -- and when threads are guaranteed to see each other's writes. Without the JMM, every concurrent Java program would be at the mercy of platform-specific memory ordering.

The JMM guarantees three things **when proper synchronization is used**:

1. **Visibility**: Changes made by one thread become visible to other threads.
2. **Ordering**: The sequence of operations observed by threads is constrained.
3. **Atomicity**: Certain operations (reads/writes of references and most primitives) are atomic.

**Critical nuance**: The JMM provides **no guarantees at all** for programs with data races (accesses to shared mutable state without synchronization). A data-racy program can observe literally any value for a read -- stale, partially constructed, or entirely fabricated by the compiler.

### 1.2 Happens-Before Relationship -- Complete Rules

The happens-before (HB) relation is the core abstraction. If action A happens-before action B, then:
- A's effects are **guaranteed visible** to B
- A is **ordered before** B from the perspective of the JMM

The complete set of HB rules (JLS 17.4.5):

| # | Rule | Description |
|---|------|-------------|
| 1 | **Program order** | Within a single thread, each action happens-before every action that follows it in program order |
| 2 | **Monitor lock** | An unlock on a monitor happens-before every subsequent lock on that same monitor |
| 3 | **Volatile field** | A write to a volatile field happens-before every subsequent read of that same field |
| 4 | **Thread start** | A call to `Thread.start()` happens-before any action in the started thread |
| 5 | **Thread join** | All actions in a thread happen-before another thread successfully returns from `Thread.join()` on that thread |
| 6 | **Thread interruption** | A thread calling `interrupt()` on another thread happens-before the interrupted thread detects the interruption (via `InterruptedException` or `Thread.interrupted()`) |
| 7 | **Finalizer** | The end of a constructor happens-before the start of the finalizer for that object |
| 8 | **Transitivity** | If A happens-before B, and B happens-before C, then A happens-before C |
| 9 | **Final fields** | A write to a final field in a constructor, followed by the constructor completing, happens-before any read of that field (provided `this` does not escape during construction) |
| 10 | **`java.util.concurrent` utilities** | Actions in a thread prior to submitting a `Runnable`/`Callable` to an `ExecutorService` happen-before any action taken by that task; similarly for `CountDownLatch.countDown()` -> `await()`, `CyclicBarrier`, `Semaphore.release()` -> `acquire()`, `Phaser`, `Future.get()`, etc. |

**Transitivity is the most powerful rule.** It allows you to chain guarantees:

```java
// Thread 1                          // Thread 2
x = 42;                              // (no HB yet)
volatileFlag = true;                 // volatile write HB
                                     if (volatileFlag) {    // volatile read
                                         assert x == 42;    // GUARANTEED by transitivity:
                                                            // program-order HB (x=42 -> volatile write)
                                                            // + volatile HB (write -> read)
                                                            // + program-order HB (volatile read -> assert)
                                     }
```

### 1.3 Memory Visibility vs. Atomicity

These are **independent** guarantees that developers frequently confuse.

**Visibility** = "Can Thread B see the write that Thread A made?"
- Without a happens-before edge, Thread B may see a stale cached value indefinitely.
- `volatile` guarantees visibility. `synchronized` guarantees visibility (at unlock/lock boundaries).

**Atomicity** = "Is the operation indivisible?"
- A 64-bit `long` or `double` write is NOT atomic on 32-bit JVMs (can see half-written values). Making it `volatile` fixes this.
- `volatile` guarantees atomicity of **individual** reads and writes, but NOT of compound operations like `count++` (which is read-modify-write: three separate operations).

```java
// BROKEN: volatile gives visibility but NOT atomicity for compound ops
private volatile int count = 0;

public void increment() {
    count++;  // NOT atomic! This is: tmp = count; tmp = tmp + 1; count = tmp;
              // Two threads can both read 5, increment to 6, write 6. Lost update.
}

// FIX: Use AtomicInteger
private final AtomicInteger count = new AtomicInteger(0);

public void increment() {
    count.incrementAndGet();  // Atomic CAS-based operation
}
```

### 1.4 Reordering -- What Can and Cannot Be Reordered

The JVM, JIT compiler, and CPU can reorder instructions for performance. The constraints:

**Within a single thread**: Reordering is allowed as long as the result is identical to sequential execution (**as-if-serial semantics**). The thread itself cannot observe the reordering.

**Across threads**: Without synchronization, virtually any reordering is legal. Examples:

```java
// Thread 1                    // Thread 2
a = 1;                         int r1 = b;
b = 2;                         int r2 = a;
// Legal outcome: r1 = 2, r2 = 0  (Thread 1's writes reordered)
// Legal outcome: r1 = 0, r2 = 1  (Thread 2's reads reordered)
// Legal outcome: r1 = 0, r2 = 0  (both reordered or cached)
// Legal outcome: r1 = 2, r2 = 1  (sequential)
```

**What prevents reordering:**
- `volatile` read/write: prevents reordering across the volatile access (volatile write is like a "release fence"; volatile read is like an "acquire fence")
- `synchronized`: the lock acquire prevents reordering of later operations before it; the unlock prevents reordering of earlier operations after it
- `final` fields: the constructor's writes to final fields cannot be reordered past the end of the constructor (from the perspective of other threads)

**Specific reordering prohibitions for volatile (JMM):**
- A volatile write cannot be reordered with any preceding read or write
- A volatile read cannot be reordered with any following read or write
- A volatile write followed by a volatile read (to the same or different variable) cannot be reordered

### 1.5 As-If-Serial Semantics

The JVM guarantees that **within a single thread**, the result of execution is the same as if all operations were executed in program order -- even though the compiler and CPU may reorder, eliminate, or speculate on instructions.

This means:
- Single-threaded code always "works correctly" regardless of optimizations
- The JMM only restricts reordering **across threads** when synchronization is present
- Without synchronization, the JMM imposes no cross-thread ordering constraints

**Practical consequence**: A bug that only manifests on certain CPUs, under load, or with JIT compilation enabled is almost certainly a missing happens-before edge. The JMM allows the reordering; your code was relying on accidental ordering.

---

## 2. Synchronization Points / Mechanisms

### 2.1 `synchronized` Blocks/Methods

The foundational synchronization mechanism. Provides:
- **Mutual exclusion**: Only one thread holds the monitor at a time
- **Visibility**: All writes before the unlock are visible to threads that subsequently lock the same monitor
- **Reordering barrier**: Operations cannot be reordered out of the synchronized block

```java
// Method-level synchronization (lock on `this` or `ClassName.class` for static)
public synchronized void update(int value) {
    this.value = value;
}

// Block-level synchronization (preferred -- narrower scope, explicit lock object)
public void update(int value) {
    synchronized (this.lock) {
        this.value = value;
    }
}
```

**Pitfalls:**
- Locking on `this` is publicly visible -- external code can `synchronized(yourObject)` and cause contention or deadlock. Prefer a private `final Object lock = new Object()`.
- Locking on a mutable field (`synchronized(list)` where `list` can be reassigned) is a bug.
- `synchronized` is reentrant (same thread can acquire same lock multiple times), which can mask reentrancy bugs.

### 2.2 `volatile` Fields

`volatile` guarantees:
- **Visibility**: Every read sees the most recent write by any thread
- **Atomicity**: Individual reads and writes are atomic (including `long`/`double`)
- **Ordering**: Acts as a memory barrier (prevents reordering across the access)

`volatile` does NOT guarantee:
- **Atomicity of compound operations**: `count++`, `if (flag) { flag = false; }`, check-then-act -- none of these are atomic
- **Mutual exclusion**: Multiple threads can read and write simultaneously

**When to use volatile:**
- Simple flags: `volatile boolean running = true;`
- One writer, multiple readers (no compound operations)
- Publishing an immutable or effectively-immutable object reference
- Double-checked locking (the field MUST be volatile)

```java
// CORRECT use of volatile: simple flag
private volatile boolean shutdownRequested = false;

public void shutdown() { shutdownRequested = true; }
public void doWork() {
    while (!shutdownRequested) {
        // ... do work ...
    }
}

// INCORRECT: volatile cannot protect compound operations
private volatile Map<String, String> cache = new HashMap<>();
// Two threads calling cache.put() concurrently = DATA CORRUPTION
// HashMap is not thread-safe, volatile only protects the reference
```

### 2.3 `java.util.concurrent` Locks

#### ReentrantLock

Same semantics as `synchronized` but with additional capabilities:

```java
private final ReentrantLock lock = new ReentrantLock();

public void update() {
    lock.lock();  // MUST be outside try block
    try {
        // critical section
    } finally {
        lock.unlock();  // ALWAYS in finally
    }
}
```

**Advantages over `synchronized`:**
- `tryLock()` -- non-blocking attempt, with optional timeout
- `lockInterruptibly()` -- can be interrupted while waiting
- Fair mode (`new ReentrantLock(true)`) -- threads acquire in FIFO order (at performance cost)
- Can have multiple `Condition` objects (vs. single wait/notify set)

**When to use**: When you need tryLock, interruptibility, fairness, or multiple conditions. Otherwise, prefer `synchronized` (simpler, less error-prone, JVM can optimize it).

**IMPORTANT for virtual threads**: With virtual threads (JDK 21+), `synchronized` blocks can "pin" the virtual thread to its carrier thread, preventing it from being unmounted. `ReentrantLock` does not have this problem. In virtual-thread-heavy applications, prefer `ReentrantLock`.

#### ReadWriteLock / ReentrantReadWriteLock

Allows concurrent reads, exclusive writes:

```java
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

public String read(String key) {
    rwLock.readLock().lock();
    try {
        return map.get(key);
    } finally {
        rwLock.readLock().unlock();
    }
}

public void write(String key, String value) {
    rwLock.writeLock().lock();
    try {
        map.put(key, value);
    } finally {
        rwLock.writeLock().unlock();
    }
}
```

**Pitfall**: In write-heavy scenarios, `ReadWriteLock` performs worse than a simple lock due to overhead. Only beneficial when reads vastly outnumber writes.

#### StampedLock (Java 8+)

Adds optimistic reading -- no lock is actually acquired for reads, just a "stamp" that is validated after the read:

```java
private final StampedLock sl = new StampedLock();
private double x, y;

public double distanceFromOrigin() {
    long stamp = sl.tryOptimisticRead();  // Non-blocking, no lock
    double currentX = x, currentY = y;   // Read shared state
    if (!sl.validate(stamp)) {            // Check if a write occurred
        stamp = sl.readLock();            // Fall back to read lock
        try {
            currentX = x;
            currentY = y;
        } finally {
            sl.unlockRead(stamp);
        }
    }
    return Math.sqrt(currentX * currentX + currentY * currentY);
}
```

**Critical warnings:**
- NOT reentrant -- a thread that holds a write lock and tries to acquire a read lock will deadlock
- Optimistic reads must validate before using the data -- unvalidated data may be inconsistent or garbage
- Code in the optimistic read section must be **side-effect free**

### 2.4 Atomic Classes and VarHandle

#### AtomicInteger, AtomicLong, AtomicReference, etc.

Lock-free, CAS-based atomic operations:

```java
private final AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();      // atomic increment
counter.compareAndSet(5, 6);    // CAS: set to 6 only if currently 5
counter.updateAndGet(x -> x * 2);  // atomic read-modify-write with lambda
```

**AtomicReference** -- for atomically swapping object references:

```java
private final AtomicReference<Config> config = new AtomicReference<>(defaultConfig);

public void updateConfig(UnaryOperator<Config> updater) {
    config.updateAndGet(updater);  // Atomic read-modify-write
}
```

**LongAdder** (Java 8+): For high-contention counters, outperforms `AtomicLong` significantly by using striped cells:

```java
private final LongAdder requestCount = new LongAdder();
requestCount.increment();          // Low contention per cell
long total = requestCount.sum();   // Aggregates all cells (not atomic snapshot)
```

#### VarHandle (Java 9+)

Low-level memory access with explicit ordering modes. Four modes, from weakest to strongest:

| Mode | Method | Guarantees |
|------|--------|------------|
| **Plain** | `get()`, `set()` | No cross-thread guarantees. Thread-confined data only. |
| **Opaque** | `getOpaque()`, `setOpaque()` | Per-variable coherence, bitwise atomicity, eventual visibility. No cross-variable ordering. |
| **Release/Acquire** | `getAcquire()`, `setRelease()` | Causal ordering. Writes before setRelease are visible after getAcquire. Sufficient for producer-consumer. |
| **Volatile** | `getVolatile()`, `setVolatile()` | Total ordering across all volatile accesses. Sequential consistency. Most expensive. |

```java
// VarHandle for a field
private int value;
private static final VarHandle VALUE;
static {
    VALUE = MethodHandles.lookup()
        .findVarHandle(MyClass.class, "value", int.class);
}

// Release/Acquire pattern (cheaper than volatile)
VALUE.setRelease(this, 42);              // Thread 1: release-write
int v = (int) VALUE.getAcquire(this);    // Thread 2: acquire-read, sees 42
```

**When to use VarHandle**: Library/framework code that needs fine-grained control over memory ordering for performance. Application code should generally stick to `volatile`, `synchronized`, and `Atomic*` classes.

### 2.5 Coordination Utilities

#### CountDownLatch

One-shot barrier. N threads count down; waiting threads proceed when count reaches zero.

```java
CountDownLatch startSignal = new CountDownLatch(1);
CountDownLatch doneSignal = new CountDownLatch(N);

// Worker threads
for (int i = 0; i < N; i++) {
    executor.execute(() -> {
        startSignal.await();  // Wait for go signal
        doWork();
        doneSignal.countDown();
    });
}
startSignal.countDown();  // Let all workers proceed
doneSignal.await();        // Wait for all workers to finish
```

**Pitfall**: `await()` returns `boolean` (whether countdown completed or timeout expired). Ignoring the return value in tests is a common source of false-passing tests.

#### CyclicBarrier

Reusable barrier. N threads wait at the barrier; all proceed when the last one arrives.

```java
CyclicBarrier barrier = new CyclicBarrier(N, () -> {
    // Optional: runs when all threads arrive (by the last arriving thread)
    mergeResults();
});

// Each thread
barrier.await();  // Blocks until N threads have called await()
```

#### Phaser

Flexible, reusable barrier with dynamic registration. Can replace both `CountDownLatch` and `CyclicBarrier`.

#### Semaphore

Controls access to a limited number of resources:

```java
Semaphore pool = new Semaphore(10);  // 10 permits
pool.acquire();  // blocks if no permits available
try {
    useResource();
} finally {
    pool.release();
}
```

### 2.6 CompletableFuture Threading Model

**Critical behavior to understand:**

`thenApply()` (non-async) callbacks execute on:
- The thread that completed the future (if the future is not yet complete when the callback is attached)
- The thread attaching the callback (if the future is already complete)

This is **non-deterministic** and can cause problems:
- If the completing thread is a Netty I/O thread, your callback runs on the I/O thread, potentially blocking I/O processing
- If the completing thread holds a lock, your callback runs with that lock held

`thenApplyAsync()` always executes on the `ForkJoinPool.commonPool()` (or a specified executor), providing predictable thread isolation.

```java
// DANGEROUS: callback may run on an I/O thread
future.thenApply(result -> {
    database.save(result);  // Blocking I/O in possibly wrong thread!
    return result;
});

// SAFE: explicit executor
future.thenApplyAsync(result -> {
    database.save(result);
    return result;
}, dbExecutor);
```

**Other pitfalls:**
- `exceptionally()` swallows exceptions by design -- make sure you actually handle them
- Chains of `thenCompose` can create deep stacks on a single thread (non-async variants)
- Uncompleted `CompletableFuture` chains leak memory -- always ensure completion or cancellation

---

## 3. Concurrent Data Structures -- Pitfalls

### 3.1 ConcurrentHashMap

**Individual operations are atomic. Compound operations are NOT.**

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// BROKEN: check-then-act race condition
Integer val = map.get(key);
if (val == null) {
    map.put(key, 1);       // Another thread may have put a value between get() and put()
} else {
    map.put(key, val + 1); // Another thread may have changed the value between get() and put()
}

// CORRECT: atomic compute
map.compute(key, (k, v) -> v == null ? 1 : v + 1);

// CORRECT: atomic merge
map.merge(key, 1, Integer::sum);

// CORRECT: atomic putIfAbsent
map.putIfAbsent(key, 1);
```

**Compound action pitfalls:**

```java
// BROKEN: size() is an approximation, NOT exact
if (map.size() == 0) {
    // Another thread may have added entries by now
    initializeMap(map);
}

// BROKEN: iterators are weakly consistent -- they may not reflect concurrent modifications
for (Map.Entry<K,V> entry : map.entrySet()) {
    // entries may be added/removed by other threads during iteration
    // iterator will NOT throw ConcurrentModificationException
    // but you may see stale data or miss entries
}
```

**Deadlock with nested compute:**

```java
// DEADLOCK: calling compute on the same map inside compute
map.compute("a", (k, v) -> {
    return map.compute("b", (k2, v2) -> 42);  // May deadlock!
    // compute() locks the bin (hash bucket); if "a" and "b" are in the same bin,
    // this is a self-deadlock. Even in different bins, this is risky.
});

// Also dangerous: calling get() inside compute() on the same map
map.compute("a", (k, v) -> {
    int other = map.get("b");  // Can deadlock if "a" and "b" share a bin
    return v + other;
});
```

**Rule**: Never call any method on a `ConcurrentHashMap` from within a `compute()`/`computeIfAbsent()`/`merge()` lambda on the same map.

### 3.2 CopyOnWriteArrayList

Every write (add, set, remove) copies the entire underlying array. Reads are lock-free and see a snapshot.

```java
// GOOD use case: listeners/observers (rarely modified, frequently iterated)
private final CopyOnWriteArrayList<Listener> listeners = new CopyOnWriteArrayList<>();

// BAD use case: frequently modified list
private final CopyOnWriteArrayList<LogEntry> log = new CopyOnWriteArrayList<>();
log.add(entry);  // Copies entire array on EVERY add -- O(n) per write, massive GC pressure
```

**Iterators are snapshot-based:**
- Iterator reflects the state of the list at the time the iterator was created
- Modifications during iteration are NOT visible to the iterator
- `Iterator.remove()` throws `UnsupportedOperationException`

### 3.3 BlockingQueue Variants

| Implementation | Bounded | Ordering | Notes |
|---------------|---------|----------|-------|
| `ArrayBlockingQueue` | Yes (fixed) | FIFO | Fair mode available. Fixed capacity. |
| `LinkedBlockingQueue` | Optional | FIFO | Separate locks for put/take (higher throughput). Default: unbounded (danger!). |
| `PriorityBlockingQueue` | No | Priority | Natural ordering or Comparator. Not FIFO. |
| `SynchronousQueue` | Zero capacity | Hand-off | Each put() blocks until a take() -- direct handoff. |
| `DelayQueue` | No | By delay | Elements available only after their delay expires. |

**Pitfalls:**
- Unbounded `LinkedBlockingQueue` (the default!) can cause `OutOfMemoryError` if producers outpace consumers
- `offer()` returns `false` on a full bounded queue (non-blocking); `put()` blocks. Ignoring `offer()` return value = lost work.
- `PriorityBlockingQueue` ordering is not stable -- equal-priority elements have no guaranteed order

### 3.4 ConcurrentLinkedQueue / ConcurrentLinkedDeque

Non-blocking, lock-free, unbounded FIFO queue using CAS operations.

**Pitfall**: `size()` is O(n) -- it traverses the entire queue. Never use `size()` for anything performance-sensitive or to check emptiness. Use `isEmpty()` instead.

### 3.5 Collections.synchronizedXxx

```java
List<String> syncList = Collections.synchronizedList(new ArrayList<>());

// Individual operations are synchronized
syncList.add("item");     // Thread-safe
syncList.get(0);          // Thread-safe

// BROKEN: iteration is NOT synchronized
for (String s : syncList) {  // ConcurrentModificationException possible!
    process(s);
}

// CORRECT: manual synchronization for iteration
synchronized (syncList) {  // Must lock on the SAME object
    for (String s : syncList) {
        process(s);
    }
}

// BROKEN: compound operations are NOT atomic
if (!syncList.contains(item)) {
    syncList.add(item);  // Race condition: another thread may add between contains and add
}
```

---

## 4. Common Concurrency Bug Patterns

### 4.1 Race Conditions: Check-Then-Act

```java
// BROKEN: Time-of-check to time-of-use (TOCTOU)
public void transferMoney(Account from, Account to, int amount) {
    if (from.getBalance() >= amount) {        // CHECK
        // Another thread may withdraw between check and act
        from.withdraw(amount);                 // ACT
        to.deposit(amount);
    }
}

// FIX: Atomic check-and-act
public void transferMoney(Account from, Account to, int amount) {
    synchronized (from) {
        synchronized (to) {  // CAUTION: lock ordering needed to prevent deadlock
            if (from.getBalance() >= amount) {
                from.withdraw(amount);
                to.deposit(amount);
            }
        }
    }
}
```

### 4.2 Race Conditions: Read-Modify-Write

```java
// BROKEN: non-atomic read-modify-write
private int counter = 0;
public void increment() {
    counter++;  // Three operations: read, add, write. NOT atomic.
}

// BROKEN: volatile does not help!
private volatile int counter = 0;
public void increment() {
    counter++;  // STILL not atomic. volatile only helps visibility, not atomicity.
}

// FIX: AtomicInteger
private final AtomicInteger counter = new AtomicInteger(0);
public void increment() {
    counter.incrementAndGet();
}
```

### 4.3 Data Races (Shared Mutable State Without Synchronization)

```java
// BROKEN: data race on non-volatile, non-synchronized field
class DataRace {
    private boolean ready = false;  // Not volatile!
    private int value = 0;

    // Thread 1
    public void writer() {
        value = 42;
        ready = true;
    }

    // Thread 2
    public void reader() {
        if (ready) {
            // May print 0! JMM allows reordering of value=42 and ready=true.
            // Also, ready may never appear true (cached in CPU register).
            System.out.println(value);
        }
    }
}

// FIX: make ready volatile (piggyback on volatile's HB guarantee)
private volatile boolean ready = false;
// Now: value=42 HB ready=true (program order)
//      ready=true HB read of ready (volatile HB)
//      => value=42 is visible when ready is read as true (transitivity)
```

### 4.4 Deadlocks

```java
// CLASSIC DEADLOCK: inconsistent lock ordering
// Thread 1: lock(A) -> lock(B)
// Thread 2: lock(B) -> lock(A)

public void transfer(Account from, Account to, int amount) {
    synchronized (from) {           // Thread 1 locks accountA
        synchronized (to) {         // Thread 1 waits for accountB
            // ...                  // Thread 2 locked accountB, waits for accountA
        }                           // DEADLOCK
    }
}

// FIX: Consistent lock ordering using a natural order
public void transfer(Account from, Account to, int amount) {
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;
    synchronized (first) {
        synchronized (second) {
            // Safe: always lock lower-ID account first
        }
    }
}

// SUBTLE DEADLOCK: locking inside concurrent collection compute
ConcurrentHashMap<String, List<String>> map = new ConcurrentHashMap<>();
map.computeIfAbsent("key", k -> {
    // This may deadlock if another thread is also computing on a key
    // that happens to be in the same hash bucket
    return map.getOrDefault("otherKey", List.of());  // DON'T DO THIS
});
```

### 4.5 Livelocks

```java
// LIVELOCK: threads keep responding to each other without making progress
public void politeWorker(Lock lock1, Lock lock2) {
    while (true) {
        lock1.lock();
        if (!lock2.tryLock()) {
            lock1.unlock();  // "After you!"
            // Both threads may release and retry in lockstep forever
            continue;
        }
        try {
            doWork();
            return;
        } finally {
            lock2.unlock();
            lock1.unlock();
        }
    }
}

// FIX: Add randomized backoff
if (!lock2.tryLock()) {
    lock1.unlock();
    Thread.sleep(ThreadLocalRandom.current().nextInt(1, 10));
    continue;
}
```

### 4.6 Starvation

Occurs when a thread cannot access shared resources because other threads monopolize them.

- Using unfair `ReentrantLock` (default) under high contention: newly arriving threads can jump ahead of threads that have been waiting longer.
- Thread priority abuse: setting priorities does not guarantee scheduling behavior and varies by OS.
- Reader starvation in `ReentrantReadWriteLock`: if writers are frequent, readers may be starved (or vice versa depending on fairness mode).

### 4.7 Lost Updates

```java
// BROKEN: lost update with AtomicReference
AtomicReference<List<String>> listRef = new AtomicReference<>(new ArrayList<>());

// Thread 1 and Thread 2 both do this:
List<String> current = listRef.get();
current.add("item");  // MUTATING the shared list! Both threads add to the same list
                       // but if they swap it, one thread's add may be lost

// FIX: use immutable snapshots with CAS
listRef.updateAndGet(list -> {
    List<String> newList = new ArrayList<>(list);
    newList.add("item");
    return newList;
});
```

### 4.8 Double-Checked Locking Done Wrong

```java
// BROKEN (pre-Java 5 style, or missing volatile)
class BrokenSingleton {
    private static BrokenSingleton instance;  // NOT volatile!

    public static BrokenSingleton getInstance() {
        if (instance == null) {                    // First check (no lock)
            synchronized (BrokenSingleton.class) {
                if (instance == null) {             // Second check (with lock)
                    instance = new BrokenSingleton(); // PROBLEM:
                    // The JVM may reorder this as:
                    // 1. Allocate memory
                    // 2. Assign reference to instance (instance is now non-null!)
                    // 3. Call constructor (object not yet fully constructed)
                    // Another thread sees non-null instance, uses partially constructed object
                }
            }
        }
        return instance;
    }
}

// CORRECT: volatile prevents reordering past the write
class CorrectSingleton {
    private static volatile CorrectSingleton instance;

    public static CorrectSingleton getInstance() {
        CorrectSingleton local = instance;  // Single volatile read into local
        if (local == null) {
            synchronized (CorrectSingleton.class) {
                local = instance;
                if (local == null) {
                    instance = local = new CorrectSingleton();
                }
            }
        }
        return local;
    }
}

// BEST: Holder class idiom (lazy initialization, no synchronization needed)
class BestSingleton {
    private static class Holder {
        static final BestSingleton INSTANCE = new BestSingleton();
    }
    public static BestSingleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

### 4.9 Publishing Partially Constructed Objects

```java
// BROKEN: 'this' escapes during construction
class BrokenListener {
    private final int value;

    public BrokenListener(EventSource source) {
        source.registerListener(this);  // 'this' escapes before constructor finishes!
        value = 42;                      // Another thread may see value=0 via the listener
    }

    public void onEvent(Event e) {
        System.out.println(value);  // May print 0!
    }
}

// FIX: Use a static factory method
class SafeListener {
    private final int value;

    private SafeListener() {
        value = 42;
    }

    public static SafeListener create(EventSource source) {
        SafeListener listener = new SafeListener();  // Fully constructed
        source.registerListener(listener);            // Safe to publish
        return listener;
    }
}
```

**Safe publication idioms (from JCIP):**
1. Initialize from a static initializer
2. Store into a `volatile` field or `AtomicReference`
3. Store into a `final` field of a properly constructed object
4. Store into a field guarded by a lock

### 4.10 Mutable Objects as Keys in Concurrent Maps

```java
// BROKEN: mutable key in ConcurrentHashMap
class MutableKey {
    int id;
    public int hashCode() { return id; }
    public boolean equals(Object o) { return o instanceof MutableKey mk && mk.id == id; }
}

MutableKey key = new MutableKey();
key.id = 1;
map.put(key, "value");
key.id = 2;  // hashCode changed! Entry is now in the wrong bucket
map.get(key);  // Returns null -- entry is lost!
```

### 4.11 Non-Atomic Compound Operations on Concurrent Collections

```java
// BROKEN: compound check-then-act on ConcurrentHashMap
ConcurrentMap<String, List<String>> map = new ConcurrentHashMap<>();

// Thread 1 and Thread 2 may both execute the put:
if (!map.containsKey("key")) {
    map.put("key", new ArrayList<>());  // Race: both threads create separate lists
}
map.get("key").add("value");  // One list gets thrown away

// CORRECT:
map.computeIfAbsent("key", k -> new ArrayList<>()).add("value");
// But NOTE: the add("value") is outside the atomic scope of computeIfAbsent.
// If the list itself is shared, it may need its own synchronization.
// Better: use a CopyOnWriteArrayList or synchronize on the list.

// CORRECT with full atomicity:
map.compute("key", (k, v) -> {
    List<String> list = v != null ? v : new ArrayList<>();
    list.add("value");
    return list;
});
```

### 4.12 Trusting size()/isEmpty() on Concurrent Collections

```java
// BROKEN: size() is a moving target
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();

if (map.size() < MAX_SIZE) {
    map.put(key, value);  // By now, size may be >= MAX_SIZE
}

// BROKEN: isEmpty() followed by action
if (!map.isEmpty()) {
    String first = map.values().iterator().next();  // NoSuchElementException possible!
    // Map may have been emptied between isEmpty() and iterator().next()
}
```

### 4.13 ThreadLocal Storage Leaks

```java
// BROKEN: ThreadLocal not cleaned up in thread pool
private static final ThreadLocal<DatabaseConnection> connectionHolder =
    ThreadLocal.withInitial(() -> new DatabaseConnection());

public void handleRequest(Request req) {
    DatabaseConnection conn = connectionHolder.get();
    // ... use connection ...
    // MISSING: connectionHolder.remove()
    // The connection stays associated with this thread FOREVER in a pool
    // => memory leak, stale connection state, cross-request data contamination
}

// CORRECT: always remove in finally
public void handleRequest(Request req) {
    try {
        DatabaseConnection conn = connectionHolder.get();
        // ... use connection ...
    } finally {
        connectionHolder.remove();  // ALWAYS clean up
    }
}

// ALSO BROKEN: ThreadLocal.set(null) does NOT remove the entry
connectionHolder.set(null);  // Entry still exists with null value! Memory leak.

// For virtual threads (JDK 21+): prefer ScopedValue over ThreadLocal
static final ScopedValue<DatabaseConnection> CONNECTION = ScopedValue.newInstance();

ScopedValue.where(CONNECTION, new DatabaseConnection()).run(() -> {
    // Connection is available via CONNECTION.get()
    // Automatically scoped -- no cleanup needed
});
```

---

## 5. Code Review Checklist

### 5.1 Shared Mutable State Identification

- [ ] **Identify all shared mutable fields.** For each field accessed by multiple threads, is there a happens-before edge protecting every access?
- [ ] **Check for hidden sharing.** Does the class escape `this` in the constructor? Are mutable collections returned from getters without defensive copying? Are method parameters stored and later accessed from other threads?
- [ ] **Static mutable state is shared by definition.** Any non-final, non-volatile static field is a red flag.
- [ ] **Thread-safety documentation.** Is the class annotated or documented as `@ThreadSafe`, `@NotThreadSafe`, or `@Immutable`? Are fields annotated with `@GuardedBy("lock")`?

### 5.2 Lock Scope Analysis

- [ ] **Too broad**: Does the synchronized block contain I/O, network calls, or long computations? This kills throughput.
- [ ] **Too narrow**: Is a compound operation (check-then-act) split across multiple synchronized blocks? This introduces race conditions.
- [ ] **Lock object**: Is the lock object `private` and `final`? Locking on `this`, `Class`, or mutable fields is dangerous.
- [ ] **Lock ordering**: If multiple locks are acquired, is the ordering documented and consistent across all code paths?
- [ ] **Lock-try pattern**: Is `lock.lock()` called BEFORE the try block (not inside it)? If `lock()` is inside try and throws, `finally` calls `unlock()` on an unheld lock, causing `IllegalMonitorStateException`.

```java
// CORRECT lock-try pattern
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}

// WRONG: lock() inside try
try {
    lock.lock();  // If this throws, unlock() in finally is wrong
    // critical section
} finally {
    lock.unlock();  // IllegalMonitorStateException if lock() failed
}
```

### 5.3 Atomicity of Compound Operations

- [ ] **Check-then-act on concurrent collections**: `if (!map.containsKey(k)) map.put(k, v)` -- use `putIfAbsent()` or `computeIfAbsent()`.
- [ ] **Read-modify-write**: `counter++` on shared non-atomic fields. Use `AtomicInteger.incrementAndGet()`.
- [ ] **Multiple field updates**: If two related fields must be updated atomically, are they updated within the same critical section? Consider a snapshot POJO in an `AtomicReference`.
- [ ] **ConcurrentHashMap compute nesting**: No calls to the same map from within `compute()`/`merge()` lambdas.

### 5.4 Visibility Guarantees

- [ ] **Is there a happens-before edge?** Trace the HB chain from every write to every read. If you cannot find one, it is a data race.
- [ ] **Volatile is not enough for compound ops.** `volatile` only guarantees visibility of individual reads/writes.
- [ ] **64-bit types**: Are `long` and `double` fields that are read/written from multiple threads either `volatile` or protected by a lock?
- [ ] **Lazy initialization**: Is the DCL pattern used correctly (volatile field, local variable optimization, assignment as last operation)?

### 5.5 Safe Publication

- [ ] **Immutable objects**: Are objects that need no synchronization truly immutable (all fields `final`, no `this` escape, no mutable collections)?
- [ ] **Effectively immutable**: Objects not modified after publication should be published via `volatile`, `final`, or within a lock.
- [ ] **Constructor escape**: Does `this` leak during construction (e.g., registering listeners, starting threads, passing `this` to other methods)?
- [ ] **Collection publishing**: Is a mutable collection published via `Collections.unmodifiableList()` wrapping? The underlying collection must not be modified after publication.

### 5.6 Thread-Safety of Third-Party Code

- [ ] **Is the library thread-safe?** Read the Javadoc. `SimpleDateFormat`, `HashMap`, `ArrayList`, `StringBuilder` -- all NOT thread-safe.
- [ ] **Servlet/Controller state**: Are there mutable instance fields in servlets, Spring controllers, or similar shared-instance classes? These are effectively global shared mutable state.
- [ ] **Connection pools**: Are database connections, HTTP clients, etc., thread-safe or properly pooled?

### 5.7 Proper Shutdown of ExecutorServices

- [ ] **Shutdown is called.** Every `ExecutorService` created must have `shutdown()` or `shutdownNow()` called, typically in a shutdown hook, `@PreDestroy`, or `close()`.
- [ ] **awaitTermination is checked.** After `shutdown()`, call `awaitTermination()` and check its return value. If it returns `false`, tasks are still running.
- [ ] **Cached thread pools with unbounded tasks.** `Executors.newCachedThreadPool()` can create unlimited threads. Use bounded pools in production.

```java
// PROPER shutdown pattern
executor.shutdown();
if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
    List<Runnable> dropped = executor.shutdownNow();
    log.warn("Executor did not terminate. {} tasks dropped.", dropped.size());
    if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
        log.error("Executor still running after shutdownNow.");
    }
}
```

### 5.8 Exception Handling in Concurrent Code

- [ ] **Runnable swallows exceptions.** Exceptions from `Runnable.run()` submitted to an executor are silently lost unless the `Future` is checked or an `UncaughtExceptionHandler` is set.
- [ ] **CompletableFuture exception chains.** Every chain should end with `exceptionally()`, `handle()`, or `whenComplete()` to avoid silent failures.
- [ ] **Thread.setUncaughtExceptionHandler.** Is there a global or per-thread handler for unexpected exceptions?

```java
// BROKEN: exception silently swallowed
executor.submit(() -> {
    throw new RuntimeException("Oops");  // Nobody ever sees this
});

// FIX: check the Future
Future<?> f = executor.submit(() -> {
    throw new RuntimeException("Oops");
});
try {
    f.get();  // Re-throws the exception wrapped in ExecutionException
} catch (ExecutionException e) {
    log.error("Task failed", e.getCause());
}

// FIX: Use execute() instead of submit() for Runnable (exception goes to UncaughtExceptionHandler)
executor.execute(() -> {
    throw new RuntimeException("Oops");  // Goes to UncaughtExceptionHandler
});
```

---

## 6. Tools and Techniques for Finding Concurrency Bugs

### 6.1 Static Analysis

#### SpotBugs (successor to FindBugs)

Detects concurrency bugs via bytecode analysis:
- **IS2_INCONSISTENT_SYNC**: Field accessed both inside and outside synchronized blocks
- **DC_DOUBLECHECK**: Broken double-checked locking
- **UG_SYNC_SET_UNSYNC_GET**: Synchronized setter but unsynchronized getter
- **LI_LAZY_INIT_STATIC**: Lazy initialization of static field not thread-safe
- **WL_USING_GETCLASS_RATHER_THAN_CLASS_LITERAL**: Synchronizing on `getClass()` (subclasses get different locks)

```xml
<!-- Maven integration -->
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.6.6</version>
</plugin>
```

#### Error Prone (Google)

Compile-time bug detection:
- **DoubleCheckedLocking**: Incorrect DCL patterns
- **SynchronizeOnNonFinalField**: Locking on a field that can be reassigned
- **NonAtomicVolatileUpdate**: `volatileField++` or `volatileField = volatileField + 1`
- **GuardedBy**: Accesses to `@GuardedBy` fields without holding the specified lock

#### IntelliJ IDEA Inspections

Built-in concurrency inspections:
- "Synchronization on a non-final field"
- "Access to field without synchronization"
- "Double-checked locking"
- "Non-atomic operation on volatile field"
- "Busy wait" (loop with `Thread.sleep()`)
- "Synchronized method can be replaced with block"

### 6.2 Dynamic Analysis

#### jcstress (Java Concurrency Stress Tests)

OpenJDK project for empirically testing JMM behaviors. Runs code under extreme contention on real hardware to expose ordering/visibility bugs.

```java
@JCStressTest
@Outcome(id = "0, 0", expect = Expect.ACCEPTABLE, desc = "Both reads before writes")
@Outcome(id = "1, 1", expect = Expect.ACCEPTABLE, desc = "Both reads after writes")
@Outcome(id = "1, 0", expect = Expect.ACCEPTABLE, desc = "Read x after write, read y before")
@Outcome(id = "0, 1", expect = Expect.ACCEPTABLE_INTERESTING, desc = "Reordering!")
@State
public class ReorderingTest {
    int x, y;

    @Actor
    public void actor1(II_Result r) {
        x = 1;
        r.r2 = y;
    }

    @Actor
    public void actor2(II_Result r) {
        y = 1;
        r.r1 = x;
    }
}
```

**When to use**: Verifying low-level concurrent algorithms, testing custom lock-free data structures, validating JMM assumptions.

#### Java Flight Recorder (JFR)

Built into the JVM. Records lock contention, thread states, and synchronization events with near-zero overhead.

```bash
# Start recording
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar

# Analyze with JDK Mission Control (JMC)
jmc recording.jfr
```

Key events for concurrency:
- `jdk.JavaMonitorWait`: Threads waiting on monitors
- `jdk.JavaMonitorEnter`: Lock contention (time spent waiting for synchronized)
- `jdk.ThreadPark`: Thread parking (Lock/Semaphore contention)
- `jdk.ThreadSleep`: Sleep calls
- `jdk.VirtualThreadPinned`: Virtual thread pinning events (JDK 21+)

#### Thread Dumps

```bash
# From command line
jstack <pid>

# Programmatic
Thread.getAllStackTraces()  // or ManagementFactory.getThreadMXBean()
```

Thread dumps show deadlocks (the JVM detects and reports `synchronized` deadlocks), lock ownership, and thread states. Take multiple dumps seconds apart to identify threads stuck in the same location.

### 6.3 Stress Testing Patterns

```java
// Pattern: multiple threads hammer shared state, then verify invariants
@Test
public void testConcurrentIncrements() throws Exception {
    final int THREADS = 100;
    final int ITERATIONS = 10_000;
    CountDownLatch startGate = new CountDownLatch(1);
    CountDownLatch endGate = new CountDownLatch(THREADS);

    for (int i = 0; i < THREADS; i++) {
        new Thread(() -> {
            try {
                startGate.await();  // All threads start simultaneously
                for (int j = 0; j < ITERATIONS; j++) {
                    sharedCounter.increment();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                endGate.countDown();
            }
        }).start();
    }

    startGate.countDown();  // Release all threads
    endGate.await(30, TimeUnit.SECONDS);

    assertEquals(THREADS * ITERATIONS, sharedCounter.get());
}
```

**Key principles:**
- Use `CountDownLatch` to synchronize thread start (maximize interleaving)
- Use more threads than CPU cores
- Run many iterations
- Verify invariants after all threads complete
- Test under `-server` JIT compilation (optimizations cause reordering)
- Run on multiple CPU architectures if possible (ARM is weakly ordered, x86 is strongly ordered -- bugs may only appear on ARM)

### 6.4 Additional Tools

| Tool | Type | Focus |
|------|------|-------|
| **Meta Infer** | Static | Thread-safety violations, race conditions, deadlocks |
| **Checker Framework** | Static (annotations) | `@GuardedBy`, `@Immutable`, `@ThreadSafe` verification at compile time |
| **Lincheck** (JetBrains) | Testing | Linearizability testing for concurrent data structures |
| **ThreadSafe** (Contemplate) | Static | Dedicated Java concurrency analyzer (commercial) |
| **Intellij Race Detector** | Dynamic | Experimental race condition detection |

---

## 7. Modern Java: Virtual Threads and Structured Concurrency

### 7.1 Virtual Threads (JDK 21+, GA)

Virtual threads are lightweight, JVM-managed threads that enable the thread-per-request model at massive scale (millions of threads). They are designed for I/O-bound workloads.

```java
// Creating virtual threads
Thread.startVirtualThread(() -> handleRequest(request));

// ExecutorService with virtual threads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    futures = requests.stream()
        .map(req -> executor.submit(() -> handleRequest(req)))
        .toList();
}
// ExecutorService is AutoCloseable -- close() calls shutdown + awaitTermination
```

**Concurrency pitfalls specific to virtual threads:**

1. **Pinning**: `synchronized` blocks/methods prevent virtual thread unmounting. The carrier thread is blocked.
   ```java
   // BAD: virtual thread gets pinned
   synchronized (lock) {
       socket.read(buffer);  // Carrier thread is blocked for the entire I/O
   }

   // GOOD: ReentrantLock allows unmounting
   lock.lock();
   try {
       socket.read(buffer);  // Virtual thread can unmount during I/O
   } finally {
       lock.unlock();
   }
   ```

2. **ThreadLocal abuse**: Virtual threads are cheap, so you may have millions. Each `ThreadLocal` consumes memory per thread. Use `ScopedValue` instead.

3. **Thread pools are an anti-pattern**: Don't pool virtual threads. Create a new one per task. Pooling them defeats their purpose and limits scalability.

4. **CPU-bound work**: Virtual threads provide no benefit for CPU-bound work. The carrier thread pool (ForkJoinPool) is limited to the number of CPU cores. Use platform threads for CPU-heavy computation.

### 7.2 Structured Concurrency (Preview, evolving through JDK 25)

Treats a group of concurrent tasks as a unit:

```java
// Java 25 API (preview)
try (var scope = StructuredTaskScope.open(Joiner.awaitAll())) {
    Subtask<String> userTask = scope.fork(() -> fetchUser(userId));
    Subtask<List<Order>> ordersTask = scope.fork(() -> fetchOrders(userId));
    scope.join();  // Wait for both

    // Both tasks completed (successfully or with exception)
    String user = userTask.get();
    List<Order> orders = ordersTask.get();
    return new UserProfile(user, orders);
}
// Scope close ensures all forked tasks are completed or cancelled
```

**Joiners** control completion behavior:
- `Joiner.awaitAll()` -- wait for all tasks
- `Joiner.allSuccessfulOrThrow()` -- fail fast if any task fails
- `Joiner.anySuccessfulResultOrThrow()` -- return first success, cancel others

**Key benefits:**
- Automatic cancellation of remaining subtasks on failure
- Thread lifetime is bounded by the scope (no leaked threads)
- Observability: thread dumps show the task hierarchy

### 7.3 ScopedValue (Preview, evolving through JDK 25)

Replacement for `ThreadLocal` in virtual-thread contexts:

```java
private static final ScopedValue<RequestContext> CONTEXT = ScopedValue.newInstance();

public void handleRequest(Request req) {
    ScopedValue.where(CONTEXT, new RequestContext(req)).run(() -> {
        // CONTEXT.get() is available here and in all called methods
        // Automatically inherited by virtual threads forked in StructuredTaskScope
        processRequest();
    });
    // CONTEXT is automatically unbound here -- no cleanup needed
}
```

**Advantages over ThreadLocal:**
- Immutable within scope (cannot be modified, only shadowed in nested scopes)
- Automatically inherited by child virtual threads
- No memory leaks (scoped lifetime)
- Better performance (no hash map lookup per thread)

---

## Key References

- Goetz et al., *Java Concurrency in Practice* (2006) -- foundational; still relevant for core concepts
- JLS Chapter 17 (Threads and Locks) -- the formal specification
- Doug Lea, *Using JDK 9 Memory Order Modes* -- VarHandle ordering modes
- [code-review-checklists/java-concurrency](https://github.com/code-review-checklists/java-concurrency) -- comprehensive review checklist
- [OpenJDK jcstress](https://github.com/openjdk/jcstress) -- concurrency stress testing
- JEP 444 (Virtual Threads), JEP 453 (Structured Concurrency), JEP 446 (Scoped Values)

## Sources

- [Mastering the Java Memory Model (JMM) -- 2025 Deep Dive](https://codefarm0.medium.com/mastering-the-java-memory-model-jmm-visibility-reordering-volatile-demystified-2025-deep-aafad27c8bcf)
- [Java Memory Model and Happens-Before Relationships Explained](https://prgrmmng.com/java-memory-model-happens-before)
- [Common Concurrency Pitfalls in Java | Baeldung](https://www.baeldung.com/java-common-concurrency-pitfalls)
- [Java Concurrency Code Review Checklist](https://github.com/code-review-checklists/java-concurrency)
- [Code Review Checklist: Java Concurrency | freeCodeCamp](https://www.freecodecamp.org/news/code-review-checklist-java-concurrency-49398c326154/)
- [The "Double-Checked Locking is Broken" Declaration](https://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)
- [Double-Checked Locking with Singleton | Baeldung](https://www.baeldung.com/java-singleton-double-checked-locking)
- [Using JDK 9 Memory Order Modes (Doug Lea)](https://gee.cs.oswego.edu/dl/html/j9mm.html)
- [Virtual Threads | Oracle Documentation](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
- [Java 25: Virtual Threads with Structured Task Scopes and Scoped Values](https://javapro.io/2025/12/23/java-25-getting-the-most-out-of-virtual-threads-with-structured-task-scopes-and-scoped-values/)
- [Is Java Concurrency in Practice Still Valid in 2024?](https://medium.com/javarevisited/is-java-concurrency-in-practice-still-valid-8bb54fc3fb7f)
- [ThreadLocal Memory Leaks | DEV Community](https://dev.to/xuan_56087d315ff4f52254e6/your-threadlocal-is-secretly-leaking-memory-fix-it-before-it-crashes-4acb)
- [CompletableFuture: thenApply vs thenApplyAsync | Baeldung](https://www.baeldung.com/java-completablefuture-thenapply-thenapplyasync)
- [The Misunderstood Thread Safety of ConcurrentHashMap](https://codefarm0.medium.com/the-misunderstood-thread-safety-of-concurrenthashmap-edf86427a641)
- [SpotBugs](https://spotbugs.github.io/)
- [OpenJDK jcstress](https://github.com/openjdk/jcstress)
- [StampedLock: Optimized Read-Heavy Locking](https://prgrmmng.com/stampedlock-optimized-read-heavy-locking)
- [JEP 453: Structured Concurrency (Preview)](https://openjdk.org/jeps/453)
- [Hunting Java Concurrency Bugs | InfoQ](https://www.infoq.com/articles/Hunting-Concurrency-Bugs-1/)
