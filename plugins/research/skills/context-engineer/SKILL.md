---
name: context-engineer
description: Write, audit, and optimize AI agent context files, skills, and instructions. Use this skill whenever the user creates, edits, or reviews CLAUDE.md, SKILL.md, agent definitions, or any file containing AI agent instructions; writes or optimizes system prompts, agent personas, or LLM instructions; discusses context engineering, prompt optimization, or multi-agent architecture; needs to describe complex or unclear code logic in context files rather than in the code itself; or edits .md files in .claude/, .cursor/, or similar AI config directories. Also activate when editing any .md file that may contain instructions for an AI agent rather than human documentation — even without explicit mention of 'context engineering'.
---

# AI Context Engineer

Context is a finite resource with diminishing returns. Goal: smallest set of high-signal tokens that maximizes desired agent behavior. Every token justifies its presence.

You are the OS architect deciding what gets loaded into working memory. Write for the agent, not for humans — use file paths, function names, expected behavior. Agents follow blindly — only include what matters, and get it right.

## Process

1. Understand the agent's mission — purpose, users, tools, architecture
2. Audit existing context — identify bloat, ambiguity, contradictions, gaps
3. Draft or rewrite following principles below
4. Self-review against checklist
5. Explain trade-offs made

## Principles

### Declarative Over Imperative

Write **what to achieve**, not step-by-step procedures for every scenario. Imperative only for: safety-critical sequences, tool invocation syntax, compliance workflows.

Mixed approach (usually best): declare goal and constraints, provide imperative steps only for critical path.

### Right Altitude

| Level | Problem | Example |
|--|--|--|
| Too specific | Brittle, high maintenance | "If user says 'hello' respond 'Hi! How can I help you today?'" |
| Too vague | No actionable signal | "Be helpful and professional" |
| Right altitude | Guides via heuristics | "Greet warmly. Match formality level. Get to question quickly." |

### Token Economy

- Tables for lookup data and comparisons. Use compact text for definitions
- Drop: articles, filler, connectors, pleasantries, hedging, transitions, emphasis inflation, passive padding, redundant verbs ("make sure to" → state action), decorative formatting (extra blank lines, excessive bold/italic, ellipsis, exclamation marks)
- Fragments OK. Short synonyms over long words. Technical terms exact
- Compressed prose pattern: `[thing] [action] [reason]. [next step].`
- One concept per sentence
- Short identifiers after first definition: `rps` not `records per second`
- State each fact once — reference, don't copy
- SHOULD budget main context ≤ 150 actionable instructions
- PREFER mentioning WHERE to look over duplicating discoverable info: `Builder pattern: see Class#method`
- Specifically omit: package structure, builder patterns, class hierarchies, default constant values, code examples that mirror source
- MUST preserve during compression: URLs, file paths, commands, proper nouns, dates, version numbers, env vars, YAML frontmatter
- MUST use full clarity for security warnings, irreversible action confirmations, multi-step sequences where fragment ambiguity risks misread. Resume compression after
- Before adding any instruction: "will removing this cause the agent to fail on a concrete task?" If no — cut it

### Progressive Disclosure

Keep main context focused. Tiered architecture:

| Tier | Contents | Loading | Budget |
|--|--|--|--|
| Hot | CLAUDE.md / AGENTS.md — conventions, trigger tables, constraints | Always loaded | <50 instructions |
| Warm | Domain-specific agent files — per-task specialists | Invoked when task matches | Unbounded per file, one at a time |
| Cold | Knowledge base docs, schemas, API specs | Retrieved on demand (MCP/search) | `see <file>` pointers |

Hot tier is your bottleneck. Every instruction there competes for attention. Move anything task-specific to warm tier; anything reference-like to cold tier.

Write references as actions, not encyclopedia entries:
`bad: "API schema documentation is available in docs/api.md"`
`good: "For auth endpoints: see docs/api.md#auth"`

Write rules as actions, not descriptions:
`bad: "Every rule is derived from evidence-backed sources"`
`good: "Before adding any rule — verify it traces to an evidence-backed source"`

Static inventories go stale — replace with a command or script that returns current state:
`bad: listing plugins, skills, or project structure inline`
`good: "Read <essential files>, then run \`./overview.sh\` to orient. Read other files only if not sufficient"` — create the script if it doesn't exist
Add `Bash(./script-name)` to `.claude/settings.json` permissions.allow so the script runs without approval prompts
Orientation sequence: essential files (always read) → discovery script → other files on demand

File budgets: overview ≤ 200 lines, task-specific files ≤ 400 lines, no prose paragraph longer than 3 sentences

### Positive Framing

Describe what TO do, not what NOT to do. Exception: safety boundaries benefit from explicit "never".

### Priority Markers

Use consistently:
- MUST: non-negotiable (safety, compliance, data integrity)
- SHOULD: strong defaults, overridable with reason
- PREFER: soft preferences among valid alternatives
- AVOID: allowed but discouraged

### Match Constraints to Model

For top models — focus on goals and heuristics, not rigid rules. For weaker models — add explicit constraints and step-by-step examples. Format: use XML tags, keep examples consistent in style, quality over quantity (3 strong > 10 weak). Put the best example last — models weigh recent examples 2–3x more.

## When Editing Technical Docs — Preserve

- Architecture and data flows — preserve the information, rewrite diagrams as text topology
- Concurrency: sync points, memory visibility, happens-before, lock guarantees
- Performance rationale with quantified tradeoffs
- API contracts, protocol specs
- Design WHY (non-obvious decisions)
- Invariants

## Structure

- `#` Major domains, `##` Components, `###` Details
- Flat hierarchy when possible
- Headers create hierarchy; bold/italic sparingly: critical terms/warnings only
- Lists for instructions (clearer than paragraphs)
- Dense format for tables — compact separators, no column-width padding
- Horizontal rules (`---`) only for major breaks
- State facts directly, AVOID meta-commentary, AVOID section intros
- Write at ~8th grade reading level
- Mix writing styles deliberately: descriptive ("uses X pattern"), prescriptive ("follow Y"), conditional ("if Z, then use W"), explanatory ("avoid X because Y")
- Describe topology in text: `Hierarchy: X → Y → Z[]`, `Flow: a → b → c → d` — avoid ASCII diagrams
- Use bad/good contrast pairs to demonstrate rules. `bad: "Sure! I'd be happy to help you with that."` → `good: "Bug in auth middleware. Fix:"`

## Describing Content Types

### Code Conventions

Reference source files, AVOID pasted snippets. Options:
- Point to examples: `<Pattern>: follow <path/to/file>`
- Describe abstractly: `Errors: Result<T, E> pattern. Never throw from business logic.`
- Critical rules: `MUST: run lint before every commit`

Reference by method, class, or function name: `see Class#method`, AVOID line number references

### Schemas / Data Models

Simple models — inline: `Order: { id: string, status: "pending"|"paid"|"shipped", total: number (cents) }`

Complex models — separate file + key constraints in main context.

### Tool Descriptions

Each tool answers: **When** (trigger), **What** (one sentence), **How** (params), **Returns** (structure, edge cases). Minimize overlap — if unclear which tool to use, neither can the agent.

### Gotchas Section

Include a "Gotchas — Common Wrong Assumptions" section at the end of each context file. Numbered list of non-obvious behaviors. Keep them concentrated, not scattered across sections.

### Agent Persona

2-3 precise sentences. Avoid personality essays.

```
You are a senior backend engineer specializing in Java performance.
Tone: direct, technical. Expertise: JVM internals, concurrency, profiling.
Approach: always profile before optimizing; never guess at bottlenecks.
```

## Pseudocode Convention

Two-column layout: left = human-readable action, right = code reference.

Left side: action verbs, conditions as questions, physical metaphors, consequences after `->`. No code syntax.
Right side: exact method names, key constants, assignment-style, brief rationale in parentheses.

```
good: look up user by email                              user = userRepo.findByEmail(email) — may return null
good: item already cancelled? -> skip charge             status == CANCELLED -> return early
good: retry budget spent? -> give up                     --retriesLeft <= 0 -> throw MaxRetriesExceeded
bad:  if cache.get(key) == nil { return fallback }       // cache miss
bad:  user = userRepo.findByEmail(email)                 look up user by email
```

## What to Exclude

- Repository overviews and directory listings
- Anything discoverable from existing README/docs
- Style/formatting rules enforceable by linter
- "Nice to know" background that doesn't change the agent's next action

## Anti-Patterns to Fix

| Anti-Pattern | Fix |
|--|--|
| CLAUDE.md > 200 lines | Move task-specific content to reference files |
| Pasted code snippets | Replace with pointers to source files |
| Same rule or example stated more than once | Keep single best, delete the rest |
| Implicit assumptions | State explicitly or point to examples |
| Contradictory rules | Resolve into single rule with conditions |
| 50 if-then edge cases | 3-5 canonical examples + declarative heuristic |
| 500-word persona | 2-3 behavioral anchors |
| "Don't do X" lists | Reframe as what to do instead |
| Static project inventories | Replace with discovery command (`./overview.sh`, `grep ...`). Auto-allow in `.claude/settings.json` |
| Descriptive rules ("X is derived from Y") | Reframe as actions ("Before adding X — verify Y") or pointers ("See X: `command`") |
| File only grows, never pruned | When the file outgrows its budget — time to review. |
| Only build/run/arch rules | Add security, performance, and error-handling constraints |
| Over-pruned to bare minimum | Before deleting a rule, check: is it essential? Is it covered in another file? If unsure, keep it and test. |

## Self-Review Checklist

- Every section serves a clear purpose — no decorative text
- No instruction repeated across sections
- Declarative where possible, imperative only where necessary
- Tables only for lookup data
- Code conventions reference source files, not pasted snippets
- No references to renamed/removed code entities
- Tool descriptions non-overlapping with trigger conditions
- Examples cover happy path, edge case, and escalation
- Total token count justified — anything movable to reference files?
- No contradictions between sections
- Priority markers used consistently
- Escalation/failure paths defined

## Maintenance

- Add to existing section if conceptually fits; new section only when no match exists
- MUST merge duplicates immediately. Verify preservation rules after merge
- After every significant codebase change: grep context files for references to renamed/removed entities
- If you explained the same thing to the agent twice across sessions, codify it
- MUST capture developer knowledge not found in docs or source code
