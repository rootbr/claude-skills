# Java Maintainability Review Checklist

Concise reference for reviewing Java code for long-term maintainability. Focus on what makes code easy to understand, change, test, and debug in production.

---

## 1. Clean Code Fundamentals

### Functions
- **Small** — maximum 20 lines, better 5-10
- **One task** — function does exactly one thing
- **0-2 arguments** — ideal 0, acceptable 2, maximum 3
- **No flags** — boolean argument = make two functions
- **No side effects** — don't change what you don't promise in the name
- **Descriptive name** — function name fully explains what it does

### Names
- **Descriptive** — name explains everything without comments
- **Pronounceable** — not `genymdhms`, but `generationTimestamp`
- **No encodings** — not `strName`, `mValue`, `userObj`
- **No magic numbers** — only named constants
- **Searchable** — not `7` or `86400`, but `DAYS_IN_WEEK`, `SECONDS_PER_DAY`

### Conditions
- **Extract to variables** — turn complex conditions into understandable variables
- **Avoid negations** — `if (isValid)` instead of `if (!isNotValid)`
- **Reads like text** — `canDrive = isAdult && hasLicense`
- **No nesting** — maximum 2 levels of if nesting. Use guard clauses and early returns

### Comments
- **Write code that explains itself.** Comments only for:
  - **Why** (not what!) — explain business decision
  - **Warnings** — important gotchas
  - **TODO** — temporary notes with owner and date
- **Never**: comment out code (delete it), duplicate code in comments (`i++; // increment i`), leave commented-out code blocks

### Code Structure
- **Variables close to usage** — not at the beginning of function
- **Related code together** — separate groups with blank lines
- **Top to bottom** — public methods on top, private below
- **Lines < 120 characters**
- **One abstraction level** — don't mix high and low level in the same method
- **Dependent functions nearby** — called function right after calling one

---

## 2. Complexity & Metrics

### Cognitive Complexity
- **SonarQube threshold: 15 per method** (rule S3776). Exceeding this is a code smell
- **How it works**: +1 for each break in linear flow (`if`, `for`, `while`, `catch`, `switch`, `&&`/`||`); **+1 additional per nesting level**. A doubly-nested `if` inside a `for` costs +3, not +1
- **Differs from cyclomatic complexity**: cyclomatic counts execution paths (testing concern); cognitive counts *human reading difficulty*. A `switch` with 10 cases has cyclomatic 10 but cognitive ~1 because it is flat and readable. Nested `if-else` chains score much higher cognitively
- **Review action**: any method over 15 — extract helper methods, replace nested conditions with guard clauses, use polymorphism

### Coupling & Cohesion (Robert C. Martin Package Metrics)

| Metric | Formula | Range | Meaning |
|--------|---------|-------|---------|
| **Ca** (Afferent coupling) | count of external classes depending on this package | 0..∞ | High = high responsibility, must be stable |
| **Ce** (Efferent coupling) | count of external classes this package depends on | 0..∞ | High = high volatility risk |
| **Instability I** | Ce / (Ce + Ca) | 0..1 | 0 = maximally stable, 1 = maximally unstable |
| **Abstractness A** | abstract types / total types in package | 0..1 | 0 = fully concrete, 1 = fully abstract |
| **Distance D** | \|A + I - 1\| | 0..1 | Ideal = 0. **D > 0.3 = warning. D > 0.5 = danger** |

- **Zone of Pain**: I ≈ 0, A ≈ 0 — stable and concrete, hard to change (utility libraries with no abstractions)
- **Zone of Uselessness**: I ≈ 1, A ≈ 1 — abstract and unstable, nobody uses it

### What to look for
- **God classes**: >500 lines, >7 dependencies, mixed concerns. A `UserService` that handles auth, email, and persistence is 3 classes
- **PMD thresholds**: `CyclomaticComplexity > 10`, `CouplingBetweenObjects > 20`, `TooManyMethods > 20`
- **Class with few fields**: 2-5 instance variables is healthy. More signals broken SRP
- **High cohesion**: methods should use the class's fields. Methods that don't touch instance state belong elsewhere or should be static

### How to detect
- **SonarQube**: cognitive complexity, duplications, package tangle index
- **JDepend**: Ca/Ce/I/A/D metrics, cycle detection
- **ArchUnit**: `slices().matching("com.company.app.(*)..").should().beFreeOfCycles()`
- **PMD**: god class detection, coupling metrics

---

## 3. SOLID Principles — What Violations Look Like

### Single Responsibility (SRP)
- **Smell**: class has multiple groups of methods that never interact with each other's fields
- **Smell**: class changes for unrelated reasons (UI formatting + business rules + database access)
- **Fix**: extract each responsibility into its own class

### Open-Closed (OCP)
- **Smell**: `switch` on type or chains of `instanceof` that grow with each new feature
- **Smell**: modifying existing code to add new behavior rather than extending
- **Fix**: polymorphism, strategy pattern, sealed class hierarchies with pattern matching

### Liskov Substitution (LSP)
- **Smell**: subtypes that throw `UnsupportedOperationException` (e.g., `Collections.unmodifiableList` where mutation is expected)
- **Smell**: declaring `ArrayList` instead of `List` — leaks implementation
- **Smell**: explicit downcasts (`(SpecificType) abstractRef`) signal LSP breakage
- **Fix**: redesign inheritance hierarchy; use composition

### Interface Segregation (ISP)
- **Smell**: "fat" interface forcing implementors to stub unused methods
- **Smell**: Spring service interface with 15+ methods where each consumer uses 2-3
- **Fix**: split into role-specific interfaces (`Readable`, `Writable`, `Deletable`)

### Dependency Inversion (DIP)
- **Smell**: `new ConcreteService()` inside business logic
- **Smell**: code that cannot be tested without a live database or message broker
- **Fix**: depend on abstractions; inject via constructor

---

## 4. API Design & Contracts

### Immutability
- **Records** (Java 16+): use for value objects, DTOs — immutable by default, auto-generate `equals`/`hashCode`/`toString`. Eliminates entire bug classes
- **Defensive copies**: return `List.copyOf()` or `Collections.unmodifiableList()` from getters exposing internal collections. SpotBugs flags this as `EI_EXPOSE_REP`
- **Immutable collections**: prefer `List.of()`, `Map.of()`, `Set.of()` for constants and config

### Sealed Classes (Java 17+)
- Restrict inheritance to a known set of subtypes
- Enable exhaustive `switch` via pattern matching — compiler catches missing cases
- Prefer sealed hierarchies for result/error types over checked exceptions or raw Optional

```java
sealed interface PaymentResult permits Approved, Declined, PendingReview {}
record Approved(String transactionId) implements PaymentResult {}
record Declined(String reason) implements PaymentResult {}
record PendingReview(Instant deadline) implements PaymentResult {}

// Compiler enforces exhaustiveness
String describe(PaymentResult result) {
    return switch (result) {
        case Approved a -> "Approved: " + a.transactionId();
        case Declined d -> "Declined: " + d.reason();
        case PendingReview p -> "Review by: " + p.deadline();
    };
}
```

### Optional
- **Use only as return type** — never as field, parameter, or collection element
- **Never call `get()` without `isPresent()`** — prefer `orElseThrow()` with a meaningful exception
- **Never use `Optional.of(nullableValue)`** — use `Optional.ofNullable()`
- **Don't wrap primitives** — use `OptionalInt`, `OptionalLong`, `OptionalDouble`

### Null Safety
- **JSpecify annotations**: apply `@NullMarked` at the package level in `package-info.java` (non-null by default); mark exceptions with `@Nullable`. Spring Framework 7 has fully migrated to JSpecify
- **Enforce with NullAway** (Error Prone plugin) — NPE detection at compile time, <10% build overhead
- **Validate preconditions** at public API boundaries: `Objects.requireNonNull(param, "param must not be null")`
- **Prefer `IllegalArgumentException`** early over returning null deep in the call chain

---

## 5. Error Handling

### Exception Strategy
- **Prefer unchecked (`RuntimeException`)** for most cases; use checked exceptions only when the caller can realistically recover
- **Custom exception hierarchy**: base `ServiceException extends RuntimeException` with domain-specific subtypes. Always include meaningful message and error code. Pass original cause via `Throwable` constructor — never discard the stack trace
- **Catch specific, not generic**: catch `FileNotFoundException` before `IOException` before `Exception`. Flag `catch (Exception e)` as a smell

### Anti-Patterns — Flag Immediately

| Anti-Pattern | Problem | Fix |
|---|---|---|
| **Log and rethrow** | Duplicate log entries across layers | Either handle (log + recover) OR rethrow (wrap + propagate). Pick one boundary for logging (typically controller layer) |
| **Swallow exception** | Empty catch block hides failures | Always handle or rethrow. No empty catch blocks |
| **Exception for flow control** | `fillInStackTrace()` costs 5-50 μs | Use return codes, Optional, sealed result types |
| **Catch `Exception`/`Throwable`** | Catches `OutOfMemoryError`, `InterruptedException` | Catch the most specific type |
| **Generic `throw new RuntimeException("error")`** | Untyped, no context | Create domain exceptions with error codes |

### Exception Translation at Boundaries
- **Repositories** throw persistence exceptions
- **Services** translate to domain exceptions
- **Controllers** translate to HTTP responses
- Use Spring's `@ControllerAdvice` / `@ExceptionHandler` to centralize translation
- Always preserve the cause chain when wrapping

---

## 6. Testing & Testability

### Test Pyramid
- **Target ratio**: ~70% unit, ~20% integration, ~10% E2E
- **Mock at HTTP boundary** (WireMock), not at method level — catches serialization, headers, timeout, and retry bugs that Mockito cannot
- **Use Testcontainers** for databases, Kafka, Redis in integration tests — real behavior, disposable containers
- **Reserve mocking** for external services you do not own and cannot run locally. Never mock the thing you are testing

### Testability Design
- **Constructor injection only** — all dependencies visible in signature; no field injection, no `new` inside business logic
- **No static methods in business logic** — static calls cannot be substituted in tests
- **Separate I/O from logic** (Functional Core / Imperative Shell) — pure logic is trivially unit-testable without mocks
- **Ban `ServiceLocator`** — hides dependencies, breaks ISP, requires global state in tests

### Test Quality

| Indicator | Threshold/Rule |
|---|---|
| Mutation testing (PIT) | Target >80% mutation kill rate. Surviving mutants = assertions that never validate outcomes |
| Assertions per test | At least one meaningful assertion. No "test runs without exception" tests |
| Mock count | Max 1-2 mocks per test. More = testing mocks, not code |
| Test independence | No shared mutable state, no required execution order, no wall-clock dependency |
| Flaky tests | P1 bug. Quarantine immediately, fix within the sprint |

### Test Anti-Patterns — Ban List
- **Testing private methods** (via reflection or making public) — test through public API. If private is unreachable, class has too many responsibilities
- **Excessive mocking** — if mock setup exceeds assertions, you're testing the mocks
- **Testing implementation, not behavior** — verify returns/outputs, not which internal methods were called. `verify()` on every collaborator locks you into current structure
- **Test pollution** — shared static state, singletons mutated between tests, `@DirtiesContext` as a band-aid
- **Anal Probe** — reading private fields via reflection to assert internal state. Assert observable outputs only

### Test Maintainability
- **Test Data Builders**: `aUser().withName("Alice").withRole(ADMIN).build()` — readable, isolates from constructor changes
- **Custom AssertJ assertions**: `assertThat(order).isConfirmed().hasTotalAbove(100)` — reads like a spec
- **`@Nested` classes in JUnit 5** — group by method or scenario (max 2 levels). `@DisplayName` for readable output
- **AAA structure** — every test: Arrange, Act, Assert. No logic (if/else/loops) inside tests
- **Use AssertJ over bare JUnit assertions** — fluent, type-safe, clear failure messages

---

## 7. Documentation

### Javadoc Rules

| Element | Javadoc Required? | Notes |
|---|---|---|
| Public API classes/interfaces | **Yes** | Purpose, thread safety, usage example |
| Public methods | **Yes** | `@param`, `@return`, `@throws` for non-obvious behavior |
| Abstract / interface methods | **Yes** | Define the contract for implementors |
| Private methods | **No** | If you need to document a private method, it's too complex — extract or simplify |
| Getters / setters | **No** | Obvious from the name |
| Overridden methods | **No** | Unless behavior differs from parent |
| `package-info.java` | **Yes** for public packages | Describe package purpose and key abstractions |
| `module-info.java` | **Yes** for JPMS modules | Document exported packages and dependencies |

### Architecture Documentation
- **ADRs (Architecture Decision Records)**: record significant decisions in `docs/adr/` using the `NNNN-title.md` format (Status, Context, Decision, Consequences). Keep in the repo, not in wiki
- **README.md**: how to build, run, test. High-level architecture diagram (C4 Level 1-2). Link to ADRs for "why" questions
- **Code as the source of truth**: docs that duplicate code drift. Document intent and constraints, not implementation

### Test as Documentation
- **BDD naming**: `should_rejectPayment_when_cardExpired()` or `@DisplayName("rejects payment when card is expired")`
- **Given/When/Then structure**: makes test readable as a specification
- **One test = one behavior**: each test demonstrates one rule of the system

### Changelog & Versioning
- **Semantic Versioning**: MAJOR (breaking), MINOR (feature), PATCH (fix)
- **Conventional Commits**: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:` — enables automated CHANGELOG generation
- **CHANGELOG.md**: maintained per release. Sections: Added, Changed, Deprecated, Removed, Fixed, Security

---

## 8. Logging & Observability

### Structured Logging
- **JSON output** (Logback/Log4j2 JSON encoder) so logs are machine-parseable by ELK/Splunk/Loki
- **MDC for context**: put `requestId`, `userId`, `traceId` into MDC at entry filter; clear on exit. **MDC is thread-local — propagate explicitly to async threads / `CompletableFuture`**
- **Correlation IDs**: generate or accept `X-Correlation-ID` header; place in MDC so every log line is traceable across services
- **Parameterized logging**: `log.info("Order {} processed", orderId)` — never string concatenation

### Log Level Discipline

| Level | When to Use | Production Default |
|---|---|---|
| **ERROR** | Needs immediate human attention (pager-worthy) | Always on |
| **WARN** | Degraded but recoverable, self-healed | Always on |
| **INFO** | Significant business events: request served, job completed, deployment | Always on |
| **DEBUG** | Diagnostic detail for troubleshooting | Off in prod (enable per-class dynamically) |
| **TRACE** | Framework-level internals | Off in prod |

### What NOT to Log
- Passwords, tokens, API keys, session IDs
- PII (names, emails, SSNs, credit cards) — GDPR/HIPAA risk
- Full request/response bodies in production (data leakage + volume)
- Use masking/redaction patterns for any potentially sensitive data

### Observability Stack
- **Metrics (Micrometer)**: instrument custom business metrics (orders processed, cache hit ratio) with dimensional tags. **Avoid high-cardinality labels** (e.g., `userId` as tag destroys Prometheus)
- **Health checks**: Spring Boot Actuator `/health` with custom `HealthIndicator` for each critical dependency. Distinguish liveness vs. readiness probes for Kubernetes
- **Distributed tracing**: Micrometer Tracing + OpenTelemetry exporter. W3C Trace Context propagation is automatic in Spring Boot 3+. Ensure `traceId`/`spanId` appear in log lines via MDC
- **Sampling**: don't trace 100% of requests in production — use probability or rate-limiting sampler

---

## 9. Dependency Injection

### Constructor Injection (Preferred)
- Enables `final` fields (immutability)
- Makes dependencies explicit and visible
- Exposes circular dependencies at startup via `BeanCurrentlyInCreationException`

### Field Injection — Anti-Pattern
- Hides dependencies from callers
- Prevents `final` fields
- Couples class to Spring container
- Silently masks circular dependencies (injected lazily)

### Circular Dependencies
- **Banned by default since Spring Boot 2.6**
- If detected: refactor — extract shared logic into a third bean, use events, or redesign
- **Never re-enable `spring.main.allow-circular-references=true`** as a permanent fix

---

## 10. Project Structure & Dependencies

### Package Design
- **Package by feature > by layer**: higher cohesion, allows package-private visibility, reduces cross-package coupling. Package-by-layer forces nearly everything `public`
- **Acyclic Dependencies Principle (ADP)**: package dependency graph must be a DAG. Cycles cause unpredictable ripple effects. Fix by introducing interfaces or extracting a new package
- **Stable Dependencies Principle (SDP)**: depend in the direction of stability. UI packages must not be depended upon by domain packages
- **Stable Abstractions Principle (SAP)**: stable packages should be abstract so they can be extended without modification

### Architecture Tests (ArchUnit)
```java
// Enforce no cycles between packages
slices().matching("com.company.app.(*)..").should().beFreeOfCycles()

// Enforce no field injection
noClasses().should().beAnnotatedWith("org.springframework.beans.factory.annotation.Autowired")
    .because("Use constructor injection")

// Enforce layer boundaries
noClasses().that().resideInAPackage("..domain..")
    .should().dependOnClassesThat().resideInAPackage("..controller..")
```

### Dependency Management
- **Transitive dependency hell**: Maven uses "nearest-wins" (depth-first), Gradle uses "highest-version-wins". Both can silently resolve incompatible versions
- **BOM (Bill of Materials)**: use `<dependencyManagement>` with `<scope>import</scope>` to lock consistent versions (Spring Boot BOM, Jackson BOM). BOMs don't prevent overrides — enforce with `maven-enforcer-plugin`
- **CI checks**: run `mvn dependency:tree` / `gradle dependencies` in CI; fail build on convergence errors (`requireUpperBoundDeps` in Gradle, `dependencyConvergence` enforcer rule in Maven)
- **Shading risks**: increases JAR size, breaks source navigation/debugging, hides CVEs from scanners. Use only as last resort for genuine classpath conflicts

### JPMS (Java Platform Module System)
- For libraries: define explicit `exports` and `requires` in `module-info.java` to enforce encapsulation at the module level
- Internal packages stay hidden by default — stronger than package-private

---

## 11. Configuration Management

### Externalized Config
- No hardcoded URLs, credentials, or feature switches in code
- Use `application.yml` + environment-specific profiles (`application-prod.yml`)
- **Typed config with validation**: `@ConfigurationProperties` with `@Validated` and JSR-303 annotations (`@NotBlank`, `@Min`). App fails fast at startup if config is invalid

### Secrets
- **Never in `application.yml` or git**
- Use Spring Cloud Vault, AWS Secrets Manager, Kubernetes Secrets (mounted as files via `spring.config.import=configtree:`), or environment variables
- Review for any plaintext secrets in config files — this is a P0 finding

### Feature Flags
- Use a dedicated system (Unleash, LaunchDarkly, FF4J) rather than config properties for runtime toggles
- Allows enable/disable without redeployment
- Ensure flags have an **owner** and **expiry date** — stale flags become technical debt

### Fail-Fast on Missing Config
- Required properties should cause startup failure with a clear error, not a cryptic NPE at runtime
- `@Value` with no default on a missing property fails at injection time — this is correct behavior, don't suppress it

---

## 12. Static Analysis Toolkit

| Tool | Key Rules | What They Catch |
|---|---|---|
| **SonarQube** | S3776 (cognitive complexity > 15), S1192 (string literals duplicated 3+), S1066 (collapsible if), S2259 (null dereference) | Readability, duplication, bugs |
| **Error Prone** | `NullAway` (NPE at compile time, <10% overhead), `MissingOverride`, `ImmutableEnumChecker`, `ReturnValueIgnored` | Correctness, thread safety |
| **SpotBugs** | `EI_EXPOSE_REP` (mutable field exposure), `NP_NULL_ON_SOME_PATH`, `SE_BAD_FIELD` (non-serializable field) | Encapsulation, null safety |
| **PMD** | `GodClass`, `CyclomaticComplexity > 10`, `CouplingBetweenObjects > 20`, `TooManyMethods > 20` | Design smells, coupling |
| **ArchUnit** | Cycle detection, layer enforcement, DI pattern enforcement | Architecture violations |
| **PIT** (mutation testing) | Surviving mutants = weak assertions | Test quality |

### Pragmatic Setup
- **Build phase**: Error Prone + SpotBugs (fast feedback, fail on violations)
- **CI phase**: SonarQube quality gate: 0 new bugs, 0 new vulnerabilities, cognitive complexity < 15, duplications < 3%
- **Weekly/sprint**: PIT mutation testing on critical modules

---

## 13. Quick Review Heuristic

When reviewing Java code for maintainability, scan in this order:

1. **Can I understand this method in 30 seconds?** If not — too complex (check cognitive complexity, extract methods, add guard clauses)
2. **Names tell the story?** Methods, variables, classes — do they explain *what* and *why* without reading the body?
3. **Functions small and focused?** < 20 lines, one responsibility, ≤ 2 arguments?
4. **No duplication?** Same logic in multiple places = extract a method or class
5. **Single responsibility?** Does each class have one reason to change? God class symptoms?
6. **Dependencies explicit?** Constructor injection, no `new` in business logic, no `ServiceLocator`?
7. **Errors handled at the right layer?** No log-and-rethrow, no swallowed exceptions, no `catch (Exception e)`?
8. **Tests exist and test behavior?** Not implementation, not private methods, meaningful assertions?
9. **Contracts documented?** Public API has Javadoc, nullability annotated, preconditions validated?
10. **Logging useful for debugging?** Structured, correlation IDs, appropriate levels, no PII leakage?

---

## Checklist Before Commit

- [ ] Function < 20 lines?
- [ ] Function does ONE thing?
- [ ] Arguments ≤ 2?
- [ ] Names explain everything?
- [ ] No magic numbers?
- [ ] Complex conditions extracted to variables?
- [ ] No duplication?
- [ ] Can it be simpler?
- [ ] Code reads like prose?
- [ ] No commented-out code?
- [ ] Exceptions handled at the right layer?
- [ ] Null safety annotations present on public API?
- [ ] Tests cover behavior, not implementation?
- [ ] Javadoc on public API?
- [ ] Logging is structured with correlation IDs?

---

## Golden Rule

> **"Any fool can write code that a computer can understand.
> Good programmers write code that humans can understand."**
> — Martin Fowler

**If in doubt — choose the simpler option.**

---

Sources:
- Clean Code — Robert C. Martin
- Effective Java — Joshua Bloch
- [SonarQube Cognitive Complexity](https://www.sonarsource.com/blog/5-clean-code-tips-for-reducing-cognitive-complexity)
- [Baeldung: Cognitive Complexity](https://www.baeldung.com/java-cognitive-complexity)
- [Robert C. Martin Package Metrics](https://en.wikipedia.org/wiki/Software_package_metrics)
- [JSpecify and NullAway — Spring Blog](https://spring.io/blog/2025/03/10/null-safety-in-spring-apps-with-jspecify-and-null-away/)
- [Error Prone Bug Patterns](https://errorprone.info/bugpatterns)
- [NullAway — Uber](https://github.com/uber/nullaway)
- [SpotBugs Bug Descriptions](https://spotbugs.readthedocs.io/en/stable/bugDescriptions.html)
- [ArchUnit User Guide](https://www.archunit.org/userguide/html/000_Index.html)
- [Victor Rentea: Exception Handling in Java](https://victorrentea.ro/blog/exception-handling-guide-in-java/)
- [Philipp Hauer: Modern Testing in Java](https://phauer.com/2019/modern-best-practices-testing-java/)
- [Software Testing Anti-Patterns — Codepipes](https://blog.codepipes.com/testing/software-testing-antipatterns.html)
- [Baeldung: Circular Dependencies in Spring](https://www.baeldung.com/circular-dependencies-in-spring)
- [Baeldung: Field Injection Cons](https://www.baeldung.com/java-spring-field-injection-cons)
- [Package by Feature vs Layer](https://medium.com/sahibinden-technology/package-by-layer-vs-package-by-feature-7e89cde2ae3a)
- [Maven BOM Usage — Reflectoring](https://reflectoring.io/maven-bom/)
- [Gradle: Five Ways Out of Dependency Hell](https://gradle.com/blog/five-ways-dependency-hell-maven/)
