# Question Map: Java Code Review — Concurrency

## Questions by Dimension

### Memory model, visibility, and safe publication
1. What does `volatile` actually guarantee here, what does it not guarantee, and why is the reviewer allowed to assume the author knows the difference?
2. For every mutable shared field, is it either guarded by a lock, declared `volatile`, or safely published through a sanctioned idiom (static initialiser, `final` field, `volatile`/`AtomicReference`, guarded-by-lock)?
3. For each "effectively immutable" object shared across threads, are all its fields `final` so the JMM's initialisation-safety guarantee actually applies?
4. Which reads are paired with which writes via `volatile`, `synchronized`, `Lock`, `Thread.start`, `Thread.join`, or `final`-field publication — where exactly are the happens-before edges this diff relies on?
5. Is "immutable" holding only for the reference and not for the mutable collection/array/date reachable from it?
6. Under what reachable code path is a freshly constructed object handed to another thread before its constructor's writes have a happens-before edge to that thread's read (CERT TSM03-J: `this` escape)?
7. Which fields used for inter-thread signalling lack `volatile`, `AtomicX`, or a lock, and where is the happens-before edge actually established?
8. Which lazily-initialised fields visible to multiple threads use `final`, `volatile`, the Holder idiom, or `AtomicReference` for safe publication — and which rely on the broken pre-JSR-133 DCL?
9. On which JDK/field-modifier combination does double-checked locking quietly regress to broken (non-`volatile` holder, partially-constructed object) and produce a usable-looking but half-initialised singleton?
10. Which compound action (read-modify-write, check-then-act, iterate-then-update) silently assumes atomicity that `volatile` alone cannot provide?
11. Does the code treat "atomic" as JMM-atomic (indivisible wrt other threads) or the colloquial "happens in one go" — and does the comment vocabulary match the semantics?
12. Does the code rely on "benign" data races that Shipilev's *JMM Pragmatics* warns can surface torn or stale values on real hardware?
13. Has the JSR-133 out-of-thin-air rule been checked for any race judged "harmless" — does the implementation prohibit fabricated values?
14. Does any comment or `@GuardedBy` annotation still reference pre-JSR-133 memory-model assumptions ("volatile only prevents caching") that the implementation would not actually satisfy?
15. Does the reviewer's implicit model of "correct concurrency" secretly assume x86 TSO in ways that break on Graviton / Apple Silicon / ARM?
16. If escape analysis, lock elision, or AOT (GraalVM native-image) changes the bytecode-to-source mapping, which review conclusions survive?

### Synchronisation primitives and their misuse
1. How does a reader tell, from reading the class alone, whether a field needs `synchronized`, `volatile`, an `AtomicReference`, or nothing at all?
2. Why is this synchronisation primitive the right choice — would `AtomicReference`, `StampedLock`, `ReentrantReadWriteLock`, or an immutable snapshot be simpler and more correct?
3. For the `{primitive} × {operation}` matrix — `synchronized`, `ReentrantLock`, `StampedLock`, `Atomic*`, `volatile`, lock-free CAS against read / write / RMW / iterate / bulk update — is the chosen primitive actually sufficient, or does an iteration leak non-atomic reads?
4. On which invocation does the `synchronized` lock object get reassigned, boxed, or replaced so that two threads enter the block holding different monitors?
5. Does a `synchronized` method on a Spring-proxied / CGLIB / AOP-woven bean actually serialise on the intended monitor, or on the proxy?
6. Is the lock released on every exit path including exceptions (try/finally, try-with-resources on `Lock.lock()`)?
7. For every `wait()`, is the predicate checked in a `while` loop so spurious wakeups and notify-before-wait orderings are tolerated?
8. For every `ReentrantLock` / `ReentrantReadWriteLock` / `Semaphore` introduced, is fairness an explicit decision (not an accident of the default constructor)?
9. If a counter uses `count++` on a shared `long`, is arithmetic actually atomic, or does it need `LongAdder` / `AtomicLong`?
10. Where a custom synchroniser is introduced, has the author considered whether `AbstractQueuedSynchronizer` already provides the primitive, avoiding a hand-rolled CAS loop?
11. If the primitive claims lock-free, wait-free, or obstruction-free progress, is the condition actually proven, or is "uses CAS somewhere" being confused with non-blocking progress per Herlihy–Shavit?
12. For custom concurrent data structures, has linearisability (not just mutual exclusion) been argued, and are the linearisation points identified?
13. For memory reclamation in custom lock-free structures, is a named scheme used (hazard pointers, epoch-based, RCU), or is reclamation ad-hoc?
14. Does any lock-acquisition path nest a monitor inside a `ReentrantLock` region (or vice versa) in ways Doug Lea's concurrency-interest list calls out?
15. If each `synchronized` block were replaced with an explicit `ReentrantLock`, which properties (fairness, `tryLock`, interruptible acquire, condition variables) would become available, and which would the reviewer then demand?

### Concurrent collections and compound actions
1. Why is `HashMap` dangerous under concurrent access when the Javadoc doesn't say "will crash" — what is the real failure mode visible in a diff?
2. Which caller-side sequence of `get`-then-`put`, `containsKey`-then-`put`, or iterate-then-update reintroduces a lost-update race that a `ConcurrentHashMap`'s per-entry atomicity does not cover?
3. Does every compound action on a `ConcurrentHashMap` use `putIfAbsent` / `compute` / `computeIfAbsent` / `merge`, rather than a racy sequence of individual calls?
4. For the `{collection} × {access-pattern}` matrix — `ConcurrentHashMap`, `CopyOnWriteArrayList`, `ConcurrentSkipListMap`, `BlockingQueue`, `synchronizedMap` against single-key get/put, compute/merge, multi-key update, snapshot iteration, size-based decision — is the pattern safe for the operation?
5. Which mutable collections cross thread boundaries without a concurrent variant, a defensive copy, or a snapshot?
6. If a `Collections.synchronizedMap` is iterated, who — caller or callee — owns the external lock, and is the documented "conditionally thread-safe" contract respected?
7. What isolation level does this in-memory concurrent code actually provide (read-uncommitted, read-committed, snapshot, serialisable), and does the API name it — or does each caller guess?
8. Is there a "lost update" pattern (read, decide, write) that a database reviewer would immediately mark as needing a version column or `SELECT FOR UPDATE` equivalent, but that Java reviewers miss because no SQL triggers the reflex?
9. Where MVCC-like copy-on-write is used, has the review checked for write-skew — two readers seeing consistent snapshots that jointly violate an invariant no single operation violates?
10. If the concurrent structure were replaced by a CRDT, would the merge semantics reveal a hidden commutativity assumption the lock-based code silently depends on?

### Liveness: deadlock, livelock, starvation, and fairness
1. Which `synchronized` block or held lock can invoke foreign / overridable / user-supplied code (listener, observer, lambda), and what prevents that code from re-entering the lock graph in a different order?
2. Where are locks acquired in a non-deterministic order, creating the possibility of an AB-BA deadlock between two threads?
3. If two locks were acquired in opposite orders on two code paths, why does the lock-ordering contract exist only in a comment rather than in a `LockOrdering` enum or a `tryLock`-with-timeout wrapper?
4. Which code path could a bounded-queue producer block on `put` while holding another subsystem's lock, creating a cross-subsystem stall?
5. Where could a thread pool deadlock itself by having dependent tasks wait for workers that are all blocked waiting for those same dependent tasks?
6. What is the backoff/jitter strategy for optimistic retries, and how do we prove it terminates rather than livelocks under contention?
7. Is a `ReentrantReadWriteLock` starving writers under continuous reader arrival, and what is the worst-case wait for the minority operation?
8. What prevents a nested-monitor-lockout pattern (the classic Doug Lea *Concurrent Programming in Java* hazard) from arising in the custom lock protocol?
9. Is there a deadlock-detector, watchdog, or `tryLock(timeout)` strategy, and does the telemetry expose a thread-dump signature distinctive enough for on-call diagnosis?
10. Treating threads as cars and locks as intersections, where does this change introduce gridlock geometry (cycles in the wait-for graph) versus merely adding a stop sign?

### Cancellation, interruption, and task lifecycle
1. How is `Thread.interrupt()` handled — is it swallowed, and does that break cancellation and liveness for the surrounding task?
2. Do callers of `Future.get`, `CompletableFuture.join`, `BlockingQueue.take` actually handle `InterruptedException` correctly, or is interruption routinely dropped?
3. Does `Future.cancel(true)` or timeout actually interrupt the work, or only release the waiter while the task continues mutating the shared result?
4. How is cancellation propagated — via `Future.cancel`, interruption, explicit cancel flags, or structured concurrency scopes — and is the mechanism consistent across the call graph?
5. What exceptions can escape a `Runnable` / `Callable` submitted to an `ExecutorService`, and how are they surfaced so tasks don't silently disappear?
6. Are exceptions thrown from inside a `synchronized` block, a `try`-with-resources lock, or a `CompletableFuture` stage handled without leaving shared state half-updated?
7. Is `ExecutionException.getCause` dropped or logged-and-swallowed anywhere, and what diff patterns must the reviewer refuse in future PRs?
8. What is the supervision strategy when a worker thread dies mid-task — restart, escalate, stop — and is the choice explicit the way an Erlang supervisor tree would be, rather than silent death inside an `ExecutorService`?
9. When a background task outlives the scope that submitted it, can it still reference shutdown or GC-unfriendly resources?
10. Is every `Thread`, `ExecutorService`, and `ScheduledExecutorService` owned by something with a defined shutdown path, and is there a test that fails if it outlives its owner?
11. When the `ScheduledExecutorService` owner is replaced, is the executor shut down, or will `ScheduledFutureTask` queue quietly accumulate until OOM three days later?
12. Does the shutdown path drain, cancel, and join every thread/executor/scheduled task/`ThreadLocal`, or does the code assume the JVM will exit?

### Executors, thread pools, backpressure, and resource budgets
1. How is bounded resource usage enforced — queue capacity, rejected-execution policy, semaphore permits, back-pressure — and what happens at saturation?
2. Where are threads and executors created, who owns their shutdown, and is there a platform-level guardrail (factory, lint rule, architecture test) that forces every async boundary through a vetted, named, monitored pool?
3. Has `Executors.newFixedThreadPool` / `newCachedThreadPool` been replaced with a directly constructed `ThreadPoolExecutor` that sets a bounded queue and an explicit `RejectedExecutionHandler`?
4. Is a new `ExecutorService` spun up per-request, and was that lifecycle considered against thread-creation cost, shutdown-leak risk, and loss of back-pressure?
5. If this module saturates its executor, does the blast radius cascade into shared pools, HTTP clients, or the common `ForkJoinPool`?
6. On which `CompletableFuture` stage does work silently fall back to the common `ForkJoinPool` and block a carrier thread other critical tasks depend on?
7. Is `parallelStream` or `CompletableFuture.supplyAsync` leaking into the common pool a long-running or blocking task?
8. How does container CPU quota interact with `ForkJoinPool.commonPool()` sizing and `Runtime.availableProcessors()` when the JVM sees host cores but is CFS-throttled to one vCPU?
9. Does retry logic, failover, or caller-side parallelism combine with the diff to multiply load on a contended resource rather than relieve it?
10. Are timeouts, retry policies, and circuit-breaker semantics still monotonic under load, or does this change introduce thread-starvation amplification?
11. Does the new work hold threads idle while waiting on I/O — paying for stack memory and context-switch CPU — when a non-blocking or virtual-thread design would shrink the fleet?
12. What locks are held across remote HTTP calls, DB calls, or circuit-breaker open states, and can a 30 s remote call pin a lock per request?

### Virtual threads, structured concurrency, and Project Loom
1. Are virtual threads being adopted here, and does the review catch the currently-hot pitfalls — `synchronized` pinning (pre-JDK 24 / JEP 491), native/FFM calls pinning, `ThreadLocal` memory amplification?
2. Does the diff pool virtual threads (an Alan Bateman anti-pattern) instead of creating a fresh virtual thread per task?
3. Do `ThreadLocal` values allocated on platform threads remain safe on pooled carriers, or do they leak and amplify memory across millions of virtual threads?
4. Is the code still treating thread-pool sizing as a live tuning concern now that virtual threads invert that assumption, and is the reviewer asking whether the pool is even necessary?
5. Is `synchronized` still being used knowing whether the target JDK has removed pinning, and does the review flag the version-dependent correctness?
6. Would a Loom-native rewrite expose which of today's synchronisation primitives are incidental complexity versus load-bearing?
7. Would structured concurrency (JEP 453/480) make the current try/finally/shutdown patterns obsolete, and should the reviewer push for a forward-compatible shape today?
8. Does the ownership of cancellation, deadlines, and exception propagation become clearer under `StructuredTaskScope` than under a `CompletableFuture` pipeline?
9. At what thread-count threshold (hundreds → millions) do review heuristics (pool sizing, contention profiling, `ThreadLocal` audit) stop being relevant and which new ones (scope confinement, pinning audit, carrier-thread saturation) take over?

### ThreadLocal, scoping, and thread-confined state
1. What `ThreadLocal` values are allocated, and what is the leak story on thread-pool-managed and virtual-thread carriers?
2. In which pooled-thread scenario does a stale `ThreadLocal` value leak across requests and silently authorise, tag, or bill the wrong user?
3. Is every `ThreadLocal.set` paired with a `remove` in a `finally` block reached on exception, timeout, and async handoff (including `CompletableFuture` stage boundaries)?
4. Which thread-confinement contracts break silently when a `CompletableFuture` callback runs on the common pool and the `ThreadLocal` disappears, or reappears from a previous request?
5. For the `{scope} × {escape}` matrix — local variable, field, static field, `ThreadLocal`, DI-scoped bean, request-scoped bean against accidental sharing, intentional sharing, escape via collection, escape via listener registration, escape via `this` in constructor — where does the diff create a new escape route?
6. Where does shared mutable state leak across module / package / framework boundaries (servlets, filters, DI scopes, actor mailboxes)?

### State design, invariants, and architectural refactoring
1. Who owns each piece of mutable state in the change, and which threads are authorised to read or write it?
2. Does the review ever ask "should this be concurrent and mutable at all?" before asking "is this concurrent code correct?"
3. Could the shared state be pushed into a request-scoped object, a thread-local, or a database row with optimistic locking rather than protected with a lock?
4. Does every public class document its thread-safety contract (immutable / thread-safe / conditionally thread-safe / thread-hostile) in Javadoc the way JCIP prescribes?
5. Is every mutable shared field annotated with `@GuardedBy` (or a type-level equivalent), naming the monitor that protects it?
6. What invariants span multiple fields, and are those fields updated as a single atomic compound action under one lock?
7. For each `synchronized` method, is the invariant the lock is supposed to protect written down, and does every read/write path respect it?
8. Does the type being passed between threads satisfy the Java analogue of Rust's `Send` (safe to transfer) and `Sync` (safe to share), and is the property explicit in annotations or a confinement contract?
9. What if every `synchronized` keyword were removed — which invariants still hold because state is thread-confined or already immutable, and which collapse, revealing what the lock was silently protecting?
10. Would replacing mutable shared state with an `AtomicReference` copy-on-write immutable snapshot eliminate a class of races, and which call sites break?
11. Would replacing shared state with message-passing or per-request copies mean the remaining concurrency no longer needs a memory-model argument?
12. How can protected regions be simultaneously large enough to preserve a multi-step invariant and small enough to not serialise the critical path (separation in time, space, condition, level)?
13. How can shared data be simultaneously mutable for writes and immutable for readers (copy-on-write, persistent data structures, versioned snapshots, read-write separation)?
14. Would a structurally different design — actor, STM, single-writer event loop, lock-free queue, Loom virtual threads, copy-on-write snapshot — eliminate the class of bug this PR is patching around?
15. Why has the team's concurrency model evolved by ad-hoc `synchronized` patches instead of a deliberate architectural choice (immutable domain + single-writer, actor, STM, reactive)?

### Scale, performance, and mechanical sympathy
1. Does the primitive choice match the actual observed contention profile (low-contention reads, high-contention reads, write-heavy, read-mostly, bursty) or an imagined one?
2. What changes qualitatively if thread count grows 10× — does a coarse `synchronized` become a hotspot, a bounded queue overflow, or a `ThreadLocal` cache explode memory?
3. What changes qualitatively if critical sections shrink 10× (fine-grained striping) — do invariants that held under one big lock break across stripe boundaries?
4. Does any hot shared field cause false sharing across cache lines, and is `@Contended` / manual padding on the review's radar?
5. Does the shared mutable state display "mechanical sympathy" — layout-aware padding, single-writer principle, lock-free queues — or is a naive `BlockingQueue` used where an LMAX-style Disruptor would remove contention entirely?
6. Does the chosen concurrency structure produce a throughput curve that degrades gracefully under overload or cliffs at saturation?
7. Could a long-running loop delay or prevent a JVM safepoint (per David Holmes's safepoint work), distorting tail latency?
8. Does latency-sensitive code suffer from coordinated-omission of safepoint-induced pauses that silently distort percentile measurements?
9. Does GC choice (G1 / ZGC / Shenandoah / Epsilon) change the probability profile of a race the review is reasoning about?

### Review process, tooling, and team practice
1. How should the reviewer budget attention across the diff so the 10% of lines carrying 90% of the concurrency risk get disproportionate scrutiny?
2. Is a human code review the right instrument for this question, or should the change be gated by a tool (ErrorProne `@GuardedBy`, SpotBugs MT, jcstress, ThreadSanitizer) the reviewer can rely on?
3. Which of today's OSS concurrency-analysis tools (jcstress, Lincheck, ErrorProne `@GuardedBy`, SpotBugs MT, IntelliJ inspections) actually run on this repository's CI today, and what is their false-negative rate?
4. Has the reviewed path been run under a race detector (FastTrack-style tooling, Java Pathfinder, jcstress) on the specific interleavings the change claims to forbid?
5. For a typical PR, what is the smallest jcstress harness a reviewer can demand as a merge-blocking artefact, and who on the team can write it?
6. Does the PR description attach a thread dump, JFR event, or async-profiler view when the change touches a lock, executor, or blocking call?
7. Which concurrency behaviours in this diff are even testable with JUnit, which need jcstress / Lincheck, and which are only observable under production load?
8. Is the test suite tolerating flakiness that is actually a reproducible race, and how would a reviewer tell the difference from the diff?
9. Are there defined merge-blocking "no-fly" conditions — any new shared mutable field without `@GuardedBy`, any new `Thread.sleep` inside a `synchronized` block, any new `wait` without a `while` — enforced independent of reviewer judgement?
10. When a concurrency bug escapes, does the team perform a no-blame root-cause analysis that asks "why did the review not catch this?" and updates the checklist?
11. When two team members independently review the same concurrency-sensitive change, do they converge on the same issues, and if not, what does the divergence say about the shared model?
12. Is the author using "thread-safe" as a virtue signal rather than a falsifiable property — can anyone state the invariant being preserved and the interleaving being ruled out?
13. Does the author sound defensive or hand-wavy when describing the locking strategy, and what does that emotional signal suggest about author confidence?
14. Which five diff patterns should a junior reviewer be told to always escalate to a senior, and what is the written rubric?
15. How is the team's concurrency review calibrated — is there a post-mortem archive of concurrency incidents every reviewer should mentally consult, and is this change touching an implicated system?

### Production diagnosis, observability, and cross-paradigm boundaries
1. If this deadlocks or hangs a thread pool at 02:00, what thread-dump signature will on-call see, and can they recognise it from this PR alone?
2. Has the author added enough observability (queue depths, lock-wait metrics, timeouts with distinct exception types) for an SRE to diagnose a concurrency incident without reading the code?
3. Given a thread dump showing N threads BLOCKED on monitor X, which review-checklist item, if it had existed, would have prevented this signature?
4. When executor queue depth climbs without bound overnight, which review-time question about backpressure, rejection policy, or producer lifecycle should have fired before merge?
5. Are logs emitted from concurrent paths still causally ordered enough to reconstruct what happened for a given transaction ID?
6. Is an in-JVM happens-before relation still a happens-before relation once the component is deployed across JVMs — does the review flag in-process assumptions that dissolve at a network boundary?
7. Does the code confuse intra-JVM synchronisation with cross-node consistency (e.g. `synchronized` protecting a cache that is actually shared via Redis)?
8. Where is the true serialisation point for a uniqueness invariant — application-level "if not exists" logic or a database unique constraint / `SELECT FOR UPDATE SKIP LOCKED`?
9. When this class is called from Kotlin coroutines, Scala `Future`s, or Groovy synthetic bridges, do cancellation, interruption, and `@GuardedBy` contracts still hold?
10. If a JNI or FFM call is reachable from a `synchronized` block, what carrier-thread and safepoint assumptions does it violate?
11. When a Kotlin `suspend` function is bridged to a Java `CompletableFuture`, which thread is the continuation resumed on, and is that the reviewer's assumption?
12. Could a race condition here be weaponised — TOCTOU on an auth check, iterator over a mutated permissions map, double-spend, request smuggling through a shared buffer?
13. Does any lock held across an untrusted operation create a denial-of-service lever an attacker can pull with a slow or malformed request?
14. If this service runs in regulated jurisdictions (GDPR / HIPAA / PCI-DSS / financial FIFO), which specific concurrency failure modes become compliance-reportable, and does the reviewer know them by name?
15. Is every state transition touching regulated data serialisable, auditable, and reconstructible from logs even under concurrent contention?

## Frontier Questions
> Ten questions whose answers would most change understanding of the topic.

1. What is the declared thread-safety category of each class under review — immutable, thread-safe, conditionally thread-safe, or thread-hostile — and is it documented in Javadoc? — Before debating primitives, the reviewer needs a contract; without it, every comment is an argument about what the code *ought* to have meant.
2. For each `synchronized` method, is the invariant the lock is supposed to protect written down anywhere, and does every read/write path respect it? — Most concurrency bugs are not "missing lock" but "lock protects no coherent invariant"; naming the invariant moves review from taste to falsifiability.
3. Where exactly are the happens-before edges — which reads are paired with which writes via `volatile`, `synchronized`, `Lock`, `Thread.start`, `Thread.join`, or `final`-field publication? — Memory visibility is the dimension reviewers most often hand-wave past; forcing explicit edge identification is the single largest step-change in review quality.
4. Which caller-side sequence of `get`-then-`put`, `containsKey`-then-`put`, or iterate-then-update reintroduces a lost-update race that a `ConcurrentHashMap`'s per-entry atomicity does not cover? — This is the single most cross-validated duplicate across agents; "we used `ConcurrentHashMap`, so it's fine" is the most common false claim the review must learn to reject.
5. Which `synchronized` block or held lock can invoke foreign / overridable / user-supplied code (listener, observer, lambda), and what prevents that code from re-entering the lock graph in a different order? — Almost every AB-BA deadlock in production Java starts with a callback-under-lock; making it a standing review question eliminates a whole class of 3 a.m. incidents.
6. Does the review ever ask "should this be concurrent and mutable at all?" before asking "is this concurrent code correct?" — Reframing review from "is this safe?" to "does this need to exist?" deletes entire sections of the checklist by deleting the code the checklist applies to.
7. Are virtual threads being adopted here, and does the review catch the currently-hot pitfalls — `synchronized` pinning (pre-JDK 24), native / FFM pinning, `ThreadLocal` memory amplification, pooling-virtual-threads anti-pattern? — The field is mid-phase-transition; the answer to this question rewrites the meaning of "pool size", "synchronized", and "thread-local" for everything that follows.
8. What if the assumption that mutexes compose (that locking two thread-safe components yields a thread-safe composition) is false in general — what review question drops out of that failure? — Most team disagreements about "is this thread-safe?" stem from an unarticulated belief in compositionality; surfacing it changes how reviewers think about library boundaries forever.
9. Is a human code review the right instrument for this question, or should it be gated by a tool (ErrorProne `@GuardedBy`, SpotBugs MT, jcstress, Lincheck) the reviewer can rely on? — Deciding what belongs to humans and what belongs to machines is the meta-decision that determines whether the team's review budget is spent on solvable problems or unsolvable ones.
10. Why has the team's concurrency model evolved by ad-hoc `synchronized` patches instead of a deliberate architectural choice (immutable domain + single-writer, actor, STM, reactive)? — A PR-level review cannot fix an architecture-level error; naming the pattern is the only way to escalate from line-comments to a design conversation.

## Open Terrain
> Dimensions still underexplored after all agents ran.

- **Economics and time-budget calibration of review**: Agent 10 gap-filled the existence of the question, but nobody knows the dollar value of a missed lost-update vs a deadlock vs a visibility bug for a given product — without those numbers, "spend more time on concurrent diffs" stays aspirational.
- **Novice-reviewer onboarding and rubric transfer**: The expertise to spot a broken DCL is tacit and Socratic; the field has no agreed-upon curriculum, worked-example gallery, or competency assessment, so each team reinvents the mentoring path and most teams don't.
- **Externalities and fairness**: When a concurrency decision shifts cost from developer time to energy consumption, user fairness, or tenant isolation, no stakeholder at the review table owns the externality — and until the review checklist names who speaks for it, the question literally cannot be asked.
- **Empirical reviewer accuracy**: There is no CVE-style public registry of concurrency bugs that escaped review, so the field cannot measure what fraction of reviews actually find concurrency bugs, cannot A/B test checklist changes, and cannot rank reviewers or tools by detection rate.
- **Provenance and stability of happens-before across build tooling**: Lock ordering, annotations, and memory barriers survive source-level review but may be rewritten by AOP weaving, bytecode transformers, Kotlin synthetic bridges, GraalVM native-image, or profile-guided JIT — no routine review artefact confirms the guarantee round-trips.
- **Schedule-space coverage as a reviewable property**: "We tested it" almost never means "we explored the schedule space"; the field has no accepted metric analogous to line/branch coverage for schedules, so the question "how much of the interleaving space does the test suite actually exercise?" remains nearly unaskable.
- **Regulatory and audit-trail reconstruction under concurrency**: Serialisability of regulated state transitions, causal log ordering, and cross-tenant leakage disclosure timelines are named in compliance regimes but not in any Java concurrency review checklist — the seam between compliance language and JMM language is still open territory.

---

*Raw per-agent outputs: /Users/aleksei/tmp/coverage-map-java-review-concurrency/outputs/. Coverage audit: /Users/aleksei/tmp/coverage-map-java-review-concurrency/outputs/10-coverage-audit.md.*

<!-- COMPLETE -->
