# Java Performance Review Checklist

Concise reference for finding performance bugs during code review. Focus on subtle, non-obvious issues.

## Table of Contents
1. Object Layout & CPU Cache -- header size, 8-byte alignment, field reordering, false sharing, NUMA. JEP 519 compact headers (JDK 25)
2. Data Structures -- HashMap traps, collection choice table, String traps (Compact Strings JEP 254)
3. Synchronization & Locking -- lock cost ladder (x86/ARM), lock states after JEP 374, StampedLock, virtual-thread pinning (JEP 491)
4. JIT Compilation & Code Shape -- inlining (MaxInlineSize=35 vs FreqInlineSize=325), megamorphic call sites, escape analysis, intrinsics, 11 hot-path bug patterns with bad:/good: code pairs
5. GC & Allocation Pressure -- TLAB fast path, humongous allocations, autoboxing, review checklist (14 items)
6. I/O & Networking -- socket tuning, ByteBuffer, zero-copy, common traps
7. Common Anti-Patterns in Hot Paths -- lookup table
8. Profiling Toolkit -- async-profiler 4.3, JFR, JOL, JMH 1.37, JITWatch, JVM flags
9. Quick Review Heuristic -- 8-step scan order

Then:
- Key References -- inline foundational list
- Research Questions to Cross-Check Against Sources -- 10 lettered buckets (A-J), 48 questions
- Sources -- 6 sub-groups (Normative, JEPs, Foundational, Tooling, Checklists, Industry)
- Gotchas -- 18 common wrong assumptions with stable IDs (G-01..G-18)

---

## 1. Object Layout & CPU Cache

### What to look for
- **Object header overhead**: 12 bytes (compressed oops) or 16 bytes without compression. Shipilev *Objects Inside Out* confirms the 8-byte alignment rule
- **Compact Object Headers** (JDK 25, JEP 519 Final): `-XX:+UseCompactObjectHeaders` shrinks headers to 8 bytes via a 22-bit class pointer. SPECjbb2015: 22% less heap, 8% less CPU. Amazon production: up to 30% CPU reduction. Was experimental in JDK 24; production-ready in JDK 25
- **Object alignment**: all objects padded to 8-byte boundary. A class with a single `boolean` field still occupies 16 bytes (12 header + 1 field + 3 padding). Source: Shipilev Quark #24 Object Alignment
- **Field reordering**: HotSpot sorts fields by size (longs/doubles first, then ints, shorts, bytes, references) to minimize padding. Declaring fields in a different order does not change layout. Source: Shipilev *Objects Inside Out*
- **False sharing**: two frequently-written fields from different threads landing on the same cache line. Common in counters, flags, ring-buffer indices. Measured impact: 170 ns/op vs 66 ns/op; L1 cache misses drop 8x with `@Contended` (Dehghani, alidg.me false-sharing)
  - Fix: `@jdk.internal.vm.annotation.Contended` (JEP 142, JDK 8) adds 128 bytes padding (2 cache lines — covers prefetch-adjacent-line behavior). Configurable via `-XX:ContendedPaddingWidth` (multiple of 8, 0-8192). Requires `-XX:-RestrictContended` for non-JDK classes. Source: Oaks *Java Performance* 2e p.296; JEP 142
  - **Cache line size**: 64 B on x86-64; **128 B on ARM/Apple Silicon M1/M2/M3** (7-cpu.com/cpu/Apple_M1). HotSpot defaults to 128 B padding to cover both architectures
- **Array vs linked structures**: `ArrayList` elements are contiguous (cache-friendly); `LinkedList` nodes are scattered (pointer chasing = cache miss per node; 24 bytes/node overhead)
- **Traversal order matches memory layout**: Java arrays are row-major — `a[i][j]` places `a[i][0..n]` contiguously, and `a[i+1]` is a separate heap-allocated sub-array. Iterating with `i` outer, `j` inner walks memory linearly and hits L1 once per cache line; swapping the loops (`j` outer, `i` inner) strides across sub-arrays and can produce near-100% L1 misses, ~10–100× slower. The same reasoning applies to `ByteBuffer` strides, packed struct layouts, and image/matrix traversal. The JIT will not rewrite the loop order for you. Source: Herlihy & Shavit, *The Art of Multiprocessor Programming*, Appendix B "Hardware Basics" (spatial / temporal locality and cache-line granularity); Drepper, *What Every Programmer Should Know About Memory* (Red Hat, 2007) §3.3
- **NUMA effects**: on multi-socket systems, accessing memory from a remote NUMA node costs 2-3x. JVM flag: `-XX:+UseNUMA` enables NUMA-aware allocation

### How to detect
- **JOL** (Java Object Layout): `java -jar jol-cli.jar internals com.example.MyClass` — shows exact byte layout, padding, alignment
- **IdeaJol** plugin for IntelliJ — visual object layout in IDE
- `perf c2c` (Linux) — detects false sharing by tracking cache-line contention across cores
- `perf stat -e cache-misses,cache-references` — L1/L2/L3 cache miss ratio

---

## 2. Data Structures

### HashMap traps
Source: `HashMap` Javadoc; Oaks Ch 12 p.392-395.
- **Initial capacity**: default 16, load factor 0.75. If size known, `new HashMap<>(expectedSize * 4 / 3 + 1)` avoids rehashing; each rehash copies the entire table
- **Bad hashCode()**: if keys cluster, O(1) degrades to O(log n) via treeification (≥8 entries/bucket, since JDK 8) or O(n) before treeification
- **Key mutability**: if key is mutated after insertion, `get()` never finds it (hash changed but bucket unchanged)
- **Treeification threshold**: buckets convert to red-black trees at 8 entries and revert to lists at 6. With good hash, never triggers

### Collection choice
Sources: Oaks Ch 12 p.392-395; Baeldung Eclipse Collections primer; respective Javadocs.

| Scenario | Use | Avoid | Why |
|--|--|--|--|
| Random access | `ArrayList` | `LinkedList` | `LinkedList`: 24 bytes/node overhead, cache-hostile |
| Enum keys | `EnumMap` / `EnumSet` | `HashMap` / `HashSet` | `EnumMap` is a flat array indexed by ordinal — O(1), zero hashing |
| Small immutable | `List.of()`, `Map.of()` | `new ArrayList<>()` | `List.of()` uses compact field-based storage (0-2 elements) or array without unused slots |
| Primitives (int, long) | Eclipse Collections, HPPC, fastutil | `HashMap<Integer,V>` | Boxed: 36 bytes/entry. Primitive: 8 bytes/entry. ~77% less memory, up to 10x faster (Eclipse Collections benchmarks) |
| Concurrent reads | `ConcurrentHashMap` | `Collections.synchronizedMap` | CHM uses lock striping (per-bucket CAS); synced map locks entire map |
| Sorted data | `TreeMap` (if needed) | `LinkedList` as sorted list | Insertion sort in `LinkedList` is O(n) per insert |

### String traps
- **Compact Strings** (JDK 9+, JEP 254): strings with only Latin-1 characters use 1 byte/char via `byte[] + LATIN1 coder`; mixing in a single non-Latin-1 char doubles the backing array to UTF-16. Source: JEP 254; Oaks p.363
- **`String.intern()`**: native hashtable, 60013 buckets default since JDK 7u40. At scale, becomes a bottleneck. Prefer manual deduplication with `ConcurrentHashMap.computeIfAbsent`. Source: Shipilev Quark #10 String.intern
- **String concatenation with `+`**: since JDK 9, compiled to `invokedynamic` → `StringConcatFactory`. Fine for single concatenations; inside a loop, each iteration still allocates a new `String` — use `StringBuilder` for loops. Source: JEP 280 (Indify String Concatenation); Oaks Ch 12

---

## 3. Synchronization & Locking

### Lock cost ladder (approximate, single-socket, uncontended)

| Operation | x86-64 cost | ARM (AArch64) cost | Notes / source |
|--|--|--|--|
| Normal field read/write | ~1 ns | ~1 ns | |
| `volatile` read (uncontended) | ~1 ns (TSO plain load) | ~2-5 ns (needs `LDAR`) | Brooker 2012/09/10. **Contended** volatile reads ~25x slower due to cache coherency |
| `volatile` write | ~20-50 ns (`LOCK`ed store / `mfence`) | ~10-30 ns (`STLR`) | Brooker 2012/11/13 |
| CAS (`compareAndSet`) | ~15-30 ns (L1 hit) | ~15-40 ns (LDXR/STXR loop) | Multi-socket NUMA: 100-300 ns. Brooker 2012/11/13 (`lock cmpxchg`) |
| `synchronized` (uninflated, uncontended) | ~100-300 ns | ~100-300 ns | **Oaks p.288: "on the order of a few hundred nanoseconds"** — not ~20 ns as commonly claimed |
| `synchronized` (first contention → inflation) | ~10 μs | ~10 μs | OS mutex allocation, context switch (Oaks p.288) |
| `synchronized` (inflated, contended) | ~1-10 μs per acquire | ~1-10 μs | OS thread park/unpark, syscall (Oaks Ch 9) |
| `ReentrantLock` (uncontended) | ~20-50 ns | ~20-50 ns | CAS on state field (Oaks p.287) |

### Lock states (JDK 15+ — biased locking disabled by default, JEP 374)
JEP 374 (JDK 15, Sep 2020) **disabled biased locking by default** and deprecated `-XX:+UseBiasedLocking`. Flag still exists but warns; it is not fully removed. Modern lock states:
1. **Thin (uninflated) lock** (uncontended): CAS on object mark word. Oaks p.288: "a few hundred nanoseconds"
2. **Spinning**: brief busy-wait before inflation. Tuned by JVM
3. **Fat (inflated) lock**: OS mutex. Park/unpark = syscall + context switch. Source: Shipilev Quark #19 Lock Elision

### What to look for
- **Lock granularity**: single lock for large data structure → bottleneck. Use lock striping or `ConcurrentHashMap`. Source: Oaks Ch 9; JCIP §11
- **Lock ordering**: inconsistent order → deadlock. Source: JCIP Ch 10
- **`synchronized` on `this` or `ClassName.class`**: any external code can also lock on these → unexpected contention. Prefer `private final Object lock = new Object()`. Source: SpotBugs `WL_USING_GETCLASS_RATHER_THAN_CLASS_LITERAL`
- **StampedLock optimistic reads**: for read-heavy workloads, `tryOptimisticRead()` avoids acquiring any lock — the stamp **must** be passed to `validate(stamp)` before trusting the read. Not reentrant → self-deadlock if misused. Source: `StampedLock` Javadoc (JDK 25)
- **VarHandle memory modes** (JDK 9+): `getOpaque`/`setOpaque` < `getAcquire`/`setRelease` < `getVolatile`/`setVolatile`. Use the weakest mode that suffices — overkill ordering = wasted fences. Source: Doug Lea *JDK 9 Memory Order Modes*
- **Virtual threads pinning (JDK 21-23)**: `synchronized` blocks pin virtual threads to carrier threads, destroying scalability. **Fixed in JDK 24 (JEP 491, March 2025)** — virtual threads now acquire/hold/release monitors without binding to the carrier. Remaining pinning sources: JNI native methods, FFM API (Kowalski, mikemybytes 2025/04/09). `-Djdk.tracePinnedThreads` is obsolete in JDK 24; use JFR `jdk.VirtualThreadPinned` event instead
- **Thread pool + ThreadLocal leaks**: thread pools reuse threads → `ThreadLocal` values leak across tasks. Clean up in `finally` or migrate to `ScopedValue` — **final in JDK 25 (JEP 506)**. `ScopedValue` carries a value only for the duration of a bounded scope; no leak possible. Source: JEP 506; Oracle JDK 25 `ScopedValue` Javadoc

### How to detect
- **JFR**: `jdk.JavaMonitorWait`, `jdk.JavaMonitorEnter` events — shows which locks are contended and for how long
- **async-profiler** 4.3+: `./asprof -e lock <pid>` — lock contention profiling; v4.0 added native-lock profiling
- `-Djdk.tracePinnedThreads=full` (JDK 21-23 only; obsolete in JDK 24). On JDK 24+ use JFR `jdk.VirtualThreadPinned` event

---

## 4. JIT Compilation & Code Shape

### Inlining
- Two independent thresholds control inlining (Oaks *Java Performance* 2e p.110):
  - `-XX:MaxInlineSize=35` (default **35 bytes**) — upper limit for **non-hot** methods
  - `-XX:FreqInlineSize=325` (default **325 bytes**) — upper limit for **hot** methods (inlined based on call-count profiling)
- Common misreading: "325 bytes is the limit for inlining" — wrong. 35 bytes is the cold-path limit; 325 bytes is reserved for hot methods only. Reviewers frequently conflate these two flags
- Getters/setters are trivially under 35 bytes → always inlined. Don't fear abstraction for tiny methods
- **Megamorphic call sites**: if a call site sees **>2 concrete receiver types**, HotSpot's inline cache gives up → no inlining, no escape analysis, no scalar replacement, no loop optimization on that path. Measured: monomorphic ~325 ns/300-iter vs megamorphic ~1070 ns/300-iter (Shipilev Quark #16 Megamorphic Virtual Calls)
  - Symptoms: interface with 3+ implementations dispatched at same call site
  - Fix: restructure to limit polymorphism at hot call sites, or use `instanceof` checks to create monomorphic branches. Source: Shipilev *Black Magic Method Dispatch*

### Escape analysis
- If an object doesn't escape the method (after inlining), it can be **scalar-replaced** (fields go to registers/stack, no heap allocation at all). Source: Oaks p.110-111; Shipilev Quark #18 Scalar Replacement
- **What breaks it**: object passed to non-inlined method, stored in field, returned from method, too-deep inlining chain, megamorphic call sites
- Verify with `-XX:+PrintEscapeAnalysis` (debug JVM) or JITWatch

### Loop optimizations
Source: Oaks Ch 4; Shipilev Quark #22 Safepoint Polls.
- **Counted loops** (`for (int i = 0; i < n; i++)`): eligible for unrolling, vectorization (SIMD), safepoint removal
- **Non-counted loops** (while with complex conditions, iterators): no unrolling, safepoints on every back-edge
- **Safepoints in loops**: JIT inserts safepoint polls in non-counted loops. Tight non-counted loops can block GC for seconds. Restructure as counted loops (Shipilev Quark #22)
- `Stream.forEach` over large collections: the lambda is an inner class → often megamorphic at the internal `accept()` call → less optimization than a plain for loop (Shipilev Quark #16)

### Intrinsics — hand-optimized by HotSpot
Source: Dehghani alidg.me/blog/2020/12/10/hotspot-intrinsics; Oaks Ch 4 p.112. Measured: `Math.log` intrinsic 309 M ops/s vs 151 M ops/s hand-rolled. Disable with `-XX:-InlineMathNatives`.
- `System.arraycopy`, `Arrays.copyOf` — memcpy-level performance
- `Arrays.mismatch`, `Arrays.equals` — SIMD comparisons
- `Math.min/max/abs/sqrt/log` — single CPU instructions (AVX on supporting CPUs)
- `StringLatin1.indexOf`, `String.equals` — SIMD + Compact Strings aware
- `Integer.bitCount`, `Long.numberOfLeadingZeros` — `POPCNT`/`LZCNT` instructions
- `Object.hashCode`, `System.identityHashCode`
- `Unsafe.compareAndSet`, `VarHandle.getAndAdd` — single `lock cmpxchg` / `lock xadd`
- Using these is always better than hand-rolling equivalent logic

### How to detect
- `-XX:+PrintCompilation` — shows which methods are compiled, deoptimized
- `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` — shows inlining decisions and failures ("too big", "no static binding", "not inlineable")
- **JITWatch** — visual tool for analyzing HotSpot JIT log output
- **async-profiler** 4.3+ CPU flame graph — wide flat frames indicate non-inlined hot code

### 4.1 Autoboxing in Hot Loops

```java
bad:
long sum = 0L;
for (Long v : values) {      // each element unboxed per iteration
    sum += v;                 // if sum declared Long, also boxed per step
}

good:
long sum = 0L;
for (long v : valuesPrimitive) {   // primitive array or LongStream
    sum += v;
}
```

`Integer` cache only covers -128..127 (§G-07); values outside allocate new boxed objects. Source: Oaks *Java Performance* 2e Ch 12 pp.392-395.

### 4.2 `Pattern.compile()` Per Call

```java
bad:
void process(String s) {
    if (s.matches("^[A-Z]+$")) { ... }   // recompiles regex every call
}

good:
private static final Pattern UPPER = Pattern.compile("^[A-Z]+$");
void process(String s) {
    if (UPPER.matcher(s).matches()) { ... }
}
```

Compile cost ~1-10 μs per call. Source: `java.util.regex.Pattern` Javadoc; Oaks Ch 12.

### 4.3 `SimpleDateFormat` in Multi-Thread Code

```java
bad:
private static final SimpleDateFormat FMT = new SimpleDateFormat("yyyy-MM-dd");
// FMT.format(...) from many threads → mutates Calendar internal state,
// returns garbled strings, throws ArrayIndexOutOfBoundsException

good:
private static final DateTimeFormatter FMT =
    DateTimeFormatter.ofPattern("yyyy-MM-dd");   // immutable, thread-safe
```

Source: `SimpleDateFormat` Javadoc ("Date formats are not synchronized"); `DateTimeFormatter` Javadoc.

### 4.4 Megamorphic Call Site in Hot Loop

```java
bad:
interface Op { int apply(int x); }   // 4+ implementations in production
int run(Op[] ops, int x) {
    for (Op op : ops) x = op.apply(x);   // call site sees >2 types → megamorphic
    return x;
}

good:
// Either group by concrete type:
for (AddOp op : addOps) x = op.apply(x);      // monomorphic
for (MulOp op : mulOps) x = op.apply(x);      // monomorphic
// Or branch explicitly:
if (op instanceof AddOp a) x = a.apply(x);    // monomorphic per branch
else if (op instanceof MulOp m) x = m.apply(x);
```

Measured cost: monomorphic ~325 ns vs megamorphic ~1070 ns (Shipilev Quark #16).

### 4.5 `synchronized` on Boxed `Integer` / `Long`

```java
bad:
private Integer counter = 0;
synchronized (counter) {             // counter autoboxed → different cache entries
    counter++;                        // AND mutates the lock reference
}

good:
private final Object lock = new Object();
private int counter = 0;
synchronized (lock) { counter++; }
```

`Integer` values -128..127 share cache entries → unrelated threads contend on the same lock. Source: `Integer.valueOf` Javadoc; Oaks Ch 9.

### 4.6 Unbounded `newCachedThreadPool`

```java
bad:
ExecutorService exec = Executors.newCachedThreadPool();
// Under a burst, creates unlimited platform threads → OOM / OS limits

good:
ExecutorService exec = new ThreadPoolExecutor(
    coreSize, maxSize, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(queueCapacity),
    new ThreadPoolExecutor.CallerRunsPolicy());
// Or, for I/O-bound: Executors.newVirtualThreadPerTaskExecutor() on JDK 21+
```

Source: `Executors` Javadoc; JCIP §8.3.

### 4.7 Direct ByteBuffer Allocated Per Request

```java
bad:
void write(SocketChannel ch, byte[] data) {
    ByteBuffer buf = ByteBuffer.allocateDirect(data.length);   // ~1 μs alloc
    buf.put(data).flip();
    ch.write(buf);   // Cleaner releases native memory lazily → possible OOM
}

good:
// Pool direct buffers (e.g., Netty PooledByteBufAllocator) or
// use a ThreadLocal<ByteBuffer> sized to a common max.
```

Direct alloc ~1 μs vs heap alloc ~10 ns. Monitor `-XX:MaxDirectMemorySize`. Source: Oaks Ch 8; `ByteBuffer` Javadoc.

### 4.8 `toString()` Evaluated Despite Disabled Log Level

```java
bad:
log.debug("state=" + state.toString() + " user=" + user.toString());
// Concat + toString runs even when DEBUG is off

good:
log.debug("state={} user={}", state, user);    // SLF4J parameterized — lazy toString
// Or guard:
if (log.isDebugEnabled()) log.debug("state={}", expensive());
```

Source: SLF4J FAQ; Oaks Ch 12.

### 4.9 `Thread.sleep(1)` for Microsecond Timing

```java
bad:
while (!ready) Thread.sleep(1);   // min granularity ~1-15 ms on many OSes

good:
while (!ready) LockSupport.parkNanos(1_000L);   // ns precision
// Or use a condition variable / CountDownLatch.
```

Source: `Thread.sleep` Javadoc ("subject to the precision and accuracy of system timers"); `LockSupport` Javadoc.

### 4.10 Varargs in Hot Logger Call

```java
bad:
log.debug("metrics: {}, {}, {}", a, b, c, d, e);
// Object[] allocated on every call, even when DEBUG is disabled

good:
// SLF4J ≥ 2.0 fluent API avoids Object[] when level disabled:
log.atDebug().addArgument(a).addArgument(b).log("metrics: {}, {}");
// Or guard:
if (log.isDebugEnabled()) log.debug("metrics: {}, {}, {}", a, b, c, d, e);
```

Source: SLF4J 2.0 manual; Oaks Ch 12.

### 4.11 Exception for Control Flow

```java
bad:
try { return Integer.parseInt(s); }
catch (NumberFormatException e) { return -1; }   // fillInStackTrace ~5-50 μs

good:
if (isNumeric(s)) return Integer.parseInt(s);
return -1;
// Or use Integer.parseInt in a guarded wrapper that returns OptionalInt.
```

Throwing cost is dominated by `Throwable.fillInStackTrace()` walking the entire call stack. Source: Shipilev *Exceptional Performance* (2014).

---

## 5. GC & Allocation Pressure

### Key metrics
- **Allocation rate** (MB/s): the single most important GC metric. High allocation rate → frequent young GC → latency spikes. Measure with JFR `jdk.ObjectAllocationInNewTLAB` / `jdk.ObjectAllocationOutsideTLAB`
- **TLAB allocation** (fast path): bump-a-pointer in a thread-local buffer in Eden. Disabling via `-XX:-UseTLAB` degrades allocation rate ~5x and single-threaded execution time ~10x (20x with 2 threads). Source: Shipilev Quark #4 TLAB Allocation
- **Outside-TLAB allocation** (slow path): shared-Eden CAS / lock. Triggered by large objects or TLAB exhaustion
- **Humongous allocations** (G1): objects **larger than half the region size** are allocated in old-gen humongous regions, bypassing young-gen collection. Default region auto-sizes between 1, 2, 4, 8, 16, 32 MB to target ~2048 regions (JDK-8276929 allows up to 512 MB via explicit `-XX:G1HeapRegionSize`). Fix: shrink the object or raise the region size. Source: Slusarski krzysztofslusarski.github.io/2020/11/10/humongous; Oracle HotSpot Tuning Guide
- **Promotion rate**: objects surviving young GC → old gen. High promotion = frequent mixed/full GC
- **Generational ZGC** (JDK 21 JEP 439, default in JDK 23 per JEP 474; non-generational mode removed in JDK 24 per JEP 490) — sub-millisecond pauses on multi-TB heaps

### What to look for
Sources: Oaks Ch 12 pp.392-395; SLF4J 2.0 manual; `java.lang.ref.Cleaner` Javadoc.
- **Autoboxing in loops**: `for (int i : map.values())` — each value unboxed. `map.put(key, i + 1)` — boxed back. Thousands of throwaway `Integer` objects. `Integer` cache only covers -128 to 127 (§4.1, §G-07)
- **String concatenation in loops**: `result += str` creates a new `String` per iteration. Use `StringBuilder`
- **`String.format()` in hot paths**: parses format string every call. Pre-format or use `StringBuilder`
- **Varargs in hot paths**: `void log(Object... args)` allocates an `Object[]` on every call, even if the log level is disabled. Guard with level check or use SLF4J 2.0 fluent API (§4.10)
- **Excessive temporary objects**: iterators on custom `Iterable`, capturing lambdas (fresh object per invocation unless stateless), `Optional` in tight loops
- **Finalizers**: create a `Finalizer` reference per object, processed on a dedicated low-priority thread. Delay GC by ≥2 cycles. Use `Cleaner` or try-with-resources instead
- **Soft/Weak references**: each adds ~32 bytes overhead plus reference-queue processing per GC cycle

### How to detect
- **JFR**: allocation profiling (`jdk.ObjectAllocationInNewTLAB`), GC pauses, promotion stats
- **async-profiler** 4.3+ (Jan 2026): `./asprof -e alloc <pid>` — allocation flame graph. v4.0 added native-memory profiling and heatmaps
- `-verbose:gc` or `-Xlog:gc*` — GC log analysis with GCEasy or GCViewer
- **jmap -histo**: quick object count/size histogram

### 5.1 Allocation Review Items
- [ ] Hot path contains no autoboxing of primitives in loops (§4.1) — source: Oaks Ch 12
- [ ] Hot-path regex is compiled once into `static final Pattern` (§4.2) — source: `Pattern` Javadoc
- [ ] Date formatting uses `DateTimeFormatter`, never `SimpleDateFormat` (§4.3) — source: `DateTimeFormatter` Javadoc
- [ ] Log calls use parameterized `{}` placeholders; heavy `toString` guarded by `isDebugEnabled` or passed lazily (§4.8, §4.10) — source: SLF4J manual
- [ ] Direct `ByteBuffer` buffers are pooled or thread-cached, not allocated per request (§4.7) — source: Oaks Ch 8
- [ ] No finalizers — `Cleaner` or try-with-resources instead (finalizers delay collection ≥2 cycles) — source: `java.lang.ref.Cleaner` Javadoc

### 5.2 GC Tuning Review Items
- [ ] Objects routinely larger than half the G1 region size are either split or the region size is raised via `-XX:G1HeapRegionSize` (§5 Key metrics) — source: Slusarski humongous post
- [ ] G1 / ZGC choice matches latency target (ZGC sub-ms pauses, generational by default since JDK 23 JEP 474) — source: JEP 439, 474, 490
- [ ] `-Xlog:gc*` enabled with rotation in production — source: Oracle JDK Logging config
- [ ] On JDK 25+, consider `-XX:+UseCompactObjectHeaders` (JEP 519) for allocation-heavy workloads (22% heap saving in SPECjbb2015) — source: JEP 519, InfoQ June 2025

### 5.3 Hot-Path Structural Review Items
- [ ] Call sites in hot loops see ≤2 concrete receiver types — no megamorphism (§4.4) — source: Shipilev Quark #16
- [ ] `synchronized` lock targets are `private final Object lock = new Object()`, never boxed primitives, `this`, or `Class` literals (§4.5) — source: SpotBugs WL_*; Oaks Ch 9
- [ ] Executors are bounded or virtual-thread per task; no `Executors.newCachedThreadPool` without explicit caps (§4.6) — source: JCIP §8.3
- [ ] No exceptions used for control flow in hot paths (§4.11) — source: Shipilev *Exceptional Performance*

---

## 6. I/O & Networking

### Socket tuning
Source: HN thread "It's always TCP_NODELAY" (#40310896); Netty `ChannelOption` Javadoc.

| Setting | Default | Issue | Fix |
|--|--|--|--|
| `TCP_NODELAY` | `false` (Nagle ON) | Nagle buffers small writes → ~40 ms delay via delayed-ACK interaction | Set `true` for latency-sensitive protocols. Netty defaults to `true` |
| `SO_SNDBUF` / `SO_RCVBUF` | OS-dependent (usually 128 KB) | Too small for high-throughput, too large wastes memory | Set based on BDP (bandwidth × RTT). 1 Gbps × 10 ms RTT = ~1.2 MB |
| `SO_LINGER` | OFF | Closing socket may lose buffered data | Set for reliable shutdown (with timeout) |
| `SO_REUSEADDR` | `false` | `TIME_WAIT` prevents quick server restart | Set `true` on server sockets |

**JDK-internal socket overhead**: Chronicle Software measured a 19% latency improvement (6.8 → 5.7 μs 50th pct) by bypassing `SocketChannelImpl` synchronization/interrupt overhead in a custom transport — not by tuning `TCP_NODELAY`. Relevant only when standard Java NIO is the bottleneck on sub-10 μs paths.

### ByteBuffer performance
Source: Oaks Ch 8 p.246; `ByteBuffer` Javadoc; Xu 2016 zero-copy post.
- **Heap ByteBuffer**: data in Java heap. Every native I/O call copies to a temporary direct buffer internally
- **Direct ByteBuffer**: data in native memory. No copy for I/O, but allocation is expensive (~1 μs vs ~10 ns for heap). Reuse direct buffers, don't allocate per-request (§4.7)
- **Direct buffer leak**: not freed by GC promptly. Freed only when the `Cleaner` runs. Under memory pressure, can cause `OutOfMemoryError: Direct buffer memory`. Monitor with `-XX:MaxDirectMemorySize`
- **MappedByteBuffer** (mmap): maps file into virtual address space. Good for random access to large files. Costs: page faults (~1-10 μs on miss), TLB pressure, no explicit `unmap()` in Java (GC-dependent → file-handle leak risk)
- **FileChannel.transferTo()**: true zero-copy on Linux (`sendfile` syscall). Data goes kernel→NIC, never enters user space. Use for file serving

### Common I/O traps
Sources: `InetSocketAddress` Javadoc; `BufferedOutputStream` Javadoc; Netty docs.
- **DNS in NIO**: `InetSocketAddress(hostname, port)` resolves DNS synchronously in constructor. In NIO event loops, this blocks the selector thread. Resolve async or pre-resolve
- **Selector wakeup cost**: `selector.wakeup()` writes a byte to a pipe → syscall. Don't call on every event
- **Epoll spin bug** (JDK < 11): `Selector.select()` can return 0 events in a tight loop burning CPU. Netty has a workaround (rebuilds selector)
- **Buffered streams**: raw `OutputStream.write(byte)` → one syscall per byte. Always wrap with `BufferedOutputStream` (8 KB default) or write byte arrays
- **io_uring** (JDK 21+ via Panama/JNI): async I/O with ring buffer. Zero syscalls in steady state. Still experimental in Java ecosystem (Netty has io_uring transport)

---

## 7. Common Anti-Patterns in Hot Paths

Concentrated lookup table. Detailed bad:/good: code pairs in §4.1-§4.11. Sources per row in the linked subsection.

| Anti-pattern | Cost | Fix | See |
|--|--|--|--|
| `Pattern.compile()` every call | ~1-10 μs per compile | Cache as `static final Pattern` | §4.2 |
| `SimpleDateFormat` in multi-thread | Not thread-safe + slow | `DateTimeFormatter` (immutable, thread-safe) | §4.3 |
| Exception for control flow | `fillInStackTrace()` walks entire stack ~5-50 μs | Use return codes, `Optional`, sentinel values | §4.11 |
| `Class.forName()` | Triggers classloading, acquires locks | Cache the `Class` reference | Oaks Ch 12 |
| Reflection `getDeclaredMethod()` + `invoke()` | ~50-100 ns after warmup, prevents inlining | Cache `MethodHandle` or use code generation | `MethodHandle` Javadoc |
| `Thread.sleep(1)` for timing | Minimum granularity ~1-15 ms (OS-dependent) | `LockSupport.parkNanos()` — ns precision | §4.9 |
| `Collections.unmodifiableList(new ArrayList<>(list))` | Wraps + copies | `List.copyOf(list)` — single compact copy | `List.copyOf` Javadoc |
| `toString()` in log arguments | Evaluated even if log level is off | Parameterized `log.debug("x={}", v)` or lazy supplier | §4.8, §4.10 |
| `synchronized` on boxed `Integer`/`Long` | Integer cache → unrelated threads share lock | Lock on dedicated `Object` | §4.5 |

---

## 8. Profiling Toolkit

| Tool | What it measures | When to use |
|--|--|--|
| **async-profiler** (4.3, Jan 2026) | CPU, allocations, locks, wall clock, native memory (v4.0+), heatmaps | First line of investigation. Low overhead (<5%), no safepoint bias. Manual: Slusarski 2022/12/12 async-manual |
| **JFR** (Java Flight Recorder) | Everything (CPU, GC, I/O, threads, locks, allocations, `jdk.VirtualThreadPinned`) | Production-safe continuous profiling. ~1% overhead |
| **JOL** | Object memory layout | Reviewing data-structure designs, finding padding/false sharing. See `github.com/openjdk/jol` |
| **JMH** (1.37, Aug 2023) | Method-level throughput/latency | Validating optimization hypotheses. Handles warmup, JIT, GC correctly. JEP 230 Microbenchmark Suite |
| **JITWatch** | JIT compilation decisions | Understanding why code isn't inlined/optimized |
| **jcmd** | Thread dumps, heap info, JFR control | Quick runtime diagnostics |
| **GCEasy / GCViewer** | GC log analysis | Understanding GC behavior, pause distribution |
| `perf` + `perf-map-agent` | Hardware counters (cache misses, branch mispredictions, TLB) | Low-level CPU performance investigation |

### Key JVM diagnostic flags
```
# Compilation (Oaks Ch 4)
-XX:+PrintCompilation
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining
# Defaults: -XX:MaxInlineSize=35 (non-hot), -XX:FreqInlineSize=325 (hot)

# GC (Oaks Ch 5-6)
-Xlog:gc*:file=gc.log:time,uptime,level,tags
-XX:+HeapDumpOnOutOfMemoryError

# JFR (always-on, low overhead)
-XX:StartFlightRecording=filename=app.jfr,maxsize=500m,settings=profile

# Virtual threads
-Djdk.tracePinnedThreads=full   # JDK 21-23 only; obsolete in JDK 24
# On JDK 24+: enable JFR event jdk.VirtualThreadPinned

# Object layout (JEP 519, JDK 25)
-XX:+UseCompactObjectHeaders

# Leyden AOT cache (JEPs 483, 515; JDK 24/25)
-XX:AOTMode=record / -XX:AOTMode=auto
```

---

## 9. Quick Review Heuristic

When reviewing Java code for performance, scan in this order:

1. **Hot path identification**: what runs per-request, per-event, per-iteration?
2. **Allocations in hot path**: any `new`, autoboxing, varargs, string concat, lambdas with captures?
3. **Lock contention**: any `synchronized`, `ReentrantLock`, `AtomicX` on shared data?
4. **Collection choice**: right data structure? Right initial capacity? Primitive types boxed unnecessarily?
5. **Call site polymorphism**: interfaces with many implementations dispatched in hot loops?
6. **I/O in hot path**: buffered? Async where needed? DNS resolved outside event loop?
7. **Regex/date/reflection cached?**: static final Pattern? DateTimeFormatter? Cached MethodHandle?
8. **Exception paths**: exceptions used for control flow? Large catch blocks hiding performance issues?

---

## Key References

- **Scott Oaks, *Java Performance* 2nd ed., O'Reilly 2020, ISBN 978-1-492-05611-9.** Covers JDK 8 & 11. Chapter-to-§ mapping: Ch 2 (JMH, §8) · Ch 3 (JFR, §8) · Ch 4 inlining p.109-111, escape analysis p.110-111, AVX intrinsics p.112 (§4) · Ch 5-6 GC (§5) · Ch 7 heap best practices (§5) · Ch 8 compressed oops p.246, direct buffers (§1, §6) · Ch 9 locking p.287-288, false sharing p.294-298, @Contended p.298, biased locking p.300 (§1, §3) · Ch 12 Compact Strings p.363, intern p.364-371, collections p.392-395 (§2, §4)
- **Aleksey Shipilev — primary JVM internals**: *Objects Inside Out*, *JVM Anatomy Quarks* (30 topics), *Black Magic Method Dispatch* (2015), *Exceptional Performance* (2014)
- **Herlihy & Shavit, *The Art of Multiprocessor Programming*, Revised Reprint (Morgan Kaufmann 2012)**, ISBN 978-0-12-397337-5 — §7 cache coherence and false sharing
- **JLS Chapter 17** — formal JVM memory model
- **Doug Lea, *Using JDK 9 Memory Order Modes* (2018)** — VarHandle ordering modes
- **Erik Ostermueller, *Troubleshooting Java Performance*, Apress 2017**, ISBN 978-1-4842-2979-8 — P.A.T.h. field checklist
- JEPs: 142 (@Contended), 230 (JMH), 254 (Compact Strings), 374 (disable biased locking), 439 (Generational ZGC), 474 (ZGC generational by default), 483 (Leyden AOT class loading), 490 (remove non-gen ZGC), 491 (VT without pinning), 506 (ScopedValue final), 508/529 (Vector API), 515 (AOT method profiling), 519 (Compact Object Headers)

## Research Questions to Cross-Check Against Sources

The checklist is organized to answer the following questions about a diff or PR. Each question cites the section(s) and primary source(s).

### A. Object layout & cache

1. **What is the actual byte size of each data-structure instance, and where does padding sit?** (§1) — source: Shipilev *Objects Inside Out*; JOL `internals` mode.
2. **Do JDK 25+ workloads enable `-XX:+UseCompactObjectHeaders`?** Headers shrink 12 → 8 B; SPECjbb2015 saves 22% heap. (§1, §5.2) — source: JEP 519; InfoQ June 2025.
3. **Are hot, per-thread fields padded against false sharing with `@jdk.internal.vm.annotation.Contended`?** (§1) — source: JEP 142; Oaks p.296; alidg.me false-sharing.
4. **Is the target cache-line size 64 B (x86) or 128 B (ARM/Apple Silicon)?** (§1) — source: 7-cpu.com Apple M1; Oaks p.296.
5. **Does the JVM flag `-XX:+UseNUMA` match the host topology for multi-socket systems?** (§1) — source: HotSpot docs.
6. **Are large hot collections array-backed (`ArrayList`) rather than node-based (`LinkedList`)?** (§1, §2) — source: Oaks Ch 12 p.392-395.
6a. **Does every multi-dimensional-array or `ByteBuffer` traversal walk memory in row-major, stride-1 order?** In Java, `a[i][j]` with `i` outer / `j` inner is linear; the reverse strides across sub-arrays and produces near-100% L1 misses. (§1) — source: Herlihy-Shavit Appendix B "Hardware Basics"; Drepper 2007 §3.3.

### B. Data-structure choice

7. **Is `HashMap` sized with `expectedSize * 4 / 3 + 1` when size is known?** (§2) — source: `HashMap` Javadoc.
8. **Do boxed-primitive collections justify the 36 B/entry overhead, or can Eclipse Collections / fastutil / HPPC replace them?** (§2) — source: Eclipse Collections docs (≈77% memory saving, up to 10x speedup).
9. **Are `EnumMap` / `EnumSet` used where keys are enums?** (§2) — source: `EnumMap` Javadoc (flat array by ordinal).
10. **Are `List.of` / `Map.of` used for small immutable collections instead of `ArrayList` / `HashMap`?** (§2) — source: `List.of` Javadoc (compact representation).
11. **Do Latin-1-only strings benefit from Compact Strings? Is any single non-Latin-1 char unexpectedly doubling the backing array?** (§2) — source: JEP 254; Oaks p.363.

### C. Synchronisation cost

12. **Is the uncontended `synchronized` cost budgeted correctly (a few hundred ns, not ~20 ns)?** (§3) — source: Oaks p.288.
13. **Is biased locking referenced as disabled by default since JDK 15 (JEP 374), not "removed"?** (§3) — source: JEP 374.
14. **Do volatile reads pay the cross-architecture cost (x86 TSO ≈ plain load vs ARM `LDAR`)?** (§3) — source: Brooker 2012/09/10.
15. **Under contention, is the volatile-read cost ~25x higher than uncontended due to cache coherency?** (§3) — source: Brooker 2012/09/10.
16. **Does lock-free code use the appropriate VarHandle mode (Plain < Opaque < Acquire/Release < Volatile)?** (§3) — source: Doug Lea *JDK 9 Memory Order Modes*.
17. **Are `StampedLock.tryOptimisticRead` returns validated via `validate(stamp)` before the read is trusted?** (§3) — source: `StampedLock` Javadoc.
18. **On JDK 21-23, are hot `synchronized` blocks around blocking I/O replaced with `ReentrantLock` to avoid virtual-thread pinning? On JDK 24+, is this no longer required?** (§3) — source: JEP 491; mikemybytes 2025/04/09.

### D. JIT & inlining

19. **What is the correct meaning of `-XX:MaxInlineSize=35` vs `-XX:FreqInlineSize=325`?** (§4) — source: Oaks p.110.
20. **Does any hot call site see >2 concrete receiver types (megamorphic)?** (§4, §4.4) — source: Shipilev Quark #16 (mono ~325 ns vs mega ~1070 ns).
21. **Are intrinsics (`System.arraycopy`, `Math.log`, `StringLatin1.indexOf`, `Arrays.mismatch`, `VarHandle`) used where applicable?** (§4) — source: alidg.me/2020/12 hotspot-intrinsics; Oaks Ch 4 p.112.
22. **Does scalar replacement require non-escaping objects (no field store, no non-inlined call with the ref)?** (§4) — source: Shipilev Quark #18 Scalar Replacement.
23. **Are tight loops counted (`for (int i=0; i<n; i++)`) so the JIT can unroll and vectorize?** (§4) — source: Oaks Ch 4.
24. **Do JDK 24+ workloads consider Leyden AOT cache (JEP 483, 515) to cut startup?** (§8) — source: JEP 483, 515; inside.java 2026/01/09.

### E. Allocation pressure & GC

25. **Is TLAB allocation enabled (`-XX:+UseTLAB` is default)?** Disabling degrades allocation rate ~5x. (§5) — source: Shipilev Quark #4 TLAB Allocation.
26. **Are allocations > half the G1 region size classified as humongous?** (§5) — source: Slusarski humongous post.
27. **Is the G1 region auto-sized to 1/2/4/8/16/32 MB to target ~2048 regions? Beyond 32 MB, is `-XX:G1HeapRegionSize` set explicitly (JDK-8276929 allows up to 512 MB)?** (§5) — source: Oracle HotSpot Tuning Guide.
28. **Is Generational ZGC (JEP 439, default since JDK 23 per JEP 474) considered for multi-TB latency-critical heaps?** (§5) — source: JEP 439, 474; inside.java gen-zgc-explainer.
29. **Are finalizers eliminated in favour of `Cleaner` / try-with-resources?** (§5, §5.1) — source: `Cleaner` Javadoc.
30. **Are autoboxing, `Pattern.compile` per call, `SimpleDateFormat` use, direct-buffer per request, and `toString()` in disabled log statements eliminated?** (§4.1-4.10, §5.1) — source: Oaks Ch 12; SLF4J manual.

### F. I/O & networking

31. **Is `TCP_NODELAY` set true on latency-sensitive sockets?** (§6) — source: HN #40310896; Netty `ChannelOption` Javadoc.
32. **Are `SO_SNDBUF` / `SO_RCVBUF` sized to BDP (bandwidth × RTT)?** (§6) — source: TCP tuning references.
33. **Are direct `ByteBuffer` instances pooled or thread-cached instead of allocated per request?** (§4.7, §6) — source: Oaks Ch 8; `ByteBuffer` Javadoc.
34. **Does large-file serving use `FileChannel.transferTo` to hit the kernel `sendfile` path?** (§6) — source: Xu 2016 zero-copy post.
35. **Do synchronous NIO event loops avoid `InetSocketAddress(hostname, port)` (blocking DNS)?** (§6) — source: `InetSocketAddress` Javadoc.
36. **Are raw `OutputStream.write(byte)` calls wrapped in `BufferedOutputStream`?** (§6) — source: `BufferedOutputStream` Javadoc.

### G. Hot-path anti-patterns

37. **Are exceptions confined to exceptional control flow (not validation / parsing / early-exit)?** (§4.11, §7) — source: Shipilev *Exceptional Performance* (fillInStackTrace walks full stack).
38. **Are `Class.forName` / `Method.invoke` cached into `MethodHandle` / static fields?** (§7) — source: `MethodHandle` Javadoc; Oaks Ch 12.
39. **Is `Thread.sleep(1)` avoided when sub-millisecond timing is needed?** (§4.9, §7) — source: `Thread.sleep` Javadoc; `LockSupport.parkNanos`.
40. **Are `Collections.unmodifiableList(new ArrayList<>(x))` patterns replaced with `List.copyOf(x)`?** (§7) — source: `List.copyOf` Javadoc.

### H. Profiling discipline

41. **Is async-profiler 4.3+ used as the first investigation tool (low overhead, no safepoint bias)?** (§8) — source: github.com/async-profiler/async-profiler; Slusarski 2022/12/12 async-manual.
42. **Is JFR continuous profiling enabled in production for post-hoc analysis?** (§8) — source: Oaks Ch 3.
43. **Are optimization hypotheses validated via JMH (1.37)?** (§8) — source: JEP 230; github.com/openjdk/jmh.
44. **Are inlining decisions verified via `-XX:+PrintInlining` / JITWatch before declaring code "hot"?** (§4, §8) — source: Oaks Ch 4.

### I. Virtual threads & modern JDK features

45. **Are virtual threads (JEP 444 GA in JDK 21) used for I/O-bound work — not pooled, not for CPU-bound tasks?** (§3) — source: Oracle JDK 25 Virtual Threads docs.
46. **On JDK 24+, has `-Djdk.tracePinnedThreads` been replaced with JFR `jdk.VirtualThreadPinned`?** (§3, §8) — source: mikemybytes 2025/04/09; Oracle JDK 25 docs.
47. **Does context propagation use `ScopedValue` (JEP 506 final in JDK 25) instead of `ThreadLocal` in pool-based executors?** (§3) — source: JEP 506; `ScopedValue` Javadoc.

### J. Compact object headers / JEP 519

48. **On JDK 25 workloads sensitive to heap or allocation rate, is `-XX:+UseCompactObjectHeaders` enabled?** Real-world: SPECjbb2015 22% heap, 8% CPU; Amazon up to 30% CPU reduction. (§1, §5.2) — source: JEP 519; InfoQ June 2025.

---

## Sources

### Normative / specification
- [JLS Chapter 17: Threads and Locks (Java SE 25)](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html)
- [StampedLock Javadoc (JDK 25)](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/StampedLock.html) — optimistic read + `validate` contract
- [ScopedValue Javadoc (JDK 25)](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/ScopedValue.html) — final in JDK 25
- [Virtual Threads (Oracle JDK 25)](https://docs.oracle.com/en/java/javase/25/core/virtual-threads.html) — pinning, `jdk.VirtualThreadPinned` JFR event
- [HotSpot Synchronization wiki](https://wiki.openjdk.org/display/HotSpot/Synchronization) — lock state machine (resolves in browsers; WebFetch-blocked)

### JEPs
- [JEP 142: Reduce Cache Contention on Specified Fields (JDK 8)](https://openjdk.org/jeps/142) — `@Contended`
- [JEP 230: Microbenchmark Suite](https://openjdk.org/jeps/230) — JMH
- [JEP 254: Compact Strings (JDK 9)](https://openjdk.org/jeps/254)
- [JEP 374: Deprecate and Disable Biased Locking (JDK 15)](https://openjdk.org/jeps/374)
- [JEP 439: Generational ZGC (JDK 21)](https://openjdk.org/jeps/439)
- [JEP 474: ZGC Generational Mode by Default (JDK 23)](https://openjdk.org/jeps/474)
- [JEP 483: Ahead-of-Time Class Loading & Linking (JDK 24)](https://openjdk.org/jeps/483)
- [JEP 490: Remove the Non-Generational ZGC Mode (JDK 24)](https://openjdk.org/jeps/490)
- [JEP 491: Synchronize Virtual Threads without Pinning (JDK 24)](https://openjdk.org/jeps/491)
- [JEP 506: Scoped Values, Final (JDK 25)](https://openjdk.org/jeps/506)
- [JEP 508: Vector API, Tenth Incubator (JDK 25)](https://openjdk.org/jeps/508)
- [JEP 515: Ahead-of-Time Method Profiling (JDK 25)](https://openjdk.org/jeps/515)
- [JEP 519: Compact Object Headers (JDK 25 Final)](https://openjdk.org/jeps/519)
- [JEP 529: Vector API, Eleventh Incubator (JDK 26)](https://openjdk.org/jeps/529)

### Foundational / authoritative
- **Scott Oaks, *Java Performance* 2nd ed. (O'Reilly 2020)**, ISBN 978-1-492-05611-9 — the primary reference for this checklist
- **Aleksey Shipilev — JVM Anatomy Quarks** (30 topics, primary source for JVM micro-behaviors):
  - [Objects Inside Out](https://shipilev.net/jvm/objects-inside-out/)
  - [Quark #4: TLAB Allocation](https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation/) — disabling TLAB: 5-10x slowdown
  - [Quark #10: String.intern](https://shipilev.net/jvm/anatomy-quarks/10-string-intern/) — StringTable 60013 buckets since JDK 7u40
  - [Quark #16: Megamorphic Virtual Calls](https://shipilev.net/jvm/anatomy-quarks/16-megamorphic-virtual-calls/) — mono 325 ns vs mega 1070 ns
  - [Quark #18: Scalar Replacement](https://shipilev.net/jvm/anatomy-quarks/18-scalar-replacement/)
  - [Quark #19: Lock Elision](https://shipilev.net/jvm/anatomy-quarks/19-lock-elision/)
  - [Quark #22: Safepoint Polls](https://shipilev.net/jvm/anatomy-quarks/22-safepoint-polls/)
  - [Quark #23: Compressed References](https://shipilev.net/jvm/anatomy-quarks/23-compressed-references/)
  - [Quark #24: Object Alignment](https://shipilev.net/jvm/anatomy-quarks/24-object-alignment/)
  - [Black Magic Method Dispatch (2015)](https://shipilev.net/blog/2015/black-magic-method-dispatch/) — mono/bi/megamorphic
  - [Arrays of Wisdom of the Ancients (2016)](https://shipilev.net/blog/2016/arrays-wisdom-ancients/)
  - [Exceptional Performance (2014)](https://shipilev.net/blog/2014/exceptional-performance/) — fillInStackTrace cost
- [Marc Brooker: Volatile Reads on x86](https://brooker.co.za/blog/2012/09/10/volatile.html) — contended volatile ≈25x slower
- [Marc Brooker: Atomic and Volatile on x86](https://brooker.co.za/blog/2012/11/13/increment.html) — `lock cmpxchg` / cache coherence storms
- [Doug Lea: Using JDK 9 Memory Order Modes (2018)](https://gee.cs.oswego.edu/dl/html/j9mm.html)
- [Ali Dehghani: HotSpot Intrinsics](https://alidg.me/blog/2020/12/10/hotspot-intrinsics) — `Math.log` intrinsic 309M vs 151M ops/s
- [Ali Dehghani: False Sharing](https://alidg.me/blog/2020/5/1/false-sharing) — 170 ns/op → 66 ns/op with `@Contended`
- [Krzysztof Slusarski: G1 Humongous Allocations](https://krzysztofslusarski.github.io/2020/11/10/humongous.html)

### Tooling
- [async-profiler](https://github.com/async-profiler/async-profiler) — v4.3 current (Jan 2026); v4.0 added native memory + heatmaps
- [OpenJDK JOL](https://github.com/openjdk/jol) — `jol-core`, `jol-cli`, `jol-samples`
- [OpenJDK JMH](https://github.com/openjdk/jmh) — v1.37 current (Aug 2023)
- [Krzysztof Slusarski: async-profiler manual (2022)](https://krzysztofslusarski.github.io/2022/12/12/async-manual.html)
- [JITWatch](https://github.com/AdoptOpenJDK/jitwatch) — visualizer for HotSpot `-XX:+PrintInlining` logs

### Checklists and curated reviews
- [Baeldung: @Contended and False Sharing](https://www.baeldung.com/java-false-sharing-contended) — resolves in browsers
- [Baeldung: Primitive Collections](https://www.baeldung.com/java-eclipse-primitive-collections) — Eclipse Collections, fastutil, HPPC overview (resolves in browsers)
- [code-review-checklists/java-concurrency](https://github.com/code-review-checklists/java-concurrency) — overlap on lock/state items

### Industry case studies and explainers
- [JEP 519: Compact Object Headers (InfoQ, June 2025)](https://www.infoq.com/news/2025/06/java-25-compact-object-headers/) — 22% heap, 8% CPU SPECjbb2015; Amazon up to 30% CPU reduction
- [Netflix Adopts Virtual Threads (Tech Blog)](https://netflixtechblog.com/java-21-virtual-threads-dude-wheres-my-lock-3052540e231d) — canonical virtual-thread + synchronized deadlock case study
- [Java 24: Thread Pinning Revisited (Mike Kowalski, Apr 9 2025)](https://mikemybytes.com/2025/04/09/java24-thread-pinning-revisited/) — JEP 491 empirical verification
- [Virtual Threads without Pinning (InfoQ, Nov 2024)](https://www.infoq.com/news/2024/11/java-evolves-tackle-pinning/)
- [HN: "It's always TCP_NODELAY" (#40310896)](https://news.ycombinator.com/item?id=40310896)
- [Zero-copy, mmap, Java NIO (Shawn Xu, 2016)](https://xunnanxu.github.io/2016/09/10/It-s-all-about-buffers-zero-copy-mmap-and-Java-NIO/)
- [Chronicle: How to Make Java Sockets Faster](https://chronicle.software/how-to-make-java-sockets-faster/) — bypassing `SocketChannelImpl` sync (NOT about TCP_NODELAY); 19% latency improvement
- [JDK 25 Performance Improvements (Inside Java, Oct 2025)](https://inside.java/2025/10/20/jdk-25-performance-improvements/)
- [Generational ZGC explainer (Inside Java, Nov 2023)](https://inside.java/2023/11/28/gen-zgc-explainer/)
- [Leyden AOT cache (Inside Java, Jan 2026)](https://inside.java/2026/01/09/run-aot-cache/)

---

## Gotchas — Common Wrong Assumptions

G-01. **"`-XX:MaxInlineSize=325` is the inlining threshold"** — false. 35 bytes is the default `MaxInlineSize` for non-hot methods; `FreqInlineSize=325` is the default for hot methods. Two different flags. See §4 Inlining. Source: Oaks p.110.
G-02. **"`synchronized` uncontended costs ~20 ns"** — misleading. Oaks p.288: "a few hundred nanoseconds" for an uninflated lock. See §3 Lock cost ladder.
G-03. **"Biased locking was removed in JDK 15"** — false. JEP 374 **disabled it by default** and deprecated the flag; `-XX:+UseBiasedLocking` still exists and warns. See §3. Source: JEP 374.
G-04. **"Cache line is always 64 bytes"** — false on ARM/Apple Silicon (M1/M2/M3 L2 line = 128 B). HotSpot defaults `@Contended` padding to 128 B to cover both. See §1. Source: 7-cpu.com Apple M1.
G-05. **"TLAB and outside-TLAB allocation cost similarly"** — false. Disabling TLAB degrades allocation rate ~5x and execution time ~10x single-threaded, 20x with 2 threads. See §5. Source: Shipilev Quark #4.
G-06. **"Humongous = object > G1 region size"** — false. Humongous = object > **half** the region size (allocates across contiguous regions). Default region auto-sizes 1/2/4/8/16/32 MB to target ~2048 regions. See §5. Source: Slusarski humongous post.
G-07. **"`Integer.valueOf(x)` always caches"** — only for -128..127 by default (`-XX:AutoBoxCacheMax` can raise upper bound). Values outside allocate fresh `Integer` objects → hidden GC pressure in counters. See §4.5, §5. Source: `Integer.valueOf` Javadoc.
G-08. **"`volatile` reads are free on x86"** — true only when uncontended. Contended volatile reads are ~25x slower than non-volatile reads due to cache-coherence traffic. See §3. Source: Brooker 2012/09/10.
G-09. **"`volatile` on ARM is the same cost as on x86"** — false. ARM needs `LDAR` / `STLR` instructions even for uncontended access; x86 uses plain load + `LOCK`ed store. See §3 cost ladder. Source: Brooker 2012/11/13.
G-10. **"Virtual threads always pin on `synchronized`"** — true on JDK 21-23 only. JEP 491 (JDK 24, March 2025) eliminated it. Remaining pinning: JNI native methods, FFM API. See §3. Source: JEP 491; mikemybytes 2025/04/09.
G-11. **"`-Djdk.tracePinnedThreads` is how you detect pinning on JDK 24"** — obsolete. Use JFR `jdk.VirtualThreadPinned` event instead. See §3, §8.
G-12. **"`ScopedValue` is still preview"** — false since JDK 25 (JEP 506). `ScopedValue.orElse(null)` is rejected in the final API. See §3. Source: JEP 506.
G-13. **"A call site with 3+ implementations still inlines most of the time"** — false. HotSpot inline cache is bimorphic max; a third type makes it megamorphic (~3.3x slower per call in Shipilev's benchmark). See §4.4. Source: Shipilev Quark #16.
G-14. **"`tryOptimisticRead` is free — no lock, no cost"** — false. The stamp must be passed to `validate(stamp)`; if false, the read was inconsistent and must be retried under the full read lock. See §3. Source: `StampedLock` Javadoc.
G-15. **"Compact Strings always save memory"** — only for Latin-1 content. A single non-Latin-1 character doubles the backing `byte[]` to UTF-16. See §2. Source: JEP 254; Oaks p.363.
G-16. **"Chronicle's 'How to Make Java Sockets Faster' is about TCP_NODELAY"** — false. It bypasses `SocketChannelImpl` synchronization/interrupt overhead. TCP_NODELAY guidance comes from HN #40310896 and Netty `ChannelOption` Javadoc. See §6.
G-17. **"`Executors.newCachedThreadPool()` is a safe default"** — false. Unbounded thread creation → OOM / OS limits under burst. See §4.6. Source: `Executors` Javadoc; JCIP §8.3.
G-18. **"`SimpleDateFormat` is fine if I re-create it per call"** — functionally safe but slow. `DateTimeFormatter` is immutable + thread-safe, reusable as a `static final`. See §4.3. Source: `DateTimeFormatter` Javadoc.
G-19. **"Nested-loop order over a 2-D array doesn't matter — the JIT will fix it"** — false. Java arrays are row-major and each `a[i]` is a distinct heap-allocated sub-array; swapping inner and outer loops strides across sub-arrays and can hit near-100% L1 misses, 10–100× slower. HotSpot does not interchange loops for you. See §1. Source: Herlihy-Shavit Appendix B "Hardware Basics"; Drepper 2007 §3.3.
