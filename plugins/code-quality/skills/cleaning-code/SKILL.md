---
name: cleaning-code
description: "Cleans up recently modified code: simplify, reduce nesting, remove redundancy, enforce project style. Use this skill proactively whenever you write, refactor, or modify code — not just when the user explicitly asks. Trigger on: any code change during a feature implementation, bug fix, or refactoring session; phrases like 'clean this up', 'simplify', 'refactor'. The goal is to leave every piece of code you touch cleaner than you found it. NOT for writing new features from scratch; NOT for deep architecture redesign; NOT for language-specific style rules that belong in a linter."
---

# Clean Code

Evidence base: Robert C. Martin, *Clean Code: A Handbook of Agile Software Craftsmanship* (2008), *Clean Architecture* (2017), and cleancoder.com. Chapter references cite *Clean Code* unless noted otherwise.

## Load-bearing rule — maximum simplicity

**Boy Scout Rule**: leave every piece of code cleaner than you found it (*Clean Code*, Preface). Simple is the default; complexity earns its place against a specific constraint.

## Functions (Ch 3)

- Small — "the first rule of functions is that they should be small; the second rule is smaller than that." Aim 5–10 lines; 20 is already large
- One thing — the function does exactly one thing at a single level of abstraction ("Do One Thing")
- 0–2 arguments ideal; 3 requires justification; more → group into an object ("Function Arguments")
- No boolean flags — a flag argument hides two functions in one; split them ("Flag Arguments")
- No side effects — what the name doesn't promise, the body doesn't do ("Side Effects")
- Descriptive name — a long, clear verb beats a short, vague one ("Use Descriptive Names")

## Names (Ch 2)

- Reveal intent — the name answers *why* this variable exists and *how* it is used
- Pronounceable — `generationTimestamp`, not `genymdhms`
- No type encodings — `name`, not `strName` / `m_name` / `userObj`
- Named constants — `DAYS_IN_WEEK` beats the literal `7` (magic numbers have no search anchor)
- Searchable — the identifier is distinctive enough that grep finds exactly the intended use

## Conditions (Ch 3, §"Blocks and Indenting")

- Extract complex booleans into well-named variables or helpers — `canDrive = isAdult && hasLicense` beats nested logic
- Prefer positive form — `isValid` beats `!isNotValid`
- Reads like prose — the body of an `if` should sound natural in English
- Max 2 levels of `if` nesting — deeper → extract a helper

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
- Lines < 120 characters
- One abstraction level per function — don't mix high-level flow with low-level mechanics
- Dependent functions adjacent — if A calls B, place B right after A

## Classes (Ch 10)

- Small — Single Responsibility Principle (*Clean Architecture* Ch 7)
- Few fields — typically 2–5 instance variables; more usually signals multiple responsibilities
- Hide data — expose behavior, not state
- High cohesion — most methods touch most fields
- Not hybrids — either an object (behavior, hidden data) or a data structure (exposed data, no behavior); don't mix (Ch 6: "Objects and Data Structures")

## Design principles

- Configuration at the top — constants and settings collected at the highest level of the system
- Dependency Injection — dependencies flow in from outside, not constructed in place (*Clean Architecture* Ch 11)
- Law of Demeter — an object speaks only to its immediate neighbours (Ch 6)
- Isolate concurrency — keep threading primitives in a narrow layer, not scattered (Ch 13)
- Make things configurable on the second concrete need, not on speculation

## Commit-ready checklist

- [ ] Function under 20 lines, doing one thing?
- [ ] At most 2 arguments, no boolean flags?
- [ ] Every name reveals intent; every constant named?
- [ ] Complex booleans extracted; `if` nesting ≤ 2 levels?
- [ ] No duplication (DRY)?
- [ ] No commented-out code, no restating comments?
- [ ] Code reads like prose?

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
