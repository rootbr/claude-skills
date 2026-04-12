# Java Reliability Review Checklist

Concise reference for reviewing Java code for correctness, resource management, fault tolerance, data integrity, and backward compatibility. Focus on subtle bugs that cause production incidents.

---

## 1. Resource Management

### Try-With-Resources
- Every `AutoCloseable` (streams, connections, statements, channels, readers, writers) **must** use try-with-resources
- Nested resources: declare in order so dependents close first (outer resource wrapping inner)
- `Stream` from `Files.lines()`, `BufferedReader.lines()`, JPA streaming queries **must** be closed — they hold underlying I/O resources

```java
// BROKEN: stream not closed, file handle leak
Files.lines(path).filter(line -> line.contains("ERROR")).forEach(System.out::println);

// CORRECT
try (Stream<String> lines = Files.lines(path)) {
    lines.filter(line -> line.contains("ERROR")).forEach(System.out::println);
}
```

### Connection Pool Leaks
- JDBC: every `Connection`, `Statement`, `ResultSet` in try-with-resources. A single leaked connection under load exhausts the pool → application freeze
- Enable HikariCP leak detection: `spring.datasource.hikari.leak-detection-threshold=5000` (ms) — logs stack trace of the code that borrowed the connection
- HTTP clients: close `Response` / `InputStream` from HTTP calls. Apache HttpClient `EntityUtils.consume(response.getEntity())` or use try-with-resources on `CloseableHttpResponse`

### ThreadLocal Cleanup
- `ThreadLocal.remove()` **must** be called in a `finally` block, especially in thread pools (servlet filters, `@Async`, `CompletableFuture`)
- `ThreadLocal.set(null)` is **not** equivalent to `remove()` — the entry stays in the thread's map, leaking the `ThreadLocal` key
- Pattern: set in filter/interceptor `preHandle`, remove in `afterCompletion`/`finally`

```java
// CORRECT pattern in a servlet filter
try {
    MDC.put("requestId", requestId);
    chain.doFilter(request, response);
} finally {
    MDC.clear();  // MDC uses ThreadLocal internally
}
```

### Other Resource Leaks
- **Event listeners / observers**: deregister on shutdown (`@PreDestroy`, `contextDestroyed`). Especially critical for `ApplicationListener` in child contexts
- **JDBC drivers in web apps**: explicitly deregister in `contextDestroyed` to prevent classloader leaks
- **Direct ByteBuffer**: lives off-heap, freed only by GC + Cleaner. Track with `-XX:MaxDirectMemorySize`. Pool and reuse, don't allocate per-request
- **ExecutorService**: `shutdown()` / `shutdownNow()` on app teardown. Register shutdown hook or `@PreDestroy`. Unshut thread pools keep the JVM alive
- **Classloader leaks**: caused by static references to classloader-scoped objects (JDBC drivers, thread-local values, shutdown hooks registered but not removed)

---

## 2. Data Integrity & Correctness

### Integer Overflow
- Use `Math.addExact()`, `Math.subtractExact()`, `Math.multiplyExact()` for arithmetic that must not silently overflow — throws `ArithmeticException` on overflow
- Array size calculations: `int newCapacity = oldCapacity + (oldCapacity >> 1)` can overflow silently → negative array size → `NegativeArraySizeException` or wrap-around
- `(int)(longValue)` silently truncates — check bounds first or use `Math.toIntExact()`

### Floating-Point
- **Never** use `double`/`float` for money — use `BigDecimal` initialized from `String` (`new BigDecimal("0.1")`), never from `double` (`new BigDecimal(0.1)` = 0.1000000000000000055511...)
- **Always** specify `RoundingMode` with `BigDecimal.divide()` — unspecified rounding throws `ArithmeticException` for non-terminating decimals
- Compare with `BigDecimal.compareTo()` (ignores scale), not `equals()` (`new BigDecimal("1.0").equals(new BigDecimal("1.00"))` returns `false`)
- For scientific/engineering: compare with epsilon tolerance: `Math.abs(a - b) < 1e-9`

### Boundary Conditions
- Check `isEmpty()` before `get(0)`, `iterator().next()`, or `stream().findFirst().get()`
- Off-by-one: `for (int i = 0; i <= list.size(); i++)` — one past the end. `substring(0, length - 1)` — missing last char
- Empty strings vs null: decide on convention and be consistent. Use `StringUtils.isBlank()` to handle both
- `Collections.singletonList()` and `List.of()` are immutable — `add()` throws `UnsupportedOperationException`

### Date & Time
- **Store and transmit as UTC**: use `Instant` for timestamps. Convert to `ZonedDateTime` only at display boundaries
- Use IANA zone IDs (`Europe/Berlin`, `America/New_York`), never fixed offsets (`+01:00` — doesn't handle DST)
- **DST gaps**: `LocalDateTime.of(2024, 3, 31, 2, 30)` doesn't exist in `Europe/Berlin` (clocks skip 2:00→3:00). `ZonedDateTime.of()` silently adjusts — verify this is intended
- **DST overlaps**: `LocalDateTime.of(2024, 10, 27, 2, 30)` is ambiguous in `Europe/Berlin` (occurs twice). Use `ZonedDateTime.ofStrict()` if precision matters
- **Never** use `java.util.Date`, `Calendar`, or `SimpleDateFormat` in new code — use `java.time.*`
- `LocalDate.parse("2024-02-29")` is valid (leap year), `LocalDate.parse("2025-02-29")` throws — validate dates from external input

### Character Encoding
- Use `String.codePointCount()` / `codePoints()` for strings with emoji, CJK extensions, or other supplementary Unicode characters. `String.length()` counts UTF-16 code units, not characters
- `charAt(i)` returns a `char` (16-bit) — can return half a surrogate pair for characters outside BMP
- Specify charset explicitly: `new String(bytes, StandardCharsets.UTF_8)`. Platform-default charset varies by OS/locale

### Null Safety
- `Optional` for method return types, `Objects.requireNonNull()` for constructor/method parameters
- `Map.get(key)` returns `null` for missing keys — use `getOrDefault()`, `computeIfAbsent()`, or check with `containsKey()`
- `Map.of()` / `List.of()` throw `NullPointerException` on null elements — by design, but surprising if migrating from `HashMap`/`ArrayList`
- `equals()` contract: `null.equals(x)` throws NPE. Always call `equals()` on the known non-null side: `"constant".equals(variable)`

---

## 3. Database & Persistence

### N+1 Query Problem
- **Symptom**: loading a parent entity triggers N separate SELECTs for child entities (one per parent)
- **Detection**: enable Hibernate statistics in tests (`spring.jpa.properties.hibernate.generate_statistics=true`) or use `spring.jpa.show-sql=true`
- **Fix**: `JOIN FETCH` in JPQL, `@EntityGraph`, `@BatchSize(size = 100)`, or DTO projection (most efficient)

### Transactions
- `@Transactional(readOnly = true)` for read-only operations — skips dirty checking and flush, enables replica routing
- `@Transactional` propagation:
  - `REQUIRED` (default): joins existing or creates new
  - `REQUIRES_NEW`: suspends outer transaction — verify this is intentional, it can cause consistency issues
  - **Private methods**: `@Transactional` on private methods does nothing (Spring proxy bypass). Also applies to self-invocation (`this.method()`)
- **Long transactions**: holding a transaction open during HTTP calls, file I/O, or user interaction locks database rows and exhausts connection pool
- Transaction boundary should be at the service layer, not the controller

### JPA / Hibernate Pitfalls

| Pitfall | Problem | Fix |
|---|---|---|
| Lazy loading outside session | `LazyInitializationException` or silent null | Use DTO projection, `JOIN FETCH`, or `@EntityGraph`. **Disable OSIV** (`spring.jpa.open-in-view=false`) |
| `equals()`/`hashCode()` on `@Id` | Unstable before `persist()`, breaks `Set` membership | Implement on natural/business key |
| Missing `@Version` | Lost updates under concurrent writes | Add `@Version` for optimistic locking, handle `OptimisticLockException` with retry |
| `CascadeType.ALL` + `orphanRemoval` | Deleting parent cascades to all children — may delete too much | Use specific cascade types (`PERSIST`, `MERGE`), review delete behavior |
| `FetchType.EAGER` on `@ManyToOne` | Loads entire object graph on every query | Use `FetchType.LAZY` (default for collections, not for `@ManyToOne`) |
| `save()` inside loop | N individual INSERTs instead of batch | Use `saveAll()` with `spring.jpa.properties.hibernate.jdbc.batch_size=50` |

### Connection Pool
- HikariCP formula: `connections = (core_count * 2) + effective_spindle_count`
- Monitor pool wait time — if consistently > 0, pool is too small or connections leak
- Set `connectionTimeout` (how long to wait for a connection from pool, default 30s) and `maxLifetime` (connection max age, default 30min — set slightly less than DB's `wait_timeout`)

---

## 4. Resilience & Fault Tolerance

### Circuit Breaker (Resilience4j)
- Wrap every remote call (HTTP, gRPC, database, message broker) with a circuit breaker
- Configure: `failureRateThreshold` (default 50%), `waitDurationInOpenState` (default 60s), `slidingWindowSize` (default 100 calls)
- **Half-open state**: allows a configurable number of probe calls to test recovery — don't set too high or recovery attempt floods a struggling service

### Retry
- Use exponential backoff with jitter: base delay × 2^attempt + random(0, base_delay)
- **Never** retry non-idempotent operations without an idempotency key
- Set a **maximum number of retries** (typically 3) and a **total timeout** encompassing all retries
- Retry only on transient errors (network timeout, 503, 429) — never on 4xx client errors (except 429)

### Timeouts
- **Every** remote call must have explicit connect and read timeouts. No infinite waits
- `RestTemplate` / `WebClient` / `HttpClient`: set `connectTimeout` (1-5s typical) and `readTimeout` (5-30s typical, depends on operation)
- Database query timeout: `@QueryHints(@QueryHint(name = "jakarta.persistence.query.timeout", value = "5000"))`
- Set a `TimeLimiter` wrapping the entire operation (including retries)

### Bulkhead
- Apply semaphore or thread-pool isolation so one slow downstream cannot exhaust the caller's thread pool
- Example: separate thread pools for "payment service" and "notification service" — payment timeout doesn't block notifications

### Fallback
- Every resilience-decorated call **must** have a fallback: cached value, degraded response, or meaningful error
- Never an empty catch block or silent swallow as a "fallback"

### Decorator Composition Order
```
Retry(CircuitBreaker(TimeLimiter(Bulkhead(actualCall))))
```
- Bulkhead limits concurrent calls
- TimeLimiter aborts if too slow
- CircuitBreaker tracks failures
- Retry retries the whole chain on transient failure

---

## 5. REST API Correctness

### HTTP Status Codes

| Operation | Success | Failure |
|---|---|---|
| GET (found) | `200 OK` | `404 Not Found` |
| POST (create) | `201 Created` + `Location` header | `409 Conflict` (duplicate), `422 Unprocessable Entity` (validation) |
| PUT (replace) | `200 OK` or `204 No Content` | `404 Not Found`, `409 Conflict` |
| PATCH (partial) | `200 OK` | `404`, `422` |
| DELETE | `204 No Content` | `404 Not Found` (or `204` — idempotent) |
| Rate limited | — | `429 Too Many Requests` + `Retry-After` header |
| Server error | — | `500` (unexpected), `503` (temporarily unavailable) + `Retry-After` |

### Idempotency
- **GET, PUT, DELETE**: must be idempotent by definition (safe to retry)
- **POST**: not idempotent by default. For critical operations (payments, order creation): accept `Idempotency-Key` header, store key with TTL (24-48h), return cached response on duplicate
- Idempotency implementation: store `(key, response, status)` in DB, check before processing

### Pagination
- **Cursor-based** (preferred for mutable data): `GET /items?cursor=abc&limit=20`. Return `nextCursor` and `hasMore`
- **Offset-based** (simpler, but skips/duplicates items if data changes): `GET /items?page=2&size=20`. Return `totalElements`, `totalPages`
- Always return pagination metadata in response body, not just headers
- Set a maximum page size (e.g., 100) — never allow clients to request unbounded results

### Request Validation
- Validate all input at the boundary: `@Valid` on request bodies, `@NotNull`, `@Size`, `@Pattern` on parameters
- Return structured error responses with stable error codes:
  ```json
  {"code": "VALIDATION_ERROR", "message": "...", "details": [{"field": "email", "message": "..."}]}
  ```
- Never expose internal details (stack traces, SQL, file paths) in error responses. Configure `server.error.include-stacktrace=never`

### API Versioning
- URL path versioning: `/api/v1/users` (simplest, most common)
- Accept header versioning: `Accept: application/vnd.company.v1+json` (REST-purist)
- Never break existing clients without a deprecation period (minimum one major release cycle)

---

## 6. Backward Compatibility

### Types of Compatibility

| Type | Definition | Example of Break |
|---|---|---|
| **Binary** | Existing compiled code still links | Removing a public method, changing method signature |
| **Source** | Existing source still compiles | Adding abstract method to interface (without `default`) |
| **Behavioral** | Existing code still works correctly | Changing sort order, changing default values, adding validation |

### Safe Changes (non-breaking)
- Adding new methods with `default` implementation to interfaces
- Adding new classes, new packages
- Adding new optional parameters (with default values)
- Widening access (protected → public)

### Unsafe Changes (breaking)
- Removing or renaming public/protected methods, classes, fields
- Changing method signatures (parameter types, return types)
- Adding abstract methods to public interfaces
- Narrowing access (public → protected/private)
- Changing exception types (widening catch clauses doesn't help callers with specific catches)

### Serialization Compatibility
- Declare explicit `serialVersionUID` on all `Serializable` classes. Change it **only** when you intentionally want to break deserialization of old instances
- JSON API evolution: only add fields (additive). Never rename or remove fields without versioning the endpoint
- Protobuf / Avro: **never** reuse or change field numbers/tags. Only add new optional fields. Mark removed fields as `reserved`

### Deprecation Protocol
1. Add `@Deprecated(since = "2.3", forRemoval = true)` with Javadoc specifying the replacement
2. Keep the old method/endpoint working for at least one major release cycle
3. Add a bridge/adapter that delegates to the new implementation
4. Log a warning when deprecated API is called (with migration instructions)
5. Remove in the next major version

---

## 7. Graceful Shutdown

### Spring Boot
- `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s`
- In-flight requests complete before shutdown. New requests receive `503 Service Unavailable`
- Kubernetes: set `terminationGracePeriodSeconds` >= Spring's shutdown timeout + buffer (e.g., 45s)
- `preStop` hook: `sleep 5` to allow load balancer to drain traffic before the app starts rejecting

### Cleanup Order
1. Stop accepting new requests (readiness probe goes unhealthy)
2. Complete in-flight requests (within timeout)
3. Stop scheduled tasks (`@Scheduled`, Quartz)
4. Close message consumers (Kafka, RabbitMQ)
5. Flush pending writes (logs, metrics, traces)
6. Close connection pools (database, HTTP clients)
7. Deregister from service discovery

### What to Review
- `@PreDestroy` methods exist for all resources that need cleanup
- Shutdown hooks don't have circular dependencies
- Background threads check for shutdown signal (`Thread.interrupted()`, `volatile boolean`)
- Pending Kafka/JMS messages are acknowledged or rejected, not silently lost

---

## 8. Quick Reliability Review Heuristic

When reviewing Java code for reliability, scan in this order:

1. **Resources closed?** Every `AutoCloseable` in try-with-resources? Connections returned to pool?
2. **Null handled?** `Optional` returns, `Objects.requireNonNull` at boundaries, no blind `.get()` or `iterator().next()`?
3. **Overflow checked?** `Math.addExact` for critical arithmetic? Array size calculations safe?
4. **Money as BigDecimal?** Never `double` for financial calculations?
5. **Dates in UTC?** `Instant` for storage, `ZonedDateTime` at display, IANA zone IDs?
6. **Transactions correct?** Right propagation, not on private methods, not holding during I/O?
7. **N+1 queries?** Lazy collections loaded efficiently? OSIV disabled?
8. **Timeouts set?** Every remote call has connect + read timeout? No infinite waits?
9. **Retries safe?** Only for idempotent operations or with idempotency key? Backoff with jitter?
10. **Graceful shutdown?** Resources cleaned up in `@PreDestroy`? Background threads check shutdown signal?

---

Sources:
- [Baeldung: Java Memory Leaks](https://www.baeldung.com/java-memory-leaks)
- [Baeldung: N+1 Problem in Hibernate](https://www.baeldung.com/spring-hibernate-n1-problem)
- [Baeldung: JPA Optimistic Locking](https://www.baeldung.com/jpa-optimistic-locking)
- [Baeldung: Integer Overflow in Java](https://www.baeldung.com/java-overflow-underflow)
- [Baeldung: Handling DST in Java](https://www.baeldung.com/java-daylight-savings)
- [Baeldung: Circular Dependencies in Spring](https://www.baeldung.com/circular-dependencies-in-spring)
- [SEI CERT: Detect or Prevent Integer Overflow](https://wiki.sei.cmu.edu/confluence/display/java/NUM00-J.+Detect+or+prevent+integer+overflow)
- [Oracle: Supplementary Characters in Java](https://www.oracle.com/technical-resources/articles/javase/supplementary.html)
- [Resilience4j Circuit Breaker Tutorial](https://mobisoftinfotech.com/resources/blog/microservices/resilience4j-circuit-breaker-retry-bulkhead-spring-boot)
- [Building Resilient Systems: Circuit Breakers and Retry Patterns](https://dasroot.net/posts/2026/01/building-resilient-systems-circuit-breakers-retry-patterns/)
- [REST API Best Practices — Postman](https://blog.postman.com/rest-api-best-practices/)
- [REST API Design: Idempotency, Pagination, Security — TechOps Asia](https://techopsasia.com/blog/rest-api-design-idempotency-pagination-security)
- [Protobuf Schema Evolution Guide](https://jsontotable.org/blog/protobuf/protobuf-schema-evolution)
- [Maintaining Backward Compatibility in Java — Medium](https://medium.com/@deshpandeshreenidhi244/maintaining-backward-compatibility-in-java-best-practices-every-developer-should-know-9f4ccd7dbd94)
- [JPA and Hibernate Best Practices — Collin Wilkins](https://collinwilkins.com/articles/jpa)
- [HikariCP Connection Pool Sizing — GitHub Wiki](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [Spring Boot Graceful Shutdown](https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html)
