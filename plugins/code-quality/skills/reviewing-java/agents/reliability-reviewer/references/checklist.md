# Java Reliability Bug Patterns, Fault Tolerance, and Code Review Guide

Concise reference for reviewing Java code for correctness, resource management, fault tolerance, data integrity, and backward compatibility. Focus on subtle bugs that cause production incidents.

## Table of Contents
1. Resource Management -- try-with-resources, connection pools, ThreadLocal (virtual-thread caveat), classloader leaks, off-heap memory (4 subsections)
2. Data Integrity & Correctness -- overflow, floating-point / `BigDecimal`, boundary conditions, date/time (monotonic vs wall clock, DST), encoding, null safety (6 subsections)
3. Database & Persistence -- N+1, transactions + propagation table, JPA / Hibernate pitfalls + cascade table, HikariCP sizing formula, batch-insert anti-pattern (5 subsections)
4. Resilience & Fault Tolerance -- Nygard stability antipatterns, metastable failure (DDIA), circuit breaker (Resilience4j 2.4.0), retry with jitter (AWS Full Jitter + Google SRE retry budget), POST retry bug pattern, timeouts, bulkhead, fallback, decorator composition (9 subsections)
5. REST API Correctness -- HTTP status codes (RFC 9110, RFC 6585), `Retry-After` (RFC 9110 §10.2.3), idempotency (RFC 9110 §9.2.2), pagination, validation (RFC 9457), versioning (6 subsections)
6. Backward Compatibility -- binary / source / behavioral (JLS §13), serialization / Protobuf / Avro, deprecation (JEP 277) (5 subsections)
7. Graceful Shutdown -- Spring Boot default behavior correction, Kubernetes `preStop`, cleanup order, review points (4 subsections)
8. Review Checklist -- `- [ ]` bullets (7 subsections)
9. Quick Reliability Review Heuristic -- 10-point scan order
10. Key References -- Nygard, Kleppmann DDIA, Google SRE, Goetz JCIP, RFCs, Resilience4j, HikariCP
11. Research Questions to Cross-Check Against Sources -- 52 questions (buckets A–J), each mapped to sections and sources
12. Sources -- normative / specification, JEPs, foundational / authoritative, tooling, checklists, industry case studies
13. Gotchas -- 16 common wrong assumptions (G-01..G-16)

---

## 1. Resource Management

### 1.1 Try-With-Resources
- Every `AutoCloseable` (streams, connections, statements, channels, readers, writers) **must** use try-with-resources
- Nested resources: declare in order so dependents close first (outer resource wrapping inner)
- `Stream` from `Files.lines()`, `BufferedReader.lines()`, JPA streaming queries **must** be closed — they hold underlying I/O resources

```java
// bad: stream not closed, file handle leak
Files.lines(path).filter(line -> line.contains("ERROR")).forEach(System.out::println);

// good:
try (Stream<String> lines = Files.lines(path)) {
    lines.filter(line -> line.contains("ERROR")).forEach(System.out::println);
}
```
Source: `Files.lines` Javadoc, "The returned stream encapsulates a Reader. If timely disposal of file system resources is required, the try-with-resources construct should be used."

### 1.2 Connection Pool Leaks
- JDBC: every `Connection`, `Statement`, `ResultSet` in try-with-resources. A single leaked connection under load exhausts the pool → application freeze
- Enable HikariCP leak detection: `spring.datasource.hikari.leak-detection-threshold=5000` (ms) — logs stack trace of the code that borrowed the connection
- HTTP clients: close `Response` / `InputStream` from HTTP calls. Apache HttpClient `EntityUtils.consume(response.getEntity())` or use try-with-resources on `CloseableHttpResponse`

```java
// bad: connection leaked if query or result-processing throws
Connection conn = dataSource.getConnection();
PreparedStatement ps = conn.prepareStatement(sql);
ResultSet rs = ps.executeQuery();
return map(rs);

// good: try-with-resources closes in reverse order on any exit path
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    return map(rs);
}
```
Source: HikariCP wiki, "About Pool Sizing" and `leak-detection-threshold` docs; JDBC 4.3 spec.

### 1.3 ThreadLocal Cleanup
- `ThreadLocal.remove()` **must** be called in a `finally` block, especially in thread pools (servlet filters, `@Async`, `CompletableFuture`)
- `ThreadLocal.set(null)` is **not** equivalent to `remove()` — the entry stays in the thread's map, leaking the `ThreadLocal` key
- Pattern: set in filter/interceptor `preHandle`, remove in `afterCompletion`/`finally`
- **Virtual threads (JDK 21+)**: a pooled `ThreadLocal` set times thousands of virtual threads multiplies memory. Prefer `ScopedValue` (JEP 506, final in JDK 25) for per-request context

```java
// bad: ThreadLocal entry survives the thread's return to the pool
private static final ThreadLocal<User> CURRENT = new ThreadLocal<>();
void handle(Request r) {
    CURRENT.set(r.user());
    process();
}

// good:
void handle(Request r) {
    CURRENT.set(r.user());
    try {
        process();
    } finally {
        CURRENT.remove();
    }
}
```
Source: `ThreadLocal` Javadoc; JEP 506 (Scoped Values, Final, JDK 25).

### 1.4 Other Resource Leaks
- **Event listeners / observers**: deregister on shutdown (`@PreDestroy`, `contextDestroyed`). Especially critical for `ApplicationListener` in child contexts
- **JDBC drivers in web apps**: explicitly deregister in `contextDestroyed` to prevent classloader leaks. Tomcat 6.0.24+ deregisters automatically via `JreMemoryLeakPreventionListener`; other containers may not. Alternative: place driver JAR in container `/lib`, not webapp `/WEB-INF/lib`
- **Direct ByteBuffer**: lives off-heap, freed only by GC + Cleaner. Track with `-XX:MaxDirectMemorySize`. Pool and reuse, don't allocate per-request
- **ExecutorService**: `shutdown()` / `shutdownNow()` on app teardown. Register shutdown hook or `@PreDestroy`. Unshut thread pools keep the JVM alive
- **Classloader leaks**: caused by static references to classloader-scoped objects (JDBC drivers, thread-local values, shutdown hooks registered but not removed)

Source: Nygard, *Release It!* 2nd ed. (2018) ch. 4 "Stability Antipatterns" §§4.5 Blocked Threads, 4.9 Slow Responses, 4.11 Unbounded Result Sets, 5.4 Steady State.

---

## 2. Data Integrity & Correctness

### 2.1 Integer Overflow
- Use `Math.addExact()`, `Math.subtractExact()`, `Math.multiplyExact()` for arithmetic that must not silently overflow — throws `ArithmeticException` on overflow
- Array size calculations: `int newCapacity = oldCapacity + (oldCapacity >> 1)` can overflow silently → negative array size → `NegativeArraySizeException` or wrap-around
- `(int)(longValue)` silently truncates — check bounds first or use `Math.toIntExact()`

```java
// bad: wraps to negative on overflow, silently corrupts balance
int newBalance = balance + credit;

// good: throws ArithmeticException instead of producing a wrong answer
int newBalance = Math.addExact(balance, credit);
```
Source: SEI CERT NUM00-J. Detect or prevent integer overflow; `Math.addExact` Javadoc.

### 2.2 Floating-Point and BigDecimal
- **Never** use `double`/`float` for money — use `BigDecimal` initialized from `String` (`new BigDecimal("0.1")`), never from `double` (`new BigDecimal(0.1)` = `0.1000000000000000055511...`)
- **Always** specify `RoundingMode` with `BigDecimal.divide()` — unspecified rounding throws `ArithmeticException` for non-terminating decimals
- Compare with `BigDecimal.compareTo()` (treats same-value / different-scale as equal), not `equals()`. Per the `BigDecimal` Javadoc: "the `equals` method requires both the numerical value and representation to be the same for equality to hold." `new BigDecimal("1.0").equals(new BigDecimal("1.00"))` returns `false`; `compareTo` returns `0`. `BigDecimal`'s natural ordering is inconsistent with `equals`
- For scientific/engineering: compare with epsilon tolerance: `Math.abs(a - b) < 1e-9`

```java
// bad: double literal 0.1 is not exactly 0.1 in IEEE-754
BigDecimal a = new BigDecimal(0.1);         // 0.1000000000000000055511151231257827021181583404541015625
// bad: equals is scale-sensitive — breaks HashSet / HashMap keys
Set<BigDecimal> prices = Set.of(new BigDecimal("1.0"));
prices.contains(new BigDecimal("1.00"));    // false

// good:
BigDecimal a = new BigDecimal("0.1");
if (a.compareTo(b) == 0) { ... }
```
Source: `BigDecimal` Javadoc (Java SE 21).

### 2.3 Boundary Conditions
- Check `isEmpty()` before `get(0)`, `iterator().next()`, or `stream().findFirst().get()`
- Off-by-one: `for (int i = 0; i <= list.size(); i++)` — one past the end. `substring(0, length - 1)` — missing last char
- Empty strings vs null: decide on convention and be consistent. Use `StringUtils.isBlank()` to handle both
- `Collections.singletonList()` and `List.of()` are immutable — `add()` throws `UnsupportedOperationException`

### 2.4 Date & Time
- **Store and transmit as UTC**: use `Instant` for timestamps. Convert to `ZonedDateTime` only at display boundaries
- Use IANA zone IDs (`Europe/Berlin`, `America/New_York`), never fixed offsets (`+01:00` — doesn't handle DST)
- **DST gaps**: `LocalDateTime.of(2024, 3, 31, 2, 30)` doesn't exist in `Europe/Berlin` (clocks skip 2:00→3:00). `ZonedDateTime.of()` silently adjusts — verify this is intended
- **DST overlaps**: `LocalDateTime.of(2024, 10, 27, 2, 30)` is ambiguous in `Europe/Berlin` (occurs twice). Use `ZonedDateTime.ofStrict()` if precision matters
- **Never** use `java.util.Date`, `Calendar`, or `SimpleDateFormat` in new code — use `java.time.*`
- `LocalDate.parse("2024-02-29")` is valid (leap year), `LocalDate.parse("2025-02-29")` throws — validate dates from external input
- **Durations / elapsed time**: use `System.nanoTime()` (monotonic, immune to NTP / DST / admin clock edits). Use `Instant.now()` / `System.currentTimeMillis()` **only** for wall-clock timestamps. Wall clock can jump backwards

```java
// bad: wall clock can jump (NTP step, manual change) -> negative elapsed
long start = System.currentTimeMillis();
doWork();
long elapsed = System.currentTimeMillis() - start;

// good: monotonic clock
long start = System.nanoTime();
doWork();
long elapsedNanos = System.nanoTime() - start;
```
Source: `System.nanoTime` Javadoc; OpenJDK bug [JDK-6458294](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6458294); JCIP.

### 2.5 Character Encoding
- Use `String.codePointCount()` / `codePoints()` for strings with emoji, CJK extensions, or other supplementary Unicode characters. `String.length()` counts UTF-16 code units, not characters
- `charAt(i)` returns a `char` (16-bit) — can return half a surrogate pair for characters outside BMP
- Specify charset explicitly: `new String(bytes, StandardCharsets.UTF_8)`. Platform-default charset varies by OS/locale

Source: Oracle, *Supplementary Characters in the Java Platform*; Unicode Standard Annex #29.

### 2.6 Null Safety
- `Optional` for method return types, `Objects.requireNonNull()` for constructor/method parameters
- `Map.get(key)` returns `null` for missing keys — use `getOrDefault()`, `computeIfAbsent()`, or check with `containsKey()`
- `Map.of()` / `List.of()` throw `NullPointerException` on null elements — by design, but surprising if migrating from `HashMap`/`ArrayList`
- `equals()` contract: `null.equals(x)` throws NPE. Always call `equals()` on the known non-null side: `"constant".equals(variable)`

---

## 3. Database & Persistence

### 3.1 N+1 Query Problem
- **Symptom**: loading a parent entity triggers N separate SELECTs for child entities (one per parent)
- **Detection**: enable Hibernate statistics in tests (`spring.jpa.properties.hibernate.generate_statistics=true`) or use `spring.jpa.show-sql=true`
- **Fix**: `JOIN FETCH` in JPQL, `@EntityGraph`, `@BatchSize(size = 100)`, or DTO projection (most efficient)

```java
// bad: one SELECT for orders, then N SELECTs for order.items
List<Order> orders = em.createQuery("FROM Order", Order.class).getResultList();
for (Order o : orders) {
    total += o.getItems().size();  // triggers lazy load per order
}

// good: single query
List<Order> orders = em.createQuery(
    "SELECT DISTINCT o FROM Order o JOIN FETCH o.items", Order.class).getResultList();
```
Source: Hibernate User Guide §6.3 "Fetching strategies"; Baeldung *N+1 Problem in Hibernate*.

### 3.2 Transactions
- `@Transactional(readOnly = true)` for read-only operations — skips dirty checking and flush, enables replica routing
- `@Transactional` propagation (from `org.springframework.transaction.annotation.Propagation`):

| Value | Behavior |
|---|---|
| `REQUIRED` (default) | Joins existing or creates new |
| `REQUIRES_NEW` | Suspends outer; starts new physical transaction — verify intentional |
| `SUPPORTS` | Joins if present, runs non-transactionally otherwise |
| `MANDATORY` | Requires active transaction, throws otherwise |
| `NOT_SUPPORTED` | Suspends any existing transaction |
| `NEVER` | Throws if a transaction exists |
| `NESTED` | Savepoint inside outer transaction (JDBC only) |

- **Visibility rules (Spring 6.x)**: `public`, `protected`, and package-visible methods are transactional with class-based (CGLIB) proxies (the default). Interface-based (JDK) proxies still require `public`. `private` methods are **never** supported. Self-invocation (`this.method()`) always bypasses the proxy
- **Long transactions**: holding a transaction open during HTTP calls, file I/O, or user interaction locks database rows and exhausts connection pool
- Transaction boundary should be at the service layer, not the controller

```java
// bad: self-invocation bypasses proxy -> no transaction starts
@Service
class OrderService {
    public void handle(Order o) { save(o); }        // no @Transactional here
    @Transactional public void save(Order o) { ... } // proxy not involved
}

// good: call the proxied bean, or inject self
```
Source: Spring Framework reference, *Transaction Management — Declarative Transactions* and *Using @Transactional*.

### 3.3 JPA / Hibernate Pitfalls

| Pitfall | Problem | Fix |
|---|---|---|
| Lazy loading outside session | `LazyInitializationException` or silent null | Use DTO projection, `JOIN FETCH`, or `@EntityGraph`. **Disable OSIV** (`spring.jpa.open-in-view=false`) |
| `equals()`/`hashCode()` on `@Id` | Unstable before `persist()`, breaks `Set` membership | Implement on natural/business key |
| Missing `@Version` | Lost updates under concurrent writes | Add `@Version` for optimistic locking, handle `OptimisticLockException` with retry |
| `CascadeType.ALL` + `orphanRemoval` | Deleting parent cascades to all children — may delete too much; re-parenting a shared child deletes it unexpectedly. DB `ON DELETE CASCADE` + JPA `orphanRemoval` causes double-delete SQL errors | Use specific cascade types (`PERSIST`, `MERGE`); never apply `orphanRemoval` on `@ManyToMany` or re-parented children |
| `FetchType.EAGER` on `@ManyToOne` | Loads entire object graph on every query | Use `FetchType.LAZY` (default for collections, not for `@ManyToOne`) |
| `save()` inside loop | N individual INSERTs instead of batch | Use `saveAll()` with `spring.jpa.properties.hibernate.jdbc.batch_size=50` and `hibernate.order_inserts=true` |

**Cascade type semantics** (`jakarta.persistence.CascadeType`):

| Type | Propagates |
|---|---|
| `PERSIST` | `em.persist()` |
| `MERGE` | `em.merge()` |
| `REMOVE` | `em.remove()` |
| `REFRESH` | `em.refresh()` |
| `DETACH` | `em.detach()` |
| `ALL` | All of the above |

Source: JPA 3.1 spec §11.1.11; Vlad Mihalcea, "The best way to use orphanRemoval in JPA and Hibernate".

### 3.4 Connection Pool (HikariCP)
- **Sizing formula**: `connections = ((core_count * 2) + effective_spindle_count)`. Per the HikariCP wiki: "HT is not counted" (use physical cores); "effective_spindle_count = 0 if fully cached"; and the explicit caveat — "there hasn't been any analysis so far regarding how well the formula works with SSDs". Treat as a starting point, load-test to tune
- Monitor pool wait time — if consistently > 0, pool is too small or connections leak
- Set `connectionTimeout` (how long to wait for a connection from pool, default 30s) and `maxLifetime` (connection max age, default 30min — set **strictly less** than the database's `wait_timeout` / idle connection kill threshold)
- `leak-detection-threshold` (ms): enable in staging / prod to log stack traces of borrowers that held a connection past the threshold

Source: [HikariCP wiki, About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing).

### 3.5 Batch Insert Anti-Pattern
- Calling `save()` per element inside a loop generates N flushes and N INSERT round-trips
- Use `saveAll(List)` with `hibernate.jdbc.batch_size` set (typically 20–100). Also set `hibernate.order_inserts=true` and `hibernate.order_updates=true` so Hibernate groups same-type statements. For bulk work, `em.flush()` + `em.clear()` every batch keeps the first-level cache small

```java
// bad: N flushes, N round-trips, first-level cache grows unbounded
for (Item i : items) repo.save(i);

// good: batched
int batchSize = 50;
for (int i = 0; i < items.size(); i++) {
    em.persist(items.get(i));
    if (i % batchSize == 0) { em.flush(); em.clear(); }
}
```
Source: Hibernate User Guide §12 "Batching".

---

## 4. Resilience & Fault Tolerance

### 4.1 Stability Antipatterns (Nygard)
Nygard, *Release It!* 2nd ed. (2018) catalogues the recurring pathologies. Review for each:

- **Integration Points** (ch. 4.1): every remote call is a point of cascading failure unless wrapped in timeout + circuit breaker
- **Chain Reactions** (ch. 4.3): failure of one node amplifies load on its peers until they fail too
- **Blocked Threads** (ch. 4.5): caller threads block on a stuck downstream → pool exhaustion → application freeze
- **Slow Responses** (ch. 4.9): worse than outright failure; consumes resources without releasing. "Fail fast" is preferable
- **Unbounded Result Sets** (ch. 4.11): queries without `LIMIT`, unbounded iterators, pagination ignored → OOM
- **Steady State** (ch. 5.4): log rotation, data archival, cache size caps — without them, disk fills and everything stops
- **Fail Fast** (ch. 5.5): validate inputs, check capacity, verify downstreams before doing any work
- **Handshaking** (ch. 5.6): backpressure signals between tiers (HTTP 429, queue high-water marks)
- **SLA Inversion** (ch. 4.10): your SLA can never exceed the weakest downstream's SLA multiplied across dependencies

Source: Nygard, *Release It!* 2nd ed., Pragmatic Bookshelf 2018, ISBN 978-1-68050-239-8.

### 4.2 Reliability and Metastable Failure (DDIA)
- "Reliability" = continuing to work correctly when faults happen, not being fault-free (DDIA 2e ch. 2)
- **Retry storm → metastable failure**: once load exceeds capacity, naive retries amplify load, the system degrades further, and **recovery does not happen when load drops**. The canonical defence stack: exponential backoff **plus** jitter **plus** circuit breaker **plus** token bucket / rate limiting **plus** load shedding **plus** backpressure
- Distributed systems are nondeterministic (DDIA ch. 9): a timeout cannot distinguish lost request / crashed node / lost response. TCP-level "reliable delivery" does not mean application reliability — a crash between ACK and processing is invisible to the sender

Source: Kleppmann, *Designing Data-Intensive Applications*, O'Reilly 2e (2025); ch. 2 sidebar "When an Overloaded System Won't Recover"; ch. 9 "The Trouble with Distributed Systems".

### 4.3 Circuit Breaker (Resilience4j 2.4.0)
Resilience4j 2.4.0 (March 14 2024) requires Java 17+. Current defaults (verify against `CircuitBreakerConfig`):

| Property | Default |
|---|---|
| `failureRateThreshold` | 50 % |
| `slowCallRateThreshold` | 100 % (off) |
| `slowCallDurationThreshold` | 60 s |
| `permittedNumberOfCallsInHalfOpenState` | 10 |
| `slidingWindowType` | `COUNT_BASED` |
| `slidingWindowSize` | 100 |
| `minimumNumberOfCalls` | 100 |
| `waitDurationInOpenState` | 60 s |

- Wrap every remote call (HTTP, gRPC, database, message broker) with a circuit breaker
- **Sliding window does NOT cap concurrency** — pair with a `Bulkhead`
- **Half-open state**: probe calls limited by `permittedNumberOfCallsInHalfOpenState` — don't raise it so high that recovery floods a struggling service

Source: [Resilience4j docs — CircuitBreaker](https://resilience4j.readme.io/docs/circuitbreaker).

### 4.4 Retry — Exponential Backoff with Jitter
- Use exponential backoff **with jitter**. AWS "Full Jitter" formula: `sleep = random(0, min(cap, base * 2^attempt))`
- Equal Jitter: `sleep = base*2^attempt / 2 + random(0, base*2^attempt / 2)`
- Decorrelated Jitter: `sleep = min(cap, random(base, prev_sleep*3))`
- **Never** retry non-idempotent operations (POST, PATCH) without an `Idempotency-Key` — see §5.3
- Set a **maximum attempt count** (typically 3) and a **total timeout** encompassing all retries
- Retry only on transient errors (network timeout, `503`, `429`) — never on 4xx client errors (except `429`)
- **Retry budget (Google SRE)**: cap retries at 10 % of total requests per client, combined with a 3-attempt-per-request limit. Worst-case amplification is ~1.1× vs. ~3× without a budget
- AWS SDKs default to exponential backoff + jitter across all services

```java
// bad: lockstep retries cause thundering-herd retry storm
for (int i = 0; i < 3; i++) {
    try { return call(); } catch (Exception e) {
        Thread.sleep(1000L * (long) Math.pow(2, i));   // same sleep on every client
    }
}

// good: full jitter breaks lockstep
long base = 100, cap = 20_000;
for (int attempt = 0; attempt < 3; attempt++) {
    try { return call(); } catch (TransientException e) {
        long sleep = ThreadLocalRandom.current()
            .nextLong(0, Math.min(cap, base * (1L << attempt)));
        Thread.sleep(sleep);
    }
}
```
Source: Marc Brooker, "Exponential Backoff And Jitter" (AWS Architecture Blog, 2015); Beyer et al., *Site Reliability Engineering* (O'Reilly 2016) ch. 22 "Addressing Cascading Failures" — "Handling Overload" discussion of retry budgets.

### 4.5 POST Retry Without Idempotency Key
- Retrying a `POST` that created a resource on the first (silent) success creates a duplicate. Use an `Idempotency-Key` header, store key → response mapping with TTL, return the cached response on duplicate
- The header is specified in the IETF HTTPAPI draft `draft-ietf-httpapi-idempotency-key-header` (updated 2025-10-15, not yet an RFC). Stripe's public implementation predates and inspired the draft

```java
// bad: network blip after server commit -> client retries -> duplicate charge
@PostMapping("/charges") Charge charge(@RequestBody Req r) { return svc.charge(r); }

// good:
@PostMapping("/charges")
Charge charge(@RequestHeader("Idempotency-Key") String key, @RequestBody Req r) {
    return idempotencyStore.findCached(key).orElseGet(() -> {
        Charge c = svc.charge(r);
        idempotencyStore.put(key, c, Duration.ofHours(24));
        return c;
    });
}
```
Source: [IETF draft-ietf-httpapi-idempotency-key-header](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/); [Stripe API docs — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests).

### 4.6 Timeouts
- **Every** remote call must have explicit connect and read timeouts. No infinite waits
- `RestTemplate` / `WebClient` / `HttpClient`: set `connectTimeout` (1-5 s typical) and `readTimeout` (5-30 s typical, depends on operation)
- Database query timeout: `@QueryHints(@QueryHint(name = "jakarta.persistence.query.timeout", value = "5000"))`
- Set a `TimeLimiter` wrapping the **entire** operation (including retries)

Source: Nygard, *Release It!* 2nd ed. ch. 5.1 "Timeouts"; `java.net.http.HttpClient` Javadoc.

### 4.7 Bulkhead
- Apply semaphore or thread-pool isolation so one slow downstream cannot exhaust the caller's thread pool
- Example: separate thread pools for "payment service" and "notification service" — payment timeout doesn't block notifications

Source: Nygard, *Release It!* 2nd ed. ch. 5.2 "Bulkheads"; Resilience4j docs — Bulkhead.

### 4.8 Fallback
- Every resilience-decorated call **must** have a fallback: cached value, degraded response, or meaningful error
- Never an empty catch block or silent swallow as a "fallback"

### 4.9 Decorator Composition Order
Typical outer-to-inner for synchronous code: `Retry(CircuitBreaker(TimeLimiter(Bulkhead(actualCall))))`. For the async `Decorators` API the explicit chained order Resilience4j documents is `.withBulkhead().withTimeLimiter().withCircuitBreaker().withRetry().withFallback()` — note the docs also show `.withCircuitBreaker().withBulkhead().withRetry()` as an example; ordering is a deliberate business-level choice. Trade-off:

- **Retry outside CircuitBreaker**: retries fire even when CB is closed; CB counts each retry as a separate call
- **Retry inside CircuitBreaker**: retries do not fire once CB opens — but the retry exhausts before CB can open

Source: [Resilience4j docs — Getting Started / Decorators](https://resilience4j.readme.io/docs/getting-started-3).

---

## 5. REST API Correctness

### 5.1 HTTP Status Codes

| Operation | Success | Failure |
|---|---|---|
| GET (found) | `200 OK` | `404 Not Found` |
| POST (create) | `201 Created` + `Location` header | `409 Conflict` (duplicate), `422 Unprocessable Entity` (validation) |
| PUT (replace) | `200 OK` or `204 No Content` | `404 Not Found`, `409 Conflict` |
| PATCH (partial) | `200 OK` | `404`, `422` |
| DELETE | `204 No Content` | `404 Not Found` (or `204` — idempotent) |
| Rate limited | — | `429 Too Many Requests` + `Retry-After` header (see §5.2) |
| Server error | — | `500` (unexpected), `503` (temporarily unavailable) + `Retry-After` |

`429 Too Many Requests` is specified in **RFC 6585 §4** (Nottingham & Fielding, 2012) — NOT RFC 9110. RFC 6585 says the server "MAY" include `Retry-After`. All other status codes above come from RFC 9110 (HTTP Semantics, June 2022).

Source: [RFC 6585 — Additional HTTP Status Codes](https://datatracker.ietf.org/doc/html/rfc6585); [RFC 9110 — HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110).

### 5.2 Retry-After Header Values
Per RFC 9110 §10.2.3: `Retry-After` takes either a **delta-seconds** integer (`Retry-After: 120`) or an HTTP-date (`Retry-After: Fri, 31 Dec 2021 23:59:59 GMT`). Servers SHOULD emit it on `429`, `503`, and `301`. Clients must parse both forms.

Source: [RFC 9110 §10.2.3](https://datatracker.ietf.org/doc/html/rfc9110#section-10.2.3).

### 5.3 Idempotency — RFC 9110 Authoritative List
Per **RFC 9110 §9.2.2**, the idempotent request methods are: **`GET`, `HEAD`, `OPTIONS`, `TRACE`, `PUT`, `DELETE`**. `POST` and `PATCH` are **not** idempotent by default.

- Safe methods (no observable state change): `GET`, `HEAD`, `OPTIONS`, `TRACE` (§9.2.1)
- For non-idempotent operations (POST, PATCH) that must tolerate client retries: accept an `Idempotency-Key` header, store `(key, response, status)` with a TTL (24-48 h), return the cached response on duplicate. See §4.5 for the bug pattern

Source: [RFC 9110 §9.2.2](https://datatracker.ietf.org/doc/html/rfc9110#section-9.2.2); [IETF draft-ietf-httpapi-idempotency-key-header](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/).

### 5.4 Pagination
- **Cursor-based** (preferred for mutable data): `GET /items?cursor=abc&limit=20`. Return `nextCursor` and `hasMore`
- **Offset-based** (simpler, but skips/duplicates items if data changes): `GET /items?page=2&size=20`. Return `totalElements`, `totalPages`
- Always return pagination metadata in response body, not just headers
- Set a maximum page size (e.g., 100) — never allow clients to request unbounded results

Source: Kleppmann, DDIA 2e ch. 5 "Encoding and Evolution"; [RFC 5988 / RFC 8288 Web Linking](https://datatracker.ietf.org/doc/html/rfc8288) for `Link` header.

### 5.5 Request Validation
- Validate all input at the boundary: `@Valid` on request bodies, `@NotNull`, `@Size`, `@Pattern` on parameters
- Return structured error responses with stable error codes:
  ```json
  {"code": "VALIDATION_ERROR", "message": "...", "details": [{"field": "email", "message": "..."}]}
  ```
- Never expose internal details (stack traces, SQL, file paths) in error responses. Configure `server.error.include-stacktrace=never`
- Consider [RFC 9457 Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc9457) (2023) for a wire-level standard error schema

### 5.6 API Versioning
- URL path versioning: `/api/v1/users` (simplest, most common)
- Accept header versioning: `Accept: application/vnd.company.v1+json` (REST-purist)
- Never break existing clients without a deprecation period (minimum one major release cycle)

---

## 6. Backward Compatibility

### 6.1 Types of Compatibility

| Type | Definition | Example of Break |
|---|---|---|
| **Binary** | Existing compiled code still links | Removing a public method, changing method signature |
| **Source** | Existing source still compiles | Adding abstract method to interface (without `default`) |
| **Behavioral** | Existing code still works correctly | Changing sort order, changing default values, adding validation |

Binary compatibility is formally defined in [JLS §13 Binary Compatibility](https://docs.oracle.com/javase/specs/jls/se21/html/jls-13.html). Eclipse's API Evolution Rules are a concise operational codification of this spec.

Source: [Eclipse API Evolution Rules](https://wiki.eclipse.org/Evolving_Java-based_APIs_2); JLS §13.

### 6.2 Safe Changes (non-breaking)
- Adding new methods with `default` implementation to interfaces
- Adding new classes, new packages
- Adding new optional parameters (with default values)
- Widening access (protected → public)

### 6.3 Unsafe Changes (breaking)
- Removing or renaming public/protected methods, classes, fields
- Changing method signatures (parameter types, return types)
- Adding abstract methods to public interfaces
- Narrowing access (public → protected/private)
- Changing exception types (widening catch clauses doesn't help callers with specific catches)

### 6.4 Serialization Compatibility
- Declare explicit `serialVersionUID` on all `Serializable` classes. Change it **only** when you intentionally want to break deserialization of old instances
- JSON API evolution: only add fields (additive). Never rename or remove fields without versioning the endpoint
- **Protobuf**: never reuse or change field numbers / tags. Only add new optional fields. Mark removed fields as `reserved`. Changing a field type is a wire-incompatible break unless the old and new types are in the same [Protobuf compatibility category](https://protobuf.dev/programming-guides/proto3/#updating) (e.g., `int32`/`uint32`/`int64`/`uint64`/`bool` are interchangeable for unsigned-in-range values; `string`/`bytes` are interchangeable for valid UTF-8)
- **Avro**: schema evolution follows Writer/Reader compatibility rules — add fields with defaults for forward compatibility

Source: [Protobuf Language Guide (proto3) — Updating A Message Type](https://protobuf.dev/programming-guides/proto3/#updating); Kleppmann, DDIA 2e ch. 5.

### 6.5 Deprecation Protocol
1. Add `@Deprecated(since = "2.3", forRemoval = true)` with Javadoc `@deprecated` tag specifying the replacement
2. Keep the old method/endpoint working for at least one major release cycle
3. Add a bridge/adapter that delegates to the new implementation
4. Log a warning when deprecated API is called (with migration instructions)
5. Remove in the next major version

Source: [JEP 277 Enhanced Deprecation (JDK 9)](https://openjdk.org/jeps/277).

---

## 7. Graceful Shutdown

### 7.1 Spring Boot (Default Behavior)
- **Graceful shutdown is the default** on Spring Boot 3.x with Tomcat, Jetty, and Reactor Netty (Spring Boot reference — "Graceful Shutdown"). Disable with `server.shutdown=immediate`
- Timeout is configured via `spring.lifecycle.timeout-per-shutdown-phase` (default `30s`)
- During shutdown, in-flight requests complete. New requests **may** be rejected (container-specific; some containers close the listener, others return `503`)
- **Do not** re-enable `server.shutdown=graceful` in your application config thinking you are opting in — it has been the default since Spring Boot 2.3 (2020) and remains default in 3.x

Source: [Spring Boot reference — Graceful Shutdown](https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html).

### 7.2 Kubernetes Integration
- Set `terminationGracePeriodSeconds` **strictly greater** than Spring's shutdown timeout + any preStop delay (e.g., `45s` if Spring's timeout is `30s`)
- Per the Kubernetes container lifecycle docs: "The Pod's termination grace period countdown **begins before** the PreStop hook is executed." preStop is synchronous and consumes part of the total grace period
- `preStop` hook is commonly used to delay SIGTERM long enough for load balancers to notice the readiness-probe failure. **A bare `sleep` is discouraged** — prefer active deregistration verification. Ensure `terminationGracePeriodSeconds >= preStop_duration + application_shutdown_budget`

Source: [Kubernetes — Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/).

### 7.3 Cleanup Order
1. Stop accepting new requests (readiness probe goes unhealthy)
2. Complete in-flight requests (within timeout)
3. Stop scheduled tasks (`@Scheduled`, Quartz)
4. Close message consumers (Kafka, RabbitMQ)
5. Flush pending writes (logs, metrics, traces)
6. Close connection pools (database, HTTP clients)
7. Deregister from service discovery

### 7.4 What to Review
- `@PreDestroy` methods exist for all resources that need cleanup
- Shutdown hooks don't have circular dependencies
- Background threads check for shutdown signal (`Thread.interrupted()`, `volatile boolean`)
- Pending Kafka/JMS messages are acknowledged or rejected, not silently lost

---

## 8. Review Checklist

### 8.1 Resource Management
- [ ] Every `AutoCloseable` (streams, connections, statements, channels, readers, writers) in try-with-resources (§1.1)
- [ ] `Files.lines()`, `BufferedReader.lines()`, JPA streaming query results closed — see §1.1 bug pattern. Source: `Files.lines` Javadoc
- [ ] `ThreadLocal.remove()` in `finally` — never `set(null)`. Source: `ThreadLocal` Javadoc
- [ ] `spring.datasource.hikari.leak-detection-threshold` enabled in non-prod. Source: HikariCP wiki
- [ ] `ExecutorService.shutdown()` / `shutdownNow()` in `@PreDestroy`. Source: JCIP ch. 7

### 8.2 Data Integrity
- [ ] `Math.addExact` / `multiplyExact` for arithmetic that must not wrap (§2.1). Source: SEI CERT NUM00-J
- [ ] `BigDecimal` from `String`, never from `double` (§2.2). Source: `BigDecimal` Javadoc
- [ ] `BigDecimal.compareTo()` for equality checks, not `equals()` (§2.2). Source: `BigDecimal` Javadoc
- [ ] `System.nanoTime()` for elapsed time, not `currentTimeMillis()` (§2.4). Source: `System.nanoTime` Javadoc
- [ ] IANA zone IDs for dates, not fixed offsets (§2.4). Source: `ZoneId` Javadoc
- [ ] Explicit charset on `String`/`InputStream` I/O (§2.5). Source: Oracle *Supplementary Characters*

### 8.3 Database & Persistence
- [ ] N+1 queries avoided via `JOIN FETCH`, `@EntityGraph`, or DTO projection (§3.1). Source: Hibernate User Guide §6.3
- [ ] `@Transactional` on public/protected/package methods only — never private (§3.2). Source: Spring Framework reference
- [ ] No self-invocation (`this.method()`) on `@Transactional` methods (§3.2). Source: Spring Framework reference
- [ ] `spring.jpa.open-in-view=false` (§3.3). Source: Spring Data JPA docs
- [ ] `@Version` on entities with concurrent writes (§3.3). Source: JPA 3.1 spec §3.4.2
- [ ] HikariCP pool sized per formula, with SSD / load-test caveat (§3.4). Source: HikariCP wiki
- [ ] HikariCP `maxLifetime < DB wait_timeout`. Source: HikariCP wiki

### 8.4 Resilience & Fault Tolerance
- [ ] Every remote call has explicit connect and read timeouts (§4.6). Source: Nygard ch. 5.1
- [ ] Circuit breaker wraps every remote call (§4.3). Source: Resilience4j docs, Nygard ch. 5.3
- [ ] Retry uses exponential backoff **with jitter**, not lockstep delay (§4.4). Source: AWS "Exponential Backoff and Jitter"
- [ ] Retry budget capped (Google SRE: 10 % retries, 3 attempts) (§4.4). Source: Google SRE book ch. 22
- [ ] Non-idempotent operations require `Idempotency-Key` before retry (§4.5, §5.3). Source: RFC 9110 §9.2.2
- [ ] Every resilience-decorated call has a real fallback (§4.8). Source: Nygard ch. 5

### 8.5 REST API
- [ ] HTTP status codes match RFC 9110 semantics (§5.1). Source: RFC 9110
- [ ] `429` from RFC 6585, includes `Retry-After` when possible (§5.1–5.2). Source: RFC 6585, RFC 9110 §10.2.3
- [ ] Idempotent methods limited to RFC 9110 §9.2.2 list: GET, HEAD, OPTIONS, TRACE, PUT, DELETE (§5.3). Source: RFC 9110
- [ ] Maximum page size enforced server-side (§5.4)
- [ ] `server.error.include-stacktrace=never` (§5.5). Source: Spring Boot reference

### 8.6 Backward Compatibility
- [ ] No removal / rename / type-change of public API in a non-major release (§6.3). Source: JLS §13
- [ ] Explicit `serialVersionUID` on `Serializable` classes (§6.4). Source: `Serializable` Javadoc
- [ ] Protobuf changes preserve field numbers, add as `optional`, mark removed as `reserved` (§6.4). Source: Protobuf Language Guide
- [ ] `@Deprecated(since, forRemoval)` with Javadoc migration note (§6.5). Source: JEP 277

### 8.7 Graceful Shutdown & Ops
- [ ] Not redundantly enabling Spring Boot graceful shutdown — it is the default (§7.1). Source: Spring Boot reference
- [ ] `terminationGracePeriodSeconds > preStop + spring.lifecycle.timeout-per-shutdown-phase` (§7.2). Source: Kubernetes Container Lifecycle Hooks
- [ ] `@PreDestroy` on every resource-holding bean (§7.4). Source: JSR-250
- [ ] Background threads poll `Thread.interrupted()` or a `volatile boolean` (§7.4). Source: JCIP ch. 7

---

## 9. Quick Reliability Review Heuristic

When reviewing Java code for reliability, scan in this order:

1. **Resources closed?** Every `AutoCloseable` in try-with-resources? Connections returned to pool? (§1)
2. **Null handled?** `Optional` returns, `Objects.requireNonNull` at boundaries, no blind `.get()` or `iterator().next()`? (§2.6)
3. **Overflow checked?** `Math.addExact` for critical arithmetic? Array size calculations safe? (§2.1)
4. **Money as BigDecimal?** Never `double` for financial calculations? `compareTo`, not `equals`? (§2.2)
5. **Dates in UTC?** `Instant` for storage, IANA zone IDs, `System.nanoTime` for durations? (§2.4)
6. **Transactions correct?** Right propagation, not private, not holding during I/O, no self-invocation? (§3.2)
7. **N+1 queries?** Lazy collections loaded efficiently? OSIV disabled? (§3.1, §3.3)
8. **Timeouts set?** Every remote call has connect + read timeout? No infinite waits? (§4.6)
9. **Retries safe?** Only idempotent ops or with idempotency key? Full-jitter backoff? Retry budget? (§4.4, §4.5)
10. **Graceful shutdown?** Relying on Spring Boot default, not re-enabling by mistake? `@PreDestroy` for resources? (§7)

---

## Key References

- Michael T. Nygard, *Release It! Design and Deploy Production-Ready Software*, 2nd ed., Pragmatic Bookshelf 2018, ISBN 978-1-68050-239-8 — ch. 4 Stability Antipatterns (Integration Points, Chain Reactions, Blocked Threads, Slow Responses, Unbounded Result Sets, SLA Inversion); ch. 5 Stability Patterns (Timeouts, Bulkheads, Circuit Breaker, Fail Fast, Handshaking, Steady State)
- Martin Kleppmann, *Designing Data-Intensive Applications*, 2nd ed., O'Reilly 2025, ISBN 978-1-098-11926-5 — ch. 2 Reliability / metastable failure sidebar; ch. 5 Encoding and Evolution; ch. 9 The Trouble with Distributed Systems
- Beyer, Jones, Petoff, Murphy (eds.), *Site Reliability Engineering: How Google Runs Production Systems*, O'Reilly 2016, [free HTML edition](https://sre.google/sre-book/table-of-contents/) — ch. 22 Addressing Cascading Failures; *Handling Overload*
- Brian Goetz et al., *Java Concurrency in Practice*, Addison-Wesley 2006 — ch. 7 Cancellation and Shutdown
- [RFC 9110 HTTP Semantics (June 2022)](https://datatracker.ietf.org/doc/html/rfc9110) — §9.2.1 safe methods, §9.2.2 idempotent methods, §10.2.3 Retry-After, §15 status codes
- [RFC 6585 Additional HTTP Status Codes (2012)](https://datatracker.ietf.org/doc/html/rfc6585) — §4 429 Too Many Requests
- [Resilience4j 2.4.0 documentation](https://resilience4j.readme.io/docs/getting-started)
- [HikariCP — About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [Spring Boot reference — Graceful Shutdown](https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html)
- [AWS Architecture Blog — "Exponential Backoff And Jitter" (Brooker, 2015)](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)

## Research Questions to Cross-Check Against Sources

The checklist is organised to answer the following questions about a diff or PR. Each question is followed by the corresponding section(s) and the primary source(s) used to verify the answer.

### A. Resource management
1. **Is every `AutoCloseable` in a try-with-resources block, including `Stream` results from `Files.lines()` and JPA streaming queries?** (§1.1) — source: `Files.lines` Javadoc; JCIP ch. 7.
2. **Does every borrowed JDBC `Connection`, `Statement`, and `ResultSet` close on every exit path?** (§1.2) — source: JDBC 4.3 spec; HikariCP wiki.
3. **Is `leak-detection-threshold` enabled in staging/prod?** (§1.2, §8.1) — source: HikariCP wiki.
4. **Does every `ThreadLocal.set` have a matching `remove()` in `finally`, especially in pooled threads?** (§1.3) — source: `ThreadLocal` Javadoc.
5. **For virtual threads, are large `ThreadLocal` payloads replaced with `ScopedValue`?** (§1.3) — source: JEP 506 (final, JDK 25).
6. **Do web apps deregister JDBC drivers in `contextDestroyed` (or rely on container-level leak prevention)?** (§1.4) — source: Tomcat `JreMemoryLeakPreventionListener` docs.
7. **Is every `ExecutorService` shut down via `@PreDestroy`, with `shutdown()` + `awaitTermination()`?** (§1.4, §8.1) — source: JCIP ch. 7.

### B. Data integrity and correctness
8. **Is `Math.addExact` / `multiplyExact` / `toIntExact` used where wrap-around is a real risk (balances, sizes, counters)?** (§2.1) — source: SEI CERT NUM00-J.
9. **Is `BigDecimal` constructed only from `String`?** (§2.2) — source: `BigDecimal(double)` Javadoc.
10. **Is equality on `BigDecimal` done with `compareTo`, not `equals`, in `Set`/`Map` keys or tests?** (§2.2) — source: `BigDecimal` Javadoc.
11. **Does `BigDecimal.divide` call an overload that takes a `RoundingMode`?** (§2.2) — source: `BigDecimal.divide` Javadoc.
12. **Is elapsed time measured with `System.nanoTime()` rather than `currentTimeMillis()` / `Instant.now()`?** (§2.4) — source: `System.nanoTime` Javadoc; OpenJDK JDK-6458294.
13. **Are all date-storage / transmission values UTC (`Instant`) with IANA zone ids used at display time?** (§2.4) — source: `ZoneId` Javadoc.
14. **Are DST gaps and overlaps explicitly handled rather than relying on silent adjustment?** (§2.4) — source: `ZonedDateTime.of` Javadoc.
15. **Is string size measured in code points when supplementary characters are possible?** (§2.5) — source: Oracle *Supplementary Characters*.

### C. Transactions and JPA
16. **Is `@Transactional` only on `public` (interface proxy) or `public`/`protected`/package (class proxy) methods — never `private`?** (§3.2) — source: Spring Framework reference *Declarative Transactions*.
17. **Is self-invocation (`this.method()`) on a `@Transactional` method absent — or explicitly routed through a proxy / injected self?** (§3.2) — source: Spring Framework reference.
18. **Does every entity with concurrent writes have `@Version`, and is `OptimisticLockException` handled?** (§3.3) — source: JPA 3.1 §3.4.2; Baeldung *JPA Optimistic Locking*.
19. **Is `spring.jpa.open-in-view=false`?** (§3.3, §8.3) — source: Spring Data JPA docs.
20. **Does `CascadeType.ALL` + `orphanRemoval=true` actually match the delete semantics, especially for shared / re-parented children?** (§3.3) — source: JPA 3.1 §11.1.11; Vlad Mihalcea on orphanRemoval.
21. **Is `saveAll()` plus `hibernate.jdbc.batch_size` used instead of `save()` in a loop?** (§3.5) — source: Hibernate User Guide §12.

### D. Connection pool
22. **Is HikariCP pool size derived from `((core_count * 2) + effective_spindle_count)` with an explicit SSD / load-test caveat?** (§3.4) — source: HikariCP wiki.
23. **Is `maxLifetime` strictly less than the database `wait_timeout`?** (§3.4, §8.3) — source: HikariCP wiki.
24. **Is `connectionTimeout` set, and are pool-wait metrics monitored?** (§3.4) — source: HikariCP wiki.

### E. Resilience composition
25. **Is every remote call (HTTP / gRPC / DB / broker) wrapped with at least a circuit breaker and a timeout?** (§4.3, §4.6) — source: Nygard ch. 4.1, 5.1, 5.3.
26. **Is the Resilience4j decorator order deliberate, and consistent across the sync / async APIs?** (§4.9) — source: Resilience4j docs.
27. **Is a bulkhead (semaphore or thread-pool) present, given that sliding-window size does not cap concurrency?** (§4.3, §4.7) — source: Resilience4j docs.
28. **Does every decorated call have a real fallback — not an empty catch?** (§4.8) — source: Nygard ch. 5.

### F. Retries and idempotency
29. **Does retry backoff use jitter (Full Jitter, Equal Jitter, or Decorrelated Jitter)?** (§4.4) — source: Brooker, AWS 2015.
30. **Is retry capped by attempt count AND total budget, with a retry budget cap (e.g., Google SRE 10 % / 3 attempts)?** (§4.4) — source: Google SRE ch. 22.
31. **Are non-idempotent operations (POST, PATCH) retried only with an `Idempotency-Key` header?** (§4.5, §5.3) — source: RFC 9110 §9.2.2; IETF HTTPAPI idempotency-key draft.
32. **Is retry restricted to transient errors (timeout, 503, 429)?** (§4.4) — source: RFC 9110 §15.6; RFC 6585.
33. **Does the timeout on each decorator leave room for the outermost `TimeLimiter` to abort the full retried call?** (§4.6) — source: Resilience4j docs.

### G. REST API correctness
34. **Do all responses use HTTP status codes with meanings from RFC 9110 (and RFC 6585 for 429)?** (§5.1) — source: RFC 9110, RFC 6585.
35. **Does every `429` / `503` response include `Retry-After` when the server knows a reasonable delay?** (§5.1, §5.2) — source: RFC 9110 §10.2.3.
36. **Is the list of idempotent methods restricted to RFC 9110 §9.2.2: GET, HEAD, OPTIONS, TRACE, PUT, DELETE?** (§5.3) — source: RFC 9110 §9.2.2.
37. **Do server-side error bodies follow RFC 9457 Problem Details, or a stable internal schema, without leaking stack traces?** (§5.5) — source: RFC 9457; Spring Boot reference.
38. **Is maximum page size enforced server-side so clients cannot request unbounded result sets?** (§5.4, Nygard ch. 4.11) — source: Nygard *Release It!* §4.11.

### H. Backward compatibility
39. **Does the change preserve binary compatibility per JLS §13 — no removed / renamed / retyped public members?** (§6.1, §6.3) — source: JLS §13; Eclipse API Evolution.
40. **For Protobuf changes: are field numbers preserved, removed fields `reserved`, and new fields optional?** (§6.4) — source: Protobuf Language Guide — Updating A Message Type.
41. **Is `@Deprecated(since=..., forRemoval=...)` applied with a Javadoc migration note and at least one major-release overlap?** (§6.5) — source: JEP 277.

### I. Graceful shutdown and ops
42. **Is Spring Boot graceful shutdown left at its default `graceful` — not redundantly configured, not disabled?** (§7.1) — source: Spring Boot reference.
43. **Is `spring.lifecycle.timeout-per-shutdown-phase` tuned to real request durations?** (§7.1) — source: Spring Boot reference.
44. **Is `terminationGracePeriodSeconds > preStop + shutdown timeout`, recognising that preStop counts inside the grace period?** (§7.2) — source: Kubernetes Container Lifecycle Hooks.
45. **Does the preStop hook actively verify load-balancer deregistration, or at least delay long enough for probe propagation — not a bare `sleep`?** (§7.2) — source: Kubernetes docs.
46. **Is `@PreDestroy` on every resource-holding bean, in an order that closes consumers before producers?** (§7.3, §7.4) — source: JSR-250; Nygard ch. 5.4 Steady State.

### J. Testing and diagnosis
47. **Are Hibernate SQL statistics and `show-sql` enabled in tests to catch N+1?** (§3.1) — source: Hibernate User Guide.
48. **Are load / chaos tests in place for circuit-breaker transitions?** (§4.3) — source: Nygard ch. 15.
49. **Is JFR / metrics visibility in place for pool-wait times and circuit-breaker state?** (§3.4, §4.3) — source: HikariCP wiki; Resilience4j docs.
50. **Are retries measured (count, reason, outcome) so a retry storm is detectable?** (§4.4) — source: Google SRE ch. 22.
51. **Are optimistic-lock exceptions logged with entity / version so lost-update risk is visible?** (§3.3) — source: JPA 3.1 §3.4.2.
52. **Does the test suite exercise the DST gap / overlap edge cases for scheduling code?** (§2.4) — source: `ZonedDateTime.of` Javadoc.

---

## Sources

### Normative / specification
- [RFC 9110 — HTTP Semantics (IETF, June 2022)](https://datatracker.ietf.org/doc/html/rfc9110) — §9.2 safe / idempotent methods; §10.2.3 `Retry-After`; §15 status code definitions
- [RFC 6585 — Additional HTTP Status Codes (IETF, 2012)](https://datatracker.ietf.org/doc/html/rfc6585) — §4 `429 Too Many Requests`
- [RFC 9457 — Problem Details for HTTP APIs (IETF, 2023)](https://datatracker.ietf.org/doc/html/rfc9457)
- [IETF draft-ietf-httpapi-idempotency-key-header](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/) — `Idempotency-Key` header, updated 2025-10-15
- [JLS §13 Binary Compatibility (Java SE 21)](https://docs.oracle.com/javase/specs/jls/se21/html/jls-13.html)
- [JPA 3.1 (Jakarta Persistence) Specification](https://jakarta.ee/specifications/persistence/3.1/)
- [`BigDecimal` Javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/math/BigDecimal.html)
- [`ThreadLocal` Javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ThreadLocal.html)
- [`System.nanoTime` Javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/System.html#nanoTime()) — "returns the current value of the running Java Virtual Machine's high-resolution time source"
- [`Files.lines` Javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/file/Files.html#lines(java.nio.file.Path)) — try-with-resources requirement
- [SEI CERT NUM00-J. Detect or prevent integer overflow](https://wiki.sei.cmu.edu/confluence/display/java/NUM00-J.+Detect+or+prevent+integer+overflow)
- [Oracle, Supplementary Characters in the Java Platform](https://www.oracle.com/technical-resources/articles/javase/supplementary.html)
- [Spring Boot reference — Graceful Shutdown](https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html)
- [Spring Framework reference — Declarative Transactions (Annotations)](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)

### JEPs
- [JEP 277 — Enhanced Deprecation (JDK 9)](https://openjdk.org/jeps/277)
- [JEP 506 — Scoped Values, Final (JDK 25)](https://openjdk.org/jeps/506)

### Foundational / authoritative
- **Michael T. Nygard, *Release It! Design and Deploy Production-Ready Software*, 2nd ed., Pragmatic Bookshelf 2018, ISBN 978-1-68050-239-8.** [Publisher page](https://pragprog.com/titles/mnee2/release-it-second-edition/). Core chapters for reliability review: ch. 4 Stability Antipatterns (Integration Points, Chain Reactions, Cascading Failures, Users, Blocked Threads, Self-Denial Attacks, Scaling Effects, Unbalanced Capacities, Slow Responses, SLA Inversion, Unbounded Result Sets); ch. 5 Stability Patterns (Timeouts, Circuit Breaker, Bulkheads, Steady State, Fail Fast, Handshaking, Test Harness, Decoupling Middleware, Shed Load, Create Backpressure, Governor).
- **Martin Kleppmann, *Designing Data-Intensive Applications*, 2nd ed., O'Reilly 2025, ISBN 978-1-098-11926-5.** Core chapters: ch. 2 — "Reliability" (fault tolerance vs. fault-free), "When an Overloaded System Won't Recover" sidebar on metastable failure; ch. 5 — Encoding and Evolution (Protobuf / Avro compatibility); ch. 9 — The Trouble with Distributed Systems (partial failures, unbounded delays, Phi Accrual).
- **Beyer, Jones, Petoff, Murphy (eds.), *Site Reliability Engineering: How Google Runs Production Systems*, O'Reilly 2016, ISBN 978-1-491-92912-4**, [free HTML edition](https://sre.google/sre-book/table-of-contents/) — especially ch. 21 *Handling Overload* and ch. 22 *Addressing Cascading Failures* (retry budgets, throttling, graceful degradation).
- Brian Goetz, Tim Peierls, Joshua Bloch, Joseph Bowbeer, David Holmes, Doug Lea, *Java Concurrency in Practice*, Addison-Wesley 2006 — ch. 7 Cancellation and Shutdown.

### Tooling
- [Resilience4j 2.4.0 documentation](https://resilience4j.readme.io/docs/getting-started) — CircuitBreaker, Retry, Bulkhead, TimeLimiter, RateLimiter, Decorators API
- [HikariCP wiki — About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing) — `((core_count * 2) + effective_spindle_count)` formula with SSD caveat
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html) — §6.3 Fetching, §12 Batching
- [Protobuf Language Guide (proto3) — Updating A Message Type](https://protobuf.dev/programming-guides/proto3/#updating)
- [Eclipse API Evolution Rules](https://wiki.eclipse.org/Evolving_Java-based_APIs_2)

### Checklists and curated reviews
- [SEI CERT Oracle Coding Standard for Java — NUM00-J. Detect or prevent integer overflow](https://wiki.sei.cmu.edu/confluence/display/java/NUM00-J.+Detect+or+prevent+integer+overflow)
- [code-review-checklists/java-concurrency (Leventov et al.)](https://github.com/code-review-checklists/java-concurrency) — complementary to this checklist; covers the concurrency facet that this one deliberately defers to `concurrency-reviewer`

### Industry case studies and explainers
- [AWS Architecture Blog — "Exponential Backoff And Jitter" (Marc Brooker, 2015)](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — Full / Equal / Decorrelated Jitter formulas
- [Stripe API docs — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests) — `Idempotency-Key` implementation reference
- [Vlad Mihalcea — The best way to use orphanRemoval in JPA and Hibernate](https://vladmihalcea.com/orphanremoval-jpa-hibernate/)
- [Collin Wilkins — JPA and Hibernate best practices](https://collinwilkins.com/articles/jpa)
- [Baeldung — N+1 Problem in Hibernate](https://www.baeldung.com/spring-hibernate-n1-problem)
- [Baeldung — JPA Optimistic Locking](https://www.baeldung.com/jpa-optimistic-locking)
- [Baeldung — Handling DST in Java](https://www.baeldung.com/java-daylight-savings)
- [Baeldung — Integer Overflow in Java](https://www.baeldung.com/java-overflow-underflow)
- [Baeldung — Java Memory Leaks](https://www.baeldung.com/java-memory-leaks)
- [Baeldung — CascadeType.REMOVE vs orphanRemoval](https://www.baeldung.com/jpa-cascade-remove-vs-orphanremoval)
- [Kubernetes docs — Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

---

## Gotchas — Common Wrong Assumptions

G-01. **"Spring Boot graceful shutdown must be enabled with `server.shutdown=graceful`"** — false since Spring Boot 2.3 (2020). `graceful` is the default on Tomcat, Jetty, and Reactor Netty; you only set it to `immediate` to disable. See §7.1.

G-02. **"`@Transactional` on a private method just logs a warning"** — false. It is silently ignored by the proxy and no transaction starts; the caller sees the work commit or roll back with the **outer** transaction (or with autocommit if none). See §3.2.

G-03. **"Self-invocation (`this.foo()`) on a `@Transactional` method still opens a transaction because it's the same object"** — false. Spring's proxy sits around the bean; `this.foo()` bypasses the proxy and so bypasses the advice. Inject self, call via `applicationContext.getBean(Self.class)`, or split into two beans. See §3.2.

G-04. **"`BigDecimal("1.0").equals(new BigDecimal("1.00"))` is true"** — false. `BigDecimal.equals` requires identical value AND scale. Use `compareTo(...) == 0`. `BigDecimal`'s natural ordering is explicitly inconsistent with `equals`. See §2.2.

G-05. **"`new BigDecimal(0.1)` is 0.1"** — false. It is `0.1000000000000000055511151231257827021181583404541015625`. Always pass a `String`. See §2.2.

G-06. **"I can use `System.currentTimeMillis()` to measure how long an operation took"** — false. The wall clock can step backwards (NTP, DST, admin edits). Use `System.nanoTime()` for durations. See §2.4.

G-07. **"`LocalDateTime.of(2024, 3, 31, 2, 30)` at `Europe/Berlin` is a valid instant"** — false. That local time does not exist (clocks skip 02:00→03:00). `ZonedDateTime.of(...)` silently adjusts to 03:30 unless you use the strict resolver. See §2.4.

G-08. **"Open Session In View is harmless — it just keeps the session alive"** — partial. OSIV lets controllers / views trigger lazy loads across HTTP, which is convenient for read paths but hides N+1 problems, extends connection-pool hold time, and breaks clean transaction boundaries for writes. Disable with `spring.jpa.open-in-view=false`. See §3.3.

G-09. **"An entity without `@Version` just uses last-write-wins, which is fine"** — partial. Last-write-wins silently **discards** concurrent updates. Without `@Version`, two users editing the same record concurrently lose one set of edits with no warning. See §3.3.

G-10. **"Retrying a `POST` on a network error is safe — the server probably didn't receive it"** — false. A single-commit write + lost ACK produces a double-create when the client retries. Use `Idempotency-Key` header + server-side dedup. See §4.5 and §5.3.

G-11. **"`int` arithmetic overflows, but it wraps, so my totals just look weird"** — false for critical paths. A silently negative balance is worse than an exception. Use `Math.addExact` / `multiplyExact`. See §2.1.

G-12. **"Exponential backoff fixes retry storms"** — false without **jitter**. Lockstep retries from 1 000 clients at `base*2^attempt` all wake up at the same instant. Use Full Jitter (`random(0, min(cap, base*2^n))`). See §4.4.

G-13. **"`429 Too Many Requests` is in RFC 9110"** — false. It is defined in RFC 6585 §4 (2012). RFC 9110 lists other status codes but not 429. See §5.1.

G-14. **"Idempotent methods are GET, PUT, and DELETE"** — incomplete. Per RFC 9110 §9.2.2 the idempotent set is GET, HEAD, OPTIONS, TRACE, PUT, DELETE. `POST` and `PATCH` are **not** idempotent. See §5.3.

G-15. **"`CascadeType.ALL` with `orphanRemoval=true` is the simplest way to keep the object graph tidy"** — false when children can be shared or re-parented. Moving a comment from one post to another will delete it; combining JPA `orphanRemoval` with DB `ON DELETE CASCADE` causes double-delete SQL errors. Use narrow cascade types. See §3.3.

G-16. **"HikariCP `maxLifetime` just controls how often connections are rotated — it doesn't matter what value I pick"** — false. If `maxLifetime` is **greater than or equal to** the database's `wait_timeout` / idle-kill threshold, borrowed connections will be silently closed server-side and the next statement throws. Keep `maxLifetime` strictly less than the DB value. See §3.4.
