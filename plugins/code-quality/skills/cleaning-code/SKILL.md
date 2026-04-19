---
name: cleaning-code
description: "Cleans up recently modified code: simplify, reduce nesting, remove redundancy, rename for intent, apply named refactoring moves, flag code smells, tighten error handling, keep tests clean. Runs proactively during any code change — feature implementation, bug fix, or refactoring session. Trigger phrases: 'clean this up', 'simplify', 'refactor', 'extract method', 'reduce nesting', 'replace null', 'rename this', 'smells like', 'make this more readable'. Goal: each touched file leaves in a more readable, more maintainable state than it arrived. NOT for writing new features from scratch; NOT for deep architecture redesign (use /architecting-code); NOT for language-specific style rules that belong in a linter."
---

# Clean Code

Evidence base: Robert C. Martin, *Clean Code: A Handbook of Agile Software Craftsmanship* (2008) and *Clean Architecture* (2017); Martin Fowler, *Refactoring: Improving the Design of Existing Code* (1999); cleancoder.com. Chapter references cite *Clean Code* unless noted otherwise.

## Load-bearing rule — maximum simplicity

**Boy Scout Rule**: leave every piece of code cleaner than you found it (*Clean Code*, Preface). Simple is the default; complexity earns its place against a specific constraint.

## Functions (Ch 3)

- Small — "first rule: functions must be compact; second rule: functions must be *even more* compact." Aim 5–10 lines; 20 is already large; blocks inside `if/else/while` should be one line, ideally a function call (Ch 3 "Small!" + "Blocks and Indenting", p. 34–35)
- One thing — operational test: try to extract another function whose name isn't a paraphrase of the body. If the extraction is meaningful, the original did more than one thing (Ch 3 "Do One Thing" / "One Level of Abstraction", p. 36)
- 0–2 arguments ideal; 3 requires justification; more → group into an object ("Function Arguments")
- No boolean flags — a flag argument hides two functions in one; split them ("Flag Arguments")
- No side effects — what the name doesn't promise, the body doesn't do ("Side Effects")
- Descriptive name — a long, clear verb beats a short, vague one ("Use Descriptive Names")
- Return the transformed value; don't mutate the argument — `Buffer transform(Buffer in)` beats `void transform(Buffer out)` (Ch 3 "Standard Unary Forms", p. 66)
- Separate command from query — a method either *does* or *answers*, never both (Ch 3 "Command Query Separation", p. 45; Fowler Ch 10 "Separate Query from Modifier", p. 282)
- Treat parameters as `final`; if you need to change the value, copy into a local (Fowler Ch 6 "Remove Assignments to Parameters", p. 144)

## Names (Ch 2)

- Reveal intent — the name answers *why* this variable exists and *how* it is used
- Pronounceable — `generationTimestamp`, not `genymdhms`
- No type encodings — `name`, not `strName` / `m_name` / `userObj`
- Named constants — `DAYS_IN_WEEK` beats the literal `7` (magic numbers have no search anchor)
- Searchable — the identifier is distinctive enough that grep finds exactly the intended use
- Don't disinform — never reuse `l`/`O` (look like 1/0); never name a thing `accountList` unless it is a `List` (Ch 2 "Avoid Disinformation", p. 19–20)
- Pick one word per concept — `fetch` OR `retrieve` OR `get`, not all three (Ch 2 "Pick One Word per Concept", p. 26)
- Class names are nouns; method names are verbs — a `Manager` / `Processor` class name hints at an SRP violation (Ch 2 "Class Names" / "Method Names", p. 25)

## Conditions (Ch 3, §"Blocks and Indenting")

- Extract complex booleans into well-named variables or helpers — `canDrive = isAdult && hasLicense` beats nested logic
- Prefer positive form — `isValid` beats `!isNotValid`
- Reads like prose — the body of an `if` should sound natural in English
- Max 2 levels of `if` nesting — deeper → extract a helper
- Encapsulate boundary expressions in named locals — compute `int nextLevel = level + 1` once (Ch 17 G33, p. 343)
- Replace nested conditionals with guard clauses — early `return` for abnormal cases; reserve `if/else` for branches of equal weight (Fowler Ch 9 "Replace Nested Conditional with Guard Clauses", p. 253)

## Polymorphism over long if / switch chains (Ch 6, Ch 17 smell G23)

- A switch or if-chain that branches on type is a polymorphism opportunity
- Prefer a class hierarchy when branches are stable and recur across the codebase; keep the switch when branches are local and short
- Open–Closed Principle — open for extension, closed for modification (*Clean Architecture* Ch 8)

## Comments (Ch 4)

Good code speaks; comments compensate for places where it doesn't.

Comments worth writing:
- Why — business / domain reason for a non-obvious decision
- Warning of consequence — what the reader can't infer from the code
- TODO — explicit temporary note with ownership

Cut on sight:
- Commented-out code — version control already has it
- Comments that parrot the code (`i++; // increment i`)
- Obvious comments (`// constructor`)
- Journal / changelog headers — `git log` is authoritative

## Code structure (Ch 5 — "Formatting")

- Variables close to first use, not at the top of the function
- Related code grouped; blank line separates concerns
- Public methods on top, private below (stepdown rule)
- Lines < 120 characters (empirically ~1% of lines exceed 80; 120 is a firm upper bound, not a target)
- One abstraction level per function — don't mix high-level flow with low-level mechanics
- Dependent functions adjacent — if A calls B, place B right after A

## Classes (Ch 10)

- Small — a class's description fits ~25 words without using "if", "and", "or", "but"; connectives betray multiple responsibilities (SRP) (Ch 10 "Classes Should Be Small!", p. 167)
- Few fields — typically 2–5 instance variables; more usually signals multiple responsibilities
- Hide data — expose behavior, not state
- High cohesion — most methods touch most fields
- Not hybrids — either an object (behavior, hidden data) or a data structure (exposed data, no behavior); don't mix (Ch 6 "Objects and Data Structures")
- Local variables shared across 3+ helpers → promote to fields; this often reveals a new class waiting to be extracted (Ch 10 "Maintaining Cohesion Results in Many Small Classes", p. 170)
- Prefer private — only relax to package-private to satisfy a same-package test; never start with protected (Ch 10 "Encapsulation", p. 165; Fowler Ch 10 "Hide Method", p. 305)

## Error handling (Ch 7)

- Write the `try-catch-finally` first when adding code that can throw; the `try` defines the scope the caller can rely on (Ch 7, p. 133)
- Wrap the try-body in its own function; `try` must be the first token and nothing follows `catch/finally` — error handling is one thing (Ch 3 "Extract Try/Catch Blocks", p. 46)
- Prefer unchecked exceptions — checked exceptions violate OCP by rippling `throws` through every caller (Ch 7 "Use Unchecked Exceptions", p. 135)
- Every exception carries context — operation, failure type, inputs — enough for a useful log entry (Ch 7 "Provide Context with Exceptions", p. 136)
- Wrap third-party APIs in your own exception type — one type per boundary, not five from the vendor (Ch 7 "Define Exception Classes in Terms of a Caller's Needs", p. 136–137)
- Never return null; never pass null — return an empty collection, a Special Case object, or throw (Ch 7 "Don't Return Null" / "Don't Pass Null", p. 139–141)
- Throw instead of returning error codes — error-code enums become dependency magnets forcing every consumer to recompile (Ch 3 "Prefer Exceptions to Returning Error Codes", p. 46; Fowler Ch 10 "Replace Error Code with Exception", p. 312)

## Tests (Ch 9 + Fowler Ch 4)

- Test code is production code — same cleanliness standard; dirty tests rot first, get deleted, then production rots (Ch 9 "Keeping Tests Clean", p. 152)
- One concept per test — a test that checks three things forces the reader to guess which failed (Ch 9 "Single Concept per Test", p. 161)
- F.I.R.S.T. — Fast, Independent, Repeatable, Self-validating, Timely (Ch 9, p. 162)
- Bug reported → write a failing unit test first; bugs cluster, so exhaustively test around the fix (Fowler Ch 4, p. 110; Ch 17 T6, p. 355)
- Test boundary conditions explicitly — empty, off-by-one, zero, negative, past-end — don't trust intuition (Fowler Ch 4, p. 112; Ch 17 T5, p. 355)

## Smells quick-lookup (Fowler Ch 3 + Clean Code Ch 17)

When you see the smell, apply the fix.

| Smell | Fix |
|---|---|
| Long Method | Extract Method until each name replaces a comment (Fowler Ch 3, p. 87) |
| Large Class | Extract Class (group fields sharing a prefix); Extract Subclass for conditional subsets (Fowler Ch 3, p. 88) |
| Long Parameter List | Introduce Parameter Object; Preserve Whole Object; Replace Parameter with Method (Fowler Ch 3, p. 89) |
| Divergent Change | Extract Class per axis of change (Fowler Ch 3, p. 90) |
| Shotgun Surgery | Move Method / Move Field to consolidate scattered related pieces (Fowler Ch 3, p. 90) |
| Feature Envy | Move Method to the class whose data it uses most (Fowler Ch 3, p. 91) |
| Data Clumps | Extract Class when 3+ items travel together; confirm with the delete-one-item test (Fowler Ch 3, p. 92) |
| Primitive Obsession | Replace Data Value with Object (`Money`, `Phone`, `DateRange`) (Fowler Ch 3, p. 92) |
| Switch Statements | Replace Conditional with Polymorphism; bury the switch in a factory (Fowler Ch 3, p. 93; Ch 17 G23, p. 338) |
| Parallel Inheritance | Collapse parallelism — one hierarchy references the other (Fowler Ch 3, p. 94) |
| Lazy Class | Inline Class / Collapse Hierarchy (Fowler Ch 3, p. 94) |
| Speculative Generality | Delete unused abstractions, parameters, hook methods (Fowler Ch 3, p. 94) |
| Temporary Field | Extract Class to hold the field + its conditional code (Fowler Ch 3, p. 95) |
| Message Chains | Hide Delegate at the best point in the chain (Fowler Ch 3, p. 95) |
| Middle Man | Remove Middle Man when >half its methods delegate straight through (Fowler Ch 3, p. 96) |
| Inappropriate Intimacy | Move Method/Field or Change Bidirectional to Unidirectional (Fowler Ch 3, p. 96) |
| Alternative Classes w/ Different Interfaces | Rename to unify, then Extract Superclass (Fowler Ch 3, p. 97) |
| Data Class | Move methods that manipulate the data into the class (Fowler Ch 3, p. 98) |
| Refused Bequest | Push Down Method/Field; if the interface is refused, Replace Inheritance with Delegation (Fowler Ch 3, p. 98) |
| Comments (as smell) | Extract Method with the comment text as the method name (Fowler Ch 3, p. 99) |
| Dead Code | Delete immediately; VCS keeps it (Ch 17 G9, p. 330) |
| Hidden Temporal Coupling | Make each method consume the previous return value (Ch 17 G31, p. 342) |

## Named refactoring moves

Vocabulary for describing cleanups precisely, instead of "extract a helper."

- **Extract Method** — a block deserves a name (Fowler Ch 6, p. 124)
- **Inline Variable / Inline Temp** — when a temp adds no clarity over the expression (Fowler Ch 6, p. 132)
- **Replace Temp with Query** — replace a temp with a method call to eliminate local state (Fowler Ch 6, p. 133)
- **Introduce Explaining Variable** — when Extract Method is blocked by too many locals (Fowler Ch 6, p. 137)
- **Split Temporary Variable** — one temp assigned twice for different reasons → two temps, each `final` (Fowler Ch 6, p. 141)
- **Introduce Parameter Object** — when 3+ parameters travel together (Fowler Ch 10, p. 297)
- **Preserve Whole Object** — pass the object, not three of its fields (Fowler Ch 10, p. 291)
- **Replace Conditional with Polymorphism** — when the switch/if branches on type (Fowler Ch 9, p. 258)
- **Replace Nested Conditional with Guard Clauses** — early `return` for abnormal cases (Fowler Ch 9, p. 253)
- **Encapsulate Field** — make public fields private; expose accessors only when needed (Fowler Ch 8, p. 212)
- **Extract Class** — split a class by responsibility; prefix-sharing fields are the hint (Fowler Ch 7, p. 161)
- **Move Method** — a method uses another class's data more than its own (Fowler Ch 7, p. 154)
- **Rename Method** — keep trying until the name reads as a sentence (Fowler Ch 10, p. 277)
- **Hide Delegate** — replace `a.getB().getC()` with `a.getC()` on the server (Fowler Ch 7, p. 168)
- **Replace Magic Number with Symbolic Constant** — any literal whose meaning isn't self-evident (Fowler Ch 8, p. 211; Ch 17 G25, p. 339)
- **Replace Error Code with Exception** — throw, don't return `-1` (Fowler Ch 10, p. 312)
- **Introduce Null Object** — replace repeated null checks with a no-op subclass (Fowler Ch 9, p. 262)

## Design principles

- Configuration at the top — constants and settings collected at the highest level of the system
- Dependency Injection — dependencies flow in from outside, not constructed in place (*Clean Architecture* Ch 11)
- Law of Demeter — an object speaks only to its immediate neighbours (Ch 6)
- Isolate concurrency — keep threading primitives in a narrow layer, not scattered (Ch 13)
- Make things configurable on the second concrete need, not on speculation
- Defer decisions to the last responsible moment — decide when you have the information, not when you have the idea (Ch 11 "Optimize Decision Making", p. 197)
- Performance: first make it clean, then measure, then optimize the <10% the profiler flags (Fowler Ch 2 "Refactoring and Performance", p. 79)

## Commit-ready checklist

- [ ] Function under 20 lines, doing one thing?
- [ ] At most 2 arguments, no boolean flags?
- [ ] Every name reveals intent; every constant named?
- [ ] Complex booleans extracted; `if` nesting ≤ 2 levels?
- [ ] Duplication beyond the third occurrence consolidated (Rule of Three)?
- [ ] No commented-out code, no restating comments?
- [ ] Code reads like prose?
- [ ] No null returned or passed; third-party APIs wrapped in local exception types?
- [ ] Tests written/updated and clean; one concept each; F.I.R.S.T.?
- [ ] Diff free of smells from the quick-lookup table?

## Golden rule

> "Any fool can write code that a computer can understand. Good programmers write code that humans can understand." — Martin, *Clean Code* Ch 1

When in doubt, choose the simpler option.

## Gotchas

G-01. The Boy Scout Rule applies to code you already have a reason to touch. Scrubbing an untidy module you haven't edited converts zero-risk code into merge conflicts — gatekeep cleanups to files in the current diff.
G-02. "One thing" scales with the call site's abstraction level. A top-level handler "doing one thing" is still ten lines of orchestration; a leaf helper "doing one thing" is three lines of arithmetic. Apply the rule at each level, not uniformly.
G-03. Polymorphism isn't always cleaner than a switch — if branches are stable, local, and short, a single switch reads more clearly than three files and an interface. Extract a hierarchy only when branches recur or are expected to grow.
G-04. "No comments" is a misread — the rule is *no comments that duplicate the code*. Rationale comments, warnings, and TODOs still earn their place; code explains *what*, comments explain *why* (Ch 4: "Good Comments" vs "Bad Comments").
G-05. Line-count and field-count limits are heuristics, not contracts. A 30-line cohesive function can be clearer than five 6-line functions with confusing names. Lengths flag a smell; the smell must be confirmed by reading the code.
G-06. Renames in a refactor session mutate semantics as far as the type system allows — run the test suite after every rename to catch the cases the compiler doesn't.
G-07. "Cleaner" is a property of the reader, not the writer — the diff must be easier for the next person to understand than the previous version. If the only person who finds the rewrite clearer is you, it's a restyle, not a cleanup.
G-08. Rule of Three — the first time, do it. The second time, wince and duplicate. Only on the third occurrence refactor. Premature extraction pays for flexibility you may never need and locks in the wrong abstraction (Fowler Ch 2, p. 66).
G-09. Two-hats discipline — wear only one at a time. "Adding a feature" (new tests, no structural change) and "refactoring" (no new tests, no behaviour change) have different failure modes; mixing them breaks the invariant that tests pass after every step (Fowler Ch 2, p. 63).
G-10. Guard clauses contradict single-exit. The 1970s "structured programming" rule said one `return` per function; Fowler says early returns for abnormal cases read better. Follow guard clauses in new code; don't rewrite existing single-exit code just to conform (Fowler Ch 9 "Remove Control Flag", p. 248).
G-11. Checked-vs-unchecked is language-scoped. Martin's "prefer unchecked" is for *application* code in Java. In a public library, or in languages without checked exceptions (Python, C#), the rule is vacuous or inverted — don't propagate it blindly (Ch 7, p. 135; Fowler Ch 10, p. 314).
G-12. "Don't publish interfaces early" (Fowler) tensions with "design as plugin architecture" (Clean Architecture). For code-level cleanup of internal modules, keep ownership of the API so renames are cheap; cross-team boundaries belong to `/architecting-code`, not here (Fowler Ch 2, p. 73).
