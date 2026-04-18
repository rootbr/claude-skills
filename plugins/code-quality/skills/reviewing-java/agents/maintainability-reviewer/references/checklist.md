# Java Maintainability Review Checklist

Concise reference for reviewing Java code for long-term maintainability — what makes code easy to understand, change, test, and debug in production. Every rule traces to a named primary source (Clean Code, Refactoring, Clean Architecture, *Building Maintainable Software* / SIG, Effective Unit Testing, JLS / JEPs / normative tool catalogs, or empirical software-engineering research).

## Table of Contents

1. Clean Code Fundamentals — functions, names, comments, code structure, conditions (Martin, Clean Code chs 2-4)
2. Complexity & Metrics — cognitive vs cyclomatic, SIG 4-star thresholds, Martin's package metrics Ca/Ce/I/A/D (SonarQube S3776, Muñoz Barón ESEM 2020, Visser BMS chs 2-5, Clean Architecture Part IV)
3. SOLID Principles — how each violation looks in Java (Clean Architecture Part III)
4. Bug-Pattern Catalogue — 12 bad/good code pairs for God class, boolean flag, nested conditionals, primitive obsession, leaky field exposure, testing implementation, log-and-rethrow, swallowed exception, feature envy, shotgun surgery, message chain, primitive obsession (Fowler Refactoring ch. 3, Clean Code, SpotBugs, Rentea)
5. Code Review Checklist — functions & names, complexity, SOLID, API & null safety, error handling, testing & testability, documentation, logging & observability, DI & project structure, configuration & secrets, static-analysis enforcement
6. Tools and Techniques — SonarQube, Error Prone / NullAway, SpotBugs, PMD, ArchUnit, PIT, Jackson / Micrometer / OpenTelemetry tooling matrix
7. Modern Java & Ecosystem — records, sealed classes, JSpecify → Spring Framework 7 / Spring Boot 4.0 (Nov 2025), JPMS, Conventional Commits, Keep a Changelog 1.1.0, ADR Nygard template, C4 Model
8. Research Questions to Cross-Check Against Sources — 52 concrete questions (sections A–L) each mapped to sections and sources
9. Sources — normative, foundational, tooling, checklists, industry case studies
10. Gotchas — 22 common wrong assumptions with stable IDs (G-01..G-22)

---

## 1. Clean Code Fundamentals

The foundational layer — names, functions, comments, local structure. Source for the whole section unless otherwise noted: Martin, *Clean Code* (Prentice Hall 2008), chs 2-4.

### 1.1 Functions

- **Small units.** Visser's empirical SIG guideline: limit each method/constructor to **≤ 15 lines of code** (LOC excluding whitespace and comments). A 4-star SIG rating requires ≥ 56.3% of code in units ≤ 15 LOC and ≤ 6.9% in units > 60 LOC. Source: Visser et al., *Building Maintainable Software — Java Edition* (O'Reilly 2016), ch. 2.
- **One thing.** A function does one thing at one level of abstraction. If you can extract a further function whose name isn't a tautological restatement, the original did more than one thing. Source: Clean Code ch. 3 "Do One Thing".
- **Few arguments.** Ideal = 0, acceptable ≤ 2, maximum 3. Visser's SIG guideline: **≤ 4 parameters per unit**; 4-star rating requires ≥ 86.2% of code in units with ≤ 2 parameters. Source: Clean Code ch. 3; Visser BMS ch. 5.
- **No flag arguments.** Boolean parameter ⇒ split into two methods (`renderForSuite()` and `renderForSingleTest()`). Source: Clean Code ch. 3 "Flag Arguments". See Bug Pattern §4.2.
- **No side effects.** A method named `checkPassword` that also initializes the session is a lie. Source: Clean Code ch. 3 "Side Effects".
- **Command-Query Separation.** A method either does something or answers something, not both (`setAndCheckIfExists()` smell). Source: Meyer, *Object-Oriented Software Construction* (1988); Clean Code ch. 3.
- **Prefer exceptions to error codes.** Error codes force callers into deeply nested `if` chains and create dependency magnets. Source: Clean Code ch. 3 "Prefer Exceptions to Returning Error Codes".

### 1.2 Names

- **Intention-revealing.** `int d // elapsed time in days` is noise; `int elapsedTimeInDays` carries the intent. Source: Clean Code ch. 2 "Use Intention-Revealing Names".
- **Avoid disinformation.** Don't call a `Map` an `accountList`. Avoid `l` and `O` (confusable with `1` and `0`). Source: Clean Code ch. 2 "Avoid Disinformation".
- **Make meaningful distinctions.** `Product` vs `ProductInfo` vs `ProductData` are semantically identical — noise. Source: Clean Code ch. 2 "Make Meaningful Distinctions".
- **Pronounceable, searchable.** Short `j` is fine in a tight loop; long-lived names should be readable and grep-friendly. Source: Clean Code ch. 2 "Use Pronounceable Names", "Use Searchable Names".
- **No encodings.** No Hungarian notation, no `m_` prefixes, no `I` interface prefix. Source: Clean Code ch. 2 "Avoid Encodings".
- **Class names = nouns** (`Customer`, `AddressParser`). **Method names = verbs** (`postPayment`, `deletePage`). Source: Clean Code ch. 2 "Class Names", "Method Names".
- **One word per concept.** Pick `fetch`, `retrieve`, or `get` once and stick with it across the codebase. Source: Clean Code ch. 2 "Pick One Word per Concept".
- **Style conventions.** Google Java Style: UpperCamelCase for classes, lowerCamelCase for methods/fields, `UPPER_SNAKE_CASE` only for `static final` deeply-immutable constants, 100-character column limit. Source: [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html).

### 1.3 Comments

- **Code explains itself; comments explain *why*.** Comments that restate code (`i++; // increment i`) rot as code changes. Source: Clean Code ch. 4 "Comments Do Not Make Up for Bad Code".
- **Never leave commented-out code.** Source control exists. Delete. Source: Clean Code ch. 4 "Bad Comments: Commented-Out Code".
- **Valid uses**: legal text, TODO with owner, warnings of consequence, amplification, public-API Javadoc. Source: Clean Code ch. 4 "Good Comments".
- **Fowler's perspective**: a comment is a deodorant — if you feel the need, first apply Extract Method and let the method name replace the comment. Source: Fowler, *Refactoring*, 2nd ed. (2018), ch. 3 "Comments".

### 1.4 Code Structure

- **Stepdown rule.** Code reads top-to-bottom, each function one level of abstraction above the functions it calls. Source: Clean Code ch. 3 "One Level of Abstraction per Function".
- **Variables close to usage.** Declare locals as late as possible. Source: Clean Code ch. 5 "Variable Declarations".
- **Dependent functions adjacent.** Caller immediately above callee whenever possible. Source: Clean Code ch. 5 "Dependent Functions".
- **Blocks and indentation.** Bodies of `if`/`while`/`for` should be one line — typically a call to a well-named function. Max nesting 1-2 levels; use guard clauses to flatten. Source: Clean Code ch. 3 "Blocks and Indenting".

### 1.5 Conditions

- **Extract explanatory variables.** `if (isAdult && hasLicense)` replaces `if (age >= 18 && licenseId != null && !licenseRevoked)`. Source: Clean Code ch. 3; Fowler, *Refactoring* — Extract Variable.
- **Avoid negations.** `if (isValid)` over `if (!isNotValid)`. Source: Clean Code ch. 17 "Smells and Heuristics: G29 Avoid Negative Conditionals".
- **Max 2 levels of nesting.** Use guard clauses and early returns. Polymorphism or sealed-class pattern-matching replaces long `if-else` / `switch` chains. Source: Clean Code ch. 3; Visser BMS ch. 3 "Dealing with Nesting".

---

## 2. Complexity & Metrics

### 2.1 Cognitive Complexity — SonarQube S3776

- **Threshold 15 per method** (rule `java:S3776`). Source: [SonarQube rule RSPEC-3776](https://rules.sonarsource.com/java/RSPEC-3776/).
- **How it accumulates.** +1 for each break in linear flow (`if`, `for`, `while`, `catch`, `switch`, `&&`/`||`); **+1 additional per nesting level**. A doubly-nested `if` inside a `for` therefore costs 3, not 1. Source: [SonarSource, *5 Clean Code Tips for Reducing Cognitive Complexity*](https://www.sonarsource.com/blog/5-clean-code-tips-for-reducing-cognitive-complexity); Campbell, *Cognitive Complexity — A new way of measuring understandability* (SonarSource white paper, 2018).
- **Differs from cyclomatic complexity.** Cyclomatic counts execution paths (testing concern). Cognitive counts *human reading difficulty*. A flat `switch` with 10 cases has cyclomatic 10 but cognitive ~1 because it is flat and readable; nested `if-else` chains score much higher cognitively. Source: Campbell 2018.
- **Empirical validation.** Muñoz Barón, Wyrich, Wagner, "An Empirical Validation of Cognitive Complexity as a Measure of Source Code Understandability", *14th ACM/IEEE ESEM* (2020), [arXiv:2007.12520](https://arxiv.org/abs/2007.12520). Meta-analysis of ~24 000 evaluations over 427 snippets: cognitive complexity positively correlates with comprehension time and subjective understandability ratings; mixed results for correctness of comprehension tasks.
- **Review action.** Any method over 15 — extract helpers, replace nesting with guard clauses, replace type-switching with polymorphism.

### 2.2 Cyclomatic Complexity & SIG Branch-Point Threshold

- **Cyclomatic = number of independent paths = branch points + 1.** Source: McCabe, "A Complexity Measure", *IEEE TSE* (1976).
- **SIG Guideline 2: ≤ 4 branch points per unit (McCabe ≤ 5).** 4-star SIG rating requires ≥ 74.8% of code in units with McCabe ≤ 5 and ≤ 1.5% with McCabe > 25. Source: Visser BMS ch. 3.
- **PMD default.** `CyclomaticComplexity` flags methods with complexity ≥ 10; aggregate class reportLevel = 80. Source: [PMD design rules](https://pmd.github.io/pmd/pmd_rules_java_design.html).

### 2.3 SIG Quality-Profile Thresholds for 4-Star Maintainability

Visser's *Building Maintainable Software — Java Edition* (O'Reilly 2016), chapters 2-5, translates maintainability into measurable unit-level thresholds. 4-star = top 35% of industry benchmark.

| Property | Guideline | 4-star cut-off |
|---|---|---|
| Unit size | ≤ 15 LOC per method/constructor (ch. 2) | ≤ 6.9% over 60 LOC, ≥ 56.3% ≤ 15 LOC |
| Unit complexity | ≤ 4 branch points (McCabe ≤ 5) (ch. 3) | ≥ 74.8% McCabe ≤ 5, ≤ 1.5% > 25 |
| Duplication | Type-1 clone ≥ 6 LOC is the counting threshold (ch. 4) | ≤ 4.6% of LOC marked redundant |
| Unit interfacing | ≤ 4 parameters per unit (ch. 5) | ≥ 86.2% of code in units ≤ 2 params |

### 2.4 Package Metrics — Martin Ca/Ce/I/A/D

| Metric | Formula | Range | Meaning |
|---|---|---|---|
| **Ca** (Afferent coupling) | external classes depending on this package | 0..∞ | High = high responsibility, must be stable |
| **Ce** (Efferent coupling) | external classes this package depends on | 0..∞ | High = high volatility risk |
| **Instability I** | Ce / (Ce + Ca) | 0..1 | 0 = maximally stable, 1 = maximally unstable |
| **Abstractness A** | abstract types / total types in package | 0..1 | 0 = fully concrete, 1 = fully abstract |
| **Distance D** | \|A + I − 1\| | 0..1 | Ideal = 0. D > 0.3 = warning. D > 0.5 = danger zone |

- **Zone of Pain**: I ≈ 0, A ≈ 0 — stable and concrete, hard to change (large utility classes). Source: Martin, *Clean Architecture*, ch. 14.
- **Zone of Uselessness**: I ≈ 1, A ≈ 1 — abstract and unstable, nobody uses it. Source: Clean Architecture ch. 14.
- **Base source**: [Wikipedia — Software package metrics](https://en.wikipedia.org/wiki/Software_package_metrics) (summary of Martin's 1994 C++ Report article and subsequent books).

### 2.5 Component Principles — Cohesion and Coupling (Clean Architecture Part IV)

Source: Martin, *Clean Architecture* (Prentice Hall 2017), chs 13-14.

| Principle | Abbrev. | Statement |
|---|---|---|
| Reuse/Release Equivalence | REP | The granule of reuse is the granule of release |
| Common Closure | CCP | Classes that change together belong together (SRP for components) |
| Common Reuse | CRP | Classes used together belong together; don't force users to depend on things they don't need (ISP for components) |
| Acyclic Dependencies | ADP | The component dependency graph must be a DAG |
| Stable Dependencies | SDP | Depend in the direction of stability |
| Stable Abstractions | SAP | A component should be as abstract as it is stable |

---

## 3. SOLID Principles — What Violations Look Like in Java

Source for the whole section: Martin, *Clean Architecture*, Part III (chs 7-11) and PPP (2002).

### 3.1 Single Responsibility (SRP)

- **Definition**: a module has one, and only one, reason to change. Source: Clean Architecture ch. 7.
- **Smell**: class with multiple method groups that never touch each other's fields.
- **Smell**: class changes for unrelated reasons (UI formatting + business rules + database access).
- **Fowler Refactoring smell**: "Divergent Change" and "Large Class". Source: Fowler *Refactoring* 2nd ed. ch. 3.
- **Fix**: Extract Class; each responsibility lives in its own class.

### 3.2 Open-Closed (OCP)

- **Definition**: open for extension, closed for modification. Source: Clean Architecture ch. 8.
- **Smell**: `switch` on type or `instanceof` chains that grow with each new feature (Fowler smell "Switch Statements", *Refactoring* ch. 3).
- **Fix**: polymorphism, Strategy pattern, sealed class hierarchies with pattern matching (Java 17+, see §7.1).

### 3.3 Liskov Substitution (LSP)

- **Definition**: subtypes must be usable everywhere the supertype is used. Source: Clean Architecture ch. 9.
- **Smell**: subtype throws `UnsupportedOperationException` (`Collections.unmodifiableList` where mutation is expected).
- **Smell**: declaring `ArrayList` instead of `List` leaks implementation.
- **Smell**: explicit downcasts (`(SpecificType) abstractRef`) signal LSP breakage.
- **Fix**: redesign hierarchy or replace inheritance with composition. Source: Fowler *Refactoring* — Replace Inheritance with Delegation.

### 3.4 Interface Segregation (ISP)

- **Definition**: clients should not depend on methods they do not use. Source: Clean Architecture ch. 10.
- **Smell**: "fat" interface forcing implementors to stub unused methods.
- **Fix**: split into role-specific interfaces (`Readable`, `Writable`, `Deletable`).

### 3.5 Dependency Inversion (DIP)

- **Definition**: high-level modules should not depend on low-level modules; both should depend on abstractions. Source: Clean Architecture ch. 11.
- **Smell**: `new ConcreteService()` inside business logic.
- **Smell**: code that cannot be tested without a live database or message broker.
- **Fix**: depend on abstractions; inject via constructor (§7.3, §5.9).

---

## 4. Bug-Pattern Catalogue

Each pattern: short lead + `bad` / `good` pair + `Source:` line. Patterns chosen from Fowler's Refactoring ch. 3 smells, Clean Code, SpotBugs, and Rentea.

### 4.1 God Class / Large Class (SRP violation)

A `UserService` handling auth, email, profiles, and persistence is three to four classes in a trench coat. Visser's threshold: LOC/class is not the only axis — fan-in over 50 and method count over ~15 are stronger signals.

```java
// bad
public class UserService {
    public User loadUser(String id) { ... }
    public boolean doesUserExist(String id) { ... }
    public User changeUserInfo(UserInfo info) { ... }
    public List<NotificationType> getNotificationTypes(User u) { ... }
    public void registerForNotifications(User u, NotificationType t) { ... }
    public void unregisterForNotifications(User u, NotificationType t) { ... }
    public List<User> searchUsers(UserInfo info) { ... }
    public void blockUser(User u) { ... }
    public List<User> getAllBlockedUsers() { ... }
    // 300+ LOC
}
```

```java
// good — separate concerns
public class UserDirectory { User load(String id); boolean exists(String id); User update(UserInfo i); }
public class NotificationService { List<NotificationType> typesFor(User u); void register(User u, NotificationType t); ... }
public class UserSearch { List<User> search(UserInfo i); }
public class UserModeration { void block(User u); List<User> blockedUsers(); }
```

Source: Fowler *Refactoring* 2nd ed. ch. 3 "Large Class"; Visser BMS ch. 6 "Separate Concerns in Modules".

### 4.2 Boolean Flag Argument

A boolean flag is a loud admission that the method does two things. Split it.

```java
// bad
public String render(PageData page, boolean isSuite) { ... }
render(page, true);  // reader has to look up what 'true' means
```

```java
// good
public String renderForSuite(PageData page) { ... }
public String renderForSingleTest(PageData page) { ... }
```

Source: Clean Code ch. 3 "Flag Arguments".

### 4.3 Nested Conditionals Deeper Than 2 Levels

Nesting compounds cognitive complexity (§2.1). Use guard clauses; extract nested branches.

```java
// bad — cognitive complexity = 8
public int calculateDepth(BinaryTreeNode<Integer> t, int n) {
    int depth = 0;
    if (t.getValue() == n) {
        return depth;
    } else {
        if (n < t.getValue()) {
            BinaryTreeNode<Integer> left = t.getLeft();
            if (left == null) { throw new TreeException("not found"); }
            else { return 1 + calculateDepth(left, n); }
        } else {
            BinaryTreeNode<Integer> right = t.getRight();
            if (right == null) { throw new TreeException("not found"); }
            else { return 1 + calculateDepth(right, n); }
        }
    }
}
```

```java
// good — flat with guard clauses
public int calculateDepth(BinaryTreeNode<Integer> t, int n) {
    if (t.getValue() == n)               return 0;
    if (n < t.getValue() && t.getLeft() != null)  return 1 + calculateDepth(t.getLeft(), n);
    if (n > t.getValue() && t.getRight() != null) return 1 + calculateDepth(t.getRight(), n);
    throw new TreeException("not found");
}
```

Source: Visser BMS ch. 3 "Dealing with Nesting — Replace Nested Conditional with Guard Clauses"; Fowler *Refactoring* 2nd ed. "Replace Nested Conditional with Guard Clauses".

### 4.4 Primitive Obsession

A `String` customer id, a `long` price in cents, and a `String` country code passed as three `String` parameters invite swapping arguments and lose invariants. Introduce small value types.

```java
// bad
public Transfer makeTransfer(String counterAccount, long amountInCents, String currency) { ... }
```

```java
// good — records (Java 16+)
record AccountNumber(String value) { /* validate 9-digit 11-check in constructor */ }
record Money(long cents, Currency currency) { ... }
public Transfer makeTransfer(AccountNumber to, Money amount) { ... }
```

Source: Fowler *Refactoring* 2nd ed. ch. 3 "Primitive Obsession"; Visser BMS ch. 5.

### 4.5 Leaky Field Exposure (EI_EXPOSE_REP)

Returning a reference to a mutable internal collection lets callers mutate private state behind your back. SpotBugs rule `EI_EXPOSE_REP` and `EI_EXPOSE_REP2`.

```java
// bad
public class Order {
    private final List<LineItem> items = new ArrayList<>();
    public List<LineItem> getItems() { return items; }      // external mutation possible
}
```

```java
// good
public List<LineItem> getItems() { return List.copyOf(items); } // immutable snapshot
```

Source: [SpotBugs bug descriptions — EI_EXPOSE_REP](https://spotbugs.readthedocs.io/en/stable/bugDescriptions.html).

### 4.6 Testing Implementation, Not Behaviour

`verify()` on every collaborator locks the test into the current implementation. Test observable outputs and state transitions.

```java
// bad
@Test void ordersCustomerEmail() {
    service.place(order);
    verify(customerRepo).findById(any());
    verify(emailGateway).send(any());
    verify(auditLogger).log(any());    // mirrors implementation
}
```

```java
// good — verify outcome, not collaborators
@Test void rejectedOrderSendsRefund() {
    PlacementResult result = service.place(rejectedOrder);
    assertThat(result).isEqualTo(PlacementResult.refundIssued());
}
```

Source: Hauer, [*Modern Best Practices for Testing in Java*](https://phauer.com/2019/modern-best-practices-testing-java/); Oikonomou, [*Software Testing Anti-Patterns*](https://blog.codepipes.com/testing/software-testing-antipatterns.html) §5.

### 4.7 Log-and-Rethrow

Logging at each layer produces duplicate stack traces. Pick one boundary (typically controller / `@ControllerAdvice`) to log.

```java
// bad
try {
    paymentGateway.charge(request);
} catch (GatewayException e) {
    log.error("gateway failed", e);
    throw new PaymentFailedException(e);
}
// caller also logs the same exception higher up
```

```java
// good — wrap and propagate; log at the boundary only
try {
    paymentGateway.charge(request);
} catch (GatewayException e) {
    throw new PaymentFailedException("charge failed for order " + orderId, e);
}
```

Source: Rentea, [*Exception-Handling Guide in Java*](https://victorrentea.ro/blog/exception-handling-guide-in-java/) — "Log-Rethrow Anti-Pattern".

### 4.8 Swallowed Exception ("Diaper")

Empty `catch` blocks delete root-cause information. Always handle, rethrow, or at minimum log with context.

```java
// bad
try {
    config = loader.load(path);
} catch (IOException e) {
    // ignored — default config used silently
}
```

```java
// good — fail fast or handle with explicit fallback + log
try {
    config = loader.load(path);
} catch (IOException e) {
    log.warn("Using defaults because {} could not be loaded", path, e);
    config = Config.defaults();
}
```

Source: Rentea *Exception-Handling Guide* — "Diaper Anti-Pattern"; Clean Code ch. 7.

### 4.9 Feature Envy

A method that spends most of its time asking another object for data belongs on the other object.

```java
// bad
public class Invoice {
    public BigDecimal total(Order o) {
        return o.getLineItems().stream()
                 .map(li -> li.getPrice().multiply(BigDecimal.valueOf(li.getQuantity())))
                 .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

```java
// good — put behaviour with data
public class Order {
    public BigDecimal total() { ... }
}
```

Source: Fowler *Refactoring* 2nd ed. ch. 3 "Feature Envy"; refactoring Move Method.

### 4.10 Shotgun Surgery / Divergent Change

Two sides of the same coin. *Divergent Change*: one class changes for many unrelated reasons — split it (SRP). *Shotgun Surgery*: a single change modifies many classes — consolidate into one.

```java
// bad — shotgun: adding a new tax rate edits 7 files
class OrderCalculator { ... }  // uses hardcoded 19% VAT
class InvoicePdfRenderer { ... } // hardcodes 19% VAT
class EmailSummary { ... }       // hardcodes 19%
// ... 4 more
```

```java
// good — one source of truth
public record TaxRate(BigDecimal percent, String region) {}
public interface TaxPolicy { TaxRate rateFor(Order o); }
// every renderer/calculator calls TaxPolicy — new rate = edit one bean
```

Source: Fowler *Refactoring* 2nd ed. ch. 3 "Divergent Change", "Shotgun Surgery".

### 4.11 Message Chain / Train Wreck

`a.getB().getC().getD().doWork()` ties the client to the entire navigation path. Hide delegation with Extract Method.

```java
// bad
String postalCode = order.getCustomer().getAddress().getCity().getPostalCode();
```

```java
// good
String postalCode = order.customerPostalCode();
```

Source: Fowler *Refactoring* 2nd ed. ch. 3 "Message Chains"; Law of Demeter.

### 4.12 Data Class

A class with only fields, getters and setters — behaviour envies it from elsewhere. Move methods that read/write the fields into the class itself.

```java
// bad
public class Reservation { /* getters/setters only, no behaviour */ }
class ReservationOps {
    public boolean conflictsWith(Reservation a, Reservation b) { /* reads a.getStart, b.getEnd ... */ }
}
```

```java
// good
public record Reservation(Instant start, Instant end) {
    public boolean conflictsWith(Reservation other) { return !end.isBefore(other.start) && !other.end.isBefore(start); }
}
```

Source: Fowler *Refactoring* 2nd ed. ch. 3 "Data Class"; records (JEP 395, Java 16).

---

## 5. Code Review Checklist

Every bullet starts `- [ ]` and ends with a traceable source.

### 5.1 Functions & Names

- [ ] Every new method is ≤ 15 LOC or has a documented reason to exceed it. Source: Visser BMS ch. 2.
- [ ] Every new method has ≤ 4 parameters; long lists replaced by parameter objects. Source: Visser BMS ch. 5; Clean Code ch. 3.
- [ ] No boolean flag arguments introduced; method is split by behaviour. Source: Clean Code ch. 3.
- [ ] Method name describes what it does without requiring a comment; side effects are named in the method. Source: Clean Code ch. 3 "Side Effects".
- [ ] Command-query separation holds: each method either mutates or returns, not both. Source: Meyer, OOSC-2 (1988); Clean Code ch. 3.
- [ ] Class names are nouns, method names are verbs; one word per concept across the codebase. Source: Clean Code ch. 2.
- [ ] Style conventions (indent, line length, naming) follow Google Java Style or the project-declared style. Source: [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html).

### 5.2 Complexity & Metrics

- [ ] Cognitive complexity per method ≤ 15 (SonarQube rule `java:S3776`). Source: [SonarQube S3776](https://rules.sonarsource.com/java/RSPEC-3776/); Muñoz Barón et al. (ESEM 2020).
- [ ] McCabe cyclomatic complexity ≤ 5 per unit (Visser SIG 4-star). Source: Visser BMS ch. 3.
- [ ] No nesting deeper than 2 levels; guard clauses used. Source: Clean Code ch. 3; Visser BMS ch. 3.
- [ ] Package Distance D ≤ 0.3 (SonarQube / JDepend). Source: Martin, *Clean Architecture* ch. 14.
- [ ] No circular dependencies between packages (ArchUnit `slices().matching(..).beFreeOfCycles()`). Source: Martin Acyclic Dependencies Principle; [ArchUnit User Guide](https://www.archunit.org/userguide/html/000_Index.html).
- [ ] Duplication: ≤ 4.6% of LOC is redundant (SIG 4-star); no Type-1 clones ≥ 6 LOC. Source: Visser BMS ch. 4.

### 5.3 SOLID Violations

- [ ] No class changes for more than one reason (SRP). Source: Clean Architecture ch. 7.
- [ ] No `switch` / `instanceof` chain grows with each new subtype (OCP). Replaced by polymorphism / sealed class + pattern matching. Source: Clean Architecture ch. 8; JEP 409.
- [ ] No subtype throws `UnsupportedOperationException` or declares narrower preconditions than the supertype (LSP). Source: Clean Architecture ch. 9.
- [ ] No interface forces implementors to stub unused methods (ISP). Source: Clean Architecture ch. 10.
- [ ] No business logic contains `new ConcreteService()`; dependencies are injected (DIP). Source: Clean Architecture ch. 11.

### 5.4 API, Immutability, Null Safety

- [ ] Value types use `record` (JEP 395, Java 16+) for immutable data carriers. Source: [JEP 395](https://openjdk.org/jeps/395).
- [ ] Result/variant types use sealed interfaces with pattern matching (JEP 409 sealed, JEP 441 patterns). Source: [JEP 409](https://openjdk.org/jeps/409), [JEP 441](https://openjdk.org/jeps/441).
- [ ] Collections returned from getters are defensively copied (`List.copyOf`) or `Collections.unmodifiable*`; SpotBugs `EI_EXPOSE_REP` absent. Source: [SpotBugs](https://spotbugs.readthedocs.io/en/stable/bugDescriptions.html).
- [ ] `Optional` used only as return type; never as field, parameter, or collection element. Source: [Effective Java, 3rd ed., Item 55](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/).
- [ ] `Optional.of` not called on a possibly-null value (use `ofNullable`); `get()` avoided in favour of `orElseThrow` with a typed exception. Source: Effective Java 3e Item 55.
- [ ] Package is `@NullMarked` (JSpecify); nullable exceptions annotated with `@Nullable`. Source: [JSpecify](https://jspecify.dev/); [Spring Framework 7 null-safety](https://docs.spring.io/spring-framework/reference/core/null-safety.html).
- [ ] Public API boundaries validate preconditions with `Objects.requireNonNull` and descriptive messages. Source: Effective Java 3e Item 49.
- [ ] Build runs NullAway (Error Prone plugin) in JSpecify mode (`NullAway:OnlyNullMarked=true`). Source: [NullAway](https://github.com/uber/nullaway).

### 5.5 Error Handling

- [ ] No empty `catch` blocks; every exception is handled with context or rethrown. Source: Rentea, *Exception-Handling Guide*.
- [ ] No log-and-rethrow — logging done once at the topmost boundary. Source: Rentea.
- [ ] No `catch (Exception e)` / `catch (Throwable t)` unless the layer is documented as a last-resort boundary; typed exceptions caught first. Source: Rentea; Effective Java 3e Item 69.
- [ ] Custom exception hierarchy extends `RuntimeException` (Rentea) unless the caller can recover; original cause always chained. Source: Rentea; Effective Java 3e Item 72.
- [ ] No flow-control-by-exception: parsing / lookup uses `Optional` or sealed result types. Source: Effective Java 3e Item 69.
- [ ] Repository exceptions translated at service boundary to domain exceptions; HTTP translation in `@ControllerAdvice`. Source: [Spring Framework Reference — Exception Handling](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html).

### 5.6 Testing & Testability

- [ ] Test pyramid: more unit than integration than E2E — exact ratio is context-specific, Fowler does not prescribe. Source: [Martin Fowler, *Test Pyramid*](https://martinfowler.com/bliki/TestPyramid.html).
- [ ] Integration tests use Testcontainers against real DB/broker (not H2 substitutes). Source: Hauer, *Modern Best Practices*.
- [ ] HTTP boundaries use WireMock or RestAssured rather than method-level mocks of the client. Source: Hauer.
- [ ] No testing of private methods via reflection ("Anal Probe" test smell). Source: Koskela, *Effective Unit Testing* (Manning 2013), ch. 9.
- [ ] `verify()` used sparingly — outputs and state transitions are the primary assertion. Source: Hauer; Oikonomou testing anti-patterns §5.
- [ ] Each test is independent; no shared mutable state, no execution order, no wall-clock dependency (inject `Clock`). Source: Hauer.
- [ ] Flaky tests are P0 bugs: quarantined the same day and fixed in-sprint. Source: [Google Testing Blog, *Flaky Tests at Google and How We Mitigate Them*](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html).
- [ ] AAA (Arrange/Act/Assert) or Given/When/Then layout; no `if`/`for` inside tests. Source: Hauer; Koskela ch. 2.
- [ ] AssertJ used for fluent, type-safe assertions; JUnit 5 `@Nested` groups scenarios; `@DisplayName` for human-readable names. Source: Hauer.
- [ ] PIT mutation testing configured on critical modules; `mutationThreshold` set per-team risk — no universal kill-rate target. Source: [PIT 1.22.0 docs](https://pitest.org/) (Nov 2025); JavaPro, *Test Your Tests: Mutation Testing in Java with PIT* (Jan 2026).
- [ ] Test-data builders (`aUser().withRole(ADMIN).build()`) used instead of bare constructors. Source: Koskela ch. 10.

### 5.7 Documentation

- [ ] Public API classes and methods have Javadoc covering purpose, thread safety, `@param`/`@return`/`@throws` non-obvious cases. Source: [Oracle Javadoc Tool Guide](https://docs.oracle.com/en/java/javase/25/docs/specs/javadoc/doc-comment-spec.html).
- [ ] No Javadoc that merely restates the signature (`@param foo the foo`). Source: Clean Code ch. 4.
- [ ] `package-info.java` present for every public package; `module-info.java` for every JPMS module. Source: Oracle Javadoc docs; [JEP 261](https://openjdk.org/jeps/261).
- [ ] Significant architectural decisions live as ADR files in `docs/adr/NNNN-title.md` with Nygard's fields: Status, Context, Decision, Consequences. Source: Nygard, *Documenting Architecture Decisions* (2011) — canonical template at [adr.github.io](https://adr.github.io/adr-templates/).
- [ ] README contains build/run/test instructions plus C4 Level-1 (system context) and Level-2 (container) diagrams. Source: [C4 Model](https://c4model.com/).
- [ ] CHANGELOG.md follows Keep a Changelog 1.1.0 structure (Added, Changed, Deprecated, Removed, Fixed, Security) with ISO-8601 release dates. Source: [keepachangelog.com 1.1.0](https://keepachangelog.com/en/1.1.0/).
- [ ] Commits follow Conventional Commits v1.0.0 (`feat:`, `fix:`, `chore:` … `feat!:` for breaking). Source: [conventionalcommits.org v1.0.0](https://www.conventionalcommits.org/en/v1.0.0/).

### 5.8 Logging & Observability

- [ ] Logs are structured (JSON) via Logback / Log4j2 JSON encoder. Source: [Logback docs](https://logback.qos.ch/); [Log4j 2 docs](https://logging.apache.org/log4j/2.x/).
- [ ] Correlation IDs (`traceId`, `spanId`, `requestId`) placed in MDC at entry filter, cleared at exit; explicitly propagated into async executors (ScopedValue / MDC adapter). Source: [W3C Trace Context Level 2](https://www.w3.org/TR/trace-context/); [SLF4J MDC](https://www.slf4j.org/manual.html#mdc).
- [ ] Log calls use parameterized messages (`log.info("order {}", id)`) — never string concatenation. Source: SLF4J Javadoc.
- [ ] Log-level discipline: ERROR pages a human, WARN = degraded/self-healed, INFO = business event, DEBUG/TRACE off in prod. Source: [Oracle Java Logging Overview](https://docs.oracle.com/en/java/javase/25/core/java-logging-overview.html); SLF4J guidance.
- [ ] No secrets, tokens, PII, full request/response bodies in logs. Source: GDPR Art. 5(1)(c) data minimization.
- [ ] Micrometer metrics use lowercase dotted names; tag values are bounded — user-derived tags (e.g. raw URI) normalized to prevent cardinality explosion. Source: [Micrometer Naming](https://docs.micrometer.io/micrometer/reference/concepts/naming.html).
- [ ] Spring Boot Actuator `/health` distinguishes liveness vs readiness probes for Kubernetes; custom `HealthIndicator` per critical dependency. Source: [Spring Boot Actuator docs](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html).
- [ ] OpenTelemetry exporter configured; tracing uses probability or rate-limiting sampler, not 100% in prod. Source: [OpenTelemetry Java docs](https://opentelemetry.io/docs/languages/java/).

### 5.9 DI, Project Structure, Modularity

- [ ] All Spring beans use constructor injection (Spring 4.3+ inference removes need for `@Autowired`); fields are `final`. Source: [Spring Framework Reference — Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies.html).
- [ ] No `@Autowired` on a field; no field injection. Source: [Spring Framework Reference](https://docs.spring.io/spring-framework/reference/core/beans/dependencies.html); also enforced by ArchUnit `noClasses().should().notUseFieldInjection()`.
- [ ] No circular bean dependencies. Since Spring Boot 2.6 — and still in Spring Boot 4.0.0 (Nov 2025) — circular references are prohibited by default. Source: [Spring Boot Release Notes 2.6](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes); [Spring Boot 4.0.0 release](https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now/).
- [ ] Never re-enable `spring.main.allow-circular-references=true` as a permanent fix.
- [ ] Packages organised by feature (`orders`, `billing`, `shipping`) not by layer (`controllers`, `services`, `repositories`). Source: [Bal, *Package by Feature vs Package by Layer*](https://medium.com/sahibinden-technology/package-by-layer-vs-package-by-feature-7e89cde2ae3a).
- [ ] ArchUnit tests enforce: cycle-free slices, no field injection, no domain→controller dependency. Source: [ArchUnit User Guide](https://www.archunit.org/userguide/html/000_Index.html) v1.4.1.
- [ ] Libraries published as JPMS modules declare explicit `exports` / `requires` in `module-info.java`. Source: [JEP 261](https://openjdk.org/jeps/261).

### 5.10 Configuration & Secrets

- [ ] All URLs, credentials, feature switches externalised; `application.yml` + environment-specific profiles. Source: [Spring Boot Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config).
- [ ] Config binding uses `@ConfigurationProperties` + `@Validated` with JSR-380 (`@NotBlank`, `@Min`) so the app fails fast on startup. Source: [Spring Boot Type-safe Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.typesafe-configuration-properties); [JSR-380 spec](https://jakarta.ee/specifications/bean-validation/3.0/).
- [ ] No plaintext secrets in `application.yml`, `.env`, or git history. Use Spring Cloud Vault, AWS Secrets Manager, Kubernetes Secrets (`spring.config.import=configtree:`), or env vars. Source: [Spring Cloud Vault docs](https://spring.io/projects/spring-cloud-vault).
- [ ] Feature flags live in a dedicated system (Unleash, LaunchDarkly, FF4J) — not `application.yml` — and every flag has an owner and expiry.
- [ ] Missing required property causes startup failure with a clear message (`@Value` without default is correct; don't suppress). Source: Spring Boot external-config docs.

### 5.11 Static-Analysis Enforcement

- [ ] Error Prone + NullAway run in the build as ERROR (not WARNING); build fails on violations. Source: [Error Prone](https://errorprone.info/); [NullAway](https://github.com/uber/nullaway).
- [ ] SpotBugs + FindSecBugs run in the build with failing severity for `EI_EXPOSE_REP`, `NP_NULL_ON_SOME_PATH`, `SE_BAD_FIELD`. Source: [SpotBugs](https://spotbugs.readthedocs.io/en/stable/).
- [ ] PMD thresholds enforced: `CyclomaticComplexity methodReportLevel=10`, `CouplingBetweenObjects threshold=20`, `TooManyMethods maxmethods=10`, `GodClass`. Source: [PMD design rules](https://pmd.github.io/pmd/pmd_rules_java_design.html).
- [ ] SonarQube quality gate: 0 new bugs/vulnerabilities, cognitive complexity per method ≤ 15, duplications < 3%. Source: [SonarQube Quality Gates docs](https://docs.sonarsource.com/sonarqube-server/latest/user-guide/quality-gates/).
- [ ] ArchUnit tests live under `src/test/java` and run on every CI build. Source: [ArchUnit User Guide](https://www.archunit.org/userguide/html/000_Index.html).
- [ ] PIT mutation testing runs weekly or per-merge on core modules; `mutationThreshold` configured and tracked. Source: [PIT 1.22.0](https://pitest.org/).

---

## 6. Tools and Techniques

### 6.1 Static-Analysis Matrix

| Tool | Version focus (2026) | Key rules | Detects |
|---|---|---|---|
| **SonarQube** | 10.x | `S3776` cognitive ≤ 15, `S1192` duplicate literals ≥ 3, `S1066` collapsible if, `S2259` null dereference | Readability, duplication, bugs |
| **Error Prone + NullAway** | Error Prone 2.36+, NullAway 0.12+ | `NullAway` (JSpecify mode), `MissingOverride`, `ImmutableEnumChecker`, `ReturnValueIgnored` | Null safety, correctness |
| **SpotBugs + FindSecBugs** | 4.8+ | `EI_EXPOSE_REP`, `NP_NULL_ON_SOME_PATH`, `SE_BAD_FIELD`, `DM_*` | Encapsulation, null paths, security |
| **PMD** | 7.x | `GodClass`, `CyclomaticComplexity ≥ 10`, `CouplingBetweenObjects > 20`, `TooManyMethods > 10` | Design smells, coupling |
| **ArchUnit** | 1.4.1 (May 2025) | Cycle detection, layer enforcement, DI pattern enforcement | Architecture violations |
| **PIT** (mutation) | 1.22.0 (Nov 2025) | Stryker-style mutators; surviving mutant = weak assertion | Test-suite quality |

### 6.2 Pragmatic Setup

- **Pre-commit / local build**: Error Prone + NullAway + SpotBugs (fast feedback, fail on violations).
- **CI pull-request**: PMD + ArchUnit + SonarQube quality gate (more expensive checks; pass/fail visible in PR).
- **Nightly / weekly**: PIT mutation testing on critical modules; benchmark against Visser SIG 4-star cutoffs.

---

## 7. Modern Java & Ecosystem (2026 baseline)

### 7.1 Java Language Features for Maintainability

- **Records** (final in Java 16, [JEP 395](https://openjdk.org/jeps/395)) — canonical immutable DTOs / value objects.
- **Sealed classes + interfaces** (final in Java 17, [JEP 409](https://openjdk.org/jeps/409)) + **pattern matching for switch** (final in Java 21, [JEP 441](https://openjdk.org/jeps/441)) — exhaustive, compiler-checked result/variant types.
- **Text blocks** (final in Java 15, [JEP 378](https://openjdk.org/jeps/378)) — readable multiline strings.
- **`Optional<T>`** — return type only; see §5.4.

### 7.2 Null Safety — JSpecify in Spring Framework 7 / Spring Boot 4.0

- **JSpecify 1.0** released October 2024; cross-vendor standard backed by Google, JetBrains, Oracle.
- **Spring Framework 7** (GA Nov 2025) fully migrated; old `org.springframework.lang.@Nullable` / `@NonNull` / `@NonNullApi` deprecated. Source: [Null-safety :: Spring Framework](https://docs.spring.io/spring-framework/reference/core/null-safety.html).
- **Spring Boot 4.0.0** (Nov 20 2025) — entire portfolio null-marked; Kotlin 2 is the new baseline. Source: [Spring Boot 4.0.0 available now](https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now/).
- **NullAway** (Uber, <10 % build overhead) enforces at compile time; configure `NullAway:OnlyNullMarked=true`, and `NullAway:JSpecifyMode=true` on JDK 22+. Source: [Null Safety in Spring apps with JSpecify and NullAway (Mar 2025)](https://spring.io/blog/2025/03/10/null-safety-in-spring-apps-with-jspecify-and-null-away/).
- **IntelliJ IDEA 2025.3+** has first-class JSpecify support.

### 7.3 Dependency Injection — Constructor-Only

- **Constructor injection** enables `final` fields (immutability), makes dependencies visible, and exposes circular refs at startup. Source: [Spring Framework DI docs](https://docs.spring.io/spring-framework/reference/core/beans/dependencies.html).
- **Field injection (`@Autowired` on field)**: hides dependencies, prevents `final`, couples class to Spring container, hides circular deps. Anti-pattern.
- **Circular dependencies**: banned by default since Spring Boot 2.6 (Nov 2021) — still banned in Spring Boot 4.0.0 (Nov 2025). Refactor (extract bean, use events) rather than setting `spring.main.allow-circular-references=true`.

### 7.4 Project Structure & Modularity

- **Package by feature**: co-locates controller, service, repository, domain for a single feature. Enables package-private visibility, raises cohesion, lowers cross-package coupling. Source: [Bal, *Package by Feature vs Package by Layer*](https://medium.com/sahibinden-technology/package-by-layer-vs-package-by-feature-7e89cde2ae3a).
- **ADP / SDP / SAP** (Clean Architecture ch. 14) guide the component graph.
- **ArchUnit 1.4.1** (May 2025) enforces rules at CI time:

```java
slices().matching("com.company.app.(*)..").should().beFreeOfCycles();

noClasses().should().beAnnotatedWith(Autowired.class)
           .because("Use constructor injection");

layeredArchitecture()
    .layer("Controller").definedBy("..controller..")
    .layer("Service").definedBy("..service..")
    .layer("Domain").definedBy("..domain..")
    .whereLayer("Domain").mayNotAccessAnyLayer();
```

### 7.5 Dependency Management

- **BOM**: Maven `<dependencyManagement>` with `<scope>import</scope>` (or Gradle platform) locks a set of compatible versions. Source: [Reflectoring, *Maven BOM*](https://reflectoring.io/maven-bom/).
- **Resolution difference**: Maven uses "nearest-wins" (depth-first), Gradle "highest-version-wins". Both can silently converge on incompatible versions — fail the build on divergence (`maven-enforcer-plugin` `dependencyConvergence`; Gradle `failOnVersionConflict`). Source: [Gradle, *Five Ways Out of Dependency Hell*](https://gradle.com/blog/five-ways-dependency-hell-maven/).
- **CI gate**: run `mvn dependency:tree` / `gradle dependencies` and fail on convergence errors.
- **Shading**: last resort. Increases JAR size, breaks source navigation, hides CVEs from scanners.

### 7.6 Documentation — ADR, Keep a Changelog, Conventional Commits, C4

- **ADR Nygard template** (2011): Title, Status, Context, Decision, Consequences. Store as `docs/adr/NNNN-title.md`. Source: [adr.github.io](https://adr.github.io/adr-templates/).
- **Keep a Changelog 1.1.0**: sections Added / Changed / Deprecated / Removed / Fixed / Security; ISO-8601 release dates. Source: [keepachangelog.com 1.1.0](https://keepachangelog.com/en/1.1.0/).
- **Conventional Commits v1.0.0**: `feat:` → MINOR, `fix:` → PATCH, `!:`/`BREAKING CHANGE:` → MAJOR. Source: [conventionalcommits.org v1.0.0](https://www.conventionalcommits.org/en/v1.0.0/).
- **commitlint + @commitlint/config-conventional** validates in a Git pre-commit hook.
- **C4 Model**: Level 1 System Context + Level 2 Container diagrams in the README — Level 3 Component for critical subsystems; Level 4 Code almost never (let the code itself be the source). Source: [c4model.com](https://c4model.com/).

---

## 8. Research Questions to Cross-Check Against Sources

The checklist is organized to answer the following 52 questions about a diff or PR. Each question is followed by the corresponding checklist section(s) and primary source(s).

### A. Functions & names

1. **Is every new method ≤ 15 LOC, or documented as an exception?** (§1.1, §5.1) — Visser BMS ch. 2.
2. **Does every method do exactly one thing at one level of abstraction?** (§1.1) — Clean Code ch. 3 "One Level of Abstraction per Function".
3. **Is parameter count ≤ 4, and are long lists replaced by parameter objects?** (§1.1, §5.1) — Visser BMS ch. 5; Fowler *Refactoring* "Introduce Parameter Object".
4. **Are there zero boolean flag parameters?** (§4.2, §5.1) — Clean Code ch. 3 "Flag Arguments".
5. **Does every name reveal intent, with no disinformation and no Hungarian encoding?** (§1.2, §5.1) — Clean Code ch. 2.
6. **Are class names nouns, method names verbs, and one word per concept?** (§1.2, §5.1) — Clean Code ch. 2.

### B. Complexity & metrics

7. **Is cognitive complexity ≤ 15 per method (`java:S3776`)?** (§2.1, §5.2) — SonarQube S3776; Muñoz Barón et al. (ESEM 2020).
8. **Is cyclomatic complexity ≤ 5 per unit (Visser SIG)?** (§2.2, §5.2) — Visser BMS ch. 3.
9. **Is nesting depth ≤ 2, or flattened with guard clauses?** (§4.3, §5.2) — Clean Code ch. 3; Visser BMS ch. 3.
10. **Is duplication ≤ 4.6 % of LOC (SIG 4-star) with no Type-1 clones ≥ 6 LOC?** (§2.3, §5.2) — Visser BMS ch. 4.
11. **Is the package-level Distance D ≤ 0.3?** (§2.4, §5.2) — Clean Architecture ch. 14.
12. **Are there zero package cycles (ArchUnit `beFreeOfCycles()`)?** (§2.5, §7.4, §5.2) — Martin ADP; [ArchUnit 1.4.1](https://www.archunit.org/userguide/html/000_Index.html).

### C. SOLID violations

13. **Does any class change for more than one reason (SRP)?** (§3.1, §4.1, §5.3) — Clean Architecture ch. 7; Fowler Large Class.
14. **Is any `switch` / `instanceof` chain type-dispatching in a way that grows per feature (OCP)?** (§3.2, §5.3) — Clean Architecture ch. 8; JEP 441.
15. **Does any subtype throw `UnsupportedOperationException` or narrow preconditions (LSP)?** (§3.3, §5.3) — Clean Architecture ch. 9.
16. **Is any implementor forced to stub unused interface methods (ISP)?** (§3.4, §5.3) — Clean Architecture ch. 10.
17. **Does any business class `new` a concrete dependency instead of injecting it (DIP)?** (§3.5, §5.3) — Clean Architecture ch. 11.

### D. API, immutability, records, sealed types

18. **Are immutable value objects expressed as `record` (JEP 395)?** (§7.1, §5.4) — JEP 395.
19. **Are variant/result types sealed with pattern-matched exhaustive switch (JEP 409 / 441)?** (§7.1, §5.4) — JEP 409, JEP 441.
20. **Do getters that expose internal collections defensively copy (no `EI_EXPOSE_REP`)?** (§4.5, §5.4) — SpotBugs bug descriptions.
21. **Is `Optional<T>` used only as a return type, never as a field or parameter?** (§5.4) — Effective Java 3e Item 55.

### E. Null safety

22. **Is the package `@NullMarked` (JSpecify) with exceptions marked `@Nullable`?** (§7.2, §5.4) — Spring Framework 7 null-safety docs.
23. **Does the build run NullAway in JSpecify mode (`OnlyNullMarked=true`)?** (§7.2, §5.4, §6.1) — Spring blog Mar 2025; NullAway README.
24. **Are public-API preconditions validated at the boundary (`Objects.requireNonNull`)?** (§5.4) — Effective Java 3e Item 49.

### F. Error handling

25. **Are all `catch` blocks non-empty (no swallowed exceptions)?** (§4.8, §5.5) — Rentea *Exception-Handling Guide*.
26. **Is logging done once at the boundary (no log-and-rethrow)?** (§4.7, §5.5) — Rentea.
27. **Is `catch (Exception)` / `catch (Throwable)` absent except at documented last-resort boundaries?** (§5.5) — Rentea; Effective Java 3e Item 69.
28. **Does every wrapped exception chain the original cause?** (§5.5) — Effective Java 3e Item 72.
29. **Is control flow free of exception-for-flow idioms?** (§5.5) — Effective Java 3e Item 69.
30. **Is exception translation centralized in `@ControllerAdvice` / `@ExceptionHandler`?** (§5.5) — Spring Framework Reference.

### G. Testing & testability

31. **Are integration tests on real DB/broker via Testcontainers (not H2 substitutes)?** (§5.6) — Hauer *Modern Best Practices*.
32. **Does every test assert observable outcomes rather than `verify()` on every collaborator?** (§4.6, §5.6) — Hauer; Oikonomou anti-patterns §5.
33. **Are there no tests of private methods (via reflection or made-public)?** (§5.6) — Koskela ch. 9.
34. **Is every test independent, with injected `Clock`, no shared static state, no required order?** (§5.6) — Hauer.
35. **Are flaky tests treated as P0 and quarantined immediately?** (§5.6) — [Google Testing Blog (2016)](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html).
36. **Does the PR use AssertJ fluent assertions and JUnit 5 `@Nested` / `@DisplayName`?** (§5.6) — Hauer.
37. **Is PIT mutation testing configured with a per-module `mutationThreshold` (no universal target)?** (§5.6, §6.1) — PIT 1.22.0; JavaPro Jan 2026.

### H. Documentation

38. **Does every public class and method have meaningful Javadoc (purpose, thread safety, `@param`/`@return`/`@throws`)?** (§5.7) — Oracle Javadoc tool guide.
39. **Do commits follow Conventional Commits v1.0.0, and does `feat!`/`BREAKING CHANGE` trigger a MAJOR bump?** (§7.6, §5.7) — conventionalcommits.org v1.0.0.
40. **Is CHANGELOG.md kept in Keep a Changelog 1.1.0 format with ISO-8601 dates?** (§7.6, §5.7) — keepachangelog.com 1.1.0.
41. **Is every significant architecture decision captured as an ADR in `docs/adr/` using Nygard's template?** (§7.6, §5.7) — adr.github.io.
42. **Does the README carry C4 Level-1 + Level-2 diagrams?** (§7.6, §5.7) — c4model.com.

### I. Logging & observability

43. **Are logs structured (JSON) with parameterized messages?** (§5.8) — Logback/Log4j2 docs.
44. **Are correlation IDs (W3C traceparent → MDC) propagated through async boundaries?** (§5.8) — W3C Trace Context L2.
45. **Do Micrometer tags avoid user-supplied unbounded cardinality (e.g., raw URI → normalized)?** (§5.8) — [Micrometer Naming](https://docs.micrometer.io/micrometer/reference/concepts/naming.html).
46. **Are secrets, tokens, and PII absent from log output?** (§5.8) — GDPR Art. 5(1)(c).

### J. DI & project structure

47. **Are all Spring beans constructor-injected with `final` fields, no field injection?** (§7.3, §5.9) — Spring Framework DI docs.
48. **Are the packages organized by feature, not by layer, with ArchUnit tests enforcing no cycles?** (§7.4, §5.9) — Bal 2018; ArchUnit 1.4.1.

### K. Configuration & secrets

49. **Are all URLs, credentials, switches externalised via `@ConfigurationProperties` + JSR-380 validation?** (§5.10) — Spring Boot type-safe config.
50. **Are secrets absent from `application.yml` and git, managed via Vault / K8s Secrets / env vars?** (§5.10) — Spring Cloud Vault docs.

### L. Static-analysis enforcement

51. **Is the CI pipeline running Error Prone + NullAway + SpotBugs + PMD + ArchUnit at failing severity?** (§6.1, §5.11) — tool docs.
52. **Does the SonarQube quality gate block the merge on new bugs / vulnerabilities and on any method with cognitive complexity > 15?** (§6.1, §5.11) — SonarQube Quality Gates docs.

---

## 9. Sources

### Normative / specification

- [Java Language Specification, Java SE 25](https://docs.oracle.com/javase/specs/jls/se25/html/index.html).
- [Javadoc Tool Guide (Java 25)](https://docs.oracle.com/en/java/javase/25/docs/specs/javadoc/doc-comment-spec.html).
- [W3C Trace Context Level 2 (2024)](https://www.w3.org/TR/trace-context/) — `traceparent` / `tracestate` header format.
- [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/) — canonical CHANGELOG structure.
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) — commit-to-SemVer mapping.
- [JSpecify 1.0](https://jspecify.dev/) — cross-vendor null-annotation standard (Oct 2024).
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html) — naming, line length (100 chars), constants.
- [Spring Framework 7 Null Safety Reference](https://docs.spring.io/spring-framework/reference/core/null-safety.html).
- [Spring Framework Dependency Injection docs](https://docs.spring.io/spring-framework/reference/core/beans/dependencies.html).
- [Spring Boot 2.6 Release Notes — Circular References](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes).
- [Spring Boot 4.0.0 available now (Nov 20 2025)](https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now/).
- [Null-safe applications with Spring Boot 4 (Nov 12 2025)](https://spring.io/blog/2025/11/12/null-safe-applications-with-spring-boot-4/).

#### JEPs (relevant to maintainability)

- [JEP 261: Module System (JDK 9)](https://openjdk.org/jeps/261).
- [JEP 378: Text Blocks (JDK 15)](https://openjdk.org/jeps/378).
- [JEP 395: Records (JDK 16)](https://openjdk.org/jeps/395).
- [JEP 409: Sealed Classes (JDK 17)](https://openjdk.org/jeps/409).
- [JEP 441: Pattern Matching for switch (JDK 21)](https://openjdk.org/jeps/441).

### Foundational / authoritative

- **Martin, *Clean Code: A Handbook of Agile Software Craftsmanship*** (Prentice Hall 2008), ISBN 978-0-13-235088-4. Ch. 2 Meaningful Names (§1.2); ch. 3 Functions (§1.1); ch. 4 Comments (§1.3); ch. 5 Formatting (§1.4); ch. 7 Error Handling (§5.5); ch. 17 Smells and Heuristics.
- **Fowler, *Refactoring: Improving the Design of Existing Code*, 2nd edition** (Addison-Wesley 2018), ISBN 978-0-13-475759-9. Ch. 3 Bad Smells in Code (§3, §4); refactoring catalogue (§1.1, §1.5).
- **Martin, *Clean Architecture: A Craftsman's Guide to Software Structure and Design*** (Prentice Hall 2017), ISBN 978-0-13-449416-6. Part III Design Principles, chs 7-11 (§3); Part IV Component Principles, chs 13-14 (§2.4-2.5).
- **Visser, van Eck, van der Leek, Rigal, Wijnholds, *Building Maintainable Software — Ten Guidelines for Future-Proof Code, Java Edition*** (O'Reilly 2016), ISBN 978-1-4919-5352-5. Authoritative SIG guidelines with 4-star thresholds: ch. 2 Write Short Units (§2.3, §5.1); ch. 3 Write Simple Units (§2.2-2.3, §5.2); ch. 4 Write Code Once (§2.3, §5.2); ch. 5 Keep Unit Interfaces Small (§5.1); ch. 6 Separate Concerns (§4.1, §5.3); ch. 7 Couple Components Loosely (§2.4-2.5); ch. 11 Write Clean Code.
- **Koskela, *Effective Unit Testing: A Guide for Java Developers*** (Manning 2013), ISBN 978-1-93-518257-3. Part 2 Catalog of test smells (§5.6).
- **Bloch, *Effective Java*, 3rd edition** (Addison-Wesley 2018), ISBN 978-0-13-468599-1. Item 49 validate parameters; Item 55 Optional; Item 69-72 exceptions.
- Muñoz Barón, Wyrich, Wagner, "An Empirical Validation of Cognitive Complexity as a Measure of Source Code Understandability", *14th ACM/IEEE International Symposium on Empirical Software Engineering and Measurement* (ESEM 2020), [arXiv:2007.12520](https://arxiv.org/abs/2007.12520).
- Campbell, G. A., "Cognitive Complexity — A new way of measuring understandability" (SonarSource white paper, 2018).
- McCabe, T. J., "A Complexity Measure", *IEEE Transactions on Software Engineering* SE-2(4), 1976.
- Nygard, M. T., "Documenting Architecture Decisions" (blog, 2011) — canonical ADR template at [adr.github.io](https://adr.github.io/adr-templates/).
- [C4 Model](https://c4model.com/) — Brown, S., 4-level software architecture model.

### Tooling

- [SonarQube rule RSPEC-3776 (Cognitive Complexity)](https://rules.sonarsource.com/java/RSPEC-3776/).
- [SonarQube Quality Gates](https://docs.sonarsource.com/sonarqube-server/latest/user-guide/quality-gates/).
- [Error Prone bug patterns](https://errorprone.info/bugpatterns) — Google static analysis compiler plugin.
- [NullAway](https://github.com/uber/nullaway) — Uber null-pointer compile-time checker (<10 % build overhead).
- [SpotBugs bug descriptions](https://spotbugs.readthedocs.io/en/stable/bugDescriptions.html) — `EI_EXPOSE_REP`, `NP_NULL_ON_SOME_PATH`, `SE_BAD_FIELD`.
- [PMD Java design rules](https://pmd.github.io/pmd/pmd_rules_java_design.html) — `GodClass`, `CyclomaticComplexity`, `CouplingBetweenObjects`, `TooManyMethods`.
- [ArchUnit User Guide (v1.4.1, May 2025)](https://www.archunit.org/userguide/html/000_Index.html); [Release v1.4.0 notes (Feb 10 2025)](https://www.archunit.org/news/release/2025/02/10/release-v1.4.0.html).
- [PIT Mutation Testing](https://pitest.org/) — v1.22.0 (Nov 2025) on [GitHub releases](https://github.com/hcoles/pitest/releases).
- [Micrometer Naming](https://docs.micrometer.io/micrometer/reference/concepts/naming.html) — tag cardinality guidance.
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html) — liveness/readiness probes.
- [commitlint](https://github.com/conventional-changelog/commitlint) — Conventional Commits enforcer.

### Checklists and curated reviews

- [Google Testing Blog, *Flaky Tests at Google and How We Mitigate Them* (2016)](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html).
- [Martin Fowler, *Test Pyramid*](https://martinfowler.com/bliki/TestPyramid.html).
- [Oikonomou, *Software Testing Anti-Patterns*](https://blog.codepipes.com/testing/software-testing-antipatterns.html) — 13 catalogued anti-patterns.
- [Philipp Hauer, *Modern Best Practices for Testing in Java* (2019)](https://phauer.com/2019/modern-best-practices-testing-java/).
- [Victor Rentea, *Exception-Handling Guide in Java*](https://victorrentea.ro/blog/exception-handling-guide-in-java/).

### Industry case studies and explainers

- [Spring blog, *Null Safety in Spring apps with JSpecify and NullAway* (Mar 2025)](https://spring.io/blog/2025/03/10/null-safety-in-spring-apps-with-jspecify-and-null-away/).
- [Bal, *Package by Feature vs Package by Layer* (Medium)](https://medium.com/sahibinden-technology/package-by-layer-vs-package-by-feature-7e89cde2ae3a).
- [Reflectoring, *Maven BOM Usage*](https://reflectoring.io/maven-bom/).
- [Gradle, *Five Ways Out of Dependency Hell*](https://gradle.com/blog/five-ways-dependency-hell-maven/).
- [SonarSource, *5 Clean Code Tips for Reducing Cognitive Complexity*](https://www.sonarsource.com/blog/5-clean-code-tips-for-reducing-cognitive-complexity).
- [JavaPro, *Test Your Tests: Mutation Testing in Java with PIT* (Jan 2026)](https://javapro.io/2026/01/21/test-your-tests-mutation-testing-in-java-with-pit/).
- [Wikipedia — Software package metrics](https://en.wikipedia.org/wiki/Software_package_metrics) — public summary of Martin's Ca/Ce/I/A/D.

---

## 10. Gotchas — Common Wrong Assumptions

G-01. **"Cyclomatic and cognitive complexity measure the same thing"** — false. A flat `switch` with 10 cases has cyclomatic 10 but cognitive ~1 because it is flat and readable. Cognitive rewards flat code; cyclomatic only counts paths. See §2.1.

G-02. **"The 20-line function rule is canonical"** — no. Clean Code does not prescribe 20. Visser's empirically-grounded SIG guideline is **15 LOC**, with 4-star cut-offs at ≤ 6.9 % of LOC over 60. See §1.1, §2.3.

G-03. **"Boolean flag arguments are fine if well-named"** — false. A flag means the method does at least two things; split it (`renderForSuite` / `renderForSingleTest`). See §4.2.

G-04. **"Getters returning mutable collections are safe because Java is pass-by-value"** — false. The reference value *is* the shared object; callers mutate through it. Return `List.copyOf(...)` (SpotBugs `EI_EXPOSE_REP`). See §4.5.

G-05. **"`Optional<T>` makes a good field / parameter type"** — false. `Optional` exists to signal *absence on return*; using it as a field or parameter violates its contract (Effective Java Item 55). See §5.4.

G-06. **"JSpecify is still a proposal"** — false. JSpecify 1.0 released October 2024; Spring Framework 7 / Spring Boot 4.0.0 (Nov 20 2025) are fully migrated. See §7.2.

G-07. **"Field injection and constructor injection are equivalent"** — false. Field injection prevents `final`, hides dependencies, couples the class to the container, and masks circular refs. Always constructor-inject. See §7.3, §5.9.

G-08. **"`spring.main.allow-circular-references=true` is a valid long-term fix"** — no. Circular references are banned by default since Spring Boot 2.6 (2021), still in Spring Boot 4.0.0 (2025); the flag is a temporary escape hatch. Refactor. See §7.3, §5.9.

G-09. **"Log the exception, then rethrow"** — false. This produces duplicate stack traces across layers. Log once at the outer boundary (e.g. `@ControllerAdvice`), throw plain up to it. See §4.7, §5.5.

G-10. **"`catch (Exception e)` is acceptable to be safe"** — false. It swallows `InterruptedException`, `Error`, and forthcoming checked types; it disguises the type signature. Catch the most specific. See §5.5.

G-11. **"`verify()` on every collaborator is thorough testing"** — false. It locks the test into the current implementation; refactoring breaks it even when behaviour is preserved. Verify outcomes, not interactions. See §4.6, §5.6.

G-12. **"100 % line coverage means the tests are good"** — false. PIT surviving mutants show assertions that never validate outcomes. Mutation testing measures test quality better than coverage. See §6.1; PIT docs.

G-13. **"The test pyramid mandates 70/20/10"** — false. Fowler's original bliki post does not prescribe percentages; only the shape "many more low-level than high-level" (*many more*, not a fixed ratio). See §5.6, [Martin Fowler Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html).

G-14. **"PIT has a canonical 80 % kill-rate target"** — false. There is no universal threshold; PIT maintainers explicitly recommend configuring `mutationThreshold` by team risk. Source: JavaPro Jan 2026; PIT docs. See §5.6.

G-15. **"Short methods hurt performance"** — false, at the method-call level. Visser ch. 2: "JVM is very good at optimizing the overhead of method invocations." Do not sacrifice maintainability for micro-optimization unless a profiler proves it matters. See §2.3, §5.1.

G-16. **"PMD `TooManyMethods` default is 20"** — false. The PMD default is `maxmethods: 10` (non-trivial methods). See §6.1, [PMD design rules](https://pmd.github.io/pmd/pmd_rules_java_design.html).

G-17. **"All lines must be under 120 characters"** — there is no canonical primary source for 120. Google Java Style is 100 characters. Use the project's style but cite it (Checkstyle / EditorConfig). See §1.2.

G-18. **"`java.util.logging` MDC works across threads automatically"** — false. MDC is thread-local; context must be propagated into async executors, `CompletableFuture`, and virtual threads (ScopedValue / MDCContext). See §5.8, W3C Trace Context L2.

G-19. **"Unbounded tags in Micrometer are harmless"** — false. A tag like `uri=/users/12345` with 10 M distinct users creates 10 M time-series and can bring a Prometheus server down. Normalize to `uri=/users/{id}` or `NOT_FOUND`. See §5.8, Micrometer Naming.

G-20. **"ADRs belong in the wiki, not the repo"** — false. ADRs in `docs/adr/NNNN-title.md` travel with the code, are diff-reviewed, and survive tool migrations. The Nygard template (Status, Context, Decision, Consequences) is the canonical shape. See §7.6, §5.7.

G-21. **"The SIG guidelines are arbitrary consultancy numbers"** — false. They are derived from ISO/IEC 25010 + the SIG/TÜViT Evaluation Criteria, benchmarked on 7.1 billion lines of code, and recalibrated annually. Maintainability rated 4-star vs 2-star predicts **issue-resolution throughput roughly 2×**. See Visser BMS ch. 1.

G-22. **"Cognitive complexity is unvalidated"** — false. Muñoz Barón, Wyrich, Wagner (ESEM 2020), meta-analysis of ~24 000 evaluations over 427 code snippets: cognitive complexity positively correlates with comprehension time and subjective understandability. It is the first solely code-based metric with empirical support for understandability. See §2.1.
