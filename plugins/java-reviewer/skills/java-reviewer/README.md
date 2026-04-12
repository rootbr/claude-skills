# java-reviewer

Deep Java code review skill for Claude Code. Runs specialist reviewers in parallel against a git diff, verifies every finding, and produces two reports — a clean review and a rejection audit trail.

## What it does

Six-phase interactive workflow: **Scope** -> **Context** -> **Agent Selection** -> **Execute** -> **Verify** -> **Report**. Confirms with you at three points so you stay in control.

| Reviewer | Scope | Selection |
|---|---|---|
| performance | Allocation, GC, JIT, cache, I/O | When diff touches loops, collections, hot paths |
| concurrency | JMM, happens-before, volatile, locks | When diff touches shared state, locks, async |
| security | OWASP Top 10, crypto, secrets, deser | When diff touches user input, auth, API endpoints |
| reliability | Leaks, timeouts, transactions, shutdown | When diff touches network, retries, resources |
| maintainability | SOLID, complexity, testability, docs | Always (lightweight) |
| project-specific | Your project's invariants and rules | Only if config exists |
| **verification** | **Challenges every finding** | **Always (post-execute)** |

Reviewers are selected dynamically based on diff content — not all run every time.

## Quickstart

```
cd <your-java-repo>
/java-reviewer
```

Works out of the box if your repo is a git working tree with Java sources and a standard base branch (`main`, `master`, or `develop`).

## Output

Two documents:

1. **Review report** (`java-review-<branch>.md`) — only verified findings. Executive summary, critical/major with suggested fix code, minor/suggestions table, positive observations.

2. **Rejection report** (`java-review-<branch>-rejections.md`) — everything the verification agent filtered out, with evidence for each rejection. Lets the author see what was flagged and push back if they disagree.

## Project Integration

For deeper reviews, add a config file to your repo:

```
<your-repo>/
└── .claude/
    └── java-reviewer/
        └── config.md
```

### config.md format

YAML frontmatter configures the skill. The markdown body becomes `project_context` injected into every reviewer.

```markdown
---
project_name: MyProject
base_branch: main
branch_pattern: '^feature/PROJ-\d+'
ticket_id_pattern: 'PROJ-\d+'
ticket_design_doc: 'docs/{TICKET_ID}.md'
review_output: 'docs/{TICKET_ID}-review.md'
documentation: 'docs/architecture'
---

# MyProject Review Context

## Writing Good Project Context

The markdown body should contain only what generic Java reviewers **cannot know**. Don't repeat OWASP, SOLID, or GC patterns — the specialist reviewers already have those.

### What to include

- **Scale numbers** — so reviewers can estimate cost-at-scale ("32 bytes × 1M entries = 32 MB leaked")
- **Lock hierarchy** — acquisition order, what deadlocks if reversed
- **Threading annotations** — how your code documents thread-safety
- **Intentional tradeoffs** — things that look like bugs but aren't ("non-volatile reads on status flags are intentional")
- **Hot-path methods** — where zero-allocation and O(1) matter
- **Module boundaries** — which packages are public API, which are internal
- **Protocol/API contracts** — wire format, message ordering, backward compatibility rules

### Invariants — the highest-value section

**Every production bug is an invariant violation.** Documenting your project's invariants is the single most impactful thing you can do for review quality.

An invariant is a property that must always hold. When violated, a specific failure occurs. Write each as:

```markdown
## Invariants

### Data Structures
1. **Sentinel at index 0**: Index 0 is reserved, never holds data.
   All iterations must use `> 0`. Violation: reads garbage.

### Accounting
16. **Size tracking**: onAdd/onRemove must be called symmetrically.
    Violation: size counter drifts.
```

Each invariant should be **numbered** (so reviewers cite "violates #9"), **verifiable** (checkable against code), and **consequence-stated** (what breaks).

### How to find invariants in your project

1. **Mine from bug history** — every post-mortem has a "should always be true" statement hiding in it
2. **Read assertions** — `assert`, `checkState()`, `Preconditions.check*()` are invariants in code
3. **Trace field relationships** — when two fields must stay consistent (`size == array.length`)
4. **Ask "what breaks?"** — for each key field, what fails if the value is wrong?
5. **Check cleanup paths** — every open must have a close, every increment a decrement
6. **Review lock ordering** — build a graph of which locks are held when acquiring others

### Common invariant categories

| Category | Example | Violation |
|---|---|---|
| Data structure | Hash load factor in [0.25, 0.75] | OOM or O(n) degradation |
| Concurrency | Lock A -> Lock B, never reversed | Deadlock |
| Accounting | `idle + active == poolSize` | Pool leak or over-allocation |
| Protocol | Wire encoding frozen across versions | Deserialization crash |
| Resource | Pooled objects null refs on return | Memory leak x scale |
| Domain | Order total == sum(items) + tax | Financial discrepancy |
```

See [Java Invariants Review Checklist](references/java-invariants-review-checklist.md) for the deep reference on discovering, documenting, and verifying invariants.