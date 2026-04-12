# Java Performance Review Checklist

Concise reference for finding performance bugs during code review. Focus on subtle, non-obvious issues.

---

## 1. Object Layout & CPU Cache

### What to look for
- **Object header overhead**: 12 bytes (compressed oops) or 16 bytes (no compression). With JDK 25+ and `-XX:+UseCompactObjectHeaders`, headers shrink to 8 bytes (Project Lilliput — 20-30% heap reduction in real workloads)
- **Object alignment**: all objects padded to 8-byte boundary. A class with a single `boolean` field still occupies 16 bytes (12 header + 1 field + 3 padding)
- **Field reordering**: HotSpot sorts fields by size (longs/doubles first, then ints, shorts, bytes, references) to minimize padding. Declaring fields in a different order does NOT change layout
- **False sharing**: two frequently-written fields from different threads landing on the same 64-byte cache line. Common in counters, flags, ring buffer indices
  - Fix: `@jdk.internal.vm.annotation.Contended` (adds 128 bytes padding — 2 cache lines due to prefetching). Requires `-XX:-RestrictContended` for non-JDK classes
  - **ARM/Apple Silicon**: cache line is 128 bytes on M1/M2/M3, not 64 — default `@Contended` padding barely covers one line
- **Array vs linked structures**: `ArrayList` elements are contiguous in memory (cache-friendly); `LinkedList` nodes are scattered (pointer chasing = cache miss per node)
- **NUMA effects**: on multi-socket systems, accessing memory from a remote NUMA node costs 2-3x. JVM flag: `-XX:+UseNUMA` enables NUMA-aware allocation

### How to detect
- **JOL** (Java Object Layout): `java -jar jol-cli.jar internals com.example.MyClass` — shows exact byte layout, padding, alignment
- **IdeaJol** plugin for IntelliJ — visual object layout in IDE
- `perf c2c` (Linux) — detects false sharing by tracking cache-line contention across cores
- `perf stat -e cache-misses,cache-references` — L1/L2/L3 cache miss ratio

---

## 2. Data Structures

### HashMap traps
- **Initial capacity**: default is 16, load factor 0.75. If you know size, use `new HashMap<>(expectedSize * 4 / 3 + 1)` to avoid rehashing. Each rehash copies the entire table
- **Bad hashCode()**: if keys cluster to few buckets, O(1) degrades to O(log n) (treeification at 8+ entries per bucket, since JDK 8) or O(n) before treeification
- **Key mutability**: if key object is mutated after insertion, `get()` will never find it (hash changed but bucket unchanged)
- **Treeification threshold**: buckets convert to red-black trees at 8 entries and back to lists at 6. With good hash, this never triggers

### Collection choice
| Scenario | Use | Avoid | Why |
|--|--|--|--|
| Random access | `ArrayList` | `LinkedList` | LinkedList: 24 bytes/node overhead, cache-hostile |
| Enum keys | `EnumMap` / `EnumSet` | `HashMap` / `HashSet` | EnumMap is a flat array indexed by ordinal — O(1), zero hashing |
| Small immutable | `List.of()`, `Map.of()` | `new ArrayList<>()` | `List.of()` uses compact field-based storage (0-2 elements) or array without unused slots |
| Primitives (int, long) | Eclipse Collections, HPPC, fastutil | `HashMap<Integer,V>` | Boxed: 36 bytes/entry. Primitive: 8 bytes/entry. **77% less memory, up to 10x faster** |
| Concurrent reads | `ConcurrentHashMap` | `Collections.synchronizedMap` | CHM uses lock striping (per-bucket CAS); synced map locks entire map |
| Sorted data | `TreeMap` (if needed) | `LinkedList` as sorted list | Insertion sort in LinkedList is O(n) per insert |

### String traps
- **Compact Strings** (JDK 9+, JEP 254): strings with Latin-1 chars use 1 byte/char (byte[] + LATIN1 coder), not 2. Mixing in a single non-Latin-1 char doubles the backing array
- **String.intern()**: uses a native hashtable with fixed bucket count. At scale, becomes a bottleneck. Prefer manual deduplication with `ConcurrentHashMap.computeIfAbsent`
- **String concatenation with `+`**: since JDK 9, compiled to `invokedynamic` → `StringConcatFactory`. Generally fine. But in a loop, each iteration still allocates a new String — use `StringBuilder` for loops

---

## 3. Synchronization & Locking

### Lock cost ladder (approximate, single-socket, uncontended)

| Operation | x86-64 cost | ARM (AArch64) cost | Notes |
|--|--|--|--|
| Normal field read/write | ~1 ns | ~1 ns | |
| `volatile` read | ~1 ns (free on x86 TSO) | ~2-5 ns (needs `LDAR`) | x86 has strong memory model — volatile reads are plain loads |
| `volatile` write | ~20-50 ns (`LOCK`ed store or `mfence`) | ~10-30 ns (`STLR`) | x86 pays more — needs store buffer flush |
| CAS (`compareAndSet`) | ~15-30 ns (L1 hit) | ~15-40 ns (LDXR/STXR loop) | Multi-socket NUMA: 100-300 ns |
| `synchronized` (uncontended) | ~20 ns thin lock | ~20 ns | CAS on mark word |
| `synchronized` (first contention) | ~10 μs (inflation to fat lock) | ~10 μs | OS mutex allocation, context switch |
| `synchronized` (contended) | ~1-10 μs per acquire | ~1-10 μs | OS thread park/unpark, context switch |
| `ReentrantLock` (uncontended) | ~20 ns | ~20 ns | CAS on state field |

### Lock states (JDK 15+, biased locking removed)
1. **Thin lock** (uncontended): CAS on object mark word. ~20 ns
2. **Spinning**: brief busy-wait before inflation. JVM-tuned
3. **Fat lock** (inflated): OS mutex. Park/unpark = syscall + context switch

### What to look for
- **Lock granularity**: single lock for large data structure → bottleneck. Use lock striping or `ConcurrentHashMap`
- **Lock ordering**: inconsistent order → deadlock
- **`synchronized` on `this` or `ClassName.class`**: any external code can also lock on these → unexpected contention. Prefer private `final Object lock = new Object()`
- **StampedLock optimistic reads**: for read-heavy workloads, `tryOptimisticRead()` avoids acquiring any lock at all. But NOT reentrant — self-deadlock if misused
- **VarHandle memory modes** (JDK 9+): `getOpaque`/`setOpaque` < `getAcquire`/`setRelease` < `getVolatile`/`setVolatile`. Use the weakest mode that suffices — overkill ordering = wasted fences
- **Virtual threads pinning (JDK 21-23)**: `synchronized` blocks pin virtual threads to carrier threads, destroying scalability. **Fixed in JDK 24** (JEP 491). But native methods still pin
- **Thread pool + ThreadLocal**: thread pools reuse threads → ThreadLocal values leak across tasks. Clean up in `finally` or use `ScopedValue` (JDK 21+ preview)

### How to detect
- **JFR**: `jdk.JavaMonitorWait`, `jdk.JavaMonitorEnter` events — shows which locks are contended and for how long
- **async-profiler**: `./asprof -e lock <pid>` — lock contention profiling
- `-Djdk.tracePinnedThreads=full` (JDK 21+) — logs virtual thread pinning stack traces

---

## 4. JIT Compilation & Code Shape

### Inlining
- Methods ≤ 325 bytes bytecode are candidates (`-XX:MaxInlineSize=325`). Hot methods up to `FreqInlineSize` (default 325)
- **Megamorphic call sites**: if a call site sees **>2 concrete receiver types**, HotSpot gives up inlining → no escape analysis, no scalar replacement, no loop optimization on that path. This is the **single most impactful** JIT deoптимization
  - Symptoms: interface with 3+ implementations dispatched at same call site
  - Fix: restructure to limit polymorphism at hot call sites, or use `instanceof` checks to create monomorphic branches
- Getters/setters are always inlined (trivial methods). Don't fear abstraction for tiny methods

### Escape analysis
- If an object doesn't escape the method (after inlining), it can be **scalar-replaced** (fields go to registers/stack, no heap allocation at all)
- **What breaks it**: object passed to non-inlined method, stored in field, returned from method, too-deep inlining chain, megamorphic call sites
- Verify with `-XX:+PrintEscapeAnalysis` (debug JVM) or JITWatch

### Loop optimizations
- **Counted loops** (`for (int i = 0; i < n; i++)`): eligible for unrolling, vectorization (SIMD), safepoint removal
- **Non-counted loops** (while with complex conditions, iterators): no unrolling, safepoints on every back-edge
- **Safepoints in loops**: JIT inserts safepoint polls in non-counted loops. Tight non-counted loops can block GC for seconds. Use `@jdk.internal.vm.annotation.IntrinsicCandidate` patterns or restructure as counted loops
- `Stream.forEach` over large collections: the lambda is an inner class → often megamorphic at the internal `accept()` call → less optimization than a plain for loop

### Intrinsics — hand-optimized by HotSpot
- `System.arraycopy`, `Arrays.copyOf` — memcpy-level performance
- `Math.min/max/abs/sqrt/log` — single CPU instructions
- `String.equals`, `Arrays.equals` — SIMD comparisons
- `Integer.bitCount`, `Long.numberOfLeadingZeros` — `POPCNT`/`LZCNT` instructions
- `Object.hashCode`, `System.identityHashCode`
- Using these is always better than hand-rolling equivalent logic

### How to detect
- `-XX:+PrintCompilation` — shows which methods are compiled, deoptimized
- `-XX:+PrintInlining` — shows inlining decisions and failures ("too big", "no static binding", "not inlineable")
- **JITWatch** — visual tool for analyzing HotSpot JIT log output
- **async-profiler** CPU flame graph — wide flat frames indicate non-inlined hot code

---

## 5. GC & Allocation Pressure

### Key metrics
- **Allocation rate** (MB/s): the single most important GC metric. High allocation rate → frequent young GC → latency spikes. Measure with JFR `jdk.ObjectAllocationInNewTLAB` / `jdk.ObjectAllocationOutsideTLAB`
- **TLAB allocation** (fast path): bump a pointer, ~10 ns. Each thread gets its own TLAB in Eden
- **Outside-TLAB allocation** (slow path): requires synchronization, ~100x slower. Triggered by large objects or TLAB exhaustion
- **Humongous allocations** (G1): objects > 50% of region size allocated directly in Old gen, skipping young gen collection. Default region is 1-32 MB (auto-sized). Fix: increase `-XX:G1HeapRegionSize` or reduce object size
- **Promotion rate**: objects surviving young GC → old gen. High promotion = frequent mixed/full GC

### What to look for
- **Autoboxing in loops**: `for (int i : map.values())` — each value unboxed. `map.put(key, i + 1)` — boxed back. Thousands of throwaway `Integer` objects. `Integer` cache only covers -128 to 127
- **String concatenation in loops**: `result += str` creates new String per iteration. Use `StringBuilder`
- **`String.format()` in hot paths**: parses format string every call. Pre-format or use `StringBuilder`
- **Varargs in hot paths**: `void log(Object... args)` allocates an `Object[]` on every call, even if the log level is disabled. Guard with level check: `if (log.isDebugEnabled())`
- **Excessive temporary objects**: iterators (`for-each` on custom Iterable), lambdas that capture variables (create a new object each time if not stateless), `Optional` in tight loops
- **Finalizers**: create a `Finalizer` reference per object, processed on a dedicated low-priority thread. Delays GC by at least 2 cycles. Use `Cleaner` or try-with-resources instead
- **Soft/Weak references**: each adds ~32 bytes overhead and a reference queue processing cost per GC cycle

### How to detect
- **JFR**: allocation profiling (`jdk.ObjectAllocationInNewTLAB`), GC pauses, promotion stats
- **async-profiler**: `./asprof -e alloc <pid>` — shows allocation flame graph (where objects are allocated)
- `-verbose:gc` or `-Xlog:gc*` — GC log analysis with GCEasy or GCViewer
- **jmap -histo**: quick object count/size histogram

---

## 6. I/O & Networking

### Socket tuning
| Setting | Default | Issue | Fix |
|--|--|--|--|
| `TCP_NODELAY` | `false` (Nagle ON) | Nagle buffers small writes → 40ms delay for small messages | Set `true` for latency-sensitive protocols. Netty defaults to `true` |
| `SO_SNDBUF` / `SO_RCVBUF` | OS-dependent (usually 128KB) | Too small for high-throughput, too large wastes memory | Set based on BDP (bandwidth × RTT). For 1 Gbps × 10ms RTT = ~1.2 MB |
| `SO_LINGER` | OFF | Closing socket may lose buffered data | Set for reliable shutdown (with timeout) |
| `SO_REUSEADDR` | `false` | `TIME_WAIT` prevents quick server restart | Set `true` on server sockets |

### ByteBuffer performance
- **Heap ByteBuffer**: data in Java heap. Every native I/O call copies to a temporary direct buffer internally
- **Direct ByteBuffer**: data in native memory. No copy for I/O, but allocation is expensive (~1 μs vs ~10 ns for heap). Reuse direct buffers, don't allocate per-request
- **Direct buffer leak**: not freed by GC promptly. Freed only when the `Cleaner` runs. Under memory pressure, can cause `OutOfMemoryError: Direct buffer memory`. Monitor with `-XX:MaxDirectMemorySize`
- **MappedByteBuffer** (mmap): maps file into virtual address space. Good for random access to large files. Costs: page faults (~1-10 μs on miss), TLB pressure, no explicit `unmap()` in Java (GC-dependent → file handle leak risk)
- **FileChannel.transferTo()**: true zero-copy on Linux (`sendfile` syscall). Data goes kernel→NIC, never enters user space. Use for file serving

### Common I/O traps
- **DNS in NIO**: `InetSocketAddress(hostname, port)` resolves DNS synchronously in constructor. In NIO event loops, this blocks the selector thread. Resolve async or pre-resolve
- **Selector wakeup cost**: `selector.wakeup()` writes a byte to a pipe → syscall. Don't call on every event
- **Epoll spin bug** (JDK < 11): `Selector.select()` can return 0 events in a tight loop burning CPU. Netty has a workaround (rebuilds selector)
- **Buffered streams**: raw `OutputStream.write(byte)` → one syscall per byte. Always wrap with `BufferedOutputStream` (8KB default) or write byte arrays
- **io_uring** (JDK 21+ via Panama/JNI): async I/O with ring buffer. Zero syscalls in steady state. Still experimental in Java ecosystem (Netty has io_uring transport)

---

## 7. Common Anti-Patterns in Hot Paths

| Anti-pattern | Cost | Fix |
|--|--|--|
| `Pattern.compile()` every call | ~1-10 μs per compile | Cache as `static final Pattern` |
| `SimpleDateFormat` in multi-thread | Not thread-safe + slow | `DateTimeFormatter` (immutable, thread-safe) |
| Exception for control flow | `fillInStackTrace()` walks entire stack ~5-50 μs | Use return codes, Optional, sentinel values |
| `Class.forName()` | Triggers classloading, acquires locks | Cache the Class reference |
| Reflection `getDeclaredMethod()` + `invoke()` | ~50-100 ns after warmup, but prevents inlining | Cache `MethodHandle` or use code generation |
| `Thread.sleep(1)` for timing | Minimum granularity ~1-15 ms (OS-dependent) | `LockSupport.parkNanos()` — microsecond precision |
| `Collections.unmodifiableList(new ArrayList<>(list))` | Wraps + copies | `List.copyOf(list)` — single compact copy |
| `toString()` in log arguments | Evaluated even if log level is off | `log.debug("x={}", () -> expensive())` |
| `synchronized` on boxed `Integer`/`Long` | Integer cache means different values share the same object → unrelated threads contend | Lock on dedicated `Object` |

---

## 8. Profiling Toolkit

| Tool | What it measures | When to use |
|--|--|--|
| **async-profiler** | CPU, allocations, locks, wall clock | First line of investigation. Low overhead (<5%), no safepoint bias |
| **JFR** (Java Flight Recorder) | Everything (CPU, GC, I/O, threads, locks, allocations) | Production-safe continuous profiling. ~1% overhead |
| **JOL** | Object memory layout | Reviewing data structure designs, finding padding/false sharing |
| **JMH** (Java Microbenchmark Harness) | Method-level throughput/latency | Validating optimization hypotheses. Handles warmup, JIT, GC correctly |
| **JITWatch** | JIT compilation decisions | Understanding why code isn't inlined/optimized |
| **jcmd** | Thread dumps, heap info, JFR control | Quick runtime diagnostics |
| **GCEasy / GCViewer** | GC log analysis | Understanding GC behavior, pause distribution |
| `perf` + `perf-map-agent` | Hardware counters (cache misses, branch mispredictions, TLB) | Low-level CPU performance investigation |

### Key JVM diagnostic flags
```
# Compilation
-XX:+PrintCompilation
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining

# GC
-Xlog:gc*:file=gc.log:time,uptime,level,tags
-XX:+HeapDumpOnOutOfMemoryError

# JFR (always-on, low overhead)
-XX:StartFlightRecording=filename=app.jfr,maxsize=500m,settings=profile

# Virtual threads
-Djdk.tracePinnedThreads=full

# Object layout (JDK 25+)
-XX:+UseCompactObjectHeaders
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

Sources:
- [Shipilev: Objects Inside Out](https://shipilev.net/jvm/objects-inside-out/)
- [OpenJDK JOL](https://github.com/openjdk/jol)
- [JEP 519: Compact Object Headers](https://www.infoq.com/news/2025/06/java-25-compact-object-headers/)
- [Marc Brooker: Volatile Reads](https://brooker.co.za/blog/2012/09/10/volatile.html)
- [Marc Brooker: Atomic and Volatile on x86](https://brooker.co.za/blog/2012/11/13/increment.html)
- [Doug Lea: JDK 9 Memory Order Modes](https://gee.cs.oswego.edu/dl/html/j9mm.html)
- [DZone: False Sharing in JVM](https://dzone.com/articles/what-false-sharing-is-and-how-jvm-prevents-it)
- [DZone: Megamorphic Call Sites](https://dzone.com/articles/too-fast-too-megamorphic-what)
- [JVM Advent: JIT, Inlining, Escape Analysis](https://www.javaadvent.com/2015/12/jit-compiler-inlining-escape-analysis.html)
- [HotSpot Intrinsics](https://alidg.me/blog/2020/12/10/hotspot-intrinsics)
- [G1 Humongous Allocations](https://krzysztofslusarski.github.io/2020/11/10/humongous.html)
- [Zero-copy, mmap, Java NIO](https://xunnanxu.github.io/2016/09/10/It-s-all-about-buffers-zero-copy-mmap-and-Java-NIO/)
- [Chronicle: Faster Java Sockets](https://chronicle.software/how-to-make-java-sockets-faster/)
- [HN: TCP_NODELAY Discussion](https://news.ycombinator.com/item?id=40310896)
- [JEP 491: Virtual Threads without Pinning](https://www.infoq.com/news/2024/11/java-evolves-tackle-pinning/)
- [Java 24: Thread Pinning Revisited](https://mikemybytes.com/2025/04/09/java24-thread-pinning-revisited/)
- [OpenJDK: Synchronization Internals](https://wiki.openjdk.org/display/HotSpot/Synchronization)
- [Baeldung: Primitive Collections](https://www.baeldung.com/java-eclipse-primitive-collections)
- [IT Hare: CPU Cycle Costs](http://ithare.com/infographics-operation-costs-in-cpu-clock-cycles/)
- [StampedLock Guide](https://www.javacodegeeks.com/2024/07/a-comprehensive-guide-to-optimistic-locking-with-javas-stampedlock.html)
