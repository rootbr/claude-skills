---
name: architecting-code
description: "Structural and boundary-level design rules: dependency direction, layering, component cohesion/coupling, plugin architecture, use-case isolation, boundary patterns, mutable-state segregation, concurrency isolation at the architectural seam. Applies when deciding where code lives, whether to split a module, how to cross a boundary, or how to keep a framework/DB/web layer replaceable. Trigger phrases: 'where should X live', 'design the boundary', 'package structure', 'should this be its own service', 'separate the core from the framework', 'dependency direction', 'plugin architecture', 'clean architecture', 'segregate mutable state', 'isolate threading'. NOT for line-level cleanup (use /cleaning-code); NOT for writing greenfield features or framework configuration; NOT for method/class-granularity refactoring moves."
---

# Architecting Code

Evidence base: Robert C. Martin, *Clean Architecture: A Craftsman's Guide to Software Structure and Design* (Pearson, 2017). Chapter references cite *Clean Architecture* unless noted.

## Load-bearing rule — architecture is measured by cost of change

Architecture is "the significant design decisions that shape a system, where significant is measured by cost of change" (Booch, via Henney, Foreword). Optimise the kinds of changes the system actually experiences to be cheap, not rare — and size the architecture to the problem, not to a textbook.

## The Dependency Rule

- Source-code dependencies always point toward higher-level policy; details implement interfaces owned by the policy layer — the DIP mechanism that makes the rule enforceable (PART III SOLID summary; Appendix A ROSE)
- When crossing a boundary, invert the dependency so the caller holds an interface the detail implements (Appendix A, Union Accounting)
- High-level policy compiles with zero knowledge of UI, database, or framework types — only of the abstractions it owns (Ch 17; Appendix A 4-Tel Vectorization)

## SOLID at the module level

- **SRP** — every module has exactly one reason to change, defined by one actor/stakeholder (Ch 7; PART III summary)
- **OCP** — add new behaviour by adding new code, not by editing existing, already-tested code (Ch 8; PART III summary)
- **LSP** — a subtype's contract must allow substitution anywhere the supertype is used, including in tests (Ch 9)
- **ISP** — never force a module to depend on interface members it doesn't use; unused methods still pin recompile/redeploy (Ch 10)
- **DIP** — high-level policy never imports concretions; details implement policy-owned interfaces; factories sit at the boundary (Ch 11)

## Component cohesion (what belongs together)

- **REP — Reuse/Release Equivalence** — the granule of reuse is the granule of release; unreleased code is not reusable (Ch 13)
- **CCP — Common Closure** — classes that change together belong in the same component (component-level SRP) (Ch 13)
- **CRP — Common Reuse** — classes that are used together belong in the same component; never drag unrelated classes along (component-level ISP) (Ch 13)

## Component coupling (how components relate)

- **ADP — Acyclic Dependencies** — the component dependency graph must have no cycles; break cycles by inverting a dependency or extracting a shared component (Ch 14)
- **SDP — Stable Dependencies** — depend toward stability: volatile components may depend on stable ones, never the reverse. Measure instability `I = fan-out / (fan-in + fan-out)`; arrows must flow from higher-I to lower-I (Ch 14)
- **SAP — Stable Abstractions** — a stable component must also be abstract; a volatile component must be concrete. The zone of pain (stable + concrete) and the zone of uselessness (volatile + abstract) are architectural smells (Ch 14)

## Boundaries

- Defer database, web-framework, and UI decisions behind an interface until you actually need to decide (Ch 15 "Keeping Options Open"; Ch 17 "Drawing Lines")
- The database is a detail — SQL and schema live behind a gateway, never smeared through business code (Ch 30; Appendix A VRS)
- The web is a detail — business rules never import from the web framework (Ch 31)
- Frameworks are tools, not ways of life — do not inherit from framework base classes in domain code (Ch 32 "Asymmetric Marriage")
- Isolate hardware and device control behind an abstract interface; hardware changes must not ripple into business rules (Ch 29; Appendix A 4-Tel modem)
- Isolate the use-case layer: it orchestrates entities but knows nothing about delivery, framework, or persistence (Ch 16, Ch 20)
- Pass data across boundaries as simple request/response DTOs — never pass entities out of the use-case layer (Ch 20)
- Partial / one-dimensional boundaries and façades are a cheap way to buy optionality — full bidirectional boundaries have costs too (Ch 24)
- A boundary is about data flow, not call-site coupling — pick the decoupling mode (source, deployment, service) per boundary, not for the whole system (Ch 18)

## Plugin architecture

- Build the system as a plugin architecture — high-level policy ships without knowing which UI, DB, or framework plugs in (Ch 17; Appendix A 4-Tel Vectorization)
- Payoff: a single feature can ship without rebuilding the other N components
- Polymorphic dispatch is the structural mechanism — the interface goes in the policy module, the implementation in the detail module (Ch 5; Appendix A Vectorization)
- Recognise cleaving lines early — when two physical concerns (e.g. dialing vs. measuring) live in one module, separating them later costs a fortune (Appendix A pCCU)

## Structure

- **Screaming Architecture** — the top-level package reveals the use cases (`Billing/`, `Inventory/`), not the framework (`controllers/`, `models/`) (Ch 21)
- Prefer "package by component" over "package by layer" — each component exposes its public API and hides controllers, gateways, and entities behind it (Ch 34)
- Encode boundaries with the language's access modifiers (or build-time dependency rules such as ArchUnit) — folder conventions alone are not boundaries (Ch 34)

## Testability

- **Humble Object** — split anything hard to test into a thin humble wrapper (UI, driver, view) plus a testable core (Ch 23)
- Design for testability as a first-class client — fragile tests are a symptom of architectural coupling, not of flaky tests (Ch 28)
- Tests consume the same stable abstractions as production; do not let tests `new` the database or the web framework (Ch 28 "Testing API")

## Main and wiring

- **Main** is the dirtiest component — concentrate concrete wiring (DI container, framework bootstrapping, env reading) there and only there. Everything else depends only on abstractions (Ch 26)
- Factories belong at boundaries — the policy module asks a factory interface for an instance; the detail module implements the factory (Ch 11)

## Services

- Services — even micro-services — are components, subject to the same cohesion/coupling rules as in-process modules (Ch 27)
- Service boundaries must be message-based, not pointer-sharing or shared mutable state — otherwise the "services" are one distributed monolith (Appendix A CDS 3DBB)
- Cross-cutting features shotgun-change every "independent" service unless decomposed by feature (Ch 27 "The Kitty Problem")

## Concurrency and mutation

- Concentrate thread/process-switching logic (locks, semaphores, reentrancy) into one small supervisor module; keep business rules thread-unaware (Appendix A SAC / BOSS scheduler)
- Prefer immutability by default; segregate mutable state into a small, well-fenced component (Ch 6 "Segregation of Mutability")

## Commit-ready checklist

- [ ] Does a change touch 3+ packages? If yes, Shotgun Surgery — check CCP
- [ ] Do any business-rule files import from the web framework, ORM, or UI toolkit?
- [ ] Can this component be released (versioned, tagged) without dragging siblings?
- [ ] Do all dependency arrows point from volatile to stable?
- [ ] Is each use case testable without the framework or the database running?
- [ ] Are boundary crossings passing DTOs, not entities?
- [ ] Is `Main` the only place concrete classes are constructed?

## Golden rule

> "The goal of the architect is to minimize the human resources required to build and maintain the required system." — Martin, *Clean Architecture* Ch 1

When in doubt, keep deferrable decisions deferrable.

## Gotchas

G-01. Architecture is a hypothesis, not a frozen artefact — one chosen once and defended forever drifts out of sync with reality (Henney / Gilb, Foreword).
G-02. "Just-enough design up front with plenty of refactoring" is the only stable equilibrium — Big Design Up Front and zero-design both failed (Gorman, Afterword).
G-03. Size the architecture to the problem — ROSE failed in part because of over-architecture with many more layers than needed; extra layers are only free on PowerPoint (Appendix A ROSE).
G-04. Micro-services don't make boundaries automatically honest — if services share mutable state or pass pointers, they aren't independent. Service boundaries must be message-based (Appendix A CDS 3DBB; Ch 27 "Kitty Problem").
G-05. Forking a codebase to serve a new market almost never works — SAC Europe failed three times. Extract a configuration/boundary seam first, then vary behind it (Appendix A SAC "Europe").
G-06. "You can't make a reusable framework until you first make a usable framework" — ETS wasted a year on 45,000 lines of speculative framework. Build it in concert with *several* reusing applications, not in isolation (Appendix A Architects Registry Exam).
G-07. Locked-down / officially-immutable code is a symptom, not a fix — if nobody dares change a module, the module is already broken. Restore it to something humans can edit (Appendix A Dispatch Determination).
G-08. Several architectures can be valid for the same problem — two developers independently solved DLU/DRU with a pipeline and a per-terminal monolith, both worked. Architectural opinion is not architectural necessity (Appendix A DLU/DRU).
G-09. "Cost of change" is the metric — if a line of code costs $500 to change, change isn't happening. The architect's job is to keep the marginal cost low enough that the business can afford the change it actually needs (Gorman, Afterword).
G-10. Architecture is independent of language and hardware era — the same rules apply to a 16 K embedded system and a cloud service; modern hardware just makes following the rules affordable, not optional (Preface).
