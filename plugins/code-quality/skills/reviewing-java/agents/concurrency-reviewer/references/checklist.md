# Java Concurrency Bug Patterns, JMM, and Code Review Guide

## Table of Contents
1. Java Memory Model (JMM) Fundamentals -- happens-before, reordering, Herlihy-Shavit consensus hierarchy & progress conditions, the canonical jcstress invariants
2. Synchronization Points / Mechanisms -- `synchronized`, `volatile`, `j.u.c` locks, atomics, VarHandle, coordination utilities, CompletableFuture
3. Concurrent Data Structures -- Pitfalls -- `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`, `ConcurrentLinkedQueue`, `Collections.synchronizedXxx`
4. Common Concurrency Bug Patterns -- 18 patterns from check-then-act to torn reads, incoherent reads, unsafe publication, ABA, false sharing, `ThreadLocal` leaks
5. Code Review Checklist -- shared state, lock scope, atomicity, visibility, safe publication, third-party code, shutdown, exceptions, interruption, parallel streams, locking-strategy selection, invariant-driven review, virtual threads, liveness hardening, primitive hygiene, collection patterns beyond CHM, async-boundary hygiene, executor backpressure, architectural reframing, scale & mechanical sympathy, production diagnosis, cross-paradigm & distributed boundaries, security & compliance, build-tool transformations, review process & team practice
6. Tools and Techniques -- static (SpotBugs, Error Prone, IDEA), dynamic (jcstress, JFR, thread dumps, Fray), stress patterns
7. Modern Java: Virtual Threads & Structured Concurrency -- JDK 21 baseline, JEP 491 pinning fix (JDK 24), structured concurrency evolution (JEP 453→525), ScopedValue final (JEP 506)
8. Research Questions to Cross-Check Against Sources -- 58 concrete questions the checklist answers (sections A–K), each mapped to sections and sources
9. Sources -- normative, foundational, tooling, checklists, industry case studies
10. Gotchas -- 35 common wrong assumptions with stable IDs (G-01..G-35)

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

### 1.6 Consensus Hierarchy and Progress Conditions (Herlihy-Shavit)

Herlihy's theorem (1991; *Art of Multiprocessor Programming* chs. 5–6) assigns every synchronization primitive a **consensus number** — the maximum number of threads for which it can solve wait-free consensus. The hierarchy is strict: a primitive at level *n* cannot build a wait-free object at level > *n* from any number of copies.

| Primitive | Consensus number | Review implication |
|---|---|---|
| Plain read/write (atomic register) | **1** | Alone, cannot build a wait-free queue, stack, or set for 2+ threads (Corollary 5.2.1). `volatile` alone is insufficient for lock-free data structures. |
| `getAndSet`, `getAndIncrement`, `getAndAdd`, FIFO queue, stack, set | **2** | Two-thread consensus only. Cannot be composed into wait-free *n*-thread algorithms. |
| `compareAndSet` / LL-SC | **∞** | Universal primitive for wait-free construction of any object (Theorem 5.8.1). "The king of all wild things." — Herlihy-Shavit |

**Review rule**: if a diff implements a lock-free or wait-free data structure using only `AtomicInteger.getAndIncrement()` or `AtomicReference.getAndSet()` without `compareAndSet`, it almost certainly does not scale past two threads correctly. Flag it.

**Progress condition hierarchy** (strict subset relation, strongest to weakest):

| Condition | Definition | Example |
|---|---|---|
| **Wait-free** | Every method call finishes in a bounded finite number of steps, regardless of other threads | `AtomicInteger.get()`, `Thread.currentThread()` |
| **Lock-free** | At least one thread always makes progress in a finite number of steps (system-wide progress, not per-thread) | Michael-Scott lock-free queue; `ConcurrentLinkedQueue` |
| **Obstruction-free** | A thread finishes in a finite number of steps if it eventually runs alone (no contention) | Software Transactional Memory; `StampedLock` optimistic read |
| **Starvation-free** | Every thread that tries to acquire a lock eventually does (assuming fair scheduler) | `new ReentrantLock(true)` (fair mode); `synchronized` in practice |
| **Deadlock-free** | Some thread eventually makes progress from any state (assuming fair scheduler) | `synchronized` under OS assumption |

**Review rule**: match the progress claim in the Javadoc/comments to the actual primitive. Code claiming "lock-free" but using `synchronized` or `ReentrantLock` is mislabeled. Code claiming "wait-free" but looping on a CAS retry is lock-free, not wait-free.

Source: Herlihy & Shavit, *The Art of Multiprocessor Programming*, Revised Reprint, §5 "The Relative Power of Primitive Synchronization Operations" (Definitions 5.1.1–5.1.2, Theorems 5.2.1, 5.4.1, 5.6.1, 5.7.1, 5.8.1).

### 1.7 Core JMM Invariants (jcstress BasicJMM series)

The OpenJDK jcstress project publishes a canonical set of stress tests demonstrating what the JMM does and does not guarantee. Each named invariant below corresponds to a test file in `jcstress-samples/src/main/java/org/openjdk/jcstress/samples/jmm/basic/` and is the formal ground truth against which to review code.

| Invariant | What it guarantees | Forbidden outcome | jcstress sample |
|---|---|---|---|
| **Access atomicity** | Reads/writes of primitives except non-volatile `long`/`double` are atomic — never a "half-written" value | Tearing on `int`, `ref`, `boolean`, etc. | `BasicJMM_02_AccessAtomicity` |
| **No word tearing** | Writes to one field/array element never corrupt adjacent ones | A concurrent `byte[]` element update losing a neighbor | `BasicJMM_03_WordTearing` (JLS 17.6) |
| **Progress** | Synchronized operations make forward progress; the JVM does not indefinitely delay a volatile read | Infinite hoisting of a volatile read out of a loop | `BasicJMM_04_Progress` |
| **Coherence** | Writes to a single shared variable are observed in a total order. Two consecutive reads of the same variable by one thread cannot see `(new, old)` | `(1, 0)` from two consecutive reads of `x` after `x = 1` — allowed on plain fields, forbidden on volatile/opaque | `BasicJMM_05_Coherence` |
| **Causality** | If A is ordered before B by HB, every effect of A is visible to B (transitive) | A visible effect without a HB edge | `BasicJMM_06_Causality` |
| **Consensus** | All threads agree on a single total order of synchronization actions. Only volatile (sequential consistency) guarantees this; acquire/release does not | `(0, 0)` in Dekker's idiom — forbidden on volatile, allowed on acquire/release | `BasicJMM_07_Consensus` |
| **Final-field initialization safety** | Writes to `final` fields in the constructor are visible to any thread that observes the fully-constructed object — provided `this` does not escape | Reading a default value (0/null) from a `final` field | `BasicJMM_08_Finals` (JLS 17.5) |
| **No out-of-thin-air** | Values read are justified by some actual write; no speculative "fabricated" values | A read returning a value never assigned anywhere | `BasicJMM_10_OOTA` |
| **SC-DRF** | Data-race-free programs execute sequentially consistent (Manson-Pugh-Adve 2005) | Any race-free code behaving non-SC | foundational; underlies all above |

**Coherence is subtle**: two plain-field reads of the same variable in the same thread can legally return `(1, 0)` — the second read may be fulfilled from a stale cache or register even after the first returned the newer value. Promoting to `volatile` or using `VarHandle.getOpaque()` restores per-variable total order.

**Consensus is stronger than release/acquire**: Dekker-style mutual-exclusion attempts require *global* agreement on write order across multiple variables. Acquire/release only constrains the handshake between two specific writes and reads; it is insufficient for multi-variable consensus (jcstress `BasicJMM_07_Consensus` AcqRelDekker proves this empirically, ~6% `(0,0)` outcomes observed).

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

**When to use**: tryLock, interruptibility, fairness, or multiple conditions. Otherwise, prefer `synchronized` (simpler, less error-prone, JVM can optimize it).

**Virtual threads — JDK version matters**:
- **JDK 21–23**: `synchronized` pins the virtual thread to its carrier. Blocking I/O inside `synchronized` caused a production-scale deadlock at Netflix in 2024 (5 virtual threads, 1 platform thread, all fighting for a lock; 4 pinned to carriers, starving the 5th).
- **JDK 24+**: JEP 491 rewrote monitor internals. Virtual threads now acquire/release monitors independently of carriers. `synchronized` no longer pins.
- **Remaining pinning in JDK 24+**: native methods, FFM API (JEP 454).
- **Review rule**: if the project targets JDK ≤ 23 AND the `synchronized` section performs blocking I/O, rewrite with `ReentrantLock`. If JDK ≥ 24, no action needed.

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

**Recursive update is detected and thrown**:

Javadoc (JDK 9+): "Throws `IllegalStateException` — if the computation detectably attempts a recursive update to this map that would otherwise never complete." On Java 8 ≤ u40, the same pattern could hang in an infinite loop (JDK-8062841).

```java
// JDK 9+: throws IllegalStateException "Recursive update"
map.compute("a", (k, v) -> map.compute("b", (k2, v2) -> 42));

// Also problematic: any mutating call to the same map inside a remapping function
map.compute("a", (k, v) -> {
    int other = map.get("b");  // read is safer but still risky across bins
    return v + other;
});
```

**Rule**: Never call any mutating method on a `ConcurrentHashMap` from within a `compute()`/`computeIfAbsent()`/`merge()` lambda on the same map. The lambda "must not attempt to update any other mappings of this Map" (JDK 21 Javadoc).

**Size semantics**:
- `size()` returns an `int` and is a transient estimate under concurrent updates.
- `mappingCount()` returns `long` — use it when the map may contain more than `Integer.MAX_VALUE` entries (Javadoc: "This method should be used instead of `size` because a ConcurrentHashMap may contain more mappings than can be represented as an int").

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

// SUBTLE: calling back into a ConcurrentHashMap from inside its own remapping function
// throws IllegalStateException "Recursive update" in JDK 9+ (not a deadlock, but equally fatal)
ConcurrentHashMap<String, List<String>> map = new ConcurrentHashMap<>();
map.computeIfAbsent("key", k -> map.getOrDefault("otherKey", List.of()));  // DON'T DO THIS

// JDK 21-23 VIRTUAL THREAD DEADLOCK: N virtual threads pinned to N carrier threads
// inside synchronized blocks, waiting on a lock held by another pinned VT.
// Real incident: Netflix 2024 (InfoQ case study). Fixed by JEP 491 in JDK 24+.
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

### 4.13 Torn 64-bit Reads and Writes

JLS 17.7: a single non-volatile `long` or `double` read/write MAY be split into two 32-bit halves. Under concurrent mutation, a reader can observe an impossible interleaved value.

```java
// BROKEN: concurrent writers with a non-volatile long
private long seqNum = 0;
public void tick()  { seqNum++;                    }  // three ops, not atomic
public long read() { return seqNum;                }  // may observe a torn value

// FIX: volatile guarantees 64-bit atomicity (and visibility)
private volatile long seqNum = 0;
// FIX: or use AtomicLong for compound updates
```

Verified empirically by jcstress `BasicJMM_02_AccessAtomicity`. Hardware may make the write atomic in practice (x86-64 for aligned 8-byte accesses), but the JMM does not require it — relying on platform behavior is non-portable.

### 4.14 Incoherent Reads of the Same Plain Field

Two reads of the same plain (non-volatile, non-opaque) variable in the same thread can legally return `(new, old)`:

```java
class Bad {
    int x = 0;
    // Thread 1: x = 1
    // Thread 2:
    int r1 = x;       // may read 1
    int r2 = x;       // may read 0 (from a cached register, after r1 already saw 1)
}
```

jcstress `BasicJMM_05_Coherence.SameRead` reproduces `(1, 0)` on plain fields. Promoting `x` to `volatile` forbids it (coherence = total order on writes to one variable).

**Rule**: if any read of a field can race with a write, either the field is `volatile` / read via `VarHandle.getOpaque()`, or the reviewer has written a proof that incoherent reads are tolerable (rare; usually a bug).

### 4.15 Unsafe Publication of Final-Field Objects

`final` fields carry an initialization-safety guarantee ("freeze action") — writes in the constructor are visible to any thread that observes the object — **only** if the object is not leaked before construction completes.

```java
// BROKEN: unsafe publication defeats final-field freeze
class Holder {
    final int[] data;
    Holder() { this.data = new int[]{1, 2, 3}; }
}
static Holder h;                           // non-volatile, non-final static
// Thread 1: h = new Holder();
// Thread 2: Holder local = h;
//           if (local != null) use(local.data);   // may see NULL or [0,0,0]

// FIX: publish via a volatile/final/synchronized container
static volatile Holder h;                  // volatile publication
static final Holder H_CONST = new Holder();// holder idiom via class init lock
```

Shipilev's *Safe Publication and Safe Initialization in Java* (2014): "final fields can save you from observing the under-initialized object, but cannot save you from not seeing the object itself." The reference used to reach the object must itself be safely published.

### 4.16 The ABA Problem

In lock-free algorithms based on `compareAndSet`, a reference may change from `A` to `B` and back to `A`. The CAS then succeeds though the data structure state has changed underneath the caller. Herlihy-Shavit §10.6: "Typically, a reference about to be modified by a `compareAndSet()` changes from `a`, to `b`, and back to `a` again. As a result, the `compareAndSet()` call succeeds even though its effect on the data structure has changed, and no longer has the desired effect."

Classic scenario — lock-free stack with node recycling:
```
Thread 1: reads top = N1, prepares CAS(top, N1, N1.next)
Thread 2: pops N1, pops N2, pushes N1 back (N1 now has a different next!)
Thread 1: CAS(top, N1, stale_N1.next) succeeds — stack is corrupted
```

**Fixes in Java**:
- **`AtomicStampedReference<T>`** — atomically updates `(reference, int stamp)`; increment the stamp on every modification so ABA is detected.
- **`AtomicMarkableReference<T>`** — atomically updates `(reference, boolean mark)`; useful for lazy deletion.
- **Avoid node recycling**: rely on the GC to reclaim nodes. Herlihy-Shavit §9.3: "We do not attempt to reuse list nodes that have been removed from the list, relying instead on a garbage collector to recycle that memory." Java code rarely sees ABA for GC'd object graphs; it appears most often in long-held pooled objects, sequence numbers, and version counters.

```java
// Fix: AtomicStampedReference for ABA-safe CAS on a pointer
private final AtomicStampedReference<Node> top = new AtomicStampedReference<>(null, 0);

public void push(T value) {
    int[] stampHolder = new int[1];
    while (true) {
        Node oldTop = top.get(stampHolder);
        Node newTop = new Node(value, oldTop);
        if (top.compareAndSet(oldTop, newTop, stampHolder[0], stampHolder[0] + 1)) return;
    }
}
```

Note: on some architectures, **load-linked/store-conditional** primitives intrinsically avoid ABA because SC fails if *any* write to the location has occurred between LL and SC — not merely a value comparison. The JVM does not expose LL-SC directly; use `AtomicStampedReference` instead.

Source: Herlihy & Shavit §10.6, "The ABA Problem"; `AtomicStampedReference` Javadoc.

### 4.17 False Sharing

Two unrelated fields that happen to share a CPU cache line (typically 64 bytes on x86-64, 128 on some ARM) cause cross-thread invalidations on every write, even when threads touch disjoint fields. The correctness is intact; throughput collapses.

Herlihy-Shavit §7.5.1 & Fig. 7.8 demonstrate the pattern with the `ALock` queue lock: array entries on the same cache line cause each `flag[i]` write to invalidate `flag[i+1]` in neighbor threads' caches. The fix is to pad each slot to its own cache line.

```java
// BROKEN: counter[0] and counter[1] likely share a cache line
long[] counter = new long[2];
// Thread 0 increments counter[0]; Thread 1 increments counter[1];
// Every write bounces the cache line between cores.

// FIX 1: @Contended (JDK 8+, requires -XX:-RestrictContended or --add-opens)
@jdk.internal.vm.annotation.Contended  // or sun.misc.Contended pre-JDK 9
volatile long counter0;
@jdk.internal.vm.annotation.Contended
volatile long counter1;

// FIX 2: LongAdder — internally stripes cells across cache lines
LongAdder counter = new LongAdder();

// FIX 3: manual padding (legacy; @Contended is preferred)
class PaddedLong {
    public volatile long value;
    public long p1, p2, p3, p4, p5, p6, p7;  // pad to 64 bytes
}
```

Detection: `perf stat -e cache-misses` shows elevated rates under contention; JMH with `@State(Scope.Thread)` and bench variants with/without padding reveals the delta.

**Lock-and-data colocation heuristic (Herlihy-Shavit Appendix B "Hardware Basics", p.17–18).** The rule inverts by contention profile:
- **Contended lock + frequently-modified data → put them on *different* cache lines.** Waiters spinning on (or invalidating) the lock line will otherwise thrash the neighbouring data, so every release has to re-fetch it. Pad the lock object, or pad the first protected field.
- **Uncontended lock + usually-quiet data → put them on the *same* cache line.** Acquiring the lock prefetches the data as a side effect; a second miss is avoided. Useful for per-connection state, per-shard counters, per-entry locks under hash striping.

This heuristic cannot be checked from the source alone — it requires knowing the contention level. Flag any diff where the author has clearly chosen one layout without naming the contention assumption.

Source: Herlihy & Shavit, *The Art of Multiprocessor Programming*, Revised Reprint, Appendix B "Hardware Basics" (p.17–18); Herlihy-Shavit §7.5.1, *Fig. 7.8* (array-based queue lock with padding); JEP 142 (Contended annotation).

### 4.18 ThreadLocal Storage Leaks

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
- [ ] **Transfer vs share (the Rust `Send`/`Sync` distinction).** A type may be safe to *transfer* ownership of between threads (handoff through a blocking queue) yet unsafe to *share* between two live threads simultaneously. Java has no compiler enforcement; state the property explicitly in Javadoc or the confinement contract ("thread-confined but reassignable", "immutable once published") so reviewers and callers do not conflate the two. Source: Goetz *Java Concurrency in Practice* §3.3 Thread Confinement; Rust Reference §"Send and Sync" as conceptual analogue.

### 5.2 Lock Scope Analysis

- [ ] **Too broad**: Does the synchronized block contain I/O, network calls, or long computations? This kills throughput.
- [ ] **Too narrow**: Is a compound operation (check-then-act) split across multiple synchronized blocks? This introduces race conditions.
- [ ] **Lock object**: Is the lock object `private` and `final`? Locking on `this`, `Class`, or mutable fields is dangerous.
- [ ] **Lock ordering**: If multiple locks are acquired, is the ordering documented and consistent across all code paths?
- [ ] **Structural lock-ordering enforcement.** A comment saying "always take A before B" decays. Prefer a `LockOrdering` enum with `Comparator`-based acquisition, a `MultiLock` helper that sorts by `System.identityHashCode` / stable ID, or `tryLock(timeout)` with backoff — enforcement the compiler or runtime can see. Source: JCIP §10.1.2 "Lock Ordering"; Doug Lea *Concurrent Programming in Java* §2.3.
- [ ] **No bounded-queue `put`, `BlockingQueue.put`, or `Future.get` while holding any lock.** A full downstream queue blocking the producer while it holds an upstream lock creates a cross-subsystem stall indistinguishable from deadlock. Rework to drop the lock before the blocking call, or use `offer(timeout)` with a rejection policy. Source: JCIP §10.1 "Liveness Hazards"; Doug Lea *Concurrent Programming in Java* §3.3.4 "Nested Monitor Lockouts".
- [ ] **No lock held across a remote call (HTTP, RPC, DB, circuit-breaker open).** A 30 s downstream latency becomes a 30 s held lock, multiplied by every queued requester. Snapshot what the call needs, release the lock, then call out; apply results via a second short critical section (`AtomicReference.updateAndGet` or a merge). Source: JCIP §11.4 "Reducing Lock Contention".
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
- [ ] **JMM vocabulary matches JMM semantics.** When a comment or Javadoc says "atomic", does it mean JMM-atomic (indivisible wrt other threads) or the colloquial "happens in one go"? `volatile long x = x + 1` is *not* atomic even though the store itself is. Flag any comment or name that uses "atomic", "synchronized", or "thread-safe" without a falsifiable invariant behind it. Source: JLS §17.4; Shipilev *JMM Pragmatics* §"Access Atomicity".
- [ ] **Audit pre-JSR-133 wording in existing comments and `@GuardedBy` annotations.** Phrases like "volatile prevents caching", "synchronized flushes memory", or "only one thread at a time reads it" signal a pre-2004 mental model that the current implementation does not satisfy. When touching such code, fix the explanation alongside the behavior. Source: Manson/Pugh/Adve *The Java Memory Model* (POPL 2005) — JSR-133 as the cutover line.
- [ ] **No "benign race" without a falsifiable argument.** Any race deemed harmless must (a) identify the specific torn/stale/out-of-thin-air outcome that is being tolerated, (b) cite the jcstress sample or JMM rule that confirms it is allowed, and (c) explain why the tolerated outcome does not propagate to an invariant. Absent (a)–(c), treat "benign" as a code smell; promote the field to `volatile` or `VarHandle.getOpaque()`. Source: Shipilev *Java Memory Model Pragmatics* §"Benign Races"; jcstress `BasicJMM_09_BenignRaces`.
- [ ] **Out-of-thin-air rule on any race judged harmless.** JSR-133 forbids reads returning values never written anywhere, but the rule is easy to overlook for "harmless" races. Confirm the read domain is constrained (bounded range, sentinel values) so that a speculatively fabricated value cannot corrupt a downstream invariant. Source: JLS §17.4.8; jcstress `BasicJMM_10_OOTA`.
- [ ] **No implicit x86 TSO assumption.** Code tested only on x86 developer laptops may fail on ARM (Graviton, Apple Silicon) or POWER because those platforms reorder more aggressively. For any new HB-sensitive code, require jcstress or integration-test evidence on a weakly-ordered architecture before merge, or explicitly mark the change "x86-only". Source: JSR-133 Cookbook (Doug Lea); ARMv8 memory model (ARMv8 Architecture Reference Manual §B2).

### 5.5 Safe Publication

- [ ] **Immutable objects**: Are objects that need no synchronization truly immutable (all fields `final`, no `this` escape, no mutable collections)?
- [ ] **Effectively immutable**: Objects not modified after publication should be published via `volatile`, `final`, or within a lock.
- [ ] **Constructor escape**: Does `this` leak during construction (e.g., registering listeners, starting threads, passing `this` to other methods)?
- [ ] **Collection publishing**: Is a mutable collection published via `Collections.unmodifiableList()` wrapping? The underlying collection must not be modified after publication.

### 5.6 Thread-Safety of Third-Party Code

- [ ] **Is the library thread-safe?** Read the Javadoc. `SimpleDateFormat`, `HashMap`, `ArrayList`, `StringBuilder` -- all NOT thread-safe.
- [ ] **`HashMap` failure-mode enumeration.** The Javadoc says "must be synchronized externally"; it does not enumerate what concretely breaks. Reviewers should recognize the four canonical failure modes on sight: (1) *infinite loop* during concurrent `put` on JDK ≤ 7 (linked-list cycle during resize — still cited in older posts); (2) *lost updates* when two threads resize concurrently; (3) *NullPointerException* from a half-resized `table`; (4) *wrong bucket* after a concurrent rehash so `get` returns `null` for a key that was inserted. JDK 8+ treeified buckets mitigate the infinite-loop mode but not the others. Source: Goetz *Java Concurrency in Practice* §5.1.1; Bloch *Effective Java*, 3rd ed., Item 79; [Naresh's JDK-6423457](https://bugs.openjdk.org/browse/JDK-6423457).
- [ ] **Spring / CGLIB / AOP proxy boundaries.** A `synchronized` method on a Spring-proxied bean serializes on the *proxy instance*, not the proxied target — and when the same bean calls itself (`this.method()`) the call bypasses the proxy entirely. Two threads can enter the "same" `synchronized` method holding different monitors, or zero monitors. Rewrite as a `synchronized` block on a `private final Object lock` inside the method, use a `ReentrantLock` field, or inject a self-reference through the proxy. Source: Spring Framework Reference "Understanding AOP Proxies"; JDK `java.lang.reflect.Proxy` Javadoc.
- [ ] **Servlet/Controller state**: Are there mutable instance fields in servlets, Spring controllers, or similar shared-instance classes? These are effectively global shared mutable state.
- [ ] **DI scopes, servlet filters, actor mailboxes, and framework callbacks are shared-state boundaries.** Request-scoped beans wrapped in a singleton proxy still route through a scope resolver that must itself be thread-safe. Servlet filters (`ThreadLocal` set, chain call, remove) must restore state even on exception. Actor mailboxes present single-threaded semantics to the actor body but cross the thread boundary at enqueue/dequeue — any object placed on a mailbox must satisfy the "safe to transfer" contract (§5.1). Framework event callbacks (Spring `@EventListener`, reactive subscribers) may run on any thread the framework chooses. Source: Spring Framework Reference "Bean Scopes"; Servlet 6 Spec §2.3.3.3; Akka *Concurrency Guide*.
- [ ] **Connection pools**: Are database connections, HTTP clients, etc., thread-safe or properly pooled?

### 5.7 Proper Shutdown of ExecutorServices

- [ ] **Shutdown is called.** Every `ExecutorService` created must have `shutdown()` or `shutdownNow()` called, typically in a shutdown hook, `@PreDestroy`, or `close()`.
- [ ] **awaitTermination is checked.** After `shutdown()`, call `awaitTermination()` and check its return value. If it returns `false`, tasks are still running.
- [ ] **Cached thread pools with unbounded tasks.** `Executors.newCachedThreadPool()` can create unlimited threads. Use bounded pools in production.
- [ ] **Replace `Executors.newFixedThreadPool` / `newCachedThreadPool` with a directly-constructed `ThreadPoolExecutor`.** The factory methods use `LinkedBlockingQueue` (effectively unbounded) and `AbortPolicy`. Direct construction forces the author to pick a bounded queue, a named `ThreadFactory`, and an explicit `RejectedExecutionHandler` (AbortPolicy / CallerRunsPolicy / DiscardOldestPolicy / a custom load-shedding policy). Source: Goetz *Java Concurrency in Practice* §8.3.1 "Thread Creation and Teardown" and §8.3.3 "Saturation Policies".
- [ ] **No per-request `ExecutorService`.** An executor constructed per request leaks threads if shutdown is missed, loses back-pressure (each request has its own unbounded queue), and pays thread-creation cost on every request. Use a shared, named, bounded pool with a documented owner — or a virtual-thread executor (JDK 21+) where per-request lifetime is acceptable. Source: JCIP §6.2.2.
- [ ] **Owned lifecycle with a test that fails if it outlives the owner.** For every `Thread`, `ExecutorService`, and `ScheduledExecutorService`, the PR should name the owner (bean, `AutoCloseable`, try-with-resources scope) and include a test that asserts `awaitTermination(...)` returns `true` within a bounded time after owner teardown. A leaked `ScheduledExecutorService` accumulates `ScheduledFutureTask` entries silently until OOM. Source: JCIP §7.2 "Stopping a Thread-Based Service".
- [ ] **Replacing a `ScheduledExecutorService` owner shuts down the previous executor.** Common leak: a bean is re-initialized (hot reload, DI rebind, test context swap) and the old scheduler keeps enqueuing tasks. Enforce via `@PreDestroy` and a leak-watcher test: construct, replace, assert the old executor terminates. Source: `ScheduledExecutorService` Javadoc; JCIP §7.2.5.
- [ ] **Background task cannot outlive the resources it captures.** A task submitted to a long-lived pool that references a request-scoped object, a closed connection, or a GC-rooted cache keeps those objects live. Capture-by-value (copy the fields you need) or pass a short-lived cancellation token the owner can trip on teardown. Source: JCIP §6.2 "Executor Framework".
- [ ] **Holistic shutdown drain.** Shutdown must drain/cancel/join every thread, executor, scheduled task, `ThreadLocal`, and pending `CompletableFuture` — not assume the JVM will exit. Test one level up: after `owner.close()`, does any non-daemon thread remain? Any `ScopedValue` binding? Any orphaned `CompletableFuture` still attached to a stage? Source: JCIP §7.2; Oracle JDK Concurrency Tutorial "Application Shutdown".

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
- [ ] **Exceptions never leave shared state half-updated.** An exception from inside a `synchronized` block, a try-with-resources lock region, or a `CompletableFuture` stage releases the lock but does *not* roll back partial writes. For every multi-field mutation under a lock, the catch/finally path must either complete the invariant or restore the pre-mutation state (snapshot-and-swap, try/commit/rollback, or `AtomicReference.updateAndGet` with an idempotent updater). Source: JCIP §10.1.4 "Open Calls"; Eric Lippert on "exception safety".
- [ ] **`ExecutionException.getCause()` is never dropped or silently logged.** A PR that catches `ExecutionException` and logs without surfacing the cause trades a user-visible bug for a ticket nobody files. Required forms: rethrow wrapped in a domain exception, surface via a metric + structured log with the cause class + stacktrace, or translate to a typed failure result. Flag any `catch (ExecutionException e)` that does not do one of these. Source: `Future.get` Javadoc; Goetz *Java Concurrency in Practice* §7.1.4.
- [ ] **Explicit supervision strategy when a worker dies mid-task.** A thread that dies inside a `ThreadPoolExecutor` is replaced silently; a thread that dies inside `ForkJoinPool.commonPool()` causes the task to complete exceptionally but does not notify the submitter. Pick one explicitly: (a) *restart* — `Thread.UncaughtExceptionHandler` that re-submits; (b) *escalate* — propagate via the `Future`/scope and let the caller decide; (c) *stop* — shut down the pool, page on-call. The choice should appear as code or configuration, not an assumption. Source: JCIP §7.2.5; Armstrong, *Programming Erlang* (supervisor tree analogy).

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

### 5.9 Thread Interruption and Future Cancellation

Interruption is cooperative: `Thread.interrupt()` only sets a flag (or wakes a thread blocked in an interruptible method). The target thread must cooperate.

- [ ] **InterruptedException is never swallowed.** Two correct responses: rethrow it (preferred — propagate up the stack with `throws InterruptedException`), or restore the interrupt flag via `Thread.currentThread().interrupt()` before continuing. Silently catching and discarding is a bug.
- [ ] **Long-running loops poll `Thread.interrupted()`.** CPU-bound code that does not invoke interruptible blocking methods must check the flag periodically and exit cleanly.
- [ ] **`Future.cancel(true)` is the right cancellation signal.** It sets the interrupt flag on the executing thread; the task code must honor it.
- [ ] **Resource cleanup on interrupt.** Cleanup (closing sockets, releasing locks) belongs in `finally`; the interrupt arrives at an arbitrary point.
- [ ] **Cancellation mechanism is consistent across the call graph.** If the top of the call tree uses `Future.cancel`, the middle uses a volatile `AtomicBoolean` flag, and the bottom uses `Thread.interrupted()`, cancellation signals are dropped at each translation boundary. Pick one mechanism per scope (or use `StructuredTaskScope.shutdown()`) and verify it flows end-to-end — every blocking call either responds to interruption or to the chosen flag, with no silent bridges. Source: JCIP §7.1 "Task Cancellation"; JEP 505/525 `StructuredTaskScope`.

```java
// BROKEN: interrupt status silently lost — upstream callers cannot cancel
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // swallowed
}

// FIX A (preferred): rethrow
void doWork() throws InterruptedException {
    Thread.sleep(1000);
}

// FIX B: restore status when the method signature cannot throw
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();  // preserve for upstream code
    return;                               // then exit promptly
}
```

Source: Oracle Concurrency Tutorial § Interrupts; *Java Concurrency in Practice* ch. 7; code-review-checklists/java-concurrency § Thread Interruption.

### 5.10 Parallel Streams and Collectors

- [ ] **No shared mutable state in lambdas** passed to `map`/`filter`/`forEach` on a parallel stream. The pipeline runs on `ForkJoinPool.commonPool()` threads.
- [ ] **Custom `Collector.combiner` must be associative.** `Collectors.toList()` and other JDK collectors satisfy this; hand-rolled ones often do not.
- [ ] **Do not use a parallel stream inside a `synchronized` block.** The stream's tasks run on worker threads that do not hold the lock.
- [ ] **Parallel streams share the common ForkJoinPool.** A single long-running parallel stream can starve every other parallel stream in the JVM. For I/O-bound work, use a dedicated `ForkJoinPool` or virtual-thread executor instead.

Source: code-review-checklists/java-concurrency § Parallel Streams; JDK Javadoc `java.util.stream.Collector`.

### 5.11 Locking-Strategy Selection (Herlihy-Shavit §9)

For any shared container, one of five strategies is appropriate. Review that the choice matches the workload:

| Strategy | How | When it fits | Example |
|---|---|---|---|
| **Coarse-grained** | One lock per object | Low contention; simple correctness | `Collections.synchronizedMap(new HashMap<>())` |
| **Fine-grained** | Lock per bucket/node (*hand-over-hand* / *lock coupling* for lists) | Read/write mix, moderate contention | Pre-JDK 8 `ConcurrentHashMap` segment locks |
| **Optimistic** | Traverse without locks; lock the target; validate it has not changed | Reads >> writes; searches dominate | `StampedLock.tryOptimisticRead`; optimistic linked list (§9.6) |
| **Lazy** | Logical removal (mark bit) now, physical unlink later | Remove is frequent but can tolerate brief inconsistency | `ConcurrentSkipListMap`; lazy linked list (§9.7) |
| **Nonblocking (lock-free)** | Only `compareAndSet` / `AtomicStampedReference`; no locks | Contention high; blocking unacceptable; GC handles memory reclamation | `ConcurrentLinkedQueue`, `AtomicReference`-based caches |

- [ ] **Strategy matches contention profile?** Coarse-grained under heavy contention is a bottleneck; nonblocking under low contention is unnecessary complexity (Herlihy-Shavit §9.1).
- [ ] **Optimistic validation present?** Every optimistic read must re-check its predecessor/target under the lock before committing. Missing validation = torn read accepted.
- [ ] **Lazy removal visible to readers?** A logically-removed node must still be traversable by concurrent reads until the GC reclaims it; code must never NPE on `null` out a field of a logically-removed node.
- [ ] **Nonblocking code relies on GC?** Java lock-free algorithms typically avoid the ABA problem by relying on the collector for "freedom from interference" (§9.3). Flag any code that manually recycles nodes — it needs `AtomicStampedReference` or LL-SC equivalent.

### 5.12 Invariant-Driven Review for Concurrent Data Structures (Herlihy-Shavit §9.3)

Before approving any concurrent data-structure change, enumerate the representation invariants and check each mutation:
1. **Does the invariant hold at construction time?**
2. **For every add/remove/update, does the code preserve the invariant — considering that other threads may be between any two of its instructions?**

Standard invariants to check for linked structures:
- Keys are sorted (for ordered sets/lists).
- `head` is always reachable to `tail` via `next` pointers.
- Every inserted node is reachable before its insertion is linearized.
- No node is concurrently reachable and freed (**freedom from interference** — in Java, guaranteed by the GC for nodes internal to the class).
- Sentinel nodes (head/tail) are never removed.

Herlihy-Shavit §9.3 definition: "the key to understanding a concurrent data structure is to understand its invariants: properties that always hold. We can show that a property is invariant by showing that: (1) The property holds when the object is created, and (2) Once the property holds, then no thread can take a step that makes the property false."

Distinguish:
- **Abstract value** (what the data structure models — the set, the queue contents)
- **Concrete representation** (how it is stored — nodes, pointers)
- **Representation invariant** (which concrete states are valid)
- **Abstraction function** (concrete → abstract mapping)

Reviews that reason only about abstract behavior miss races on representation fields.

- [ ] **Linearization points are named.** For every public method on a custom concurrent data structure, the author must identify the single instruction at which the operation "takes effect" for observers — the CAS on the head pointer, the volatile flag flip, the write under the lock. If no single instruction linearizes the operation, it probably is not linearizable. Source: Herlihy & Wing, "Linearizability: A Correctness Condition for Concurrent Objects", *ACM TOPLAS* 12(3), 1990; Herlihy-Shavit §3.6.
- [ ] **Memory-reclamation scheme is named or GC-based.** Custom lock-free structures that recycle nodes without a named scheme (hazard pointers — Michael 2004; epoch-based — Fraser 2004; QSBR/RCU — McKenney 2001) are almost certainly wrong under ABA or use-after-free. Java's GC provides "freedom from interference" for the common case (Herlihy-Shavit §9.3); if the PR deliberately sidesteps the GC, require a citation to the scheme it uses instead. Source: Michael, *IEEE TPDS* 15(6), 2004 (hazard pointers); Fraser, PhD thesis 2004 (epoch-based).
- [ ] **"Thread-safe" claim is falsifiable.** An author who calls a class "thread-safe" must state (a) which invariant is preserved, (b) which interleaving is ruled out, and (c) which scheduler-admissible interleaving would *violate* it absent the current synchronization. If the author cannot answer (c), the claim is a virtue signal, not a property. Source: Popper on falsifiability as applied to concurrency claims (JCIP ch. 2 framing); Herlihy-Shavit §3.6 "Correctness" (need for a proof obligation).

### 5.13 Virtual Thread Review Rules (JDK 21+)

- [ ] **Do not pool virtual threads.** Create one per task via `Thread.ofVirtual().start()` or `Executors.newVirtualThreadPerTaskExecutor()`. Pooling defeats their purpose.
- [ ] **CPU-bound work stays on platform threads.** Virtual threads help I/O-bound workloads; a `ForkJoinPool` sized to `Runtime.getRuntime().availableProcessors()` is correct for CPU-bound.
- [ ] **Backpressure.** When spawning millions of virtual threads, bound concurrent downstream calls with a `Semaphore` — databases, HTTP peers, and the common ForkJoinPool still have finite capacity.
- [ ] **Heap sizing.** Virtual thread stacks live on the heap. High-concurrency services need higher `-XX:MaxRAMPercentage` (empirical guidance: 35%+ when migrating from platform threads).
- [ ] **Pinning (JDK 21–23 only)**: `synchronized` around blocking I/O. Enable `-Djdk.tracePinnedThreads=short` or the JFR `jdk.VirtualThreadPinned` event (default 20ms threshold). Resolved by JEP 491 on JDK 24+.
- [ ] **ThreadLocal churn.** Each VT gets its own copy; millions of VTs × any cached state = heap pressure. Prefer `ScopedValue` (final in JDK 25, JEP 506).
- [ ] **Is the pool even necessary?** On JDK 21+, "what size should this pool be?" is often the wrong question — the right one is "does this need to be a pool at all?" A `newVirtualThreadPerTaskExecutor()` or `StructuredTaskScope` eliminates sizing tuning for I/O-bound workloads and often simplifies shutdown. Require an explicit justification for any new pool in code that will run on JDK 21+. Source: JEP 444; Oracle JDK 25 Virtual Threads docs; Alan Bateman, *Virtual Threads: State of Play* (JVMLS 2023).
- [ ] **Loom-native rewrite thought experiment.** For any concurrency-heavy change, ask: if I rewrote this with virtual threads + `StructuredTaskScope` tomorrow, which of today's primitives would disappear (incidental complexity)? Which would remain (load-bearing — protecting a real cross-thread invariant)? The answer reveals which `synchronized` blocks, executors, and `ThreadLocal`s are worth reviewing in depth and which to simply refactor away. Source: JEP 505/525; Ron Pressler, *Project Loom* design rationale.
- [ ] **Forward-compatible shape today.** Even in projects not yet on structured concurrency, prefer shapes that port cleanly: bounded task lifetimes tied to a scope; cancellation through a single channel; exceptions propagated rather than logged-and-swallowed. Avoid free-floating `CompletableFuture` pipelines whose lifetime is not tied to any owner. Source: JEP 505 "Goals"; Pressler.
- [ ] **`StructuredTaskScope` vs `CompletableFuture` clarity test.** If the diff uses `CompletableFuture.allOf` / `anyOf` with manual cancellation and exception joining, write the equivalent `StructuredTaskScope` sketch in the PR description. If it is visibly clearer, the existing form is carrying accidental complexity. Source: JEP 505 §Motivation.
- [ ] **Review-heuristic transition at scale.** Pool-sizing, contention-profile, and `ThreadLocal`-audit heuristics lose relevance once active thread count reaches the millions. Substitute: scope confinement (every virtual thread has a clear owner), pinning audit (native/FFM calls, pre-JDK 24 `synchronized` I/O), carrier-thread saturation audit (common FJP parallelism = available processors), `ScopedValue` adoption. Source: Oracle JDK 25 Virtual Threads docs; Cashfree *7 lessons*.

Source: Oracle JDK 25 "Virtual Threads" documentation; JEP 444; JEP 491; Netflix case study (InfoQ, Aug 2024); Cashfree "7 lessons from Java 21 Virtual Threads in Production".

### 5.14 Liveness Hardening Beyond Basic Deadlock Avoidance

- [ ] **No alien method invoked while a lock is held.** A `synchronized` block (or held `Lock`) must not call: a listener, an observer, a `Consumer`/`Runnable`/lambda passed from outside, an overridable method on `this`, or any method on a parameter whose concrete type is not under this module's control. Every production AB-BA deadlock in Java traces back to a callback invoked under a lock, re-entering the lock graph in a different order. Snapshot the state, release the lock, then invoke the alien method. Source: JCIP §10.1.3 "Open Calls"; Goetz, "Java theory and practice: Safe construction techniques" (IBM, 2004).
- [ ] **Thread-pool self-deadlock is ruled out.** If tasks submitted to pool P wait for the completion of other tasks in P, a pool saturated at N workers with N such tasks deadlocks itself. Flag any `Future.get` / `CountDownLatch.await` / `join` reached from a task whose executor is the same pool that would need to run the awaited task. Use a separate pool for dependents, or restructure with `CompletableFuture` composition so no task blocks waiting on its siblings. Source: JCIP §8.1.1 "Thread Starvation Deadlock".
- [ ] **Nested-monitor lockout is impossible.** A thread holding outer lock `A` that calls `wait()` on inner monitor `B` releases only `B`; another thread cannot enter the guarded region to signal `B` because it would first need `A`. The result is a hang that does *not* show up as a classic deadlock cycle. Avoid `wait` on a monitor while holding any other lock; use `Condition` variables paired to their governing `Lock`. Source: Doug Lea, *Concurrent Programming in Java*, 2nd ed., §3.3.4 "Nested Monitor Lockouts".
- [ ] **Writer-starvation bound on `ReentrantReadWriteLock`.** With continuous reader arrival in unfair mode, writers can wait indefinitely. Either use fair mode (`new ReentrantReadWriteLock(true)`) if bounded latency matters, or switch to `StampedLock` (which upgrades optimistic reads to a write lock) or a snapshot-based design. Document the expected worst-case wait for the minority operation. Source: `ReentrantReadWriteLock` Javadoc "Fairness"; Herlihy-Shavit §8.3.
- [ ] **Deadlock detection or `tryLock(timeout)` is in place, with a distinctive thread-dump signature.** For any module introducing a new lock graph, include at least one of: `Lock.tryLock(timeout)` with a typed exception on timeout, a watchdog that periodically calls `ThreadMXBean.findDeadlockedThreads()`, or named lock objects and named threads so `jstack` output unambiguously identifies the cycle. An on-call responder at 02:00 should be able to distinguish *this* PR's deadlock signature from every other one. Source: `ThreadMXBean.findDeadlockedThreads` Javadoc; Holmes, *Concurrency: A Practitioner's Guide* (JavaOne 2015).
- [ ] **Wait-for graph is acyclic by construction.** When a change introduces a new lock acquisition, draw the wait-for graph (nodes = locks, edges = "Thread holding L1 requests L2") for every code path. The union across paths must be a DAG. If the DAG cannot be proven, require `tryLock(timeout)` with backoff so a cycle becomes a retryable failure, not a deadlock. Source: Coffman, Elphick & Shoshani, "System Deadlocks", *ACM Computing Surveys* 3(2), 1971 — the canonical four conditions.

### 5.15 Synchronization Primitive Hygiene

- [ ] **The field's synchronization story is self-evident from the class alone.** A reviewer opening the file should know in five seconds whether a field needs `synchronized`, `volatile`, `AtomicReference`, or nothing — via (a) a `@GuardedBy` annotation naming the monitor, (b) a naming convention such as `volatile` or atomic types visible in the declaration, (c) a class-level Javadoc stating the thread-safety contract (immutable / thread-safe / conditionally thread-safe / thread-hostile). Fields reachable only through a chain of inference are a review-time hazard. Source: JCIP §2.4 "Guarding State with Locks"; JCIP §4.5 "Documenting Synchronization Policies".
- [ ] **Primitive × operation matrix is satisfied.** For each shared data structure, verify the chosen primitive (`synchronized`, `ReentrantLock`, `StampedLock`, `Atomic*`, `volatile`, lock-free CAS) actually supports the operation the code performs — including iteration and bulk read. A `volatile Map` reference protects assignment but not iteration. An `AtomicReference<List>` protects swap but not in-place mutation. A `synchronizedList` protects individual calls but not compound operations. Enumerate the operations (read / write / RMW / iterate / bulk update / snapshot) and verify each is covered. Source: JCIP ch. 5; `Collections.synchronizedMap` Javadoc ("conditionally thread-safe").
- [ ] **Every `wait()` sits inside a `while` predicate loop.** `if (cond) wait()` tolerates spurious wakeups and notify-before-wait orderings incorrectly. The canonical idiom is `while (!cond) wait();` inside a synchronized block; for `Condition` variables, `while (!cond) condition.await();`. Flag any `if`-gated `wait` as a bug. Source: `Object.wait` Javadoc ("should always occur in a loop"); JCIP §14.2.2.
- [ ] **Fairness is an explicit constructor argument, not a default.** For every `new ReentrantLock(...)`, `new ReentrantReadWriteLock(...)`, and `new Semaphore(...)`, the PR must show the fairness argument — even if that is `false`. Accepting the default without comment hides a latency/throughput trade-off from review. Source: `ReentrantLock(boolean fair)` Javadoc; JCIP §13.3.
- [ ] **Consider `AbstractQueuedSynchronizer` before rolling a custom CAS loop.** Hand-rolled synchronizers routinely get interruption, timeout, and condition variables wrong. AQS subclasses (`tryAcquire`/`tryRelease`) inherit a tested framework for all of these. A PR introducing a custom synchronizer should either extend AQS or justify why AQS doesn't fit (typical valid reasons: obstruction-free progress requirements, lock-free queue semantics). Source: Doug Lea, "The java.util.concurrent Synchronizer Framework" (PODC 2004); `AbstractQueuedSynchronizer` Javadoc.
- [ ] **Monitor / `ReentrantLock` nesting is deliberate.** Mixing `synchronized` blocks with `ReentrantLock` regions around the same state risks ordering and reentrance bugs — the two primitives cannot be acquired in one atomic step, and a single thread can accidentally hold both. Pick one per code path. If nesting is unavoidable, document the order explicitly and cover it with a test that acquires in reverse and asserts a `tryLock`-timeout rejection. Source: Doug Lea, *Concurrent Programming in Java* §2.2.3; JCIP §13.1.
- [ ] **Swap-to-`ReentrantLock` thought experiment.** For each `synchronized` block or method, ask: if this were a `ReentrantLock`, which of {fairness, `tryLock`, `lockInterruptibly`, multiple `Condition`s} would the diff's requirements actually demand? If two or more, the `synchronized` form is hiding a requirement the author has not articulated. Source: `ReentrantLock` Javadoc "Advantages over synchronized"; JCIP §13.1.

### 5.16 Concurrent Collection Patterns Beyond ConcurrentHashMap

- [ ] **Collection × access-pattern safety matrix.** For each concurrent collection in the diff (`ConcurrentHashMap`, `CopyOnWriteArrayList`, `ConcurrentSkipListMap`, `BlockingQueue`, `Collections.synchronizedXxx`), cross-check the access patterns actually used: single-key get/put, `compute`/`merge`, multi-key update, snapshot iteration, size-based control flow. The matrix reveals mismatches a prose review misses — e.g., `ConcurrentSkipListMap` gives navigable-map operations but snapshot iteration is *not* an atomic operation; `CopyOnWriteArrayList` is safe for iteration but writes are O(n). Source: `java.util.concurrent` package-info; JCIP ch. 5.
- [ ] **Isolation level is named, not guessed.** Every concurrent in-memory data structure should document the isolation level it provides — read-committed, snapshot, serializable, or (more commonly) a weaker variant with caveats. Callers then do not have to reverse-engineer what "thread-safe" meant. If the author cannot state the level, the data structure probably does not have one, and compound operations are races waiting to happen. Source: Berenson et al., "A Critique of ANSI SQL Isolation Levels" (SIGMOD 1995) — framework imported from DB literature.
- [ ] **Write-skew check for COW / MVCC designs.** When two readers each take a consistent snapshot, each preserves the invariant individually, but their combined writes can violate it. Example: two sibling counters whose sum must be ≥ 0 — each transaction decrements one and checks the sum, but two concurrent decrements produce a sum < 0. Under snapshot isolation, neither transaction sees the other; serializable isolation is required, or a version counter / optimistic lock on the combined invariant. Source: Berenson et al. (1995); Fekete et al., "Making Snapshot Isolation Serializable", *ACM TODS* 30(2), 2005.
- [ ] **CRDT / commutativity thought experiment.** For each lock-based update on shared state, ask: if two replicas applied these updates independently and later merged, would the result match the synchronized outcome? If not, the lock is silently relying on a *non-commutative* operation order — which means the operation is not safely parallelizable at all, and any reordering (cache reorder, JIT reorder, retry replay) risks corruption. Source: Shapiro et al., "Conflict-Free Replicated Data Types" (SSS 2011) — used here as a reasoning probe, not a prescription.

### 5.17 Async Boundary Hygiene and Context Propagation

- [ ] **`ThreadLocal.set` / `remove` survives async handoff.** In a `CompletableFuture` pipeline where stages run on different carriers, `ThreadLocal` values set on stage 1 are absent on stage 2. Use a context-capturing wrapper (`Executor` that copies the MDC / ThreadLocal map into the task Runnable), `ScopedValue` (JDK 25+), or make context an explicit parameter. Verify every `set` has a matching `remove` on every exit — including exception and timeout — for every stage boundary the value is expected to cross. Source: `ThreadLocal` Javadoc; Oracle JDK 25 `ScopedValue` docs; JCIP §3.3.
- [ ] **Scope × escape matrix.** For each variable in the diff, map its scope ({local, field, static field, `ThreadLocal`, `ScopedValue`, DI-scoped bean, request-scoped bean}) against its escape routes ({accidental sharing, intentional sharing, escape via collection, escape via listener registration, escape via `this` in constructor, escape via lambda capture, escape via return value}). Every new cell in the matrix is a reviewable boundary. Source: JCIP §3.2 "Publication and Escape" (extended with scope taxonomy).
- [ ] **`ThreadLocal` contracts on pooled carriers.** A `ThreadLocal` written in a `CompletableFuture.thenApply` stage may execute on any carrier — pooled platform thread, common ForkJoinPool worker, or virtual-thread carrier. Stale values from a previous request silently authorize, tag, or bill the wrong user. For cross-stage context, prefer `ScopedValue` (immutable, scoped lifetime, inherited by forked tasks); if `ThreadLocal` is unavoidable, force the value to flow through the stage's input rather than its thread. Source: Oracle JDK 25 "Virtual Threads", section "ThreadLocal variables"; JEP 506 §Motivation.

### 5.18 Executor Backpressure, Saturation, and Resource Budgets

- [ ] **Saturation behavior is named and tested.** For each executor introduced, the PR must answer: what happens when the queue fills? What happens when the pool saturates? Required answers: a specific `RejectedExecutionHandler` (Abort / CallerRuns / DiscardOldest / custom load-shedder), an observable metric (rejection count, queue depth histogram), and at minimum one test that drives the executor into rejection and asserts the documented behavior. Source: `ThreadPoolExecutor` Javadoc "Rejected tasks"; JCIP §8.3.3.
- [ ] **Platform-level guardrail funnels every async boundary through a vetted pool.** A team that allows `Executors.newCachedThreadPool()` anywhere will eventually have one in production. Enforce a factory (e.g., `Executors.defaultThreadFactory` wrapped with naming + `UncaughtExceptionHandler`), an Error Prone or Architecture-Test rule that forbids raw `new Thread(...)` / `Executors.*` calls outside that factory, and a Prometheus / Micrometer integration that registers every pool automatically. Source: JCIP §8.3.1; Google *Software Engineering at Google* §20 "Static Analysis".
- [ ] **Blast radius is bounded.** If this module saturates its executor, which other modules stall? A shared HTTP client pool, the common `ForkJoinPool`, a shared DB connection pool, or a Spring `@Async` bean are all common cross-module coupling points. Name the blast-radius boundary in the PR, or size the executor so its saturation is a local event. Source: Hystrix *How it Works* (Netflix); Fowler *Patterns of Distributed Systems* "Bulkhead".
- [ ] **Container CPU quota vs `availableProcessors()` and common ForkJoinPool.** In a CFS-throttled container with cpu.max of 1 vCPU, the JVM may still report `Runtime.availableProcessors()` = host cores if ergonomics read `/proc/cpuinfo` rather than the cgroup. `ForkJoinPool.commonPool()` then sizes itself for host cores and competes with the quota. JDK 17+ `UseContainerSupport` (default on) reads the cgroup cpu.max; verify the rollout JDK honors it, and pin `-Djava.util.concurrent.ForkJoinPool.common.parallelism=<quota>` when quotas are fractional. Source: JEP 436 (Container-aware JVM ergonomics); OpenJDK JDK-8281181.
- [ ] **Retry / failover / parallelism does not multiply load on saturation.** When a downstream is already saturated, client-side retries multiply the load, not relieve it. Verify retry budgets (token-bucket, retry-per-request cap), circuit breakers (open → fail fast, not queue), and caller-side parallelism (do not fan out when a circuit is open). Source: Google SRE Book ch. 22 "Addressing Cascading Failures"; Resilience4j "RetryConfig" / "CircuitBreakerConfig".
- [ ] **Timeout, retry, and circuit-breaker policies are monotonic under load.** Non-monotonic policies — timeouts that shorten under stress, retry caps that grow, circuit breakers that close prematurely — produce thread-starvation amplification. Verify that each policy's parameters are fixed or moving in the load-reducing direction. Source: SRE Book §22.2 "Preventing Server Overload".
- [ ] **Platform threads idle on I/O are an anti-pattern on JDK 21+.** Each platform thread idle on a blocking socket pays ~512 KiB stack + context-switch CPU. For I/O-bound services on JDK 21+, prefer `Executors.newVirtualThreadPerTaskExecutor()` — the fleet shrinks and throughput scales with connections, not threads. Flag any new fixed pool of platform threads serving blocking I/O. Source: JEP 444 §Goals; Cashfree *7 lessons*.
- [ ] **Queue-depth runaway maps back to a review question.** When an executor queue depth climbs without bound overnight, exactly one of the following checklist items should have fired on the PR that introduced it: (a) unbounded queue (§5.7), (b) missing `RejectedExecutionHandler` (§5.18 saturation), (c) producer without backpressure (§5.18 saturation test), (d) task-holding-lock cascading into upstream pile-up (§5.2). Every queue-depth runaway that reaches production is a checklist gap to close. Source: SRE Book ch. 21 "Handling Overload".
- [ ] **`CompletableFuture` stage scheduling is explicit.** Non-async variants (`thenApply`, `thenAccept`, `thenCompose`) may execute on the completing thread — a Netty I/O thread, a DB-driver callback thread, the submitter's thread if the future was already complete. For any stage that does blocking work or requires a specific execution context, use the `*Async(fn, executor)` overload and pass an explicit, named executor. Source: `CompletableFuture` Javadoc "Async execution"; Baeldung *thenApply vs thenApplyAsync*.

### 5.19 Architectural Reframing of Concurrency

- [ ] **"Should this be concurrent and mutable at all?" is asked first.** Before debating primitives, attempt the three reductions that delete the question: (1) make the state immutable (every field `final`, no setters); (2) confine the state to one thread (request-scoped, `ThreadLocal`, single-writer queue); (3) push the state to an external serializer (DB row with optimistic lock, single-threaded coordinator). Any reduction that succeeds deletes a section of this checklist. Only if all three fail does "how do we synchronize?" apply. Source: JCIP ch. 3 "Sharing Objects"; Hickey, *Simple Made Easy* (Strange Loop 2011).
- [ ] **Alternative scoping considered.** For each piece of shared mutable state, ask whether it could live in a request-scoped bean, a `ThreadLocal` / `ScopedValue`, a database row with optimistic locking, or a Kafka partition (single-consumer) instead of behind an in-JVM lock. Each alternative eliminates the memory-model argument entirely. Source: JCIP §3.3 "Thread Confinement"; Kleppmann, *Designing Data-Intensive Applications* ch. 7 (on optimistic locking).
- [ ] **"Remove all `synchronized` keywords" thought experiment.** Mentally delete every `synchronized` in the class. Which invariants still hold because state is thread-confined or immutable? Which collapse? The collapsing ones name what the lock was silently protecting; the surviving ones reveal locks to remove. A lock with no collapsing invariant is dead weight. Source: JCIP §4.1 "Designing a Thread-safe Class" (invariant identification).
- [ ] **`AtomicReference` copy-on-write snapshot is evaluated.** For each mutable shared object, ask whether replacing it with an immutable snapshot held in an `AtomicReference` (`updateAndGet` for changes) eliminates a race class. Concrete win conditions: read-mostly access pattern, small object size (< ~1 KiB), no iteration-under-mutation requirement, no writer-writer contention requirement. Identify which call sites would break (mutating API → pure-function update). Source: JCIP §15.2 "Atomic Variable Classes"; Rich Hickey, *Values and Change* (Strange Loop 2009).
- [ ] **Message-passing or per-request copies evaluated.** If shared state were replaced with messages to a single-writer actor, or per-request copies merged at a commit point, would the memory-model argument disappear entirely? Often yes — at the cost of queue depth and latency. The choice is explicit, not an accident. Source: Hewitt, *Actor Model* (MIT AI Memo 197, 1973); Akka *Concurrency Guide*; Armstrong, *Programming Erlang*.
- [ ] **Four-axis critical-section sizing.** Protected regions must be large enough to preserve the multi-field invariant and small enough not to serialize the hot path. Four axes of separation: (1) *time* — split sequential phases into separate locks; (2) *space* — stripe by key / shard / partition; (3) *condition* — separate locks for read vs write, hot vs cold; (4) *level* — fine-grained inner locks under a coarse outer one, acquired in fixed order. Name which axis the diff uses. Source: Herlihy-Shavit §9 (striping); JCIP §11.4 "Reducing Lock Contention".
- [ ] **Persistent structures / versioned snapshots / read-write separation considered.** Beyond copy-on-write, evaluate: persistent data structures (Bagwell hash array mapped tries — Clojure-style, O(log32 n) update with structural sharing); versioned snapshots (`AtomicReference<VersionedState>`, readers pin a version); read-write separation (separate types for read API and mutation API, enforced at compile time). Each turns mutable-shared-state review into value-semantics review. Source: Bagwell, "Ideal Hash Trees" (EPFL TR, 2001); Okasaki, *Purely Functional Data Structures* (Cambridge UP, 1998).
- [ ] **Architectural-alternative review when patching a bug.** When a PR patches a concurrency bug with another `synchronized`, another `AtomicBoolean`, another retry, ask whether a structurally different design — actor, STM, single-writer event loop, lock-free queue, virtual threads + structured concurrency, copy-on-write snapshot — would eliminate the bug *class*, not just this instance. File a follow-up for the architectural change; do not block the hotfix on it. Source: Beck, *Tidy First?* (O'Reilly 2023) on separating hotfix from structural change.
- [ ] **Escalate ad-hoc patches to a deliberate model.** If the current diff is the third or fourth `synchronized` added to the same class in six months, the team is accumulating patches instead of choosing a model (immutable domain + single-writer, actor, STM, reactive). Flag the pattern and escalate: a PR-level review cannot fix an architecture-level drift. Source: Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" (CACM 15(12), 1972) as a generic warning about accumulated coupling.

### 5.20 Scale, Performance, and Mechanical Sympathy

- [ ] **10× thread-count stress: what breaks first?** For each concurrency design, simulate — on paper, in a benchmark, or via code review — what happens at 10× current thread count. Candidates that crack first: a coarse `synchronized` becoming a hotspot; a bounded queue overflowing; a `ThreadLocal` cache exploding heap; the common ForkJoinPool starving; a retry storm amplifying. Name the first failure and whether the design accommodates or prevents it. Source: JCIP §11.3 "Reducing Lock Contention"; Google SRE Book §22.
- [ ] **10× fine-graining: do invariants survive?** If the critical section were striped 10× finer (per-bucket lock → per-entry lock, per-partition queue → per-key queue), which invariants that held under one coarse lock still hold across stripe boundaries? Multi-key operations, size-based decisions, and "total balance ≥ 0" style invariants typically break. Stripe-level correctness is a separate proof from object-level correctness. Source: Herlihy-Shavit §9.2 "Fine-Grained Synchronization".
- [ ] **Mechanical sympathy: single-writer / Disruptor / ring buffer considered.** For the highest-contention queues, naive `BlockingQueue` allocates per message and serializes on a lock. An LMAX-style Disruptor (single-writer principle, pre-allocated ring buffer, cache-line padding) can reach tens of millions of ops/s with zero GC pressure. Evaluate whether the workload justifies the complexity — typically yes for > 1M ops/s sustained, no for < 100K. Source: Thompson et al., "Disruptor: High performance alternative to bounded queues for exchanging data between concurrent threads" (LMAX TR 2011); Fowler, *The LMAX Architecture* (martinfowler.com 2011).
- [ ] **Graceful degradation under overload.** The throughput curve should degrade gracefully past saturation, not cliff. Symptoms of cliffs: latency explodes, throughput drops below peak, recovery requires operator intervention. Mitigations: bounded queues, explicit load shedding (429 / 503), adaptive concurrency limits (TCP Vegas-style, Netflix `concurrency-limits`), priority-based queueing. Source: Google SRE Book §22 "Handling Overload"; Netflix, *Adaptive Concurrency Limits* (techblog, 2018).
- [ ] **Long loops do not delay JVM safepoints.** A counted `for (int i=...)` with no safepoint poll inside will prevent the JVM from reaching a safepoint for the duration of the loop — stop-the-world pauses, JFR sampling, and deopt all stall. Long uncounted loops, per contrast, insert a safepoint check each iteration. For latency-sensitive paths, prefer uncounted loops (long-indexed or while with interruption check), or enable `-XX:+UseCountedLoopSafepoints`. Source: Holmes, *Safepoints: Meaning, Side Effects and Overheads* (JVMLS 2015); JDK `-XX:+UseCountedLoopSafepoints` option.
- [ ] **Coordinated omission corrected in latency measurements.** If the test rig pauses its sender during a safepoint-induced response-time spike, it omits the exact samples that proved the problem — percentiles then understate tail latency. Use HdrHistogram with `recordValueWithExpectedInterval`, or tools like wrk2 / rt-bench that pre-schedule requests. Source: Gil Tene, *How NOT to Measure Latency* (Strange Loop 2015); HdrHistogram Javadoc.
- [ ] **GC choice affects race-window shape.** A 200 ms G1 pause widens every race window by 200 ms; a 10 µs ZGC pause barely moves any. If the review assumes a race "almost never" triggers, the GC choice is part of the assumption. ZGC / Shenandoah narrow most windows; Epsilon eliminates them but caps heap lifetime; Parallel's world-stops dilate them. State the assumed GC and test on it. Source: Oracle *HotSpot Virtual Machine Garbage Collection Tuning Guide* (JDK 21); ZGC JEP 377.
- [ ] **Custom spin-loops have an adaptive deschedule fallback for long waits.** A fixed spin (infinite `while (!cond)` on a `volatile`, or an unbounded CAS retry) wastes CPU while the awaited event is > ~2 × context-switch cost. The competitive-scheduling result (Herlihy-Shavit Appendix B "Hardware Basics", Exercise 219): spin up to *c* cycles, then park — the strategy is provably within 2× of any clairvoyant scheduler. Java tools: `LockSupport.parkNanos`, `Thread.onSpinWait()` hint (JEP 285) during the spin phase, or `Condition.await` with a budget. Bare `while (!cond) { }` on a custom primitive is a defect. Source: Herlihy-Shavit Appendix B, Exercise 219; JEP 285 (`Thread.onSpinWait`); `LockSupport` Javadoc.
- [ ] **Prefer `getAndAdd` over a CAS retry-loop for additive updates.** A `do { v = read(); } while (!cas(v, v + delta))` loop issues `LOCK CMPXCHG` and retries on every contended write; under high thread counts each failure re-reads and re-CASes, amplifying cache-coherence bus traffic. `AtomicInteger.getAndAdd(delta)` / `AtomicLong.getAndAdd(delta)` / `VarHandle.getAndAdd` emit a single `LOCK XADD` — one bus transaction, no retry path. Reserve CAS-loops for updates whose new value is a *non-additive* function of the current value (state-machine transition, bitwise mask, multiplicative factor). Source: Intel *64 and IA-32 Architectures Software Developer's Manual* Vol. 2A, XADD instruction entry; OpenJDK `AtomicInteger.getAndAdd` implementation (`jdk.internal.misc.Unsafe.getAndAddInt` — intrinsified to `LOCK XADD` on x86); Herlihy-Shavit §5.1 "Consensus Numbers" (fetch-and-add is sufficient for additive operations — applying a stronger primitive than needed introduces avoidable retry complexity).
- [ ] **`availableProcessors()` ≠ throughput ceiling on SMT hardware.** Intel Hyper-Threading and AMD SMT-2 report two logical processors per physical core; the pair shares execution units, L1 / L2 caches, and the TLB. Benchmarks that saturate `2N` logical CPUs on an `N`-core host can look artificially good (memory-bound work hidden by SMT) or artificially bad (CPU-bound pairs fighting for a single ALU). For CPU-bound pools, `Runtime.availableProcessors()` is an over-count; size by physical cores (`lscpu -p` / JFR `CPUInformation`) or explicitly justify SMT saturation. Source: Herlihy-Shavit Appendix B "Hardware Basics" ("Multi-Core and Multi-Threaded Processors"); Intel *64 and IA-32 Architectures Software Developer's Manual* Vol. 3A §9.5 "Processor Topology".

### 5.21 Production Diagnosis and Observability

- [ ] **Thread-dump signature pre-mortem.** For each new lock acquisition or executor, the PR author should predict the thread-dump signature a responder would see at 02:00 if it deadlocked or saturated — the thread names, the monitor object names, the stack frames. If the prediction is "I can't tell", the lock or pool lacks identifying information. Fix by naming threads (`Thread.ofPlatform().name("ingest-worker-", 0)`), naming lock objects (wrap in a named class), and keeping stack frames narrow enough to be meaningful. Source: JDK `jstack` Javadoc; Kabutz, *Hunting Java Concurrency Bugs* (InfoQ 2014).
- [ ] **SRE can diagnose a concurrency incident from metrics alone.** For every lock, executor, or blocking call the diff introduces, expose: (a) *wait time* — how long threads blocked acquiring — as a histogram; (b) *queue depth* — `ThreadPoolExecutor.getQueue().size()` or equivalent — as a gauge; (c) *rejection count* — distinct from error count — as a counter; (d) distinct exception types for timeout vs saturation vs downstream failure, so an SRE can tell which without reading the code. Source: Micrometer `ExecutorServiceMetrics`; Google SRE Book ch. 6 "Monitoring Distributed Systems".
- [ ] **Thread-dump signature → checklist gap mapping.** When a production thread dump implicates a signature no checklist item would have flagged, file a concrete update to this checklist. Over time, the checklist converges with the team's actual failure modes. Examples of signatures that should map to existing items: N threads BLOCKED on the same monitor (§5.14 deadlock detection); threads parked in `common-pool-worker` with `CompletableFuture` stacks (§5.18 `CompletableFuture` stage scheduling). Source: Google SRE Book ch. 13 "Emergency Response" (blameless post-mortem tradition).
- [ ] **Causal log ordering across concurrent paths.** Logs from a single transaction must be reconstructible post-incident. Enforce: transaction ID in MDC, timestamp at millisecond precision or better, MDC propagation across `CompletableFuture` stages and `StructuredTaskScope` forks, log aggregator with per-transaction grouping. Source: OpenTelemetry *Tracing Specification* §trace context; Sigelman et al., "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure" (Google TR 2010).

### 5.22 Cross-Paradigm and Distributed Boundaries

- [ ] **In-JVM happens-before does not cross network boundaries.** A `volatile` write in JVM A establishes an HB edge for readers in JVM A only. Across the network, a TCP socket, a Kafka topic, or a shared Redis key is needed to establish any cross-node ordering — and the ordering semantics are those of that medium, not the JMM. Flag any comment, invariant, or Javadoc that claims cross-node ordering from an in-JVM primitive. Source: JLS §17.4 (JMM scoped to a single JVM); Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System" (CACM 1978).
- [ ] **`synchronized` is not a distributed lock.** A `synchronized` block protecting a value that is actually shared through Redis, Hazelcast, Kafka, or a database serializes the *local JVM's* access only — every other JVM runs freely. If the invariant spans nodes, the serialization must be in the shared medium (Redis `SETNX` + lease, Hazelcast `ILock`, Kafka partition, `SELECT FOR UPDATE`). Source: Fowler, *Patterns of Distributed Systems*, "Leader and Followers"; Kleppmann, *DDIA* §9 "Consistency and Consensus".
- [ ] **True serialization point for uniqueness invariants.** For any "at most one" invariant (one pending order per user, one elected leader, one owner of a token), name the actual serialization point: a DB unique constraint, `SELECT FOR UPDATE SKIP LOCKED`, a Kafka partition single-consumer, a ZooKeeper ephemeral node. Application-level "check-if-exists, then insert" is almost never sufficient across nodes. Source: Kleppmann, *DDIA* §7 "Transactions"; *DDIA* §8.1 on application-level TOCTOU.
- [ ] **Kotlin coroutines, Scala Futures, Groovy bridges preserve Java contracts.** A class callable from another JVM language must specify how cancellation (Kotlin `CancellationException`, Scala `Future.cancel` semantics, Groovy `ExecutorService` bridges) maps to `Thread.interrupt`; how `@GuardedBy` annotations are respected when the caller is a coroutine context that hops dispatchers; how `InterruptedException` is surfaced through continuations. The simplest path: document the class as "callable from Java only" or provide explicit cross-language adapters. Source: Kotlin *Coroutines Guide* — Cancellation; Scala `Future` Javadoc; Kotlin/Java interop reference.
- [ ] **JNI / FFM calls reachable from `synchronized` violate carrier-thread assumptions.** A native or FFM (JEP 454) call pins a virtual thread to its carrier on JDK 21+. Nested under a `synchronized` block, the pin extends to the lock hold. Safepoints also can not occur inside the native call, delaying GC and JFR sampling. Call-site rule: either keep native calls outside `synchronized` regions, move them off virtual threads onto dedicated platform threads, or accept the pinning and size the carrier pool for the worst case. Source: JEP 454 (FFM); JEP 491 §"Remaining Limitations"; Holmes, *Safepoints* (JVMLS 2015).
- [ ] **Kotlin `suspend` → `CompletableFuture` continuation-thread is named.** When a `suspend` function is bridged to a Java `CompletableFuture` (via `kotlinx-coroutines-jdk8` `asCompletableFuture()`, Ktor, or similar), the continuation resumes on whichever dispatcher the coroutine was launched with — `Dispatchers.IO`, `Dispatchers.Default`, a custom `ExecutorCoroutineDispatcher`. The Java caller's `.thenApply` then runs on that dispatcher, not on the Java caller's executor. Verify the assumed continuation thread matches the reviewer's model. Source: kotlinx-coroutines-jdk8 API docs; Kotlin coroutines design docs §"Dispatchers".

### 5.23 Concurrency Security and Compliance

- [ ] **Race weaponization check.** For each race condition, ask whether an attacker can amplify the timing window: TOCTOU on an authorization check (is the auth result used *later* than it was computed? race-condition CVEs on file descriptors, Java capability tokens), iterator over a permissions map mutated under attacker-controllable load, double-spend between two balance updates, request smuggling through a shared buffer whose bounds are checked once. The bar for "exploitable" is low; the bar for "patchable" is smaller than one might think. Source: Wei et al., "Toward Automatically Generating Privilege-Separated Code from Information-Flow Specifications" (USENIX Security 2013); CERT Oracle Secure Coding Standard for Java — TOCTOU rules (FIO01-J).
- [ ] **Lock across untrusted input is not a DoS lever.** A lock held during parsing of user-controlled input, a slow downstream HTTP call triggered by a malformed request, or a regex with catastrophic backtracking (*ReDoS*) inside a `synchronized` block gives an attacker a single-request denial-of-service primitive. Parse and validate before taking the lock; time-box regex; move user-controlled downstreams outside the critical section. Source: OWASP *Denial of Service Cheat Sheet*; Weidman, *Penetration Testing* §ReDoS.
- [ ] **Regulated-jurisdiction failure modes are named.** For services under GDPR / HIPAA / PCI-DSS / MiFID-II / SEC Rule 613 (CAT), specific concurrency failures become compliance-reportable: cross-tenant data leakage via a shared `ThreadLocal`; audit-log loss under race; broken FIFO ordering for financial transactions; unverified consent timestamp due to causal-log gap. The reviewer should know these by name when reviewing code that touches regulated data. Source: GDPR Art. 32 "Security of processing"; PCI-DSS v4.0 §10; SEC CAT NMS Plan.
- [ ] **Serializable, auditable, reconstructible state transitions on regulated data.** Every state transition touching regulated data must be (a) serializable under concurrent contention — a DB transaction, a single-writer actor, a Kafka partition; (b) auditable — the write is durably logged with actor identity and timestamp; (c) reconstructible — from the log stream alone, the order of transitions for any entity can be replayed. An in-JVM lock alone satisfies none of these. Source: NIST SP 800-92 *Guide to Computer Security Log Management*; PCI-DSS §10.2 audit-trail requirements.

### 5.24 Build-Tool Transformations and Bytecode Rewrites

- [ ] **Review conclusions survive AOT / GraalVM native-image.** Escape analysis, lock elision, and AOT compilation (GraalVM native-image) rewrite the bytecode-to-native mapping; a synchronization guarantee at source level may elide at runtime (e.g., lock elision on a biased monitor — removed in JDK 15 — or a `StringBuffer` whose lock the JIT eliminates after escape analysis). For projects targeting GraalVM, verify that the review's HB reasoning still holds after the native-image build: run the concurrency tests against the native binary, not just the JVM. Source: Oracle GraalVM *Native Image Reference*; Biased Locking removal JEP 374.
- [ ] **AOP weaving / bytecode transformers do not invalidate review.** Spring AOP (CGLIB / JDK dynamic proxies), AspectJ compile-time or load-time weaving, Byte Buddy agents, and Kotlin compiler plugins all rewrite bytecode between source and runtime. A `synchronized` method may become a proxied call that synchronizes on a different object; a `final` field may be set after construction; a `@GuardedBy` annotation may not survive weaving. When the project uses any such tool, require proof (a trivial test harness, a jcstress harness run on the deployed jar) that the review's claims survive the transform. Source: Spring Framework *Understanding AOP Proxies*; Byte Buddy *User Guide*.
- [ ] **Kotlin synthetic bridges preserve Java thread-safety annotations.** Kotlin compiles `@JvmStatic`, `@JvmField`, and data-class `copy`/`componentN` to synthetic methods that may or may not carry the original `@GuardedBy` / `@ThreadSafe` annotations. When a class is intended to be thread-safe and is also called from Java, spot-check the bytecode for synthetic bridges that bypass the source-level guarantees. Source: Kotlin reference *Java interop* §"Annotation targets"; Kotlin `@file:JvmName` docs.

### 5.25 Review Process, Tooling, and Team Practice

- [ ] **Reviewer attention budget is allocated by risk.** In a concurrency-touching diff, roughly 10% of lines carry 90% of the risk: any `synchronized` / lock acquisition; any executor submission or `CompletableFuture` stage; any shared-field write; any new `ThreadLocal`; any `wait`/`notify`/`Condition`; any virtual-thread or FFM call. Spend disproportionate attention there; skim the rest. Source: Cohen, *Best Kept Secrets of Peer Code Review* (SmartBear 2006) — reviewer attention studies.
- [ ] **Human review vs tool gate decision is explicit.** For each concurrency property the reviewer wants to enforce, ask: can Error Prone `@GuardedBy`, SpotBugs MT detectors, IntelliJ inspections, jcstress, Lincheck, Fray, or Checker Framework enforce this mechanically? If yes, gate the build on the tool rather than a reviewer's attention — humans miss, tools do not. Source: Google *Software Engineering at Google* §20; Beller et al., "Analyzing the State of Static Analysis: A Large-Scale Evaluation in Open Source Software" (SANER 2016).
- [ ] **Repository's CI tool coverage is known.** Know which of {jcstress, Lincheck, Fray, Error Prone `@GuardedBy`, SpotBugs MT, IntelliJ inspections, Checker Framework} actually run on this repo's CI today, and their false-negative rate (most static analyzers find < 50% of the real concurrency bugs in SIR-style evaluations). A bug class the repo's tooling cannot catch needs disproportionate human review. Source: Beller et al. (SANER 2016); Li et al., *Fray* (OOPSLA 2025) — reports SIR benchmark coverage.
- [ ] **Race-detector path coverage on forbidden interleavings.** For each interleaving the PR claims to forbid, has *some* tool actually tried to execute it? jcstress for JMM-level forbidden outcomes; Lincheck for linearizability violations; Fray (OOPSLA 2025) for general interleavings. If nothing in CI tried to reproduce the forbidden interleaving, the claim is unverified. Source: Li et al., *Fray* (OOPSLA 2025); jcstress README.
- [ ] **Smallest jcstress harness demandable as merge-blocking.** For PRs touching memory-ordering or safe publication, define the smallest jcstress harness the reviewer can demand as a merge-blocking artefact — typically one `@State` class, one `@Actor`, one `@Arbiter` or `@Outcome`, 30 minutes of author time. Identify who on the team can author one. If nobody, that is a training gap, not a reviewer gap. Source: OpenJDK jcstress README; Shipilev *Java Concurrency Stress Tests — What, Why, How* (JVMLS 2015).
- [ ] **PR description attaches diagnostic artefacts when touching lock / executor / blocking call.** A PR that adds a lock, an executor, or a blocking call without attaching one of: a thread dump under realistic load, a JFR recording showing lock-wait or pinning events, an async-profiler lock-contention view — is asking the reviewer to guess at production behavior. Require the artefact. Source: Oracle *JFR User's Guide*; async-profiler GitHub README.
- [ ] **Testability triage: JUnit / jcstress / production-only.** For each concurrency behavior in the diff, classify: testable with JUnit (observable from a single thread, or with `CountDownLatch`-based stress); requires jcstress / Lincheck / Fray (depends on specific interleavings or weak memory orderings); only observable under production load (cache effects, GC interactions, CPU count). The classification dictates where the test goes — and "production only" classifications deserve extra review, not a pass. Source: JCIP ch. 12 "Testing Concurrent Programs".
- [ ] **Flaky test vs reproducible race rubric.** When a concurrency test is flaky, the diff rarely lets a reviewer tell race from infrastructure. Required discriminators: does the failure rate correlate with CPU count / load / JVM version? Does `-XX:+UnlockDiagnosticVMOptions -XX:+CheckJNICalls` expose anything? Does a jcstress harness targeted at the same interleaving reproduce it? A test that flakes once per thousand runs under load and never under a debugger is almost always a race. Source: Luo et al., "An Empirical Analysis of Flaky Tests" (FSE 2014); JCIP §12.1.
- [ ] **Merge-blocking "no-fly" conditions enforced independent of reviewer judgment.** The team should publish (in CODEOWNERS, CI config, or review rubric) conditions that block merge automatically: any new shared mutable field without `@GuardedBy`; any new `Thread.sleep` inside `synchronized`; any new `wait` without a `while` loop; any `Executors.newCachedThreadPool()` outside the approved factory; any `CompletableFuture.get()` without a timeout. These are automatable; do not rely on a human catching them. Source: Google *Software Engineering at Google* §9 "Code Review"; Error Prone bug-patterns list.
- [ ] **Post-incident blameless RCA updates the checklist.** When a concurrency bug escapes review and reaches production, the RCA must ask "which checklist item, had it existed, would have caught this?" — and either add the item or note why the item existed but was not applied. Over time this closes the tacit-knowledge gap that keeps the checklist lagging production. Source: Beyer et al., *Site Reliability Engineering* ch. 15 "Postmortem Culture"; Allspaw, *Blameless PostMortems and a Just Culture* (Etsy 2012).
- [ ] **Two-reviewer calibration on the same change.** Periodically — quarterly, or after an incident — have two reviewers independently review the same concurrency-sensitive change and compare findings. Convergence suggests a shared mental model; divergence reveals a training gap or a rubric gap. Source: Cohen, *Best Kept Secrets of Peer Code Review* §"Reviewer Calibration".
- [ ] **"Thread-safe" is a falsifiable property on this PR, not a vibe.** The author and reviewer should together state: (i) the invariant; (ii) an interleaving that would violate it absent the current synchronization; (iii) the happens-before edge that forbids that interleaving. If any of (i)–(iii) is missing, the claim is not a property — it is a virtue signal. Source: JCIP ch. 2 framing; Herlihy-Shavit §3.6 correctness obligations.
- [ ] **Author's emotional signal is a probe target.** Hand-wavy, defensive, or over-confident descriptions of the locking strategy ("it's fine, we tested it", "it can't happen because…", "the JIT handles it") correlate with missing invariants. Probe gently: ask for the interleaving that would break it, the jcstress harness that would prove it, or the thread-dump signature that would diagnose it. Confidence backed by artefacts is a different signal from confidence backed by assertion. Source: reviewer hands-on experience; Kahneman, *Thinking, Fast and Slow* on overconfidence in technical claims.
- [ ] **Junior-to-senior escalation rubric.** Maintain a written rubric of the five diff patterns a junior reviewer must always escalate: (1) any new lock-acquisition graph touching more than one monitor; (2) any custom CAS loop or hand-rolled synchronizer; (3) virtual-thread adoption on JDK 21–23; (4) any executor whose bounded-queue / rejection-handler tuning is being touched; (5) `ThreadLocal` used in pooled or virtual-thread context. Source: Google *Software Engineering at Google* §9 "Code Review" (review-authority gradient).
- [ ] **Post-mortem archive consulted on every touch to implicated systems.** Maintain a team archive of concurrency incidents (date, system, root-cause checklist-item, fix). When a PR touches a previously-implicated system, surface the past incident in the review so the reviewer's attention is primed. Over time the archive becomes the team's concurrency curriculum. Source: Beyer et al., *SRE* ch. 15 (post-mortem culture); Google *Project Oxygen* on institutional memory.

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
| **Fray** (CMU PASTA, OOPSLA 2025) | Controlled concurrency testing | Deterministic interleaving exploration; 70% more bugs than JPF, 77% more than RR chaos mode on SCTBench/JaConTeBe; discovered 18 real bugs in mature JVM projects |
| **Lincheck** (JetBrains) | Testing | Linearizability testing for concurrent data structures |
| **Meta Infer** | Static | Thread-safety violations, race conditions, deadlocks |
| **Checker Framework** | Static (annotations) | `@GuardedBy`, `@Immutable`, `@ThreadSafe` verification at compile time |
| **ThreadSafe** (Contemplate) | Static | Dedicated Java concurrency analyzer (commercial) |
| **Intellij Race Detector** | Dynamic | Experimental race condition detection |

Fray integrates with JUnit 5 via `@FrayTest`, supports Gradle/Maven, and offers deterministic replay of failing interleavings. Source: Li et al., "Fray: An Efficient General-Purpose Concurrency Testing Platform for the JVM," *Proc. ACM Program. Lang.* 9, OOPSLA2, Article 417 (Oct 2025); arXiv:2501.12618; github.com/cmu-pasta/fray.

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

1. **Pinning (JDK 21–23, mostly fixed in JDK 24+)**: In JDK 21–23, `synchronized` blocks/methods prevented virtual thread unmounting; blocking I/O inside `synchronized` pinned the carrier. This caused the Netflix production deadlock (Aug 2024, InfoQ case study): five virtual threads contended a `ReentrantLock` while four were pinned inside a `synchronized` block, occupying all four carriers and starving the fifth. JEP 491 rewrote monitor internals in **JDK 24 (March 2025)** so virtual threads acquire/hold/release monitors independently of carriers. In JDK 24+, remaining pinning sources are narrow: native methods and FFM API (JEP 454).
   ```java
   // JDK 21–23 only: avoid blocking I/O inside synchronized
   lock.lock();
   try {
       socket.read(buffer);  // Virtual thread can unmount during I/O
   } finally {
       lock.unlock();
   }
   // JDK 24+: the synchronized version also unmounts correctly.
   ```
   Detect with `-Djdk.tracePinnedThreads=short` or the JFR event `jdk.VirtualThreadPinned` (enabled by default, 20ms threshold).

2. **ThreadLocal abuse**: Each VT gets its own ThreadLocalMap; with millions of VTs, cached values multiply heap pressure. Oracle's JDK 25 docs explicitly recommend replacing `ThreadLocal<SimpleDateFormat>` with a shared immutable `DateTimeFormatter`. For context propagation, use `ScopedValue` (final in JDK 25, JEP 506).

3. **Thread pools are an anti-pattern**: Don't pool virtual threads. Create a new one per task via `Executors.newVirtualThreadPerTaskExecutor()` or `Thread.ofVirtual().start()`. Pooling limits scalability and reintroduces platform-thread semantics.

4. **CPU-bound work stays on platform threads**: Virtual threads provide no benefit for CPU-bound work. The carrier `ForkJoinPool` is capped at `availableProcessors()`. Use a fixed-size platform pool for CPU-heavy computation.

5. **Backpressure**: Millions of concurrent virtual threads can overwhelm databases, HTTP peers, and the common ForkJoinPool. Bound outstanding concurrent calls with a `Semaphore`.

6. **Heap sizing**: Virtual thread stacks live on the heap (not native memory). Migrating from platform threads typically requires higher heap allocation (e.g., `-XX:MaxRAMPercentage=35.0` or greater).

### 7.2 Structured Concurrency (Preview; JEP 453 → 462 → 480 → 499 → 505 → 525)

Treats a group of concurrent tasks as a unit. Preview status across JDK 21 (JEP 453) through JDK 26 (JEP 525); API shape shown below is the JDK 25 (JEP 505) form. Requires `--enable-preview`.

```java
// JDK 25 (JEP 505) API
try (var scope = StructuredTaskScope.open(Joiner.awaitAll())) {
    Subtask<String> userTask = scope.fork(() -> fetchUser(userId));
    Subtask<List<Order>> ordersTask = scope.fork(() -> fetchOrders(userId));
    scope.join();

    String user = userTask.get();
    List<Order> orders = ordersTask.get();
    return new UserProfile(user, orders);
}
// Scope close ensures all forked tasks are completed or cancelled
```

**Joiners** (JDK 25):
- `Joiner.awaitAll()` -- wait for all; no throw on subtask failure
- `Joiner.awaitAllSuccessfulOrThrow()` -- wait for all; throw on first failure
- `Joiner.allSuccessfulOrThrow()` -- stream of subtasks; throw on first failure
- `Joiner.anySuccessfulResultOrThrow()` -- return first success, cancel others
- Custom joiner: implement `Joiner<T,R>` with `onFork`, `onComplete`, `result`.

**JEP 525 (JDK 26) delta — API changes reviewers will see in JDK 26+ code**:
- `anySuccessfulResultOrThrow()` → renamed to `anySuccessfulOrThrow()`
- `allSuccessfulOrThrow()` returns `List<T>` directly (not a stream of `Subtask`)
- New `onTimeout()` callback on `Joiner` for fallback/partial results
- `Configuration` parameter changed from `Function` to `UnaryOperator`

**Key benefits:**
- Automatic cancellation of remaining subtasks on failure
- Thread lifetime is bounded by the scope (no leaked threads)
- Observability: thread dumps show the task hierarchy (JSON thread-dump format extended in JDK 25)

### 7.3 ScopedValue (Final in JDK 25, JEP 506)

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

**Final API note (JEP 506)**: `ScopedValue.orElse(null)` is rejected — pass a non-null fallback or call `isBound()` / `orElseThrow()` first.

---

## Key References

- Goetz et al., *Java Concurrency in Practice* (Addison-Wesley 2006) -- foundational; still relevant for core concepts
- Herlihy & Shavit, *The Art of Multiprocessor Programming*, Revised Reprint (Morgan Kaufmann 2012) -- formal treatment of consensus hierarchy (§5), spin locks and cache coherence (§7), linked-list locking strategies (§9), queues and the ABA problem (§10). The definitive textbook on concurrent algorithms; Java used as the implementation language throughout.
- Manson, Pugh, Adve, "The Java Memory Model" (POPL 2005) -- original JSR-133 formalization
- JLS Chapter 17 (Threads and Locks) -- the formal specification; §17.4.5 happens-before, §17.5 final fields, §17.6 word tearing, §17.7 non-atomic long/double
- Shipilev, *Java Memory Model Pragmatics* (2014) -- SC-DRF, OoTA, coherence, consensus, causality with worked examples
- Shipilev, *Safe Publication and Safe Initialization in Java* (2014) -- four publication idioms and their failure modes
- Doug Lea, *Using JDK 9 Memory Order Modes* (2018) -- VarHandle ordering modes
- [code-review-checklists/java-concurrency](https://github.com/code-review-checklists/java-concurrency) -- comprehensive review checklist
- [OpenJDK jcstress](https://github.com/openjdk/jcstress) -- concurrency stress testing; `BasicJMM_01..10` samples are the canonical invariants demonstrated
- JEPs: 142 (Contended annotation), 444 (Virtual Threads, JDK 21), 446 (Scoped Values preview), 453 (Structured Concurrency preview), 491 (Synchronize Virtual Threads without Pinning, JDK 24), 505 (Structured Concurrency 5th preview, JDK 25), 506 (Scoped Values final, JDK 25), 525 (Structured Concurrency 6th preview, JDK 26)

## Research Questions to Cross-Check Against Sources

The checklist is organized to answer the following questions about a diff or PR. Each question is followed by the corresponding checklist section(s) and the primary source(s) used to verify the answer.

### A. Shared mutable state and visibility
1. **Which fields are shared across threads, and which are mutable?** (§5.1 Shared Mutable State) — source: JCIP ch. 2; code-review-checklists § Documentation, § Insufficient Synchronization.
2. **Is there a happens-before edge from every write to every read?** (§1.2 Happens-Before, §5.4 Visibility) — source: JLS 17.4.5; Shipilev *JMM Pragmatics*.
3. **Is `volatile` used where visibility (not atomicity) is required?** (§2.2 Volatile) — source: JLS 17.4.5 rule 3; Shipilev.
4. **Are all accesses to the same plain field coherent?** Could two consecutive reads return `(new, old)`? (§1.6 Coherence, §4.14 Incoherent Reads) — source: jcstress `BasicJMM_05_Coherence`.
5. **Can any 64-bit `long`/`double` read observe a torn value?** (§4.13 Torn 64-bit) — source: JLS 17.7; jcstress `BasicJMM_02`.
6. **Does any field/array access risk word tearing?** (§1.6 Word Tearing) — source: JLS 17.6; jcstress `BasicJMM_03`.

### B. Atomicity of compound operations
7. **Is every read-modify-write on shared state atomic?** (§4.2, §5.3) — source: JCIP ch. 2; code-review-checklists § Race Conditions.
8. **Are check-then-act idioms on concurrent collections replaced with atomic primitives (`putIfAbsent`, `compute`, `merge`)?** (§3.1, §4.11) — source: `ConcurrentHashMap` Javadoc.
9. **Do nested `compute`/`computeIfAbsent`/`merge` calls avoid recursive updates to the same map?** (§3.1) — source: `ConcurrentHashMap` Javadoc (JDK 9+ throws `IllegalStateException`).
10. **Do compound updates across multiple fields share a single critical section, or sit inside one `AtomicReference` snapshot?** (§5.3) — source: JCIP ch. 4.

### C. Safe publication and initialization
11. **Is every published object fully constructed before the reference becomes visible?** (§4.9, §4.15, §5.5) — source: Shipilev *Safe Publication*; JCIP § 3.5.
12. **Does `this` escape during construction (listener registration, thread start, passing `this` to helpers)?** (§4.9) — source: JCIP § 3.2.
13. **Are all safe-publication idioms applied correctly: static initializer, `volatile`/`AtomicReference`, `final`, or lock-guarded?** (§5.5) — source: JCIP § 3.5; Shipilev.
14. **Does the DCL pattern use `volatile` on the singleton field and read-into-local optimization?** (§4.8) — source: Pugh, *Double-Checked Locking is Broken*; Baeldung DCL singleton.
15. **Does an object published via `final` fields rely on the reference itself being safely published?** (§4.15) — source: Shipilev.

### D. Synchronization mechanisms
16. **Is the lock object `private final`, not `this`/`Class`/a mutable/boxed field?** (§2.1, §5.2) — source: SpotBugs `WL_USING_GETCLASS_RATHER_THAN_CLASS_LITERAL`; Error Prone `SynchronizeOnNonFinalField`.
17. **Is the lock scope neither too broad (holding while doing I/O) nor too narrow (split check-then-act)?** (§5.2) — source: JCIP ch. 11.
18. **Is `lock.lock()` called *before* the try block, and `unlock()` in `finally`?** (§5.2) — source: `ReentrantLock` Javadoc; JCIP § 13.1.
19. **If multiple locks are acquired, is ordering consistent across all code paths?** (§4.4) — source: JCIP ch. 10.
20. **Is `StampedLock` used for read-heavy paths, and is its optimistic-read section side-effect-free and validated?** (§2.3) — source: `StampedLock` Javadoc.
21. **For virtual threads on JDK 21–23: are `synchronized` blocks around blocking I/O replaced with `ReentrantLock`? On JDK 24+: is this no longer required?** (§2.3 VT note, §7.1) — source: JEP 491; Netflix case study (InfoQ, Aug 2024); Oracle JDK 25 Virtual Threads docs.

### E. Reordering and consensus
22. **Does any multi-variable algorithm (Dekker, sequence locks, double-buffer) rely on total order across variables?** If so, volatile (not just acquire/release) is required. (§1.4, §1.7 Consensus) — source: jcstress `BasicJMM_07_Consensus`; Doug Lea *JDK 9 Memory Order Modes*.
23. **Does any optimization assume plain-field reads cannot be reordered across a volatile access?** (§1.4) — source: JLS 17.4.4, 17.4.5.
24. **Is any VarHandle mode weaker than `volatile` used?** Is the choice justified by a cited requirement (not "performance")? (§2.4) — source: Doug Lea.
24a. **If the code implements a lock-free or wait-free algorithm, does it use `compareAndSet` (consensus number ∞)?** Algorithms built only on `getAndSet`/`getAndIncrement` (consensus number 2) cannot be wait-free for 3+ threads. (§1.6) — source: Herlihy-Shavit §5.
24b. **What progress condition does the code claim, and does the primitive support it?** Wait-free must not contain retry loops; lock-free may; obstruction-free may livelock under contention. (§1.6) — source: Herlihy-Shavit §5.1, §9.8.

### F. Concurrent collections and utilities
25. **Are concurrent-collection iterators treated as weakly consistent (no `ConcurrentModificationException`)?** (§3.1) — source: `ConcurrentHashMap` Javadoc.
26. **Are `size()`, `isEmpty()`, `containsValue()` on concurrent collections treated as estimates, not truths for control flow?** (§3.1, §4.12) — source: `ConcurrentHashMap` Javadoc.
27. **On maps that may exceed `Integer.MAX_VALUE`, is `mappingCount()` used instead of `size()`?** (§3.1) — source: `ConcurrentHashMap` Javadoc.
28. **Is `CopyOnWriteArrayList` used only for read-dominant workloads?** (§3.2) — source: `CopyOnWriteArrayList` Javadoc; JCIP § 5.2.
29. **Is the `BlockingQueue` choice bounded (prevents OOM under producer overwhelm)?** (§3.3) — source: JCIP § 5.3.
30. **Are `offer()` return values inspected (non-blocking put can fail)?** (§3.3) — source: `BlockingQueue` Javadoc.
31. **Is `Collections.synchronizedXxx` iteration guarded by an external `synchronized(col)` block?** (§3.5) — source: `Collections.synchronizedList` Javadoc.

### G. Executors, futures, and async
32. **Does every `ExecutorService` have a matching shutdown path (`shutdown()` + `awaitTermination()`)?** (§5.7) — source: JCIP ch. 7.
33. **Is `Executors.newCachedThreadPool()` avoided for unbounded-task workloads?** (§5.7) — source: JCIP § 8.3.
34. **Are exceptions from `Runnable.run()` observed — via `Future.get()`, `UncaughtExceptionHandler`, or `execute()` semantics?** (§5.8) — source: `ExecutorService` Javadoc.
35. **Every `CompletableFuture` chain ends in `exceptionally`/`handle`/`whenComplete`?** (§2.6, §5.8) — source: `CompletableFuture` Javadoc.
36. **Non-async `CompletableFuture` callbacks: is it OK that they may run on the completing thread (Netty I/O, lock holders)?** (§2.6) — source: Baeldung *thenApply vs thenApplyAsync*.

### H. Interruption, cancellation, timing
37. **Is `InterruptedException` never swallowed — always rethrown or the flag restored via `Thread.currentThread().interrupt()`?** (§5.9) — source: Oracle Concurrency Tutorial; JCIP ch. 7.
38. **Do long-running loops poll `Thread.interrupted()`?** (§5.9) — source: JCIP ch. 7.
39. **Is `Future.cancel(true)` honored by the task code?** (§5.9) — source: `Future` Javadoc.
40. **Is elapsed time measured with `System.nanoTime()` (monotonic), not `currentTimeMillis()` (wall clock, can jump)?** — source: code-review-checklists § Time; `System.nanoTime` Javadoc.

### I. Virtual threads and modern concurrency (JDK 21+)
41. **Are virtual threads not pooled?** (§7.1) — source: Oracle JDK 25 Virtual Threads docs.
42. **Is CPU-bound work kept on platform threads?** (§7.1) — source: Oracle docs; Cashfree *7 lessons*.
43. **Is backpressure bounded (e.g., `Semaphore`) when spawning many virtual threads against a shared downstream?** (§7.1) — source: Cashfree *7 lessons*; Netflix case study.
44. **Is heap sizing adequate for virtual-thread stacks (on-heap)?** (§7.1) — source: Cashfree *7 lessons*.
45. **Is `ThreadLocal` replaced with `ScopedValue` (final in JDK 25, JEP 506) for context propagation?** (§4.16, §7.3) — source: JEP 506; Oracle JDK 25 `ScopedValue` Javadoc.
46. **Are structured concurrency scopes used so task lifetime is bounded?** (§7.2) — source: JEP 505 / 525.
47. **Is the target JDK version known, so pinning guidance (JEP 491) is applied correctly?** — source: JEP 491.

### J. Testing and diagnosis
48. **Are concurrency-sensitive algorithms covered by jcstress tests (for memory-ordering assumptions) or Fray tests (for interleaving bugs)?** (§6.1–6.4) — source: OpenJDK jcstress; Li et al. *Fray* (OOPSLA 2025).
49. **Is JFR / `jdk.VirtualThreadPinned` monitoring enabled in perf envs to catch pinning?** (§6.2, §7.1) — source: Oracle JDK 25 docs.
50. **Do stress tests use `CountDownLatch` to synchronize start, run for many iterations, and span >1 CPU architecture where possible?** (§6.3) — source: JCIP ch. 12; jcstress README.

### K. Lock-free algorithms, ABA, and cache effects
51. **Does any lock-free CAS-loop protect against ABA?** If the value can be re-introduced (pooled nodes, version counters), use `AtomicStampedReference` or `AtomicMarkableReference`. (§4.16) — source: Herlihy-Shavit §10.6.
52. **Is the locking strategy (coarse / fine / optimistic / lazy / nonblocking) appropriate for the read/write ratio and contention level?** (§5.11) — source: Herlihy-Shavit §9.
53. **For optimistic reads: does every traversal re-validate under a lock before committing the decision?** (§5.11) — source: Herlihy-Shavit §9.6.
54. **Does the code implement a lock-free algorithm while also recycling nodes manually?** Nonblocking algorithms rely on GC reachability for freedom from interference; manual recycling reintroduces ABA. (§4.16, §5.11) — source: Herlihy-Shavit §9.3.
55. **Are hot per-thread fields / counters padded or striped to avoid false sharing?** Look for arrays of `long` counters, per-slot flags, `AtomicLong` held in a tight loop across threads. Use `@jdk.internal.vm.annotation.Contended` or `LongAdder`. (§4.17) — source: Herlihy-Shavit §7.5.1; JEP 142.
56. **For concurrent data-structure changes: are the representation invariants explicit, and does each mutation preserve them at every interleaving point?** (§5.12) — source: Herlihy-Shavit §9.3.
57. **If the algorithm assumes sequential consistency across multiple fields, is a memory barrier present at each step?** Peterson/Bakery-style algorithms fail on modern CPUs because store buffers delay cross-variable ordering. In Java, use `volatile` or `VarHandle.setVolatile` for the step that must be globally ordered. (§1.4, §2.2) — source: Herlihy-Shavit §7.1.
58. **For contended spin/try-locks: is there randomized exponential backoff to avoid lockstep retry storms?** Applies to custom CAS loops, not to `ReentrantLock` (which parks). (§4.5) — source: Herlihy-Shavit §7.4.
59. **Is the lock object on a *different* cache line from the data it protects under contention, and on the *same* line when almost always uncontended?** Waiters invalidating the lock line must not thrash the data; acquiring an uncontended lock should prefetch the data as a side effect. (§4.17, §5.20) — source: Herlihy-Shavit Appendix B "Hardware Basics", p.17–18.
60. **Do custom spin-loops have an adaptive deschedule fallback (spin up to *c* cycles, then `LockSupport.parkNanos`)?** Fixed-decision spin on long waits wastes CPU; the 2-competitive bound (Exercise 219) is lost. (§5.20) — source: Herlihy-Shavit Appendix B "Hardware Basics", Exercise 219; JEP 285.
61. **Is thread-count ceiling sized against *physical* cores on SMT hardware, not `Runtime.availableProcessors()`?** Two SMT siblings share execution units, L1 / L2, and the TLB — CPU-bound pools sized to logical CPUs oversubscribe. (§5.20) — source: Herlihy-Shavit Appendix B "Hardware Basics" ("Multi-Core and Multi-Threaded Processors"); Intel SDM Vol. 3A §9.5.

---

## Sources

### Normative / specification
- [JLS Chapter 17: Threads and Locks (Java SE 25)](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html) -- 17.4.5 happens-before, 17.5 final-field semantics, 17.6 word tearing, 17.7 non-atomic long/double
- [ConcurrentHashMap Javadoc (JDK 25)](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html) -- weakly consistent iterators, recursive-update detection, `mappingCount`
- [StructuredTaskScope Javadoc (JDK 25)](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/StructuredTaskScope.html)
- [ScopedValue Javadoc (JDK 25)](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/ScopedValue.html) -- finalized in JDK 25
- [Virtual Threads (Oracle JDK 25)](https://docs.oracle.com/en/java/javase/25/core/virtual-threads.html) -- pinning, ThreadLocal guidance, JFR events
- [Structured Concurrency (Oracle JDK 25)](https://docs.oracle.com/en/java/javase/25/core/structured-concurrency.html)

### JEPs
- [JEP 444: Virtual Threads (JDK 21)](https://openjdk.org/jeps/444)
- [JEP 491: Synchronize Virtual Threads without Pinning (JDK 24)](https://openjdk.org/jeps/491)
- [JEP 505: Structured Concurrency, 5th Preview (JDK 25)](https://openjdk.org/jeps/505)
- [JEP 506: Scoped Values, Final (JDK 25)](https://openjdk.org/jeps/506)
- [JEP 525: Structured Concurrency, 6th Preview (JDK 26)](https://openjdk.org/jeps/525)

### Foundational / authoritative
- **Herlihy & Shavit, *The Art of Multiprocessor Programming*, Revised Reprint (Morgan Kaufmann 2012)**, ISBN 978-0-12-397337-5. [O'Reilly learning](https://learning.oreilly.com/library/view/the-art-of/9780123973375/). Core chapters for Java review: §5 primitive synchronization operations and the consensus hierarchy; §7 spin locks, cache coherence, exponential backoff, queue locks, false sharing; §9 linked-list locking strategies (coarse / fine-grained / optimistic / lazy / nonblocking); §10 concurrent queues, linearization, and the ABA problem.
- Manson, Pugh, Adve, "The Java Memory Model" (POPL 2005)
- Goetz et al., *Java Concurrency in Practice* (Addison-Wesley 2006)
- [The "Double-Checked Locking is Broken" Declaration (Pugh et al.)](https://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)
- [Using JDK 9 Memory Order Modes (Doug Lea, 2018)](https://gee.cs.oswego.edu/dl/html/j9mm.html)
- [Java Memory Model Pragmatics (Shipilev, 2014)](https://shipilev.net/blog/2014/jmm-pragmatics/) -- SC-DRF, OoTA, coherence, consensus
- [Safe Publication and Safe Initialization in Java (Shipilev, 2014)](https://shipilev.net/blog/2014/safe-public-construction/) -- four idioms with worked examples

### Tooling
- [OpenJDK jcstress](https://github.com/openjdk/jcstress) -- empirical JMM conformance tests
- [jcstress BasicJMM samples (01–10)](https://github.com/openjdk/jcstress/tree/master/jcstress-samples/src/main/java/org/openjdk/jcstress/samples/jmm/basic) -- canonical invariant demonstrations: DataRaces, AccessAtomicity, WordTearing, Progress, Coherence, Causality, Consensus, Finals, BenignRaces, OOTA
- [SpotBugs](https://spotbugs.github.io/) -- static concurrency-bug detector
- Li, Kang, Vikram et al., "Fray: An Efficient General-Purpose Concurrency Testing Platform for the JVM", *Proc. ACM Program. Lang.* 9, OOPSLA2, Article 417 (Oct 2025). [arXiv:2501.12618](https://arxiv.org/abs/2501.12618) · [github.com/cmu-pasta/fray](https://github.com/cmu-pasta/fray)

### Checklists and curated reviews
- [code-review-checklists/java-concurrency (Leventov et al.)](https://github.com/code-review-checklists/java-concurrency) -- most comprehensive public Java-concurrency review checklist
- [Code Review Checklist: Java Concurrency | freeCodeCamp (Leventov, 2019)](https://www.freecodecamp.org/news/code-review-checklist-java-concurrency-49398c326154/)

### Industry case studies and explainers
- [Netflix Adopts Virtual Threads: Performance and Pitfalls (InfoQ, Aug 2024)](https://www.infoq.com/news/2024/08/netflix-performance-case-study/) -- motivating real-world deadlock for JEP 491
- [Java 24: Thread pinning revisited (mikemybytes, Apr 2025)](https://mikemybytes.com/2025/04/09/java24-thread-pinning-revisited/) -- empirical verification of JEP 491 fix
- [7 Key Lessons from Java 21 Virtual Threads in Production (Cashfree)](https://www.cashfree.com/blog/java-21-virtual-threads-lessons-production/) -- backpressure, heap sizing, monitoring
- [Java 25: Virtual Threads with Structured Task Scopes and Scoped Values (JavaPro, Dec 2025)](https://javapro.io/2025/12/23/java-25-getting-the-most-out-of-virtual-threads-with-structured-task-scopes-and-scoped-values/)
- [Hunting Java Concurrency Bugs (Kabutz, InfoQ 2014)](https://www.infoq.com/articles/Hunting-Concurrency-Bugs-1/)
- [Mastering the Java Memory Model (Arvind Kumar, 2025)](https://codefarm0.medium.com/mastering-the-java-memory-model-jmm-visibility-reordering-volatile-demystified-2025-deep-aafad27c8bcf)
- [The Misunderstood Thread Safety of ConcurrentHashMap (Arvind Kumar, 2025)](https://codefarm0.medium.com/the-misunderstood-thread-safety-of-concurrenthashmap-edf86427a641)
- [Avoid Recursion in ConcurrentHashMap.computeIfAbsent (Lukas Eder, jOOQ blog)](https://blog.jooq.org/avoid-recursion-in-concurrenthashmap-computeifabsent/) -- JDK-8062841 context

---

## Gotchas — Common Wrong Assumptions

G-01. **"volatile makes compound ops atomic"** — false. `v++` on a volatile field is still read-modify-write. Use `AtomicInteger` / `LongAdder`. See §2.2, §4.2.
G-02. **"synchronized pins virtual threads"** — true on JDK 21–23 only. JEP 491 (JDK 24, March 2025) eliminated it. Remaining pinning: native methods, FFM API. See §2.3, §7.1.
G-03. **"Two plain-field reads of the same variable will see the same or increasing values"** — false. jcstress `BasicJMM_05_Coherence` produces `(1, 0)` on plain fields. Promote to `volatile` or `VarHandle.getOpaque()` for per-variable total order. See §1.6, §4.14.
G-04. **"acquire/release is equivalent to volatile for Dekker-style mutual exclusion"** — false. Only volatile guarantees global consensus (total order across variables). AcqRelDekker shows ~6% `(0,0)` outcomes empirically (jcstress `BasicJMM_07_Consensus`). See §1.4, §2.4.
G-05. **"final fields protect against unsafely published objects"** — partially. `final` guarantees the fields' initialized values are visible IF the object is seen at all. Unsafe publication of the reference itself can still return `null`. See §4.15.
G-06. **"DCL works if I just add the second null check"** — false. The singleton field MUST be `volatile`. Without it, the reference may be published before the constructor finishes. See §4.8.
G-07. **"`ConcurrentHashMap.size()` gives an accurate count"** — false. It is an estimate under concurrent updates. Use `mappingCount()` for counts exceeding `Integer.MAX_VALUE`. See §3.1, §4.12.
G-08. **"Nested `ConcurrentHashMap.compute()` on the same map just blocks"** — false on JDK 9+. It throws `IllegalStateException: Recursive update`. See §3.1, §4.4.
G-09. **"`Executors.newCachedThreadPool()` is a safe default"** — false. It creates unbounded threads; a burst of tasks can exhaust memory. See §5.7.
G-10. **"ScopedValue is still preview"** — false since JDK 25 (JEP 506). `ScopedValue.orElse(null)` is rejected in the final API. See §7.3.
G-11. **"`System.currentTimeMillis()` is fine for measuring elapsed time"** — false. Wall clock can jump (NTP, DST, manual edits). Use `System.nanoTime()` for intervals. See Question 40.
G-12. **"Swallowing `InterruptedException` is OK if the method doesn't declare it"** — false. Restore the flag via `Thread.currentThread().interrupt()` and exit promptly. See §5.9.
G-13. **"`CompletableFuture.thenApply()` runs on an executor"** — not necessarily. Non-async variants may run on the completing thread (Netty I/O, a lock holder). Use `thenApplyAsync(fn, executor)` when the callback's execution context matters. See §2.6.
G-14. **"64-bit `long` reads are atomic"** — only guaranteed for `volatile long`. JLS 17.7 allows non-volatile `long`/`double` to tear into two 32-bit halves. See §4.13.
G-15. **"`BitSet` is just an array of bits, so concurrent updates to different bits are safe"** — false. `BitSet` packs bits densely; concurrent updates tear at the word level (jcstress `BasicJMM_03_WordTearing` measured ~16–18% lost updates). Use `AtomicIntegerArray` or per-bit `AtomicBoolean`. See §1.7.
G-16. **"`compareAndSet` succeeded, therefore the data structure state is unchanged"** — false. The ABA problem: a value can change A → B → A between the read and the CAS. Use `AtomicStampedReference` or `AtomicMarkableReference` when references may be recycled or a sequence can repeat. See §4.16.
G-17. **"`volatile` alone is enough for a lock-free queue/stack"** — false. Atomic registers (including `volatile`) have consensus number 1; they cannot build wait-free queues/stacks/sets for 2+ threads (Herlihy-Shavit Corollary 5.2.1). Lock-free collections need `compareAndSet`. See §1.6.
G-18. **"Wait-free and lock-free mean the same thing"** — false. Wait-free: *every* call finishes in bounded steps (per-thread progress). Lock-free: *some* call finishes (system-wide progress). A CAS-retry loop is lock-free, not wait-free. See §1.6.
G-19. **"Per-thread counters in an array won't contend because threads only touch their own slot"** — false. Entries on the same cache line cause false sharing: one thread's write invalidates neighbors' cache lines. Pad with `@jdk.internal.vm.annotation.Contended` or use `LongAdder`. See §4.17.
G-20. **"Peterson's/Bakery/Dekker's algorithm gives mutual exclusion on modern CPUs"** — false. They rely on sequential consistency across two variables. Modern CPUs have store buffers that reorder writes to different addresses. In Java, rebuild the algorithm's critical transitions on `volatile` fields or use `synchronized`/`ReentrantLock` — do not roll your own mutex from plain ints. See §1.4, §7.1 of Herlihy-Shavit.
G-21. **"TTAS and TAS are equivalent — they just spin"** — false. Plain TAS (`while (state.getAndSet(true))`) causes a cache-coherence storm; TTAS first reads (on the local cache line) and only attempts CAS when the lock appears free. For custom CAS spin loops, add exponential backoff to avoid lockstep retry. See §4.5 and Herlihy-Shavit §7.3–7.4.
G-22. **"Memory barriers are cheap, just sprinkle them everywhere"** — false. Herlihy-Shavit §7.1: memory barriers cost "about as much as an atomic `compareAndSet()` instruction." Reads and writes of `volatile` fields already include a memory barrier on most architectures; double-barriering via `Unsafe.fullFence()` adds latency without benefit in most code paths.
G-23. **"An optimistic read is free — no lock, no cost"** — false. The validation step after the read is mandatory. `StampedLock.tryOptimisticRead` returns a stamp that must be passed to `validate(stamp)`; if it returns false, the read was inconsistent and must be retried under the full read lock. See §2.3.
G-24. **"`if (cond) wait()` is equivalent to `while (!cond) wait()`"** — false. Spurious wakeups and notify-before-wait orderings both mean the thread may wake with `cond` still false. Always use `while`. Source: `Object.wait` Javadoc. See §5.15.
G-25. **"`synchronized` on a Spring-proxied method serializes on `this`"** — false. It serializes on the proxy, and self-calls (`this.method()`) bypass the proxy entirely, so two calls to "the same" synchronized method can hold different monitors or none. Use a private final lock inside the method. See §5.6.
G-26. **"A `synchronized` block protects state shared via Redis / Hazelcast / DB"** — false. `synchronized` is a local-JVM primitive. Cross-node invariants require a distributed serialization point (Redis SETNX + lease, DB unique constraint, `SELECT FOR UPDATE`, Kafka single-consumer partition). See §5.22.
G-27. **"A flaky concurrency test is just infrastructure noise"** — usually false. Tests that flake once per thousand runs under load and never under a debugger are almost always reproducible races. Run jcstress against the suspected interleaving before dismissing. See §5.25. Source: Luo et al., "An Empirical Analysis of Flaky Tests" (FSE 2014).
G-28. **"`Runtime.availableProcessors()` reports the container's CPU quota"** — depends. JDK 17+ with `-XX:+UseContainerSupport` (default on) reads `cpu.max` from cgroup v2; older JDKs or container runtimes without cgroup support may report host cores. `ForkJoinPool.commonPool()` then oversizes. Pin `-Djava.util.concurrent.ForkJoinPool.common.parallelism=<quota>` when in doubt. See §5.18. Source: JEP 436.
G-29. **"A `ThreadLocal` set in a `CompletableFuture` stage is visible to later stages"** — false. Non-async stages may run on any carrier; the `ThreadLocal` on stage 2 is whatever was left by the previous task on that carrier, or empty. Use `ScopedValue` (JDK 25+) or capture context explicitly into the stage input. See §5.17.
G-30. **"`Future.cancel(true)` stops the task"** — false. It sets the interrupt flag; if the task does not cooperate (no `Thread.interrupted()` check, no interruptible blocking call), the task continues mutating shared state while the caller has moved on. See §5.9. Source: `Future.cancel` Javadoc.
G-31. **"A lock held across a remote HTTP call is fine because the SLA is short"** — false. A 99p 30 ms remote call occasionally spikes to 30 s under retries, DNS failure, or downstream GC. Every queued requester waits that long. Release the lock before the call. See §5.2, §5.22.
G-32. **"`ExecutionException` caught and logged is handled"** — usually false. If `e.getCause()` is dropped, the original exception's stacktrace and class are invisible to the triage team. Either rethrow as a domain exception carrying the cause, or log the cause explicitly. See §5.8.
G-33. **"Virtual thread = thread, so size the pool for CPU count"** — false. Virtual threads are scheduler tasks; do not pool them. Use `Executors.newVirtualThreadPerTaskExecutor()` or `Thread.ofVirtual().start()` per task. Pool sizing applies to platform threads and to downstream peer-limiting via `Semaphore`. See §5.13. Source: JEP 444; Alan Bateman, *Virtual Threads: State of Play*.
G-34. **"Lock elision / escape analysis can't change my review conclusions"** — sometimes false. On GraalVM native-image, JIT-time elisions become AOT-time elisions; a `StringBuffer` passed to a method that the AOT compiler proves non-escaping may drop its synchronization. Test on the deployed binary, not only the JVM. See §5.24. Source: Oracle GraalVM *Native Image Reference*.
G-35. **"Putting the lock next to the data it protects is always good locality"** — false. Under contention, waiters invalidating the lock's cache line thrash the adjacent data on every release; pad them onto separate lines. The heuristic *inverts* for an almost-always-uncontended lock, where colocation lets the acquire prefetch the data. See §4.17, §5.20. Source: Herlihy-Shavit Appendix B "Hardware Basics", p.17–18.
G-36. **"A `while (!cond) { }` spin on a `volatile` is fine — it's just a busy-wait"** — false for long waits. A fixed spin with no deschedule fallback wastes cycles past ~2 × context-switch cost; the competitive bound (spin for *c*, then park) is lost. Add `Thread.onSpinWait()` during the spin and `LockSupport.parkNanos` past a threshold, or use `Condition.await`. See §5.20. Source: Herlihy-Shavit Appendix B "Hardware Basics", Exercise 219; JEP 285.
G-37. **"`Runtime.availableProcessors()` is the number of cores to size a CPU-bound pool against"** — misleading on SMT hardware. It reports logical CPUs; two SMT siblings share ALUs, L1, and TLB. A CPU-bound pool sized to logical count oversubscribes and sibling-pairs fight for the same execution unit. Size against physical cores, or justify the SMT saturation explicitly. See §5.20. Source: Herlihy-Shavit Appendix B "Hardware Basics" ("Multi-Core and Multi-Threaded Processors"); Intel SDM Vol. 3A §9.5.
G-35. **"A bounded `BlockingQueue` gives you backpressure"** — partial. It gives you producer back-pressure only if producers actually block on `put` or observe `offer` return false. Producers that call `offer` and ignore the result silently drop work; producers running on a fire-and-forget path (e.g., inside a `@EventListener`) propagate load pressure nowhere. See §5.18.
