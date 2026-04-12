# Invariants in Java Codebases

> **Key insight:** Finding and documenting invariants is the single highest-ROI activity for AI code review. AI enforces what's documented, misses what's not.

---

## What Is an Invariant?

A property that **must always hold true** at defined program points — after construction, before and after every public method, across threads. When an invariant breaks silently, the bug surfaces far from the cause.

---

## Taxonomy of Invariants

### 1. Data / Representation Invariants

Rules about what constitutes a valid internal state of an object.

| Example | Domain | Invariant |
|---------|--------|-----------|
| `RatNum` | Math library | `denom > 0` and `gcd(|numer|, denom) == 1` — always reduced form |
| `CharSet` (sorted rep) | Collections | `s[0] <= s[1] <= ... <= s[n-1]` — backing string is sorted |
| `Tweet` | Social media | `author` matches `[A-Za-z0-9_]+`, `text.length <= 140` |
| `Money` | Financial | Amount is never `null`; currency code is ISO 4217; scale matches currency (e.g. 2 for USD) |
| `Email` | Any domain | Must match RFC 5322; local part + domain never empty after parsing |

**How they break:** Constructor allows invalid state; mutator forgets to re-establish the invariant; deserialization bypasses constructor.

**Enforcement:**
```java
private void checkRep() {
    assert denom > 0;
    assert gcd(Math.abs(numer), denom) == 1;
}
```
Call `checkRep()` at end of every constructor and every public mutator. Assertions are disabled by default; enable with `-ea` during development and testing.

---

### 2. Concurrency Invariants

Rules that prevent data races, visibility bugs, and atomicity violations.

**Common patterns and their violations:**

| Invariant | Violation | Fix |
|-----------|-----------|-----|
| Compound operations on `ConcurrentHashMap` must be atomic | `if (!map.containsKey(k)) map.put(k, v)` — race between check and put | `map.computeIfAbsent(k, key -> v)` |
| Related fields must update atomically | Separate getters for `balance` and `lastUpdated` — reader sees inconsistent pair | Wrap in immutable POJO, swap via `AtomicReference` |
| Non-thread-safe collections must not be read during concurrent writes | `HashMap.get()` while another thread calls `put()` — infinite loop or NPE in JDK 7, corruption in JDK 8+ | Use `ConcurrentHashMap` or external synchronization |
| `long`/`double` writes are non-atomic on 32-bit JVMs | Concurrent read of a `long` field sees torn value (high 32 bits from one write, low 32 from another) | Mark field `volatile` or guard with `synchronized` |
| `DateFormat.parse()`/`format()` mutate internal state | Shared `SimpleDateFormat` across threads — garbled output | Use `DateTimeFormatter` (immutable) or thread-local instances |
| Static mutable state requires synchronization | `static HashMap<String, Config> cache` accessed from servlet threads | `static ConcurrentHashMap<>` or `ClassValue<>` |

**Documentation invariants:**
- Every `synchronized` method/block needs a comment explaining *what* it protects and *why*
- Every `volatile` field needs a comment explaining the happens-before relationship it establishes
- Every field accessed by multiple threads needs `@GuardedBy("lockName")` annotation
- Every class with thread-safety requirements needs `@ThreadSafe`, `@Immutable`, or `@NotThreadSafe`

**Source:** [java-concurrency code review checklist](https://github.com/code-review-checklists/java-concurrency) — 100+ items organized by category.

---

### 3. Structural Invariants

Rules about data structure shape that guarantee algorithmic properties.

| Data Structure | Invariant | Guarantees |
|---------------|-----------|------------|
| Binary Search Tree | For node `n`: all left descendants `< n.key`, all right descendants `> n.key` | O(n) worst-case lookup |
| Red-Black Tree | (1) Every node is red or black. (2) Root is black. (3) No red node has a red parent. (4) Every root-to-leaf path has equal black-height | O(log n) lookup, insert, delete |
| Min-Heap | Parent `<=` both children; tree is complete (filled left-to-right) | O(1) min, O(log n) insert/extract |
| B-Tree (order m) | Every non-root node has `⌈m/2⌉` to `m` children; all leaves at same depth | O(log n) disk I/O operations |
| HashMap (JDK 8+) | Bucket is linked list when `size < 8`, red-black tree when `>= 8`, back to list when `<= 6` | Worst-case O(log n) per bucket instead of O(n) |

**How they break:** Incorrect rotation in tree rebalancing; comparator inconsistent with `equals()`; hash function changes after insertion.

---

### 4. Framework Invariants

Rules imposed by the framework that the compiler does not enforce.

#### Spring Framework

| Invariant | Violation | Consequence |
|-----------|-----------|-------------|
| `@Transactional` only works through proxy — self-invocation bypasses it | Calling `this.saveA(a)` from another method in the same bean | Transaction never starts; data may be partially written |
| `@Transactional` only works on `public` methods | Annotating `private` or `protected` method | Annotation silently ignored |
| Default rollback is only for unchecked exceptions | `catch (IOException e) { throw e; }` inside `@Transactional` | Transaction commits despite the error |
| Persistent entities must never be used as controller method parameters | `public String save(User user)` in `@Controller` | Mass assignment vulnerability — attacker sets `user.role = ADMIN` |
| `@ComponentScan` on default package scans entire classpath | Class in root package without explicit `basePackages` | Startup takes minutes; picks up unrelated beans |
| `@Autowired` on configuration fields forces eager instantiation | `@Autowired DataSource ds` in `@Configuration` class | Bean created before context is fully initialized |

**Source:** [Spring Framework Pitfalls — SonarSource](https://www.sonarsource.com/blog/spring-framework-pitfalls/)

#### JPA / Hibernate

| Invariant | Violation | Consequence |
|-----------|-----------|-------------|
| Changes to detached entities are NOT auto-persisted | Modify entity after transaction ends, expect DB update | Silent data loss |
| Cascade must be configured explicitly | Persist parent without `cascade = PERSIST` on child | `TransientObjectException` |
| `equals()`/`hashCode()` must use business key, not generated ID | Use `@Id` in `equals()` — different for transient vs. managed | Entity lost in `Set` after merge |
| `merge()` returns a *new* managed instance | Continue using the passed-in (still detached) instance | Changes on detached copy are ignored |
| Version/identity fields must not be altered on detached entities | Manual `setId()` or `setVersion()` before merge | `OptimisticLockException` or silent overwrite |

#### Servlet / Jakarta EE

| Invariant | Violation |
|-----------|-----------|
| Servlet instances are shared across requests — no mutable instance fields | `private int counter = 0` in a Servlet — race condition |
| `HttpSession` attributes must be `Serializable` for clustering | Storing a non-serializable object — session replication fails silently |
| Filter chain order matters — security filters must run before business filters | Misconfigured `web.xml` or `@Order` — authentication bypassed |

---

### 5. Business Logic / Domain Invariants

Rules from the problem domain that must be enforced in code.

#### Financial Domain
- **Account balance >= 0** (or >= overdraft limit) — must be checked atomically with withdrawal
- **Double-entry bookkeeping:** sum of all debits equals sum of all credits — system-wide invariant verified on every transaction
- **Money arithmetic uses `BigDecimal`, never `double`** — floating-point rounding violates cent-level precision
- **Currency conversion is never implicit** — `Money(100, USD) + Money(50, EUR)` must not compile or must throw

#### E-Commerce Domain
- **Order total = sum of line items × quantity × unit price − discounts + tax + shipping** — must be recalculated, never cached stale
- **Inventory count >= 0** — check-and-decrement must be atomic (optimistic lock or DB-level `CHECK` constraint)
- **Order state machine:** `CREATED → PAID → SHIPPED → DELIVERED` — no skipping states; no backward transitions except `CANCELLED`
- **Aggregate root:** modifications to `OrderLine` must go through `Order` — no direct repository for child entities

#### Healthcare Domain
- **Patient age >= 0** — derived from birth date, never stored as mutable field
- **Prescription dose within therapeutic range** — domain-specific validation, not just "not null"
- **Audit trail is append-only** — no updates, no deletes on audit records
- **PHI fields encrypted at rest** — structural invariant on the persistence layer

#### Scheduling / Calendar Domain
- **Event end time > start time** — always
- **No overlapping bookings for the same resource** — requires DB-level uniqueness or gap-lock
- **Timezone stored with every timestamp** — bare `LocalDateTime` for user-facing events is a bug

---

## How to Find Invariants

### Technique 1: Read the Bugs

Every bug report is an invariant that was never documented. Go through the last 50 closed bugs. For each one, ask: *what rule, if stated explicitly, would have prevented this?*

### Technique 2: Read the Tests

Tests encode invariants implicitly. A test that asserts `account.getBalance() >= 0` after withdrawal reveals a domain invariant. Extract these into explicit documentation.

### Technique 3: Read the `synchronized` Blocks

Every `synchronized` block, `Lock`, `volatile` field, and `Atomic*` variable exists to protect an invariant. If the invariant isn't documented next to the synchronization, it will eventually be broken by someone who doesn't understand what's being protected.

### Technique 4: Search for Defensive Checks

```
grep -rn "assert " src/
grep -rn "throw new IllegalStateException" src/
grep -rn "throw new IllegalArgumentException" src/
grep -rn "Preconditions.check" src/
grep -rn "@NotNull\|@NotBlank\|@Positive\|@Min\|@Max" src/
```

Every defensive check is a documented invariant. Every *missing* defensive check where one of the above patterns *should* exist is an undocumented invariant.

### Technique 5: Interview the Domain Expert

Ask: "What would make this data invalid?" and "What should never happen?" Domain experts think in invariants naturally — they just don't call them that.

### Technique 6: Use Daikon (Automated Dynamic Detection)

[Daikon](https://plse.cs.washington.edu/daikon/) instruments your Java program and infers likely invariants from observed executions:

1. Run DynComp to create a `.decls` file (variable comparability)
2. Run Chicory to create a `.dtrace` file (execution traces)
3. Run Daikon to infer invariants: `x > 0`, `y = 2*x + 1`, `array is sorted`, etc.

Daikon discovers invariants you didn't know existed. Use it on legacy codebases where documentation is sparse.

### Technique 7: Static Analysis Tools

| Tool | What It Finds |
|------|---------------|
| Error Prone | `@GuardedBy` violations, `@CheckReturnValue` ignoring |
| NullAway / Checker Framework | Nullability invariant violations |
| SpotBugs / FindBugs | Synchronization errors, mutable static state |
| Infer (Meta) | Null dereferences, resource leaks, thread safety violations |
| ArchUnit | Structural/architectural invariants (package dependencies, naming conventions, layer violations) |

---

## How to Document Invariants

### Level 1: In-Code (for the compiler and runtime)

```java
/**
 * Represents a non-negative monetary amount in a specific currency.
 *
 * Rep invariant:
 *   amount >= 0
 *   currency is a valid ISO 4217 code
 *   amount.scale() == currency.defaultFractionDigits()
 *
 * Abstraction function:
 *   represents the monetary value of amount in currency
 *
 * Thread safety:
 *   immutable — safe to share across threads without synchronization
 */
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    private void checkRep() {
        assert amount.compareTo(BigDecimal.ZERO) >= 0 : "negative amount";
        assert amount.scale() == currency.getDefaultFractionDigits() : "wrong scale";
    }
}
```

### Level 2: Bean Validation (for framework-enforced boundaries)

```java
public record CreateOrderRequest(
    @NotNull @Size(min = 1) List<@Valid LineItemDTO> items,
    @NotNull @ValidCurrencyCode String currency,
    @Positive BigDecimal totalAmount
) {}
```

### Level 3: ArchUnit (for structural invariants across the codebase)

```java
@ArchTest
static final ArchRule entities_not_in_controllers =
    noClasses().that().resideInAPackage("..controller..")
        .should().dependOnClassesThat().areAnnotatedWith(Entity.class)
        .because("controllers must use DTOs, never JPA entities (mass assignment risk)");

@ArchTest
static final ArchRule services_dont_access_repositories_of_other_aggregates =
    classes().that().resideInAPackage("..order.service..")
        .should().onlyAccessClassesThat()
        .resideInAnyPackage("..order..", "..common..", "java..", "org.springframework..")
        .because("aggregate boundaries must be respected");
```

### Level 4: REVIEW.md (for AI code review enforcement)

```markdown
# Review Guidelines

## Always flag
- Any new field on a JPA @Entity that lacks a Bean Validation constraint
- Any @Transactional method that catches and swallows exceptions
- Any monetary calculation using double or float instead of BigDecimal
- Any ConcurrentHashMap accessed with separate containsKey() + put() instead of compute()
- Any public API endpoint without an integration test
- Any mutable static field without @GuardedBy annotation

## Domain rules
- Order state transitions: CREATED → PAID → SHIPPED → DELIVERED (no skips)
- Account balance may never go negative — check in service layer AND database CHECK constraint
- All timestamps in API responses must include timezone offset

## Skip
- Generated code under src/gen/
- Formatting-only changes
```

---

## Why This Matters for AI Code Review

| What you document | What AI does | What AI misses |
|---|---|---|
| "`@Transactional` self-invocation doesn't work" in REVIEW.md | Flags every self-call in `@Transactional` beans | Nothing — it's a documented rule |
| Nothing about `@Transactional` self-invocation | Might catch it if the model's training data covered this pattern | Will miss it in non-obvious cases (e.g., method reference passed as lambda) |
| "Account balance >= 0" in REVIEW.md | Checks every code path that modifies balance for the guard | Nothing — it's a documented rule |
| Nothing about account balance | Has no way to know this is a domain rule | Every code path that assumes balance can go negative |

**The math is simple:** 20 minutes writing REVIEW.md → every future PR is checked against your invariants automatically. Zero minutes → AI reviews with generic heuristics and misses domain-specific bugs.

Claude Code Review dispatches multiple specialized agents per PR, each looking for a different class of issue. A verification step attempts to disprove each finding before posting. Documented invariants give these agents *concrete rules to verify against* rather than relying on general-purpose pattern matching.

---

## Practical Checklist: Discover, Document, Maintain

### Phase 1: Discovery (one-time, 2–4 hours)

- [ ] Review last 50 bug reports — extract the implicit invariant each one reveals
- [ ] `grep` for `assert`, `Preconditions.check*`, `throw new IllegalState/ArgumentException` — catalog existing invariants
- [ ] `grep` for `synchronized`, `volatile`, `AtomicReference`, `@GuardedBy` — list every concurrency invariant (documented or not)
- [ ] `grep` for `@NotNull`, `@Min`, `@Max`, `@Size`, `@Pattern` — catalog Bean Validation constraints
- [ ] Review entity classes — for each `@Entity`, list what makes an instance valid vs. invalid
- [ ] Review aggregate roots — for each aggregate, list which child modifications must go through the root
- [ ] Interview 2–3 senior engineers: "What should never happen in production?"
- [ ] Run Daikon on the test suite — review inferred invariants for surprises
- [ ] Run ArchUnit on the codebase — verify package dependency rules match intended architecture

### Phase 2: Documentation (one-time, 2–4 hours)

- [ ] Add rep invariant + abstraction function comments to core domain classes
- [ ] Add `checkRep()` methods to mutable domain objects; call at end of constructors and mutators
- [ ] Add `@GuardedBy` annotations to every field accessed under a lock
- [ ] Add `@ThreadSafe` / `@Immutable` / `@NotThreadSafe` to every public class in concurrent packages
- [ ] Write ArchUnit tests for cross-cutting structural invariants
- [ ] Create `REVIEW.md` at repo root with domain-specific rules for AI review
- [ ] Update `CLAUDE.md` with project conventions that apply beyond just reviews

### Phase 3: Maintenance (ongoing, <30 min/sprint)

- [ ] Every new bug → add the violated invariant to REVIEW.md and/or code
- [ ] Every new entity/aggregate → document rep invariant before the first PR merges
- [ ] Every new concurrent data structure → document threading model before the first PR merges
- [ ] Quarterly: review REVIEW.md for stale rules (removed features, migrated patterns)
- [ ] Quarterly: run Daikon on expanded test suite — compare new inferred invariants against documented ones
- [ ] Monitor AI review feedback: if Claude flags something repeatedly that's not a real issue, add it to `## Skip`; if it misses real issues, add the rule to `## Always flag`

---

## Tools Reference

| Tool | Purpose | Invariant Type |
|------|---------|---------------|
| `assert` / `checkRep()` | Runtime verification (dev/test) | Data / representation |
| Bean Validation (`jakarta.validation`) | Framework-enforced input validation | Data / boundary |
| ArchUnit | Compile-time architectural tests | Structural / dependency |
| Error Prone + `@GuardedBy` | Static analysis of lock discipline | Concurrency |
| NullAway / Checker Framework | Static null-safety analysis | Data / nullability |
| Infer (Meta) | Cross-procedural static analysis | Concurrency, resource leaks |
| SpotBugs | Pattern-based bug detection | Concurrency, data |
| Daikon | Dynamic invariant inference from executions | All types (discovery) |
| `REVIEW.md` | AI code review rule encoding | All types (enforcement) |
| `CLAUDE.md` | Project-wide AI assistant context | All types (guidance) |

---

## Sources

- [MIT 6.005 — Abstraction Functions & Rep Invariants](https://web.mit.edu/6.005/www/fa15/classes/13-abstraction-functions-rep-invariants/)
- [Java Concurrency Code Review Checklist (GitHub)](https://github.com/code-review-checklists/java-concurrency)
- [Code Review Checklist: Java Concurrency — Roman Leventov (freeCodeCamp)](https://medium.com/free-code-camp/code-review-checklist-java-concurrency-49398c326154)
- [Spring Framework Pitfalls — SonarSource](https://www.sonarsource.com/blog/spring-framework-pitfalls/)
- [Class Invariant — Wikipedia](https://en.wikipedia.org/wiki/Class_invariant)
- [Design by Contract — Wikipedia](https://en.wikipedia.org/wiki/Design_by_contract)
- [The Daikon Dynamic Invariant Detector](https://plse.cs.washington.edu/daikon/)
- [Protecting Invariants with Immutability — Vikas Garg](https://medium.com/@vikas.garg_7674/java-multithreading-protecting-invariants-with-immutability-82998d4b844b)
- [Java Thread Safety — TheLinuxCode](https://thelinuxcode.com/java-thread-safety-what-it-really-means-where-it-breaks-and-how-i-make-it-correct-in-2026/)
- [Enforcing Java Record Invariants with Bean Validation — Gunnar Morling](https://www.morling.dev/blog/enforcing-java-record-invariants-with-bean-validation/)
- [Code Review — Claude Code Docs](https://code.claude.com/docs/en/code-review)
- [Contracts for Java (Cofoja) — Google](https://github.com/nhatminhle/cofoja)
- [Protect Your Invariants — Marcus Biel](https://marcus-biel.com/what-are-invariants/)
- [Red-Black Tree — Wikipedia](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree)
- [Programming With Assertions — Oracle Java Docs](https://docs.oracle.com/javase/7/docs/technotes/guides/language/assert.html)
- [Retrofitting Null-Safety onto Java at Meta](https://engineering.fb.com/2022/11/22/developer-tools/meta-java-nullsafe/)
- [How to Kill the Code Review — Latent.Space](https://www.latent.space/p/reviews-dead)
- [Domain Model Pattern — Java Design Patterns](https://java-design-patterns.com/patterns/domain-model/)
