---
name: auditing-ai-context
description: Write, audit, and optimize AI agent context files, skills, and instructions. Use this skill whenever the user creates, edits, or reviews CLAUDE.md, SKILL.md, agent definitions, or any file containing AI agent instructions; writes or optimizes system prompts, agent personas, or LLM instructions; discusses context engineering, prompt optimization, or multi-agent architecture; needs to describe complex or unclear code logic in context files rather than in the code itself; or edits .md files in .claude/, .cursor/, or similar AI config directories. Also activate when editing any .md file that may contain instructions for an AI agent rather than human documentation — even without explicit mention of 'context engineering'. NOT for auto-generating context files from scratch — LLM-generated files cost +20–23% tokens with −0.5% to −2% success; audit human-written files instead.
---

# AI Context Engineer

Context is working memory. Hot tier is finite; warm-tier playbooks MAY run long when each bullet earns its place and KV-cache amortizes the cost (91.8% cache hit, 82.6% billed-cost cut at scale, Zhang 2510.04618 §4.7). Every bullet traces to evidence or a real failure — not to a token count.

You are the OS architect deciding what gets loaded into working memory. Write for the agent, not for humans — use file paths, function names, expected behavior. Agents follow blindly — only include what matters, and get it right.

## Process

1. Understand the agent's mission — purpose, users, tools, architecture
2. Audit existing context — identify bloat, ambiguity, contradictions, gaps, and body-vs-description scope drift (body that acts outside what `description` claims)
3. Draft 3+ eval queries (should-trigger + should-not-trigger) BEFORE writing body. Source rules from failed-trajectory error analysis first (~41.7% of production rules), existing-prompt mining second (~35%), LLM brainstorming last (~23.3%) — brainstorm-only rules miss real failure modes (Yang 2505.13360 §3.1)
4. Draft or rewrite following principles below
5. Self-review against checklist
6. Test against evals using **reliable@10** — 10 should-trigger + 10 should-not paraphrased cousin prompts, **all must pass jointly** (not majority). Up to 61.8% reliability drops occur between nominal accuracy and reliable@10 (Dong 2512.14754 §2.2). Numeric edits ("at most 600" vs "610") and constraint reconfiguration degrade more than pure rephrasing — lock exact numerics
7. If reliable@10 fails on 1–2 cousins, ship with rejection sampling n=3 (plateaus ~n=12). If it fails broadly, revise the skill
8. Assess skill quality directly — token consumption, step count, trigger precision. Regression on any axis blocks release

## Principles

### Declarative Over Imperative

Write **what to achieve**, not step-by-step procedures for every scenario. Imperative only for: safety-critical sequences, tool invocation syntax, compliance workflows.

Mixed approach (usually best): declare goal and constraints, provide imperative steps only for critical path.

Phased procedures: each phase opens with a one-line goal, body is terse imperatives. Hoist rationale to the goal line — keep procedure bodies free of mid-step "because…" asides.

### Right Altitude

| Level | Problem | Example |
|--|--|--|
| Too specific | Brittle, high maintenance | "If user says 'hello' respond 'Hi! How can I help you today?'" |
| Too vague | No actionable signal | "Be helpful and professional" |
| Right altitude | Guides via heuristics | "Greet warmly. Match formality level. Get to question quickly." |

Scope vague goals with audience + length + format: "300-word summary of X for <audience>" beats "Write about X" (Promptomatix §B.1.1).

### Token Economy

Baseline: qualifying hot-tier file (conventions + architecture + project description) reduces mean output tokens 20.08% and wall-clock 20.27% (Lulla 2601.20404, Codex + gpt-5.2; Claude generalization unverified).

- Tables for lookup data and comparisons. Use compact text for definitions
- Drop: articles, filler, connectors, pleasantries, hedging, transitions, emphasis inflation, passive padding, redundant verbs ("make sure to" → state action), decorative formatting (extra blank lines, excessive bold/italic, ellipsis, exclamation marks)
- Fragments OK. Short synonyms over long words. Technical terms exact
- Compressed prose pattern: `[thing] [action] [reason]. [next step].`
- One concept per sentence
- Short identifiers after first definition: `rps` not `records per second`
- State each fact once — reference, don't copy
- **Compress format, not meaning**: tables, fragments, abbreviations are safe. Merging, paraphrasing, or bulk-deleting accumulated bullets is NOT — ACE (Zhang §2.2, Fig 2) shows full-rewrite compression collapsed a playbook 18,282 → 122 tokens and dropped accuracy 66.7% → 57.1% (below baseline). Append and refine; never wholesale-rewrite
- **Bundling tax**: rules at 98.7% compliance in isolation drop to 85% when bundled with 18 others; 37.5% of requirements lose >5% when combined (Yang §3.4). Adding a rule earns a slot or replaces a weaker one — it does not squeeze in
- Ecosystem median SKILL.md ≈1.5k tokens / 2.3 KB (IQR 0.8–6.1 KB). Detailed +18.8pp and Compact +17.1pp beat Standard +10.1pp on SkillsBench; Comprehensive **hurts −2.9pp** (Li 2602.12670 Table 6)
- Budgets: hot tier ≤ 200 lines / ~2,000 tokens; warm-tier SKILL.md ≤ 500 lines / ~5,000 tokens (only 5% of 601 ecosystem skills exceed this). Prose ≤ 3 sentences per paragraph
- RECOMMENDED: mention WHERE to look over duplicating discoverable info: `Builder pattern: see Class#method`
- Specifically omit: package structure, builder patterns, class hierarchies, default constant values, code examples that mirror source
- MUST preserve verbatim: URLs, file paths, commands, proper nouns, **dates, version numbers, release markers, coreferences, cross-references**, env vars, YAML frontmatter. Flat fact-extraction "irretrievably loses" temporal markers and ephemeral updates (Pollertlam 2603.04814 §5.1)
- MUST use full clarity for security warnings, irreversible action confirmations, multi-step sequences where fragment ambiguity risks misread. Resume compression after
- **History growth**: full transcripts grow O(T²) in tokens. Summarize or truncate after ~20 turns — "context rot" past that threshold (Tokalator 2604.08290 §3.4)
- **Single-shot output fragility**: when a skill asks for output exceeding ~230 tokens, GPT-4o pass@1 collapses below 10% (Liang 2410.21647 §4.2). Skills that demand one large generated artifact are fragile — decompose into bounded outputs or iterative edits
- **Every useless instruction still gets obeyed**: repo-specific tool mentions used 2.5×/instance vs <0.01× unmentioned (Gloaguen §4.3). Instruction-following is strong, so junk rules spend budget on grep/test/write activity without improving solves — compression savings come from deletion, not paraphrase
- **Asymmetric inference**: format defaults (length, casing, layout) are inferred correctly 70.7% when unspecified; conditional / edge-case rules only 22.9% (Yang 2505.13360 §3.2). SHOULD spell out conditionals and edge-case branches; MAY omit format defaults the model handles natively
- **Counter the brevity bias**: well-functioning specialist skills tilt toward domain knowledge over behavioral rules — Vasilopoulos's working 19-agent system ran ~65% domain facts per spec (§3.2). Strip padding, not substance
- Before adding an instruction: "will the agent fail on a concrete eval without it?" If no — cut. Before deleting an accumulated playbook bullet: does an eval regress? Default for playbook bullets is keep-and-refine; default for unsourced prose is cut

### Focused Over Comprehensive

**2–3 skills optimal (+18.6pp gain); 1 skill +17.8pp; 4+ skills collapse to +5.9pp** (Li 2602.12670 Table 5). 16/84 tasks (~19%) regress when adding skills; worst case −39.3pp. Split by task boundary; don't bloat one file to cover adjacent domains.

For tasks needing isolated reasoning (parallel exploration, different model, high token budget), prefer a subagent — subagents fork a new context window rather than extending the current one (Liu 2604.14228 §6.2).

ROI: Haiku+skills (27.7%) beats Opus-without (22.0%) at $0.03–0.22/trial; average +16.2pp gain.

**Self-authoring regresses**: self-generated SKILL.md yields −1.3pp avg; Codex+GPT-5.2 **−5.6pp**; only Opus 4.6 marginal +1.4pp (Li 2602.12670 Finding 3). Gloaguen (2602.11988 §4.2): LLM-generated AGENTS.md costs +20–23% tokens with −0.5% to −2% success. Validate with a fresh agent on real tasks — never the same session that authored.

**Procedural, not instance-specific**: skills describe how to approach a class of tasks. SHOULD NOT embed task-specific filenames, paths, IDs, exact command sequences, magic constants, or references to test cases / expected outputs (Li 2602.12670 §2.4 forbid-list).

### Progressive Disclosure

Tiered architecture:

| Tier | Contents | Loading | Budget |
|--|--|--|--|
| Hot | CLAUDE.md / AGENTS.md — **(i) conventions + best practices, (ii) architecture + project structure, (iii) project description** (qualifying triple) | Always loaded | ≤ 200 lines / ~2,000 tokens |
| Warm | SKILL.md / domain-specific agent files — per-task specialists | Lazy-loaded by `SkillTool` on trigger — only frontmatter description sits in the prompt until invoked | ≤ 500 lines / ~5,000 tokens per file |
| Cold | Knowledge base docs, schemas, API specs | Retrieved on demand (MCP / search) | `see <file>` pointers |

**Retrieval is not the default for reference material.** Long-context retention beats fact-extracted memory by **33–35pp** on accuracy (LoCoMo 92.85% vs 57.68%; LongMemEval 82.40% vs 49.00%, Pollertlam §4.1). Retrieval wins only at ~10+ reuses of ~100k-token content with caching. Prefer hot/warm inclusion for content reused in-session; reserve MCP/search for rarely-touched specs.

Hot tier is your bottleneck. Every instruction there competes for attention. Move anything task-specific to warm tier; anything reference-like and rarely-touched to cold tier.

When multiple tools target the same repo, write conventions once in AGENTS.md and reference from CLAUDE.md / copilot-instructions.md — 311 outgoing CLAUDE.md → AGENTS.md references observed across 2,631 repos; AGENTS.md is the lowest-friction shared baseline (Galster §5.1).

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

### Observation Management — Prefer Masking Over Summarization

Runtime tool outputs consume ~84% of agent tokens (Lindenbauer 2508.21433 §1, Fig 1) — far more than skill prose. Simple masking of stale observations cuts cost 50.9–57.1%; LLM summarization 41.5–55.4% but adds 2.86–7.20% overhead for the summary pass itself. Favor structural rules over semantic rewrites.

Recency heuristic: M=10 turn window optimal on SWE-agent; M=20 degrades. Context-management settings are scaffold-bound — re-tune per harness.

**Pointer-ID pattern**: when a skill's tool returns large payloads, prefer returning a handle / path the agent dereferences on demand over inlining the blob — observed 16,900× and 7× token reductions in two case studies (Bulle Labate 2511.22729 §3). Complements masking and summarization as a third mechanism.

### Name the Function, Not the Surface Topic

Routers weigh name tokens heavily (3% of total, disproportionate per-token influence). Generic names lose to generic-named alternatives even when the body is specific (SkillRouter 2603.22455 Appendix L.1).

```
bad:  video-tutorial-indexer       (topic → router picks video-explorer on keyword)
good: speech-to-text               (function → 0/12 → 9/12 success)

bad:  software-dependency-audit    (generic → beaten by dependency-security)
good: trivy-offline-vulnerability-scanning  (tool + mode → 0/12 → 12/12)
```

### Positive Framing

Describe what TO do, not what NOT to do in body instructions — positive instructions are processed more reliably than negations (Promptomatix §B.1.1). Exceptions: safety boundaries ("never commit without tests"), and skill `description` negative triggers ("NOT for Vue or Svelte") — the latter improves routing precision by preventing false activation.

Frontier models (Opus 4+, GPT-5, Sonnet 4.5+) interpret prohibitions hyper-literally — "must NOT", "ONLY", "never" act as handcuffs rather than guardrails. Constraints that lifted GPT-4o 93→97% **dropped** GPT-5 96.36→94% (Khan 2510.22251 §6.2.1). Reserve ALL-CAPS prohibitions for safety / data integrity; prefer positive phrasing for style / format.

### Priority Markers

Keywords interpreted per BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in ALL CAPITALS:
- MUST / REQUIRED / SHALL: absolute requirement (safety, compliance, data integrity)
- MUST NOT / SHALL NOT: absolute prohibition
- SHOULD / RECOMMENDED: strong default, deviation requires justified reason
- SHOULD NOT / NOT RECOMMENDED: discouraged, deviation requires justified reason
- MAY / OPTIONAL: truly optional, valid alternatives coexist

Caps ≠ compliance. Anthropic's official skill-creator treats bare ALL-CAPS ALWAYS/NEVER as a "yellow flag" — pair every capitalized marker with hoisted rationale so the agent can judge edge cases rather than mimic edicts.

Target ≤ 3 ALL-CAPS prohibitions per SKILL.md; everything else as SHOULD or positive phrasing. Prohibitions compound on frontier models.

### Match Constraints to Model

Accuracy-based heuristic (Khan §6.2.1):
- Eval accuracy on target model < 90% → add hard constraints (Sculpting-style)
- Eval accuracy > 95% → keep prompts simple, goal-oriented
- 90–95% → A/B test both

Hard constraints that raise Haiku/Sonnet by +4 pts can degrade Opus/GPT-5 by ~2 pts. Implicit-rule inference varies by model: o3-mini recovers 44.7% of unstated requirements, Llama-3.3-70B only 24.5% (Yang §3.2) — a SKILL.md tuned on Opus under-specifies for smaller models. Accuracy rankings also don't predict reliability: Gemma-3-IT-27B ranks 17th on IFEval but 7th on reliable@10 (Dong Table 1).

Format: use XML tags, keep examples consistent in style. 2–5 diverse examples; **3 excellent beats 10 mediocre**. Place most-relevant example last — models weigh final examples 2–3× more. Inconsistent label/format across examples cuts effectiveness up to 40%.

When authoring trigger examples, use **cousin-prompt variants** (paraphrases of real queries) — cousin-augmented data lifts reliability >45% vs generic Alpaca-style examples, which decline reliability (Dong Fig 5).

**Harness binding**: same skill + model can swing **+13.6 to +23.3 pp** across harnesses (Li 2602.12670 §5). Match harness constraints explicitly — for JSON-only or strict tool-use protocols, include repeated format reminders in the body; one-shot instructions may not survive harness wrappers.

## Mechanism Decision Boundary

Choosing between skill / subagent / hook / MCP (Liu 2604.14228 §6.2–6.3):

| Mechanism | Use when | Context cost | Scope |
|--|--|--|--|
| Skill | Shape how the agent thinks; task-specific expertise | **Low** (frontmatter in prompt; body lazy-loaded by `SkillTool`) | Current session |
| Subagent | Isolated reasoning, parallel exploration | **New context window** | Forked; returns summary |
| Hook | Cross-cutting lifecycle (pre/post-tool, stop, automation) | **Zero** | Deterministic, settings-driven |
| MCP | External tool / data source | **High** (full tool schemas every turn) | Session |

Cost ordering: hooks < skills < plugins < MCP. Migrating an MCP tool into a skill is a context-budget win.

## Context Rot — Four Failure Modes

Beyond simple bloat (Vishnyakova 2603.09619 §9):

| Mode | Symptom | Cause | Fix |
|--|--|--|--|
| Poisoning | Hallucination reproduced every step | One bad fact cached and reused | Purge poisoned turns; restart from clean state |
| Distraction | Performance collapses past ~100k tokens | Attention dilutes | Summarize or tier into warm |
| Confusion | Irrelevant material degrades response | Low-signal tokens crowd out high-signal | Prune; move to cold |
| Clash | ~39% quality drop when one intent split across sequential turns | Fragmented prompt | Consolidate into one message |

Diagnose before compressing — the symptom is rarely "too many tokens."

## When Editing Technical Docs — Preserve

- Architecture and data flows — preserve the information, rewrite diagrams as text topology
- Concurrency: sync points, memory visibility, happens-before, lock guarantees
- Performance rationale with quantified tradeoffs
- API contracts, protocol specs
- Design WHY (non-obvious decisions)
- Invariants
- Version strings, release dates, migration timelines — verbatim, not paraphrased into relative terms

## Structure

- `#` Major domains, `##` Components, `###` Details
- Flat hierarchy: 2–3 levels typical (empirical median across 253 CLAUDE.md files: H1=1, H2=5, H3=9; H4 in <15%). Depth > 3 signals over-nesting
- Headers create hierarchy; bold/italic sparingly: critical terms/warnings only
- Lists for instructions (clearer than paragraphs)
- Dense format for tables — compact separators, no column-width padding
- Horizontal rules (`---`) only for major breaks
- State facts directly; skip section intros and meta-commentary
- Write at ~8th grade reading level
- Mix five instruction styles deliberately (Mohsenimofidi §4.2): descriptive ("uses X pattern"), prescriptive ("follow Y"), **prohibitive ("never Z")**, conditional ("if Z, then W"), explanatory ("avoid X because Y")
- Describe topology in text: `Hierarchy: X → Y → Z[]`, `Flow: a → b → c → d`. Mermaid appears in only 2 of 328 surveyed CLAUDE.md files; prose topology dominates
- Place load-bearing rules (safety, routing triggers, MUSTs) near the top of the file — retrieval accuracy drops as position moves later, effect compounds with input length (Hong 2025 Context Rot)
- Counterintuitive: shuffled / discrete content beats logically structured prose for reference material — "structural coherence consistently hurts model performance" (Hong 2025). Use bullets and tables for reference content; reserve narrative flow for places where argumentation genuinely requires it
- Use bad/good contrast pairs to demonstrate rules. `bad: "Sure! I'd be happy to help you with that."` → `good: "Bug in auth middleware. Fix:"`

## Describing Content Types

### Code Conventions

Reference source files; SHOULD NOT paste snippets by default. Code examples appear in 17.68% of Development Guidelines sections in the wild (Santos §4.3) — MAY include minimal illustrative snippets when a pointer is insufficient. Options:
- Point to examples: `<Pattern>: follow <path/to/file>`
- Describe abstractly: `Errors: Result<T, E> pattern. Never throw from business logic.`
- Critical rules: `MUST: run lint before every commit`

Reference by method, class, or function name: `see Class#method`. Avoid line numbers — they rot on edit.

For illustrative examples (teaching a pattern, not pointing to specific code) — SHOULD use generic placeholders. Real names rot on rename and confuse pattern-vs-pointer intent.

bad: `Builder pattern: see UserService.findById()`
good: `Builder pattern: see Class#method`

### Schemas / Data Models

Simple models — inline: `Order: { id: string, status: "pending"|"paid"|"shipped", total: number (cents) }`

Complex models — separate file + key constraints in main context.

### Tool Descriptions

Each tool answers: **When** (trigger), **What** (one sentence), **How** (params), **Returns** (structure, edge cases). Minimize overlap — if unclear which tool to use, neither can the agent.

Before documenting an MCP tool — consider migrating it into a skill. MCP pays full schema cost every turn; a skill's workflow description pays zero until invoked.

### Gotchas Section

Include a "Gotchas — Common Wrong Assumptions" section at the end of each context file. Numbered list of non-obvious behaviors with stable IDs (G-01, G-02 …) so individual entries can be updated without rewriting siblings. Keep them concentrated, not scattered across sections.

For "Known Failure Modes" use **symptom → cause → fix** triples — symptom-first indexing lets the agent match observed behavior to remedy faster than scanning paragraphs (Vasilopoulos §4.4.4).

### Agent Persona

2-3 precise sentences. Avoid personality essays.

```
You are a senior backend engineer specializing in Java performance.
Tone: direct, technical. Expertise: JVM internals, concurrency, profiling.
Approach: always profile before optimizing; never guess at bottlenecks.
```

### Untrusted Input Handling

Wrap every user-supplied payload in a delimited template: `<user_input>…</user_input>`. Embed safety constraints in the system-prompt tier (hot), not inline next to the payload — proactive beats reactive (Promptomatix §B.5.1).

### SKILL.md Frontmatter

Anthropic-official hard limits (auto-rejected if exceeded):
- `name`: ≤ 64 chars, lowercase letters/digits/hyphens only; gerund form. Naming strategy — see *Name the Function, Not the Surface Topic* principle
- `description`: ≤ 1024 chars, third person, non-empty; include what it does, when to trigger, and negative triggers

Third-person only — "I can…" / "you use this to…" degrades routing.

`bad: "Helps with agent files"`
`good: "Audit AI agent context files. Use when editing CLAUDE.md, SKILL.md, or agent prompts. NOT for generating new agents from scratch."`

**Routers inspect full skill text**, not just the frontmatter. Hiding body drops Hit@1 by **31–44pp** vs full-text routing (SkillRouter 2603.22455, Fig 1). Body SHOULD contain routing-signal vocabulary: tool names, trigger phrases, negative cases, function verbs. SHOULD NOT shrink body below ~400–700 words of substantive routing signal (median effective body: 704 words). The claim "description is the only signal" is **empirically false** for modern routers — it is what the agent sees post-routing, not what the router uses.

Reference files one level deep from SKILL.md — deeper nesting triggers partial reads (`head -100`). Reference files > 100 lines MUST include a table of contents so partial previews surface full scope. Body budget: see *Token Economy*.

Optional frontmatter fields (Liu §6.1 — `parseSkillFrontmatterFields` exposes 15+ fields):
- `allowed-tools`: restrict tool surface
- `model`: pin a specific model (e.g., `claude-opus-4-7`)
- `effort`: `low` / `medium` / `high` reasoning budget
- `execution`: `'fork'` for isolated context window
- `argument-hint`: CLI arg discovery
- Skill-scoped hooks register dynamically on invocation and unregister when the skill ends — use for lifecycle confined to the skill's session; global hooks belong in `settings.json`

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

- Repository overviews and directory listings — 8/12 developer AGENTS.md files include overviews, yet "context files, even developer-provided ones, are not effective at providing a repository overview" (Gloaguen §4.2). Claude Code's own generation prompt warns against listing easily-discoverable components
- Anything duplicating README/docs — deleting README flips LLM-generated AGENTS.md from −2% to +2.7% on AGENTbench (Gloaguen §4.2); duplication actively harms
- Anything auto-generated by an LLM — cost +20–23% tokens with −0.5% to −2% success
- Style/formatting rules enforceable by linter
- "Nice to know" background that doesn't change the agent's next action

## Anti-Patterns to Fix

| Anti-Pattern | Fix |
|--|--|
| CLAUDE.md > 200 lines | Move task-specific content to reference files |
| Pasted code snippets | Replace with pointers to source files |
| Same rule or example stated more than once | Keep single best, delete the rest |
| Same rule duplicated across CLAUDE.md / AGENTS.md / copilot-instructions.md | Write once in AGENTS.md; reference from others (observed: 311 CLAUDE.md → AGENTS.md references across 2,631 repos) |
| Implicit assumptions | State explicitly or point to examples |
| Contradictory rules | Resolve into single rule with conditions |
| 50 if-then edge cases | 3-5 canonical examples + declarative heuristic |
| 500-word persona | 2-3 behavioral anchors |
| "Don't do X" lists | Reframe as what to do instead |
| Task-specific leakage — filenames, paths, IDs, magic constants, expected outputs | Procedural only; skills apply to a class of tasks, not a single instance |
| Auto-generated context file (LLM-written) | Delete. Costs +20–23% tokens and −0.5% to −2% success. Human-author minimal requirements only |
| Rule with no traceable source | Cut. Brainstormed-only rules (~23% of authored rules in the wild) are the most likely dead weight |
| Skill bundles scripts / binaries without audit | Review each executable; script-bundling skills are **2.12× more likely** to be vulnerable |
| Eval set = single phrasings | Replace with cousin-prompt set — 10 paraphrases, joint-pass (reliable@10) |
| Uncached hot-tier content reused ≥ 2× | Enable prompt caching — break-even at n*=2 |
| Sub-skill inherits full parent privileges | Privilege attenuation — grant smallest slice of tools / paths / secrets needed |
| Static project inventories | Replace with discovery command (`./overview.sh`, `grep …`). Auto-allow in `.claude/settings.json` |
| Descriptive rules ("X is derived from Y") | Reframe as actions ("Before adding X — verify Y") or pointers ("See X: `command`") |
| Inline justifications mid-procedure ("because…", "this makes…") | Hoist to phase goal line or cut; keep body terse imperatives |
| File only grows, never pruned | When the file outgrows its budget — time to review |
| Only build/run/arch rules | Add security (rarest empirically at ~6 per 100 repos), tests, goals, error-handling |
| Over-pruned to bare minimum | Default for playbook bullets is keep-and-refine. ACE shows full-rewrite compression below threshold drops accuracy by 9.6pp |
| Near-duplicate skills in catalog | Audit for semantic overlap; consolidate or sharpen name / description to create routing separation. Near-duplicates cause router false negatives beyond what exact-dupe merge catches (Zheng §4) |

## Self-Review Checklist

- Frontmatter `description` is self-contained — no cross-file references, no citations, readable in isolation during skill routing
- Hot-tier file covers the qualifying triple (conventions + architecture + project description)
- Every section serves a clear purpose — no decorative text
- No instruction repeated across sections
- Declarative where possible, imperative only where necessary
- Tables only for lookup data
- Code conventions reference source files, not pasted snippets
- No references to renamed/removed code entities
- No task-specific leakage — no filenames, paths, IDs, magic constants, expected outputs
- Tool descriptions non-overlapping with trigger conditions
- Examples cover happy path, edge case, and escalation; example labels/format consistent; best-fit example placed last
- Total token count justified — anything movable to reference files?
- No contradictions between sections
- Priority markers used consistently; ALL-CAPS prohibitions ≤ 3, each tied to safety / data integrity (not style)
- Eval constraint density matches model heuristic (<90% → constrained; >95% → simple)
- Escalation/failure paths defined
- Gotchas section present at end of file with stable IDs
- Every hard rule has a dedicated validator (one per rule, not holistic — per-requirement validators hit 95.6% human-LLM agreement, Yang §A.6)
- Description passes **reliable@10** — 10 paraphrased should-trigger + 10 should-not, all must pass jointly
- Body contains routing-signal vocabulary (tool names, triggers, negative cases)
- Name describes the function performed, not the domain
- Body scope matches description — no tool / file-access / external-call drift
- Every rule / threshold traces to a named source (paper, RFC, spec, dated hands-on note) — untraceable rules MUST be cut or marked provisional
- User-input payloads wrapped in delimited template; safety instructions in system-prompt tier

## Maintenance

- Add to existing section if conceptually fits; new section only when no match exists. Most field edits are "Add instruction" (~78) vs "Modify" (~59) — prefer modifying existing rules over appending new ones
- MUST merge exact duplicates immediately (exact only — never bulk-paraphrase accumulated bullets, which risks context collapse)
- Maintain via delta updates: APPEND new bullets; UPDATE existing bullets in place; DEDUPE only via semantic-embedding similarity (Zhang §3.1–3.2)
- After every significant codebase change: grep context files for references to renamed/removed entities. For larger systems, run a drift detector that diffs Git commits against spec mappings and flags specs whose tracked paths changed (Vasilopoulos §5.2)
- Time budget: expect ~5 min/session when touching a spec + biweekly 30–45 min review = 1–2 hr/week overhead
- If you explained the same thing to the agent twice across sessions, codify it (heuristic G4: "explained twice → write it down")
- On every body edit that changes scope (new tool, new file access, new external call) — update the description in the same commit and note the delta in the commit message
- Pin model version in eval runs. Unspecified requirements are ~2× as likely to regress across model / prompt changes; a minor model bump silently dropped one rule's compliance by 48% (Yang §3.3). Re-run evals on every model upgrade
- MUST capture developer knowledge not found in docs or source code

## Gotchas

G-01. Skill fails to trigger? Check `description` first (negative triggers, third person, when-to-use). If description is solid but routing still misses — the body is under-signaling. Routers use full text; descriptions cannot substitute (31.8pp gap even for long descriptions). Optimize with reliable@10 and iterate description; also tighten the body's routing vocabulary.
G-02. Agent ignores MUST/NEVER in ALL CAPS. Pair every caps-marker with a one-sentence rationale — agents follow reasons better than edicts.
G-03. `description` is pasted verbatim into the system prompt. Third-person is mandatory; first/second person breaks routing.
G-04. Positive framing holds for body; negative triggers belong in `description` only. Don't conflate contexts.
G-05. Self-generated SKILL.md / AGENTS.md regresses: −1.3pp avg, Codex+GPT-5.2 −5.6pp, Opus 4.6 marginal +1.4pp (Li 2602.12670); +20–23% token cost for −0.5% to −2% success (Gloaguen 2602.11988). Validate with a fresh agent on real tasks.
G-06. ~26.1% of 42,447 community skills contain prompt-injection vulnerabilities; skills bundling executable scripts are **2.12× more vulnerable**; the ClawHavoc campaign compromised 1,184 published skills. Before installing third-party skills, diff against the last-consented version — trust is bound to identity, not content, so post-install edits bypass consent.
G-07. Reference files two hops from SKILL.md get partial reads. Keep references flat; link every reference directly from SKILL.md.
G-08. Numeric edits and constraint reconfiguration degrade more than pure rephrasing — "at most 600" vs "610" causes widespread failure (Dong §3.1). Lock exact numerics; re-run reliable@10 after any constraint edit.
G-09. **Bundling tax**: a rule at 98.7% in isolation can land at 85% when bundled with 18 others; 37.5% of rules lose >5% when combined (Yang §3.4). If adding rule N suddenly breaks rule N−5, it is bundling interference — not model regression. Split or prune.
G-10. **Prohibitions invert on strong models**: constraints that lifted GPT-4o 93→97% dropped GPT-5 96.36→94% (Khan §6.2.1). "MUST NOT / ONLY / NEVER" act as guardrails on Haiku/Sonnet, handcuffs on Opus/GPT-5. Re-run evals before porting a skill across model tiers.
G-11. YAML frontmatter is not a contract — the routing layer has no mechanism to verify the body stays within what `description` claims (Li 2604.02837 §3.1). Read the body as an adversary during self-review: does it do anything the description doesn't advertise?
G-12. Persistent cross-session memory files are not an observed pattern across 2,631 repos (Galster §6.3) — rely on CLAUDE.md / AGENTS.md edits for durable state; do not invent bespoke memory schemes.
G-13. Don't trust model self-report when debugging a non-trigger. Self-confidence AUROC 0.549, perplexity 0.497 (both near-random); hidden-state probing peaks at 0.757 (Dong Table 2). Diagnose by running new cousin prompts, not by asking "why did you fail?"
G-14. Cross-skill body references fail at routing time — other skills' bodies are not in context until their own `SkillTool` fires (Liu §6.3). Descriptions must be self-contained.
G-15. Window-size / context-management tuning is scaffold-specific — a setting tuned on SWE-agent regresses on OpenHands until re-tuned (Lindenbauer §6). Never port context-management hyperparameters blindly.
G-16. Context rot has four distinct failure modes (see table above): poisoning, distraction, confusion, clash (splitting one intent across sequential turns drops quality ~39%). Diagnose before compressing — the symptom is rarely "too many tokens."
G-17. LLM-as-judge drifts on ambiguous correctness. When scoring skill outputs with an LLM (not deterministic tests), prefer multi-round adversarial debate (d=3 rounds, two advocates + judge) — beats single-pass LLM-as-judge 80.3–95% win rate (Nair 2506.00178 Table 4). Single-judge evals inflate pass rates on subjective criteria.
G-18. No runtime mechanism arbitrates between simultaneously triggered skills — conflict resolution, resource sharing, and failure recovery across skills remain underdeveloped (Xu & Yan §7). Design descriptions with disjoint triggers; verify via should-not-trigger cousin prompts that your skill doesn't steal activations from adjacent ones.
G-19. Direct prompt injection remains an open problem not fully addressable by wrapper/template discipline alone (Li 2604.02837 §7.1). The Untrusted Input Handling section reduces risk but does not eliminate it — treat defense as best-effort and pair with human review for high-trust operations.
