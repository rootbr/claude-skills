# Project-Specific Reviewer — Meta-Checklist for Auditing `project_context`

## Table of Contents
1. What a well-formed `project_context` looks like -- numbered invariants, definitions dictionary, scale numbers, hot-path list, lock hierarchy, API-compatibility policy, cross-references
2. Anatomy of a good invariant -- subject, rule, scale multiplier, forbidden state, evidence, correction recipe (shape derived from Vernon's `c = a + b` + ATAM stimulus-response-measure scenario)
3. Invariant anti-patterns with bad / good pairs -- undefined terms, unscaled numbers, aspirational "should", overlapping rules, contradictory invariants, unnumbered entries, implicit rules
4. Audit procedure for a diff against `project_context` -- diff discovery, per-file traversal, per-invariant check, old-vs-new comparison via `git show $base:`, scale-multiplier math, cross-invariant contradiction check
5. Non-overlap with other reviewers -- one row per reviewer (concurrency / security / maintainability / performance / reliability); what they own; what this reviewer stays silent about
6. Output-format discipline -- every finding cites invariant number; suggested-fix code; severity triage; empty case handling
7. Modern architecture practices that ground invariants -- ADR pattern, ArchUnit fitness tests, hexagonal / Ports & Adapters boundaries, ATAM utility tree, FMEA, evolutionary-architecture fitness functions
8. Research Questions to Cross-Check Against Sources -- 52 questions (buckets A-J), each mapped to section and source
9. Sources -- normative / foundational / tooling / checklists / industry explainers
10. Gotchas -- 15 common wrong assumptions with stable IDs (G-01..G-15)

---

## 1. What a well-formed `project_context` looks like

The project-specific reviewer audits a diff against whatever rubric the caller
supplies in `project_context`. Its quality ceiling is set by the rubric quality.
Before enforcing anything, the reviewer must first recognise whether the
supplied rubric is auditable. A well-formed `project_context` contains the
seven artefacts below. When any are missing, the reviewer flags the gap instead
of guessing (§1.8).

### 1.1 Numbered invariants with stable IDs

Every rule is numbered (`Inv 1`, `Inv 2`, …) and does not renumber across
revisions. The ID is the handle every finding cites. Source: Nygard's ADR
template ([Status field, Nygard 2011](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions))
uses stable IDs for the same reason: a superseded decision keeps its number so
cross-references continue to resolve. Vernon numbers his aggregate rules-of-
thumb 1-4 (*IDDD* pp. 430-445) for the same cross-reference reason.

### 1.2 Definitions dictionary (ubiquitous language)

Every domain term that is load-bearing in a rule has one definition. `hot path`,
`steady state`, `cold start`, `tenant`, `page-cache miss` - all must be pinned
down once and only once. This is Evans's *ubiquitous language* applied at the
rule layer. If two invariants use the same word with different meanings, the
reviewer flags the collision (§3, G-12).

### 1.3 Scale numbers tied to each operational rule

Rules that depend on traffic shape carry the number: `onEvent() at ~50 M
invocations/day`, `heap budget 4 GiB steady`, `p99 latency ≤ 5 ms`. Without a
number the rule is un-auditable; a patch that adds a trivially-looking field
may push the invariant over the edge. Source: Glass *Facts and Fallacies*
Fact 5 documents that tool-hype promises "order of magnitude" gains while
measurement finds 5-35 % - the only defence is a cited number, not a feeling.

### 1.4 Hot-path list

Methods, classes, or packages that are on the critical path -- where changes
must clear extra bars (allocation, lock acquisition, exception-throw). Glass
Fact 49 documents the 80/20 defect-clustering (Endres 1975; Boehm-Basili 2001);
hot-path lists formalise which 20 % need the tightest scrutiny.

### 1.5 Lock hierarchy or synchronization order (if concurrent)

An explicit partial order of locks, monitors, or other coordination primitives
that code must obey to avoid deadlock. The project-specific reviewer uses this
only to determine whether the diff has introduced a new lock that must be
placed in the hierarchy; actual deadlock analysis belongs to the
concurrency-reviewer (§5).

### 1.6 API-compatibility policy

What counts as a breaking change (parameter reordering, enum-constant removal,
exception-type widening, nullable-to-non-null tightening). This enables the
old-vs-new comparison procedure in §4.3.

### 1.7 Cross-references between invariants

Each invariant names the others it depends on or supersedes: `Inv 7 refines
Inv 3`, `Inv 12 replaces Inv 5`. These links make the rubric navigable and let
the reviewer detect contradictions (§3.6). Source: MADR template
([adr/madr](https://github.com/adr/madr)) and ADR "Status: superseded-by"
field ([Nygard 2011](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions))
use the same pattern for architecture decisions.

### 1.8 When `project_context` is malformed

If any of §1.1-§1.7 are missing or ambiguous, the reviewer emits a `PROJ-META`
finding explaining the gap and suggesting a fix-shape, then proceeds to audit
the invariants that are well-formed. The reviewer never fabricates an
invariant from `design_intent`; intent is context, not a rule source (G-02).

---

## 2. Anatomy of a good invariant

A single invariant has six parts. The shape combines Vernon's canonical
invariant form (*IDDD* ch. 10 p. 430: "Invariant is a business rule that
always preserves its consistency … transactional consistency is instantaneous
and atomic") with the ATAM quality-attribute scenario structure (Bass,
Clements, Kazman, *Software Architecture in Practice* 3rd/4th ed. - source,
stimulus, artifact, environment, response, response measure).

| Part | What it is | Derived from |
|---|---|---|
| 1. **ID + subject** | `Inv 7 - onEvent allocation-free` | Nygard ADR Title; Vernon rule-of-thumb numbering |
| 2. **Rule** | precise constraint on state / behaviour | Vernon p. 430 `c = a + b` form |
| 3. **Scale multiplier** | numeric context making the rule auditable (`~50 M inv/day`) | ATAM response-measure; Glass Fact 5 |
| 4. **Forbidden state** | explicit negation the reviewer scans for (`any `new` expression inside the hot-path body`) | Vernon p. 430 (c ≠ 5 ⇒ violated) |
| 5. **Evidence handle** | where the rule is enforced (unit test name, ArchUnit rule name, JFR event, runbook page, benchmark) | ArchUnit user guide - rules are executable |
| 6. **Correction recipe** | the preferred fix pattern when violated (`reuse a pooled buffer`, `move work off hot path via StructuredTaskScope`) | Ford, Parsons, Kua 2017 (fitness functions guide remediation) |

Example of a complete invariant in this shape:

```
Inv 7 — allocation-free onEvent
  Subject: com.example.bus.Dispatcher#onEvent(Event)
  Rule: hot path must not execute `new`, autoboxing, varargs-spread,
        lambda-capture with captured outer state, or String.format.
  Scale: ~50 M invocations/day peak; p99 target 120 µs.
  Forbidden state: any bytecode NEW / NEWARRAY / INVOKEVIRTUAL with
        boxed-primitive signature inside Dispatcher#onEvent.
  Evidence: jcstress stress test AllocProfileTest; JFR event
        jdk.ObjectAllocationInNewTLAB recorded in canary.
  Correction: pre-allocate in constructor; use primitive-specialised
        collections (Eclipse Collections) or `int`-keyed arrays.
```

An invariant missing any of parts 3-5 is a meta-bug the reviewer surfaces
(§3). An invariant missing part 6 is tolerable but weaker; the reviewer may
still enforce it, but findings will omit the suggested fix block.

---

## 3. Invariant anti-patterns

Each anti-pattern below explains a common failure mode in a `project_context`
and gives a concrete `bad:` / `good:` rewrite pair.

### 3.1 Undefined term (vocabulary leak)

The rule uses a word that the context does not define elsewhere. Evans's
ubiquitous-language principle: every domain word has one meaning inside one
bounded context.

```
bad:  Inv 3 — the hot path must not block.
good: Inv 3 — methods listed in §hot-paths (Dispatcher#onEvent,
             RequestLoop#tick) must not execute any `Thread.sleep`,
             `Object.wait`, `LockSupport.park`, `Socket.read` without
             a timeout, or `CompletableFuture.get()` without a timeout.
```

Source: Evans *DDD* ch. 2 (ubiquitous language); Vernon *IDDD* ch. 1.

### 3.2 Unscaled number

A rule whose enforcement depends on traffic shape must carry the number.
Without it, a patch that adds a 16-byte allocation looks equivalent to one
that adds a 16-KiB one.

```
bad:  Inv 5 — avoid large objects on the heap.
good: Inv 5 — no single allocation inside Dispatcher#onEvent may exceed
             256 bytes retained. Called ~50 M times/day; any breach
             produces ~3 GiB/h of churn and triggers G1 Mixed-GC.
```

Source: Glass Fact 5 (documented gap between hype and measured wins); Bass-
Clements-Kazman ATAM quality-attribute scenario mandates a response measure.

### 3.3 Aspirational "should" without enforcement

`should`, `try to`, `prefer` aren't auditable. Either the rule is enforced
and becomes `must`, or it isn't a rule and the reviewer ignores it.

```
bad:  Inv 9 — controllers should avoid direct JPA access.
good: Inv 9 — classes in `..adapter.in.web..` must not depend on any class
             in `..adapter.out.persistence..`. Enforced by ArchUnit test
             `WebAdapterDoesNotDependOnPersistence` in module
             `architecture-tests`.
```

Source: Hombergs *Get Your Hands Dirty on Clean Architecture* (2019) ch. 10:
"enforce does not mean a senior developer doing code reviews but rules that
make the build fail when they're broken."

### 3.4 Overlapping rules

Two invariants covering the same territory invite contradictory findings.

```
bad:  Inv 2 — no blocking I/O on hot path.
      Inv 6 — RequestLoop#tick must not call Socket#read.
good: Inv 6 refines Inv 2 (Inv 6 is the project-specific evidence for
     the general Inv 2).
```

Source: ADR "Status: refines / superseded-by" field (Nygard 2011).

### 3.5 Contradictory invariants

Two rules impossible to satisfy simultaneously. The reviewer must detect this
*before* enforcing either.

```
bad:  Inv 4 — all mutations flush through a single `synchronized` region.
      Inv 5 — no method may hold a monitor across I/O.
      (Violation scenario: the flush involves a disk write.)
good: Inv 4 — mutations within the in-memory snapshot region flush under
             `ReentrantLock#lock()`. Disk persistence happens on a
             separate executor; see `PersistenceAdapter` for the handoff.
```

Source: SEI ATAM tradeoff point - a decision affecting two attributes
simultaneously needs to be surfaced, not hidden (Bass-Clements-Kazman).

### 3.6 Unnumbered or floating rule

Text like "we also want to avoid …" buried in prose, without an ID or a
place in the rule list, has no cross-reference handle. Findings cannot cite
it; subsequent revisions cannot supersede it.

```
bad:  "Also, please try to keep the Controller layer thin. We also want
      to avoid Spring annotations in the domain."
good: Inv 11 — no class in `..domain..` may import from any
             `org.springframework.*` package. Enforced by ArchUnit
             rule `DomainFreeOfSpringAnnotations`.
```

Source: Nygard ADR (decisions need IDs for traceability); Glass Fact 26
(explicit requirements explode ~50× into implicit design requirements -
without an ID the dependency graph cannot be walked).

### 3.7 Implicit rule ("obvious to us")

A rule nobody bothers to write. For this reviewer it does not exist (G-03).
If the caller says "well, everyone knows X", the correct response is to flag
the absent rule and let the author add it to `project_context` before the
next audit.

### 3.8 Untested invariant

A rule with no evidence handle (§2 part 5). The reviewer cannot objectively
re-verify the rule, only read-check the change. This is tolerable when the
rule is structural (ArchUnit-testable), but a runtime rule without a test,
benchmark, or production-signal handle is half a rule.

Source: Ford, Parsons, Kua, *Building Evolutionary Architectures* (2017):
fitness functions "provide an objective integrity assessment" - subjective
rules do not qualify.

---

## 4. Audit procedure

The reviewer follows the phases below. Every phase ends in either emitting a
finding or moving on; there is no silent dropping of evidence.

### 4.1 Orient

- [ ] Read `project_context` in full. Confirm §1.1-§1.7 artefacts are present.
      Source: Nygard ADR template.
- [ ] List invariants by number into a working map `{Inv N → rule text,
      scale, forbidden state, evidence}`. Source: §2 anatomy.
- [ ] Read `design_intent`. Note the stated goal but do NOT relax any rule on
      its basis. Source: `prompt.md` §Step 1; Glass Fact 23 (requirements /
      intent are separate from invariants).
- [ ] Flag missing artefacts as `PROJ-META` findings. Source: §1.8.

### 4.2 Discover the diff

- [ ] Run `git diff $diff_ref --name-only -- '*.java' $path_filter` to list
      changed Java files. Source: `prompt.md` §Step 2.
- [ ] Extract `$base_ref` (the part of `diff_ref` before `...` or `..`) for
      old-vs-new comparison. Source: `prompt.md` §Step 3.3.
- [ ] If the changed set is empty, emit `## No findings` as the single output
      line. Source: `prompt.md` §Empty case (silence is not an option, G-05).

### 4.3 Per-file, per-invariant traversal

For each changed file `F`:

- [ ] Run `git diff $diff_ref -- F` to see the delta.
- [ ] For every changed method / field / class, walk the invariant map.
      For each invariant `Inv N`:
  - [ ] Does `F` fall into the invariant's subject? (package, class name,
        method, annotation match.)
  - [ ] Does the change introduce the invariant's forbidden state?
  - [ ] If yes, emit a finding citing `Inv N` (§6 output format).
- [ ] For invariants that require old-vs-new comparison (hot-path discipline,
      API compatibility, lock-hierarchy additions): run `git show
      $base_ref:F` and diff the signatures / bytecode shape against the new
      version. Source: `prompt.md` §Step 3.3; §1.6.

### 4.4 Scale-multiplier arithmetic

For each change that adds a new operation (allocation, lock acquisition,
exception throw, remote call):

- [ ] Multiply the per-operation cost by the scale number from the matching
      invariant. Example: a new 48-byte allocation inside a `~50 M inv/day`
      method produces 2.4 GB/day of churn. Source: §1.3; Glass Fact 5;
      Ford-Parsons-Kua fitness-function categories (automated vs manual).
- [ ] Emit a finding if the result exceeds the invariant's budget line.

### 4.5 Cross-invariant contradiction check

- [ ] After generating findings, inspect whether two findings suggest
      incompatible fixes. If the underlying invariants conflict (§3.5),
      surface a `PROJ-META` finding at higher severity than the code
      findings: the reviewer cannot pick between contradictory rules.
- [ ] If one invariant supersedes another (§3.4, §1.7) and the diff
      complies with the newer one, suppress the older-invariant finding
      and note the supersession in the report's preface.

### 4.6 Silence discipline

- [ ] Drop every finding that any other reviewer (§5) already owns. These
      are not this reviewer's scope; emitting them is noise that inflates
      total review cost against Bosu et al. MSR 2015 findings on useful-
      review factors (focused scope correlates with usefulness).

### 4.7 Stop condition

Either findings exist and §6 emits them numbered, or no findings remain and
§6 emits `## No findings`. Source: `prompt.md` §Empty case.

---

## 5. Non-overlap with other reviewers

The project-specific reviewer is the domain specialist. Generic Java issues
that apply to any codebase belong to the other reviewers. Silence on those
topics is required, not optional - overlap inflates total review noise and
lowers per-finding signal (Bosu et al., "Characteristics of Useful Code
Reviews", MSR 2015, Microsoft).

| Other reviewer | What it owns (stay silent here) | Boundary: when project-specific re-engages |
|---|---|---|
| concurrency-reviewer | JMM, happens-before, `volatile`/`synchronized`/`j.u.c.`/`VarHandle`, deadlock, data-race, virtual threads (JEP 491, JDK 24+), lock-free consensus hierarchy, jcstress patterns | Project defines a lock *hierarchy* and the diff violates hierarchy order; or the project has named a hot path that must be allocation-free or lock-free beyond generic Java safety |
| security-reviewer | OWASP Top 10, injection, XSS, SSRF, deserialization CVEs, secret management, crypto-primitive misuse (JCA/JCE), authentication / authorization bugs | Project defines a tenant-isolation invariant, data-residency rule, or a project-specific threat model that generic OWASP checks cannot see |
| maintainability-reviewer | SOLID, Checkstyle / PMD style, naming, cyclomatic-complexity thresholds, method length, public API surface size, generic layering violations | Project `project_context` names a specific boundary (hexagonal inbound/outbound adapter, bounded context) that must not be crossed even if generic layering would allow it |
| performance-reviewer | algorithmic complexity, JIT warmup, GC tuning, escape analysis, generic micro-benchmark hygiene, JMH pitfalls, profile-first rule | Project has a scale number in `project_context` (`~50 M inv/day`) and the diff moves a previously free operation onto the hot path |
| reliability-reviewer | bulkheads, circuit breakers, retries, timeouts, idempotency, back-pressure, generic resilience (Nygard *Release It!* stability patterns) | Project `project_context` names a specific SLO, a specific external dependency with its own failure budget, or an FMEA entry tied to a concrete artefact |

Three rules apply to every row:

1. If the finding would apply to *any* Java codebase, it is not project-
   specific. Stay silent. Source: `prompt.md` §Non-overlap.
2. `FIXME` / `TODO` / `HACK` comments are tracked work items, not findings
   (G-04). Source: `prompt.md` §What NOT to Report.
3. Pre-existing issues the diff does not worsen are out of scope.

---

## 6. Output-format discipline

Every finding follows the skeleton defined in `prompt.md` §Output Format.
This section adds enforcement notes the reviewer must obey.

### 6.1 Required fields

```
### PROJ<N>: <short title>
- **Severity**: Critical / Major / Minor
- **Location**: `File.java:line`
- **Code**: `<problematic code, quoted from diff>`
- **Invariant**: #N — <invariant title>
- **Problem**: <what rule is violated>
- **Suggested fix**:
  ```java
  <code showing what to write instead>
  ```
- **Rationale**: <what breaks if unfixed>
```

- The `**Invariant**` field is mandatory. A finding without a cited invariant
  is not this reviewer's job and must be dropped. Source: `prompt.md`
  §Step 4 "Every finding cites the invariant number it violates."
- The `**Severity**` field uses three levels only; see §6.2.
- The `**Suggested fix**` block must contain code the reviewer would write,
  not prose. If the fix is non-obvious, the reviewer still sketches the
  direction rather than leaving the field empty.

### 6.2 Severity triage

Source: Glass Fact 51 ("errors are always with us. The goal is to avoid or
minimise critical errors") - severity exists because reviewers have finite
attention.

- **Critical** - invariant violation that, if merged, produces a data-
  correctness failure, a security boundary breach named in `project_context`,
  or a production outage by SLO.
- **Major** - invariant violation that produces regression in a measured
  quality attribute (latency, throughput, memory) at stated scale.
- **Minor** - invariant violation that is cosmetic or statistically small at
  the stated scale. If the rule is aspirational or absent, use `PROJ-META`
  instead and keep severity Minor.

### 6.3 Numbering

Findings are numbered `PROJ1`, `PROJ2`, … sequentially per run. `PROJ-META`
findings about the shape of `project_context` (§1.8) share the sequence.

### 6.4 Empty case

If after §4 no findings remain, the entire output is a single line:

```
## No findings
```

Silence is a reviewer bug (G-05). Source: `prompt.md` §Empty case.

### 6.5 What must NOT appear

- Findings on `FIXME`, `TODO`, `HACK` comments (G-04).
- Findings covered by another reviewer (§5).
- Style issues a linter would catch (`prompt.md` §What NOT to Report).
- Pre-existing issues not worsened by the diff.

---

## 7. Modern architecture practices that ground invariants

The practices below are where `project_context` authors *should* source their
invariants. When the reviewer finds a rule that is not grounded in one of
these practices, it tolerates the rule but notes in §1.8 that the provenance
is thin.

### 7.1 Architecture Decision Records (ADR)

Short, numbered, immutable documents capturing *why* a decision was taken.
Template fields: Title, Status, Context, Decision, Consequences. Status
values: proposed, accepted, deprecated, superseded-by.

Source: Michael Nygard, "Documenting Architecture Decisions"
[Cognitect blog, 15 Nov 2011](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions).
Tooling: [`adr-tools`](https://github.com/npryce/adr-tools) (Nygard-style bash
CLI), [MADR](https://github.com/adr/madr) (Markdown Any Decision Records),
[Log4brains](https://github.com/thomvaill/log4brains) (static-site publisher),
index at [adr.github.io](https://adr.github.io).

Use: `project_context` often starts as a distillation of the project's ADR
log. When an invariant is linked back to an ADR, the reviewer can trust the
rationale (the *why* in Context + Consequences). Conscious shortcuts should
be recorded as ADRs, not left as `TODO` comments - Hombergs *Get Your Hands
Dirty on Clean Architecture* ch. 11 p. 94 makes this explicit.

### 7.2 ArchUnit as fitness-function test

[ArchUnit User Guide](https://www.archunit.org/userguide/html/000_Index.html)
lets architecture rules be expressed as JUnit tests: layered / onion
architecture rules, slice acyclicity, modularization via `@AppModule` /
`@ApplicationModule` with `allowedDependencies` and `exposedPackages`
attributes, dependency and general-coding rules.

Enforcement pattern (Hombergs ch. 10 pp. 88-89):

```java
@Test
void domainLayerDoesNotDependOnApplicationLayer() {
    noClasses()
        .that().resideInAPackage("buckpal.domain..")
        .should().dependOnClassesThat()
            .resideInAnyPackage("buckpal.application..")
        .check(new ClassFileImporter().importPackages("buckpal.."));
}
```

Three-layer Hombergs defence-in-depth:

1. Package-private visibility where possible (p. 86).
2. Post-compile ArchUnit checks (p. 88).
3. Build-artifact separation so the Java classpath physically cannot carry
   the forbidden dependency (p. 89-92).

Caveat to preserve (Hombergs p. 89): ArchUnit tests "are not fail-safe. If we
misspell the package name … the test will find no classes and thus no
dependency violations." Invariants enforced only by string-matched ArchUnit
rules must also include a check that the rule matches *some* classes (G-10).

### 7.3 Hexagonal / Ports & Adapters

Alistair Cockburn, "Hexagonal Architecture" (originally 2005 on the c2 /
Portland Pattern Repository wiki; consolidated 2022 as *Hexagonal
Architecture Explained*). The pattern's stated intent: "allow an application
to equally be driven by users, programs, automated test or batch scripts, and
to be developed and tested in isolation from its eventual run-time devices
and databases." ([Wikipedia summary](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software))).

Vocabulary that often appears in `project_context`: inbound / driving /
primary adapter; outbound / driven / secondary adapter; port (inbound port =
use-case interface; outbound port = SPI). Hombergs's Java implementation (ch.
4-10) is the reference for how boundaries translate into `project_context`
invariants.

### 7.4 Fitness functions (Evolutionary Architecture)

Ford, Parsons, Kua, *Building Evolutionary Architectures* (O'Reilly 2017;
2nd ed. with Sadalage 2022). Definition: "An architectural fitness function
provides an objective integrity assessment of some architectural
characteristic(s)."

Seven categorical axes: atomic vs holistic, triggered vs continual, static vs
dynamic, automated vs manual, temporal, intentional vs emergent, domain-
specific.

Use: any `Inv N` enforced by a test, metric, monitor, log, or chaos-
experiment is a fitness function. Categorising each invariant on these axes
tells the reviewer whether it is continuously enforced (CI test) or only
triggered (manual review).

### 7.5 ATAM utility tree and quality-attribute scenarios

SEI ATAM (Kazman, Klein, Clements, "ATAM: Method for Architecture
Evaluation", [CMU/SEI-2000-TR-004](https://www.sei.cmu.edu/documents/629/2000_005_001_13706.pdf)).
Nine steps: present ATAM, present business drivers, present architecture,
identify architectural approaches, generate utility tree, analyze approaches,
brainstorm and prioritize scenarios, analyze approaches again, present
results.

Utility-tree shape: root = overall utility; tier 1 = quality-attribute
requirement (performance, availability, modifiability, security,
interoperability, testability, usability); tier 2 = refinement (latency,
throughput, access control, deployability); tier 3 = concrete scenario with
`source · stimulus · artifact · environment · response · response-measure`
(Bass, Clements, Kazman *Software Architecture in Practice* 3rd/4th ed.).

Output categories that show up inside a good `project_context`:

- **Risk** - a decision that may introduce unanticipated problems in another
  quality attribute.
- **Non-risk** - a sound decision.
- **Sensitivity point** - a property upon which some quality-attribute
  measure depends.
- **Tradeoff point** - a decision that affects multiple quality attributes
  simultaneously (§3.5 contradiction detector).

### 7.6 FMEA (Failure Mode and Effects Analysis)

[IEC 60812:2018](https://webstore.iec.ch/en/publication/26359), 3rd ed.,
"Failure modes and effects analysis (FMEA and FMECA)". Scope explicitly
covers software (confirmed: "applicable to hardware, software, processes
including human action, and their interfaces, in any combination"). Use
inside `project_context`: each `Inv N` may carry a forward link to an FMEA
entry numbering the failure mode, its local effect, its global effect, and
the detection mechanism.

### 7.7 Domain-Driven Design - true invariants and consistency boundaries

Vernon, *Implementing Domain-Driven Design* (Addison-Wesley 2013). Chapter 10
"Aggregates" gives four numbered rules of thumb:

1. **Model True Invariants in Consistency Boundaries** (p. 430). Definition
   of invariant: "a business rule that always preserves its consistency …
   transactional consistency is instantaneous and atomic." Example: `c = a +
   b` - if `a = 2` and `b = 3`, `c` must be 5; any other value violates.
2. **Design Small Aggregates** (p. 432) - a correctly designed aggregate is
   one that "can be altered in any way required by the business, with its
   invariants completely consistent, within a single transaction."
3. **Reference Other Aggregates by Identity** (p. 437) - cross-aggregate
   references use the root's globally unique identifier, not object pointers.
4. **Use Eventual Consistency Outside the Boundary** (p. 442) - multi-
   aggregate updates go through domain events, not a single transaction.

Evans, *Domain-Driven Design* (Addison-Wesley 2004) is the original source
for aggregate / bounded-context / ubiquitous-language vocabulary.

Kleppmann, *Designing Data-Intensive Applications* (O'Reilly 2017) ch. 7 -
ACID consistency "refers to the validity of application data according to
certain invariants. … It is up to the application to define what those
invariants are so the transaction can preserve them correctly." Canonical
example: account cannot be overdrawn beyond a limit.

Use: the reviewer's `Inv N` entries at the data-consistency layer are
strongest when they name the aggregate boundary and the transactional
discipline Vernon / Kleppmann describe.

---

## 8. Research Questions to Cross-Check Against Sources

The meta-checklist is organised to answer the questions below for any
`project_context` plus diff. Each question is answerable yes / no / not-
applicable and cites the section and the primary source.

### A. Invariant structure

1. **Is every rule in `project_context` numbered with a stable ID?** (§1.1)
   — source: Nygard, "Documenting Architecture Decisions", Cognitect 2011.
2. **Does a definitions dictionary (ubiquitous language) cover every load-
   bearing domain term?** (§1.2) — source: Evans *DDD* ch. 2; Vernon *IDDD*
   ch. 1.
3. **Does every operational rule carry a scale number?** (§1.3) — source:
   Glass *Facts and Fallacies* Fact 5; Bass-Clements-Kazman ATAM scenario
   response-measure.
4. **Is there a hot-path list enumerating the modules / methods under
   tightest scrutiny?** (§1.4) — source: Glass Fact 49 (Endres 1975; Boehm-
   Basili 2001).
5. **If the system is concurrent, is a lock hierarchy or synchronization
   order spelled out?** (§1.5) — source: Herlihy-Shavit *The Art of
   Multiprocessor Programming* ch. 9 (project context delegates to concur-
   rency-reviewer; this reviewer checks only hierarchy-additions).
6. **Does `project_context` declare what counts as a breaking API change?**
   (§1.6) — source: Nygard ADR template Consequences section.
7. **Do invariants cross-reference each other (refines / supersedes /
   depends-on)?** (§1.7) — source: MADR template (adr/madr); Nygard ADR
   "Status: superseded-by".

### B. Anatomy of one invariant

8. **Does each invariant have a subject, rule, scale, forbidden state,
   evidence handle, and correction recipe?** (§2) — source: Vernon *IDDD*
   p. 430; Bass-Clements-Kazman ATAM scenario; Ford-Parsons-Kua fitness-
   function definition (*Building Evolutionary Architectures* 2017).
9. **Is the forbidden state expressed negatively (what the code must not
   do)?** (§2 part 4) — source: Vernon p. 430 (invariant violation = `c ≠
   5`).
10. **Is the evidence handle executable (unit / ArchUnit test, JFR event,
    benchmark, runbook)?** (§2 part 5) — source: Ford-Parsons-Kua 2017
    (fitness functions "provide an objective integrity assessment").

### C. Invariant anti-patterns

11. **Are all load-bearing terms defined exactly once, with no homonym
    clashes?** (§3.1) — source: Evans *DDD* ch. 2.
12. **Does every traffic-dependent rule have a number?** (§3.2) — source:
    Glass Fact 5.
13. **Is every "should" either converted to "must + enforcement" or removed
    from the rule list?** (§3.3) — source: Hombergs *Get Your Hands Dirty on
    Clean Architecture* ch. 10.
14. **Are overlapping rules resolved via refinement / supersession links?**
    (§3.4) — source: Nygard ADR "Status: refines / superseded-by".
15. **Are contradictory invariants surfaced as tradeoff points, not hidden?**
    (§3.5) — source: Bass-Clements-Kazman ATAM tradeoff-point definition.
16. **Does every rule that code would violate have an ID (no floating prose
    rules)?** (§3.6) — source: Nygard ADR; Glass Fact 26.
17. **Does any rule carry an evidence handle, not just text?** (§3.8) —
    source: Ford-Parsons-Kua 2017.

### D. Audit discipline

18. **Did the reviewer read `project_context` in full before the diff?**
    (§4.1) — source: `prompt.md` §Step 1.
19. **Did the reviewer derive a working invariant map with subject / scale /
    forbidden state per rule?** (§4.1) — source: §2 anatomy.
20. **Does every finding cite the invariant ID it violates?** (§4.3, §6.1) —
    source: `prompt.md` §Step 4 ("Every finding cites the invariant number
    it violates").
21. **For invariants requiring old-vs-new comparison, did the reviewer run
    `git show $base_ref:<file>`?** (§4.3) — source: `prompt.md` §Step 3.3.
22. **If the diff introduces a new operation on a hot path, is scale-
    multiplied impact computed?** (§4.4) — source: Glass Fact 5; §1.3.
23. **Were generic Java findings suppressed in favour of the owning
    reviewer?** (§4.6, §5) — source: Bosu et al., "Characteristics of Useful
    Code Reviews" MSR 2015 (focused scope correlates with usefulness).
24. **Did the reviewer check for contradictions between its own findings
    before emitting?** (§4.5) — source: §3.5.
25. **If no findings remain, was `## No findings` emitted on a single line?**
    (§4.7, §6.4) — source: `prompt.md` §Empty case.

### E. Scale-multiplier analysis

26. **When a new allocation is added to a ~N M/day method, is N × size
    computed and compared to the invariant budget?** (§4.4) — source:
    Glass Fact 5; Fact 49.
27. **When a new lock acquisition is added to a hot path, is the
    contention estimate computed against the lock-hierarchy rules?** (§4.4)
    — source: Herlihy-Shavit §7; `project_context` §1.5.
28. **For new synchronous remote calls, is their median-and-p99 latency
    compared to the invariant's response-measure line?** (§4.4) — source:
    Bass-Clements-Kazman response-measure.

### F. Old-vs-new comparison

29. **Has the signature, parameter order, or nullability of any public API
    changed in a way the API-compat invariant forbids?** (§4.3, §1.6) —
    source: Nygard ADR Consequences section; `prompt.md` §Step 3.3.
30. **Has a previously allocation-free method gained any of: `new`,
    autoboxing, varargs spread, lambda capture, `String.format`?** (§4.3,
    §2 example) — source: Vernon *IDDD* ch. 5 (entities are value-object-
    heavy by design).
31. **Has an invariant that used to be enforced by a test become silently
    untested (removed or disabled test)?** (§4.3) — source: Ford-Parsons-
    Kua fitness-function definition; Hombergs ch. 10.

### G. Non-overlap with other reviewers

32. **Is every emitted finding tied to a project-specific invariant, not a
    generic Java rule?** (§5) — source: `prompt.md` §Non-overlap.
33. **Are `FIXME` / `TODO` / `HACK` comments absent from findings?** (§5 row
    3) — source: `prompt.md` §What NOT to Report.
34. **Are pre-existing issues unchanged by the diff absent from findings?**
    (§5 row 3) — source: `prompt.md` §What NOT to Report.

### H. Fitness functions and ArchUnit

35. **Is each invariant categorised on the Ford-Parsons-Kua axes (atomic vs
    holistic, triggered vs continual, static vs dynamic, automated vs
    manual, temporal, intentional vs emergent, domain-specific)?** (§7.4) —
    source: Ford, Parsons, Kua, *Building Evolutionary Architectures* 2017.
36. **Are structural invariants enforced by ArchUnit rules with matching
    `@AppModule` or `@ApplicationModule` annotations?** (§7.2) — source:
    ArchUnit User Guide; Spring Modulith Reference 2022+.
37. **Does each ArchUnit rule also verify that the rule matches some
    classes (to survive package renames)?** (§7.2 caveat) — source: Hombergs
    ch. 10 p. 89.
38. **Are layered / onion / hexagonal boundaries enforced at post-compile
    *and* build-module level?** (§7.2) — source: Hombergs ch. 10 pp. 86-92
    (three-layer defence).

### I. Utility tree, FMEA, and ADR discipline

39. **Does the rubric trace each invariant back to a quality-attribute
    scenario (source · stimulus · artifact · environment · response ·
    response-measure)?** (§7.5) — source: Bass-Clements-Kazman *Software
    Architecture in Practice*.
40. **Are risk / non-risk / sensitivity / tradeoff labels attached to
    invariants where applicable?** (§7.5) — source: SEI ATAM TR
    CMU/SEI-2000-TR-004.
41. **Do invariants on the reliability axis cross-reference FMEA entries
    (failure mode · local effect · global effect · detection)?** (§7.6) —
    source: IEC 60812:2018.
42. **Is each accepted shortcut recorded as an ADR (Status: accepted, with
    Consequences section naming the technical-debt line)?** (§7.1) —
    source: Hombergs ch. 11 p. 94 → Nygard 2011.
43. **When an invariant is superseded, does the new invariant's Status
    field name the old one as `superseded-by`?** (§1.7) — source: Nygard
    ADR template.

### J. Output-format discipline

44. **Does every finding name the invariant number in the `**Invariant**`
    field?** (§6.1) — source: `prompt.md` §Output Format.
45. **Does every finding carry a severity label (Critical / Major /
    Minor)?** (§6.2) — source: Glass Fact 51.
46. **Does the `**Suggested fix**` block contain code, not prose?** (§6.1)
    — source: `prompt.md` §Step 4.
47. **If no findings remain, is the output exactly `## No findings` on a
    single line?** (§6.4) — source: `prompt.md` §Empty case.
48. **Are `PROJ-META` findings (rubric-shape issues) numbered in the same
    sequence as `PROJ1`, `PROJ2`, …?** (§6.3) — source: `prompt.md`
    §Output Format.

---

## 9. Sources

### Normative / specification

- Nygard, Michael T., "Documenting Architecture Decisions", Cognitect blog,
  15 Nov 2011. https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions
  — defines the five-field ADR template (Title, Status, Context, Decision,
  Consequences) and the "superseded-by" status link.
- [IEC 60812:2018, 3rd edition](https://webstore.iec.ch/en/publication/26359)
  "Failure modes and effects analysis (FMEA and FMECA)" - scope explicitly
  covers software and service processes.
- Kazman, Klein, Clements, "ATAM: Method for Architecture Evaluation",
  CMU/SEI-2000-TR-004. https://www.sei.cmu.edu/documents/629/2000_005_001_13706.pdf
  - the nine-step method, utility tree, risk / non-risk / sensitivity-point /
  tradeoff-point taxonomy.

### Foundational / authoritative

- Vernon, Vaughn, *Implementing Domain-Driven Design* (Addison-Wesley 2013),
  ISBN 978-0-321-83457-7. Ch. 10 "Aggregates" (pp. 355-402 EN, pp. 423-470 RU
  ed.) - the four rules-of-thumb: Model True Invariants in Consistency
  Boundaries (p. 430); Design Small Aggregates (p. 432); Reference Other
  Aggregates by Identity (p. 437); Use Eventual Consistency Outside the
  Boundary (p. 442). Definition of invariant at p. 430. Canonical example
  `c = a + b`.
- Evans, Eric, *Domain-Driven Design: Tackling Complexity in the Heart of
  Software* (Addison-Wesley 2004), ISBN 978-0-321-12521-7 - ubiquitous
  language, bounded context, aggregate.
- Hombergs, Tom, *Get Your Hands Dirty on Clean Architecture* (Leanpub 2019).
  Ch. 10 "Enforcing Architecture Boundaries" (post-compile ArchUnit checks,
  build-artifact enforcement, package-private visibility); ch. 11 "Taking
  Shortcuts Consciously" (ADR for conscious shortcuts).
- Bass, Clements, Kazman, *Software Architecture in Practice* (Addison-Wesley
  SEI Series, 3rd ed. 2012; 4th ed. 2021) - quality-attribute scenario
  structure (source · stimulus · artifact · environment · response ·
  response-measure).
- Ford, Parsons, Kua, *Building Evolutionary Architectures* (O'Reilly 2017;
  2nd ed. with Sadalage 2022). Fitness-function definition; seven categorical
  axes; deployment-pipeline integration.
- Kleppmann, Martin, *Designing Data-Intensive Applications* (O'Reilly 2017)
  - ch. 7 Transactions (invariants as ACID consistency); ch. 9 Consistency
  and Consensus.
- Nygard, Michael T., *Release It! Design and Deploy Production-Ready
  Software* (Pragmatic Bookshelf, 2nd ed. 2018) - stability anti-patterns,
  bulkheads, circuit breakers (the reliability-reviewer's territory;
  cross-referenced here for non-overlap §5).
- Glass, Robert L., *Facts and Fallacies of Software Engineering* (Addison-
  Wesley 2002), ISBN 978-0-321-11742-7 - Fact 5 (tool-hype 5-35% vs order-
  of-magnitude claims); Fact 23 (requirements cause runaway projects);
  Fact 26 (explicit requirements explode ~50×); Fact 37 (inspections detect
  up to 90% of defects); Fact 38 (inspections complement testing);
  Fact 49 (80/20 defect clustering); Fact 51 (residual defects are
  unavoidable; goal is to minimise critical ones).

### Tooling

- [ArchUnit User Guide](https://www.archunit.org/userguide/html/000_Index.html)
  - JUnit-based architecture rules: LayeredArchitecture, OnionArchitecture,
  slice rules, modularization via `@AppModule` with `allowedDependencies` and
  `exposedPackages`, dependency rules, general coding rules.
- [ArchUnit-Examples](https://github.com/TNG/ArchUnit-Examples) — reference
  patterns for layered, hexagonal, onion rules.
- [adr-tools](https://github.com/npryce/adr-tools) - Nathan Pryce's bash CLI
  using Nygard's template.
- [MADR](https://github.com/adr/madr) - Markdown Any Decision Records, full
  and minimal templates.
- [Log4brains](https://github.com/thomvaill/log4brains) - static-site
  generator for ADR logs.
- [Spring Modulith](https://docs.spring.io/spring-modulith/docs/current/reference/html/)
  - `@ApplicationModule(allowedDependencies = …)` implementation for Spring
  Boot, building on ArchUnit.
- [Structurizr](https://structurizr.com/) - C4 model diagrams as code;
  complement to ADR logs for architecture visualisation.

### Checklists and curated reviews

- [adr.github.io](https://adr.github.io) - curated index of ADR templates,
  tooling, examples.
- Jureczko et al., "Code review effectiveness: an empirical study on selected
  factors influence", IET Software Vol. 14 No. 7, 2020.
  https://ietresearch.onlinelibrary.wiley.com/doi/full/10.1049/iet-sen.2020.0134
- Bosu, Greiler, Bird, "Characteristics of Useful Code Reviews: An Empirical
  Study at Microsoft", MSR 2015.
  https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/bosu2015useful.pdf
  - focused scope correlates with usefulness; evidence for §5 non-overlap
  discipline.

### Industry case studies and explainers

- Cockburn, Alistair, "Hexagonal Architecture" (2005, Portland Pattern
  Repository wiki; *Hexagonal Architecture Explained*, 2022, with Garrido de
  Paz).
  [Wikipedia summary](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software))
  - ports and adapters, primary vs secondary adapters, isolation from run-
  time devices.
- Ghorbani, Kalaitzakis, Anastasi, "Supporting architecture evaluation for
  ATAM scenarios with LLMs", arXiv:2506.00150, 2025 - recent academic take on
  ATAM scenarios.
- "Invariant-based Program Repair", FASE 2024, arXiv:2312.16652 - invariants
  as automated-repair specifications.

---

## 10. Gotchas -- Common Wrong Assumptions

G-01. **"The reviewer should also flag generic Java issues it sees"** — false.
Anything another reviewer owns (§5) is noise for this reviewer. Silence on
generic Java issues is required; the orchestrator composes per-reviewer
reports. See §5.

G-02. **"`design_intent` overrides invariants when the author explains the
rule is OK to break"** — false. Intent is context, not permission. Every
invariant violation must still be flagged even when `design_intent` says the
change is intentional. The author can then amend `project_context` to
supersede the invariant if the intent is correct. See §4.1, §1.7.

G-03. **"Rules that `everyone knows` but aren't written down still apply"** —
false. For this reviewer, unwritten rules do not exist. If a rule matters,
it belongs in `project_context`; otherwise it cannot be cited in a finding
and the reviewer stays silent on it. See §3.7.

G-04. **"`FIXME` / `TODO` / `HACK` comments in the diff are findings"** —
false. They are tracked work items owned by the project's backlog, not this
reviewer's output. See §5 row 3, `prompt.md` §What NOT to Report.

G-05. **"If nothing is wrong, the reviewer can return empty output"** —
false. Silence is a reviewer bug. The empty case is a single line
`## No findings`. Always emit a report. See §6.4, `prompt.md` §Empty case.

G-06. **"An invariant without a number is still enforceable"** — false.
Findings must cite an ID. A floating prose rule has no handle for
cross-reference, supersession, or audit. Surface as `PROJ-META` and move on.
See §3.6, §1.1.

G-07. **"Two invariants covering the same territory is redundant but
harmless"** — false. Overlap invites contradictory findings and makes
supersession tracking impossible. Require a refines / superseded-by link.
See §3.4, §1.7.

G-08. **"An invariant with an aspirational `should` is enforceable when
reviewers interpret it charitably"** — false. `should` / `try to` / `prefer`
cannot fail a build. Either the rule is `must` + enforcement mechanism, or
it is not a rule. See §3.3; Hombergs ch. 10.

G-09. **"Scale number can be omitted when the rule feels `obvious`"** —
false. Glass Fact 5 documents a systemic gap between expected ("order of
magnitude") and measured (5-35 %) improvements; only numbers defend against
it. See §3.2, §1.3.

G-10. **"ArchUnit tests are safe once written"** — false. Hombergs p. 89:
"If we misspell the package name … the test will find no classes and thus
no dependency violations. A single typo or a single refactoring renaming a
package can make the whole test useless." Invariants enforced by ArchUnit
rules must also assert that the rule matched some classes. See §7.2.

G-11. **"Inspections can catch every defect"** — false. Glass Fact 37: the
highest reported detection rate is ~90%, and Fact 38 makes explicit that
inspections cannot replace testing. The reviewer must not claim
completeness; residual-defect planning belongs in the reliability reviewer's
scope. See §5 reliability row.

G-12. **"Two invariants using the same word are using the same meaning"** —
false. Terminology collisions are a known DDD failure mode (Evans *DDD*
ch. 2 ubiquitous language). If `tenant` means "customer organisation" in
Inv 3 and "URL-prefixed path segment" in Inv 8, the code will obey neither.
Flag vocabulary collisions as `PROJ-META`. See §1.2, §3.1.

G-13. **"`project_context` is the reviewer's problem to interpret"** —
false. The rubric is the author's responsibility; the reviewer's job is to
audit against it. When the rubric is malformed, the correct output is a
`PROJ-META` finding describing the gap, not a best-effort guess. See §1.8.

G-14. **"An invariant without a test is still enforceable by reading the
diff"** — partially false. Structural invariants (layering, dependency
direction, naming) can be reviewed by diff-read; runtime invariants
(allocation, latency, concurrency) cannot be verified by reading and need a
fitness function. Flag the missing test when the invariant is runtime. See
§3.8, §7.4.

G-15. **"ADRs are documentation, not part of the rubric"** — false. ADRs
produce the `project_context` invariants; their `superseded-by` status is
the mechanism §1.7 relies on to detect obsolete rules. A rubric without ADR
provenance is harder to audit because the *why* behind each rule is missing.
See §7.1.
