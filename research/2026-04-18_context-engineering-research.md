# Authoring AI agent skills — arXiv evidence

## Research questions to cross-check against sources

### Questions the skill already addresses

**Authoring process**
- Should evals be drafted before or after the body?
- How do you validate a skill — same session vs. fresh agent?
- Who decides when a draft is ready to ship?

**Shape of instructions**
- Write "do X, then Y" or "achieve goal Z"? When does each win?
- What's the right altitude — how specific before brittle, how general before useless?
- Declarative vs. imperative — where's the boundary?
- Where does "why" belong — inside the procedure or hoisted to the phase goal?

**Prohibitions and permissions**
- Frame rules as "do X" or "don't do Y" — which when?
- Where is negative framing legitimate (safety, description negative triggers)?
- Do ALL CAPS markers actually change agent behavior, and under what conditions?
- MUST / SHOULD / MAY — how to pick the strength?

**Token economy**
- What can be safely compressed or dropped?
- What must be preserved verbatim (URLs, commands, versions, paths)?
- What are the line budgets for CLAUDE.md, SKILL.md, reference files?
- When are fragments acceptable vs. full sentences required?

**Context architecture**
- What loads always, what on task match, what on retrieval?
- Split a domain into one file or several?
- Narrow skill vs. broad skill — which generalizes better?
- How to structure references so they don't trigger partial reads?

**Format and structure**
- Table, list, or prose — for which data type?
- How to describe topology without ASCII art?
- How many examples, in what order, bad/good pairs?

**Describing code**
- Paste snippets or point to files?
- Real method names or placeholders?
- Are line-number references acceptable?

**Frontmatter and routing**
- How to write a `description` that triggers where intended?
- What to do when a skill fails to trigger?
- How to handle user paraphrases?

**Maintenance over time**
- When to update, merge, or delete a rule?
- How to catch references to renamed or removed code?
- When is it time to split a file?

### Adjacent questions the skill does not address (candidates for expansion)

**Eval methodology**
- How exactly do you author an eval query — format, source, who generates it?
- What counts as a "should-not trigger" — false positive vs. competing skill?
- How to automate eval runs in CI?
- How to measure reliability as a number, not by feel?

**Skill interaction**
- If two skills match at once, who wins?
- Can one skill invoke another? Chains?
- How is state passed between skills within a session?
- Priority: project skill vs. plugin skill vs. user skill?

**Model and environment binding**
- How does style concretely shift across Opus / Sonnet / Haiku?
- Does the budget change when the agent runs with 1M context vs. 200K?
- Does a skill port to Cursor / Cline / Aider, or is it Claude Code only?
- Does behavior depend on OS, shell, or CWD — and how to declare that?

**Security and trust**
- How to audit a third-party skill before installing it?
- What is prompt injection via skill body, and how to defend against it?
- What tool permissions to grant, and at what scope?
- Can skills be signed or verified?

**Observability and debugging**
- How to diagnose why a skill did *not* fire (not only why it fired wrong)?
- Are there routing-decision logs?
- How to trace which part of the skill actually affected behavior?
- Metrics: activation rate, precision/recall, cost per invocation?

**Versioning and evolution**
- How to ship breaking changes in a skill that users already depend on?
- Does a skill need a changelog?
- How to migrate older skills to newer conventions?
- Deprecation policy?

**Cost**
- How many tokens / dollars does loading a skill into the hot tier cost?
- How to compare ROI between monolith and split?
- Is the cold tier via MCP justified if the round-trip is expensive?

**Boundaries of applicability**
- When does a task *not* need a skill — system prompt or CLAUDE.md suffices?
- When should a subagent be used instead of a skill?
- When is a hook the right tool instead of a skill (automation vs. intent reaction)?

**User experience**
- How does the user learn that a skill fired or failed to fire?
- How can the user force or suppress a skill?
- Where do users report skill bugs?

**Internationalization**
- Are trigger phrases English-only, or multilingual?
- How to handle users who switch languages mid-session?

## 2024

- 2024-10 · [2410.21647v4](https://arxiv.org/html/2410.21647v4) — Liang et al. — Can Language Models Replace Programmers for Coding? RepoCod Says "Not Yet"
  - Benchmark of LLM repo-level code generation, not skill authoring — two tangential signals worth keeping:
  - **Context architecture**: RepoCod context strategies cluster tightly — GPT-4o RAGBM25 27.4%, RAGDense 27.0%, Current-File 26.8% pass@1 (§4.3, Table 3); retrieval wins by <1 pp margin, not decisively. Absolute pass@1 stays below 30% across all context strategies — repo-level code generation is context-limited regardless of tier choice. Does not overturn Pollertlam's long-context > retrieval finding on memory-style content.
  - **Token economy**: For functions >232 tokens output, GPT-4o drops below 10% pass@1; highest-complexity bucket 7.0% (§4.2) — skills demanding large single-shot outputs are fragile, prefer decomposition.

## 2025

- 2025-05 · [2505.13360v2](https://arxiv.org/html/2505.13360v2) — Yang et al. — What Prompts Don't Say: Understanding and Managing Underspecification in LLM Prompts
  - **Token economy**: Specifying all 19 requirements drops gpt-4o from 98.7% (isolated) to 85.0% combined; 37.5% of requirements suffer >5% decline when bundled (§3.4). A Bayesian optimizer selecting which rules to include gained +3.8% while cutting tokens 41–45% — fewer well-chosen rules beat exhaustive lists.
  - **Shape of instructions**: Format requirements inferred correctly 70.7% when unspecified; conditional/edge-case rules only 22.9% (§3.2) — always spell out conditionals, safely omit format defaults.
  - **Maintenance / versioning**: Unspecified requirements are "2× as likely to regress across model or prompt changes"; a gpt-4o minor bump dropped "skimmable outputs" performance by 48% (§3.3).
  - **Eval methodology**: Per-requirement validators (planner drafts logic, validator executes) reached 95.6% human-LLM agreement on 1,095 validations (§A.6) — pattern for SKILL.md CI: one validator per rule, not a holistic score.
  - **Authoring process**: 41.7% of rules discovered via error analysis, 35% from mining existing prompts, 23.3% from LLM brainstorming (§3.1) — don't rely on brainstorming alone.
  - **Model binding**: o3-mini infers 44.7% of implicit rules vs. Llama-3.3-70B 24.5% (§3.2) — a SKILL.md tuned on Opus may under-specify when ported to smaller models.
- 2025-06 · [2506.00178v2](https://arxiv.org/html/2506.00178v2) — Nair et al. — Tournament of Prompts: Evolving LLM Instructions Through Structured Debates and Elo Ratings
  - Automated evolutionary prompt optimization (DEEVO/TEAM), not human-authored skill files — two findings transfer:
  - **Shape of instructions**: Optimized prompts converged on numbered procedural steps + XML-style structural tags (`<analysis>`, `<pain_point>`) — weak evidence imperative numbered procedures + output scaffolding beat free-form (Appendix A.4).
  - **Eval methodology**: Multi-round LLM debate (d=3 rounds, two advocates + judge) outperformed single-pass LLM-as-judge 80.3–95% win rate (Table 4) — adversarial/multi-round eval materially stronger than single-judge scoring. Applicable to SKILL.md regression suites where the judge is itself an LLM.
- 2025-07 · [2507.14241v3](https://arxiv.org/html/2507.14241v3) — Murthy et al. — Promptomatix: An Automatic Prompt Optimization Framework for LLMs
  - Not directly relevant as framework; Appendix B guidelines transfer to skill authoring:
  - **Prohibitions (framing)**: "Employ 'do' statements rather than 'don't'… positive instructions are processed more reliably than negations" (§B.1.1) — reframe skill prohibitions positively.
  - **Shape of instructions**: Replace vague directives ("Write about leadership") with scoped ones ("300-word summary of transformational leadership for university administrators") (§B.1.1).
  - **Format**: Specify desired structure, length, style explicitly (§B.1.2).
  - **Token economy**: Length penalty λ=0.005 shrank prompts 29→16.5 tokens retaining 99.9% peak performance; aggressive λ=0.05 eliminated gains (Table 2).
  - **Examples (count/order)**: 2–5 diverse examples; 3 excellent > 10 mediocre; place most-relevant last ("final examples have 2–3× more influence"); inconsistent label format cuts effectiveness "by up to 40%" (§B.2.1).
  - **Security**: Wrap user inputs in structured/guarded templates; embed safety instructions in system prompt — "proactive safety instructions are more reliable than reactive filtering" (§B.5.1).
- 2025-08 · [2508.21433v3](https://arxiv.org/html/2508.21433v3) — Lindenbauer et al. — The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization for Agent Context Management
  - **Token economy**: Observations/tool outputs consume ~84% of tokens in SWE-agent turns (Fig 1) — the hot skill-text tier is not where agentic spend dominates, so don't over-compress skill prose at the expense of clarity.
  - **Context architecture**: Masking with M=10 turn window was optimal; M=20 degraded performance (Fig 10) — recent context is often sufficient, older context belongs on-demand/retrieval.
  - **Cost (compression ROI)**: Simple masking reduces cost 50.9–57.1%; LLM summarization 41.5–55.4% but adds 2.86–7.20% for summary generation — "LLM-Summary does not consistently or significantly outperform Observation Masking" (Abstract). Favor structural rules over LLM-mediated rewrites.
  - **Eval methodology**: Context-management settings are scaffold-bound — generalized to OpenHands only after re-tuning window size; evals must re-tune per harness (§5.1).
- 2025-09 · [2509.14744v1](https://arxiv.org/html/2509.14744v1) — Chatlatanagulchai et al. — On the Use of Agentic Coding Manifests: An Empirical Study of Claude Code
  - Descriptive content analysis of 253 CLAUDE.md files across 242 GitHub repos — catalogs what manifests contain, not what works. No evals, no performance metrics.
  - **Context architecture**: Shallow hierarchies — median H1=1, H2=5, H3=9; H4 appears in only 37/253 files, H5 in 5, H6 in 1 — authors converge on 2–3 heading levels.
  - **Content distribution**: Build and Run 77.1%, Implementation Details 71.9%, Architecture 64.8%, Testing 60.5%, System Overview 48.2%; only 15.4% define the agent's role/responsibilities.
  - **Authoring process (open problem)**: "The lack of comprehensive and accessible documentation for creating these manifests presents a significant challenge… trial-and-error approaches" — positions authoring guidance as unresolved.
- 2025-10 · [2510.04618v3](https://arxiv.org/html/2510.04618v3) — Zhang et al. — Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models (ACE)
  - **Shape of instructions**: Argues against "concise summaries"; frames contexts as "comprehensive, structured playbooks that continuously accumulate, refine, and organize strategies" — favors "domain-specific heuristics, tool-use guidelines, or common failure modes" (§2.2, §3).
  - **Token economy (anti-compression)**: Full-rewrite compression collapsed 18,282 → 122 tokens and dropped accuracy 66.7% → 57.1% (below 63.7% baseline, §2.2, Fig 2) — evidence against aggressive rewrites of working context.
  - **Context architecture (always-loaded)**: "LLMs are more effective when provided with long, detailed contexts and can distill relevance autonomously"; with KV-cache 91.8% of input tokens served from cache, 82.6% billed-cost reduction (§4.7) — supports large always-loaded CLAUDE.md when cache-friendly.
  - **Format**: Bullets with metadata (ID, helpful/harmful counters) + content outperform monolithic prose because bullets enable localized edits (§3.1).
  - **Maintenance**: Append new bullets; update existing in place (increment counters); semantic-embedding dedup; refinement proactive (after each delta) or lazy (only at context-window limit) (§3.1–3.2).
  - **Boundaries (narrow vs broad)**: Detailed playbooks help "detailed domain knowledge, complex tool use"; simple tasks "benefit more from concise, high-level instructions" (§5) — evidence for narrow/deep skills over broad/shallow.
- 2025-10 · [2510.16786v2](https://arxiv.org/html/2510.16786v2) — Gao & Peng — More with Less: An Empirical Study of Turn-Control Strategies for Efficient Coding Agents
  - Studies turn-count limits for SWE-bench agents, not skill authoring — three transferable signals:
  - **Token economy**: Conversation history grows "often quadratically (O(n²))" with turns (§1) — verbose always-loaded context compounds cost.
  - **Eval methodology**: Case studies repeated 10× per task; 80%/40% success gap failed to reach p<0.05 — single-shot skill evals unreliable, repetition required. Reinforces the `reliable@k` and n=3 rejection-sampling rules in SKILL.md Process.
  - **Model binding**: Same prompt produced Claude 4 Sonnet 75% / $7.80, Gemini 2.5 Pro 63% / $6.47, GPT-4.1 62% / $5.19 (§3.1.3); GPT-4.1 "graceful degradation" vs Gemini "threshold effect" — concrete cost/accuracy numbers that prompts don't port cleanly across models.
- 2025-10 · [2510.21413v4](https://arxiv.org/html/2510.21413v4) — Mohsenimofidi et al. — Context Engineering for AI Agents in Open-Source Software
  - Descriptive study of 466 OSS repos with AI context files; explicitly defers "how content/structure/style affect agent behavior" to future work.
  - **Content categories (§4.2, Table 1)**: Conventions (50), Contribution guidelines (48), Architecture (47), Build commands (40), Goals (32), Test execution (32), Metadata (29); security rarest (6).
  - **Shape of instructions (five styles)**: Descriptive, Prescriptive, Prohibitive, Explanatory (rationale inline), Conditional — authors do not rank effectiveness.
  - **Token economy (observational)**: File length means — Copilot 310 lines (SD=127), CLAUDE.md 287 (SD=112), AGENTS.md 142 (SD=231), GEMINI.md 106 (SD=65).
  - **Maintenance**: Of 155 AGENTS.md files — 50% unchanged, 23% modified once, 21% modified 2–7 times; top edit types "Add instruction" (78), "Modify instruction" (59).
  - **Adoption baseline**: Only 466/10,000 (5%) repos have any AI context file — field "still in an early stage."
- 2025-10 · [2510.22251v1](https://arxiv.org/html/2510.22251v1) — Khan — You Don't Need Prompt Engineering Anymore: The Prompting Inversion
  - Narrow empirical study on GSM8K math, not skill files — but two findings on prohibition framing transfer:
  - **Model binding**: Constrained "Sculpting" prompts beat CoT +4 pts on GPT-4o (97% vs 93%) but underperformed −2.36 pts on GPT-5 (94% vs 96.36%). Heuristic (§6.2.1): accuracy <90% → constrained; >95% → simple; else A/B test. Hard constraints helpful for Haiku/Sonnet may degrade Opus.
  - **Prohibitions (negative framing)**: Negative constraints ("must NOT", "ONLY") act as "guardrails" on weaker models but "handcuffs" on stronger — GPT-5 hyper-literal misreads of idioms (e.g., "two times older" as Y+2Y). Aggressive prohibitions in SKILL.md may hurt on frontier models.
- 2025-11 · [2511.09268v1](https://arxiv.org/html/2511.09268v1) — Santos et al. — Decoding the Configuration of AI Coding Agents: Insights from Claude Code Projects
  - Descriptive empirical study of 328 CLAUDE.md files across top GitHub repos — catalogs section frequencies, does not evaluate technique.
  - **Context architecture**: Median 7 level-2 headings (range 0–213); Architecture 72.6%, Development Guidelines 44.8%, Project Overview 39%, Testing Guidelines 35.4%, Dependencies 30.8% (Table 1). Architecture appears in all top-5 patterns (RQ3).
  - **Format**: Code examples in 17.68% of Development Guidelines (highest); external file links only 1.83% of Architecture; Mermaid diagrams in only 2 of 328 files — prose dominates.
  - **Descriptive only**: No analysis of phrasing style, CAPS/MUST/SHOULD, token economy, routing, eval methodology, skill interaction, or model binding.
- 2025-11 · [2511.22729v1](https://arxiv.org/html/2511.22729v1) — Bulle Labate et al. — Solving Context Window Overflow in AI Agents via Memory Pointers
  - Runtime memory-pointer wrapper for tool outputs, not skill authoring — one transferable pattern:
  - **Context architecture**: Replacing raw tool outputs with pointer IDs achieved ~16,900× and ~7× token reduction (20,822,181 → 1,234 tokens; 6,411 → 842 tokens, §3) — supports skills whose tools return handles/paths rather than inline blobs. Informs SKILL.md's Observation Management section.
- 2025-12 · [2512.14754v2](https://arxiv.org/html/2512.14754v2) — Dong et al. — Revisiting the Reliability of Language Models in Instruction-Following
  - **Shape of instructions**: Constraint/task reconfiguration causes greater reliability drops than rephrasing; minor numeric edits ("at most 600" vs "610") cause widespread failures (§3.1) — behavior shifts with numeric/structural edits, not just semantic.
  - **Eval methodology (reliability as number)**: Introduces `reliable@k` — all k paraphrased "cousin prompts" must pass simultaneously; IFEval++ = 541 cases × 10 paraphrases (§2.2). Apply to skill eval: measure trigger/behavior across paraphrases, not single queries.
  - **Paraphrase handling**: GPT-5 drops 18.3% from nominal to reliable@10; Qwen3-0.6B drops 61.8% relative (Table 1) — skills that "work" on author-written queries silently fail on user paraphrases.
  - **Shipping criteria**: Rejection sampling n=3 lifts Qwen3-4B above LLaMA-3.3-70B; plateaus ~n=12 (§5) — validate via multi-sample agreement, not single-shot.
  - **Observability (non-trigger diagnosis)**: Verbalized self-confidence near-random (0.549 AUROC); perplexity useless (0.497); hidden-state probing best but still unreliable (0.757, Table 2) — don't trust model self-confidence, rely on external behavioral eval.
  - **Model binding**: Gemma-3-IT-27B ranks 17th on IFEval but 7th on reliable@10; Qwen3-14B beats Qwen3-32B on reliability despite lower accuracy (§4.2) — accuracy rankings don't predict reliability.
  - **Trigger examples**: Augmented cousin-prompt training lifts reliability >45%; generic Alpaca training slightly declines (Fig 5) — adding paraphrased trigger examples to descriptions likely beats generic polish.

## Non-arxiv sources

- 2025 · [trychroma.com/research/context-rot](https://www.trychroma.com/research/context-rot) — Hong, Troynikov, Huber — Context Rot: How Increasing Input Tokens Impacts LLM Performance
  - 18-model study (GPT-4.1, Claude Opus/Sonnet 4, Gemini 2.5, Qwen3, …) — every extra token carries non-zero reliability cost, even on simple tasks.
  - **Token economy**: "Model performance varies significantly as input length changes, even on simple tasks" — the 10,000th token is not processed as reliably as the 100th. Near-perfect NIAH scores do not generalize to agentic context loads.
  - **Context architecture (focused beats full)**: On LongMemEval (~113k tokens), "across all models, we see significantly higher performance on focused prompts compared to full prompts" — reasoning mode narrows but does not close the gap. Direct evidence for tiered / on-demand loading over always-on dumps.
  - **Shape of instructions (placement)**: In Repeated Words, "accuracy is highest when the unique word is placed near the beginning of the sequence, especially as input length increases" — put load-bearing rules early in SKILL.md, not buried mid-document.
  - **Distractors compound**: "Even a single distractor reduces performance relative to the baseline; adding four distractors compounds this degradation further" — and individual distractors hurt non-uniformly. Every adjacent-but-not-applicable rule in a skill file acts as a distractor; prune aggressively.
  - **Format (counterintuitive)**: "Models perform better on shuffled haystacks than on logically structured ones… structural coherence consistently hurts model performance." Discrete bullets and tables likely beat prose narratives whose flow invites attention to chase irrelevant continuity.
  - **Routing / frontmatter**: "Lower needle-question similarity increases the rate of performance degradation" (similarity 0.445–0.829). Descriptions should lexically match user phrasing, not paraphrase it — reinforces the "name the function, not the topic" rule.
  - **Model binding**: Claude Opus 4 and Sonnet 4 "particularly conservative under ambiguity" — Opus 4 refused 2.89% of Repeated Words tasks; GPT-4.1 2.55%; GPT-3.5 Turbo excluded entirely (60.29% content-filter refusals). Skills targeting Claude must minimize ambiguity; cross-family portability is unsafe to assume.
  - **Eval methodology**: Judge calibrated to ">0.99 alignment to human judgment" over ~500–600 manually labeled outputs per experiment, across 8 input lengths × 11 needle positions — reliability reported as a number with repetition, not single-pass.

## 2026

- 2026-01 · [2601.20404v2](https://arxiv.org/html/2601.20404v2) — Lulla et al. — On the Impact of AGENTS.md Files on the Efficiency of AI Coding Agents
  - **Token economy**: Qualifying AGENTS.md reduced mean output tokens 20.08% (5,744.81 → 4,591.46), median 16.58% (2,925 → 2,440); input tokens −9.73%; wall-clock 20.27% mean / 28.64% median (162.94s → 129.91s).
  - **Shipping criteria**: "Qualifying" AGENTS.md required three categories present — (i) conventions and best practices, (ii) architecture and project structure, (iii) project description. Inclusion criterion, not a validated recipe.
  - **Eval methodology**: Paired within-task design — 10 repos / 124 PRs, OpenAI Codex (gpt-5.2-codex), pre-merge state, ≤100 LOC / ≤5 files / code-only. Usable as A/B template for context files.
  - **Model binding caveat**: Single framework (Codex + gpt-5.2); generalization to Claude explicitly unverified.
- 2026-02 · [2602.11988v1](https://arxiv.org/html/2602.11988v1) — Gloaguen et al. — Evaluating AGENTS.md: Are Repository-Level Context Files Helpful for Coding Agents?
  - **Authoring process / shipping criteria**: "Human-written context files should describe only minimal requirements"; "omit LLM-generated context files for the time being, contrary to agent developers' recommendations" (§1).
  - **Cost**: LLM-generated files reduced success 0.5% on SWE-bench Lite and 2% on AGENTbench while adding 2.45 and 3.92 steps (+20%/+23% cost); developer-written files +4% success on AGENTbench but +19% cost, +3.34 steps (§4.2). Reasoning tokens rose 22% (GPT-5.2) / 14% (GPT-5.1 mini) with LLM-generated files.
  - **Context architecture — repo overviews fail**: Overviews in 8/12 developer files and ~95–100% of LLM-generated files — yet "context files, even developer-provided ones, are not effective at providing a repository overview" (§4.2). Paper endorses Claude Code's stance that prompts "warn against listing components that are easily discoverable."
  - **Redundancy with docs**: When README, `docs/`, examples deleted, LLM-generated files flip to +2.7% gain (§4.2) — duplicating existing docs in SKILL.md/CLAUDE.md is net-negative.
  - **Instruction following is strong — prune carefully**: Tools mentioned used 1.6×/instance vs <0.01× unmentioned; repo-specific tools 2.5× (§4.3) — "the absence of improvements with context files is not due to a lack of instruction-following." Every useless instruction still gets obeyed.
  - **What to keep**: Minimal, repo-specific tooling (example: `uv`); style/overview/testing boilerplate causes more grep/test/write activity without more solves.
  - **Routing**: Codex vs Claude Code generation prompts show no consistent winner — "sensitivity to different (good) prompts is generally small" (§4.4).
- 2026-02 · [2602.12430v3](https://arxiv.org/html/2602.12430v3) — Xu & Yan — Agent Skills for LLMs: Architecture, Acquisition, Security, and the Path Forward
  - Survey paper — two durable numbers, plus naming several authoring questions as open problems.
  - **Context architecture**: Three-level progressive disclosure is "the defining architectural innovation" — Level 1 metadata (dozens of tokens, always-loaded), Level 2 instructions (on trigger), Level 3 assets (on-demand). Goal: "minimize context window consumption while maintaining access to arbitrarily deep procedural knowledge" (§3.1, Fig 1).
  - **Token economy**: Tool Search reduces overhead "up to 85%"; Programmatic Tool Calling lifts accuracy 79.5% → 88.1% (§3.4); SAGE achieves "26% fewer interaction steps and 59% fewer generated tokens" (§4.2).
  - **Security**: "26.1% of 42,447 community skills contain vulnerabilities"; skills bundling executable scripts "2.12× more likely to contain vulnerabilities" (§6.2). Proposes four-tier trust model T1–T4 with permission gates G1–G4.
  - **Routing/frontmatter**: SKILL.md uses YAML frontmatter with name/description; routing fires on description match — no concrete guidance on description length or keywords.
  - **Eval methodology (open problem)**: "Current benchmarks evaluate agents on task completion but rarely assess skill quality directly" (§7).
  - **Skill interaction (open)**: "Principled frameworks for multi-skill orchestration — conflict resolution, resource sharing, failure recovery — remain underdeveloped" (§7).
- 2026-02 · [2602.12670v3](https://arxiv.org/html/2602.12670v3) — Li et al. — SkillsBench: Benchmarking How Well Agent Skills Work Across Diverse Tasks
  - **Shape of instructions (§2.4)**: Skills must "provide procedural guidance (how to approach), not declarative answers (what to output)" and "be applied to a class of tasks, not a single instance." Explicit forbid list: task-specific filenames/paths/IDs, exact command sequences, constants, magic numbers, references to test cases or expected outputs.
  - **Authoring process / eval timing**: "A Claude Code Agent SDK-based validation agent runs in CI to detect potential Skill-solution leakage; failed tasks are rejected" (§2.4). Skills "authored independently of benchmark specifications" — fresh-session principle.
  - **Self-authoring fails (Finding 3, §4.1.2)**: Self-generated Skills yield −1.3pp avg; only Opus 4.6 marginal (+1.4pp); Codex+GPT-5.2 −5.6pp. "Models cannot reliably author the procedural knowledge they benefit from consuming."
  - **Token economy (Table 6, §4.2.2)**: Detailed +18.8pp and Compact +17.1pp beat Standard +10.1pp; Comprehensive skills hurt by −2.9pp. "Overly elaborate Skills can consume context budget without providing actionable guidance." Ecosystem median SKILL.md ≈1.5k tokens; median size 2.3 KB (IQR 0.8–6.1 KB).
  - **Narrow vs broad / chains (Finding 5, Table 5)**: 1 skill +17.8pp; 2–3 skills +18.6pp optimal; 4+ skills only +5.9pp. "Excessive Skills content creates cognitive overhead or conflicting guidance."
  - **Harness binding (§5)**: Same skill and model span +13.6 to +23.3pp across harnesses. Skills "should explicitly match harness constraints (e.g., repeated format reminders for JSON-only protocols)."
  - **ROI (§4, Table 10)**: Overall +16.2pp avg at $0.03–0.22/trial; Haiku+Skills (27.7%) > Opus-without (22.0%). 16/84 tasks have negative delta — skills can harm when model has strong priors (worst: taxonomy-tree-merge −39.3pp).
- 2026-02 · [2602.14690v2](https://arxiv.org/html/2602.14690v2) — Galster et al. — Configuring Agentic AI Coding Tools: An Exploratory Study
  - Descriptive snapshot of adoption patterns (Feb 2026); no phrasing-effectiveness analysis.
  - **Context architecture (tool choice)**: Of 2,631 repos with context files — CLAUDE.md 34.2% (1,658), AGENTS.md 32.4% (1,571), copilot-instructions.md 28.7% (1,393). CLAUDE.md created first; AGENTS.md added later. 69.0% of repos use a single tool; 18.1% use only AGENTS.md. Paper recommends AGENTS.md as "lowest-friction entry point" shared baseline.
  - **Reference structure**: 518 reference pairs between context files; CLAUDE.md had 357 outgoing references (311 to AGENTS.md); AGENTS.md received 368 incoming — hierarchical linking over duplication.
  - **Token economy (line budget)**: Of 601 Skills, only 5% exceeded the 500-line specification limit (§6.2) — the 500-line budget is empirically followed.
  - **Context architecture (bundling)**: 85.5% of Skills included no additional resources; scripts/ 5.8%, references/ 5.8%, assets/ 0.7% — Skills "function primarily as structured text rather than executable workflow bundles."
  - **Subagents**: 450 Subagents in 131 repos, median 2/repo; "no repositories storing persistent memory files" (§6.3).
- 2026-02 · [2602.20478v1](https://arxiv.org/html/2602.20478v1) — Vasilopoulos — Codified Context: Infrastructure for AI Agents in a Complex Codebase
  - **Context architecture (three tiers by access frequency, §3)**: Tier 1 constitution ~660 lines always-loaded; Tier 2 is 19 domain agents totaling ~9,300 lines loaded on dispatch; Tier 3 is 34 knowledge-base docs ~16,250 lines pulled on-demand.
  - **Shape of instructions**: "Over half of each specification's content is project-domain knowledge rather than behavioral instructions" (§3.2). Networking agent = 915 lines, ~65% domain facts — counters brevity bias.
  - **Format**: Mixed per purpose — "Correctness Pillars" as requirement tables; "Known Failure Modes" as symptom-cause-fix triples (§4.4.4). Not prose-first.
  - **Routing**: Constitution "requires the orchestrator to consult the trigger table before changes" (§3.1.1). Pre-change triggers route to designer agents; post-change to reviewer agents.
  - **Maintenance**: ~5 min/session when spec touched + biweekly 30–45 min review = 1–2 hr/week (§5.2). "Out-of-date specs can mislead sessions and lead to silent failures"; drift detector diffs Git commits against spec mappings.
  - **Eval methodology**: Observational — 283 sessions over 70 days; 2,801 human prompts, 1,197 agent invocations, 16,522 agent turns (Table 3). 87% ad-hoc vs 13% structured plan-execute-review (Fig 3); 57% of classifiable invocations hit project-specific specialists.
  - **Token economy**: 80%+ of human prompts ≤100 words (§4.3) — pre-loaded context substitutes for in-prompt explanation. Heuristic G4: "If you explained it twice, write it down"; G6: "Stale specs mislead efforts."
- 2026-03 · [2603.04814v1](https://arxiv.org/html/2603.04814v1) — Pollertlam & Kornsuwannawit — Beyond the Context Window: Fact-Based Memory vs. Long-Context LLMs for Persistent Agents
  - Chatbot-personalization study (Mem0 vs long-context); transfers by analogy to hot-vs-retrieval skill-tier decisions.
  - **Context architecture**: Long-context retention beats fact-extracted memory on accuracy — LoCoMo 92.85% vs 57.68% (−35.2pp); LongMemEval 82.40% vs 49.00% (−33.4pp) (§4.1, Table 3). Favor always-loaded over retrieval when accuracy matters.
  - **Token economy (caching)**: With 90% prompt caching, long-context per-turn cost "grows linearly with context length, whereas the memory system's per-turn read cost remains roughly fixed" (§5.1). Break-even at ~10 turns for 100k-token context.
  - **Context architecture (verbatim content)**: Flat fact extraction achieves 35:1 compression (101,601 → 2,909 tokens) but "irretrievably loses" precise temporal markers, implicit coreferences, ephemeral updates (§5.1, Table 6) — keep version strings, dates, cross-references verbatim in SKILL.md.
  - **Cost (hot vs cold tier)**: Memory read ~$0.0013/query (fixed) vs cached long-context turn ~$0.0036 at 100k. Memory system wins only after ~10 reuses; monolithic always-loaded wins for short sessions.
  - **Boundaries**: "Memory systems excel only when information suits stable, factual attributes suited to flat-typed extraction" (Abstract) — goal-based / multi-hop content belongs in always-loaded context, not retrieval.
- 2026-03 · [2603.06358v1](https://arxiv.org/html/2603.06358v1) — Liu et al. — A Scalable Benchmark for Repository-Oriented Long-Horizon Conversational Context Management
  - Benchmark (LoCoEval) for runtime conversational context management in repo tasks, not skill authoring — two transferable signals:
  - **Context architecture (composite memory)**: Memory pairing "textual description and path locations of associated repository artifacts" outperforms pure-text — Mem0R 62.22% normalized Pass@1 on multi-hop vs Vanilla RAG 52.22% (§5.3). Analog for SKILL.md's `see Class#method` pattern: pair prose with file paths, not prose alone.
  - **Token economy**: Conversations average "2.5 requirements and 50 conversational turns, resulting in total context lengths reaching up to 64K–256K tokens", ~3M total tokens per conversation (~$1 at $0.28/1M) — unmanaged long context is costly.
- 2026-03 · [2603.09619v2](https://arxiv.org/html/2603.09619v2) — Vishnyakova — Context Engineering: From Prompts to Corporate Multi-Agent Architecture
  - Corporate-architecture essay — no concrete rules for MUST/SHOULD/CAPS/line-budgets/frontmatter. Five concept-level contributions:
  - **Context architecture (JIT logistics)**: CE defined as "what to include, when to supply it, in what form, for how long, and for which sub-agent" (§6). CLAUDE.md, MEMORY.md named as "explicit declarations of goals and constraints that the agent will see in every session regardless of what has accumulated in episodic memory" — always-loaded tier.
  - **Token economy**: Cites Manus (2025) — cached vs uncached tokens ~10× cost difference; compression + caching + selective loading yield "5–10× cost reduction while preserving decision quality" (§10).
  - **Context quality (five criteria, §9)**: Relevance, Sufficiency, Isolation, Economy, Provenance. Provenance: "every element of context must be traceable to its source: which system, when, with what trust level."
  - **Skill interaction (isolation)**: "Privilege attenuation" — sub-skill "cannot transfer its full set of rights… only a strictly limited slice necessary for the subtask" (§9). StrongDM incident: agents exploited visible test scenarios for reward hacking.
  - **Context rot modes (§9)**: Poisoning (hallucination reproduced each step), distraction (Gemini 2.5 at >100k tokens), confusion (irrelevant info degrades), clash (Microsoft/Salesforce: "39% quality drop when a single prompt was split into sequential turns").
- 2026-03 · [2603.22455v4](https://arxiv.org/html/2603.22455v4) — Zheng et al. — SkillRouter: Skill Routing for LLM Agents at Scale
  - **Routing/frontmatter**: Hiding body / routing on name+description alone (progressive-disclosure default) causes **31–44 pp Hit@1 drop** vs full-text routing — BM25 31.4% → 0.0%; Qwen3-Emb-8B 64.0% → 25.3%; Qwen3-Emb-8B × Rank-8B 68.0% → 24.0% (§3, Fig 1). "Full skill text is a critical routing signal" (Abstract).
  - **Context architecture (asymmetry)**: Router inspects full body; agent sees only name+description post-selection (§2). Body access matters at retrieval stage; runtime token budget preserved.
  - **Token economy (field breakdown)**: Tokens split 96.5% body / 3.0% name / 0.5% description; median body 704 words vs description 21 words; 59% of descriptions <25 words, 18.7% <10 words (Appendix A, Table 7). Even top-quartile descriptions (>35w) trail full-text by +31.8pp (Appendix D, Table 14) — descriptions cannot substitute body signal regardless of length.
  - **Routing (case studies, Appendix L.1)**: "Software-dependency-audit" — generic `dependency-security` wins over specific `trivy-offline-vulnerability-scanning` on metadata alone; 0/12 → 12/12 agent success after correct routing. "Video-tutorial-indexer" picks `video-explorer` surface keyword over function `speech-to-text`; 0/12 → 9/12. **Name the function, not the surface topic.**
  - **Maintenance (near-duplicates)**: Near-duplicate skills degrade the router; false-negative filtering + listwise reranking loss needed (§4).
  - **Boundaries**: Claims apply to large registries with heavy overlap; metadata-only routing more competitive in smaller catalogs (§7). Downstream gains larger for capable agents — Claude Sonnet/Opus 4.6 +3.22pp task success vs +0.89pp for glm-5/Kimi-K2.5 (§5.5).
- 2026-04 · [2604.02837v1](https://arxiv.org/html/2604.02837v1) — Li, Wu, Ling, Cui, Luo — Towards Secure Agent Skills: Architecture, Threat Taxonomy, and Security Analysis
  - Security threat taxonomy (7 categories / 17 scenarios / 3 attack layers), validated against 5 real incidents. Relevant to Security, Routing-as-attack-surface, and Versioning; nothing on authoring craft.
  - **Security (audit, signing, permissions)**: Proposes mandatory pre-publication review, cryptographic verified-publisher signatures with namespace-ownership binding, per-capability grants "analogous to Android", capability-tiered sandboxing (§7.1). Empirical scope: 42,447 skills scanned; 26.1% contained prompt-injection vulnerabilities; ClawHavoc campaign compromised 1,184 skills (~1 in 5 in OpenClaw).
  - **Routing/frontmatter (attack surface)**: SKILL.md YAML frontmatter + body have "no mechanism to verify that the instructions body is consistent with—or confined to—that declaration" (§3.1) — description is unreliable as a contract.
  - **Versioning**: Trust is bound to skill identity, not a "cryptographically committed version" — post-install content changes bypass consent. Recommends delta-based re-consent when "content diff exceeds configurable behavioral threshold"; changelogs surfaced "before next activation" (§3.3).
  - **Open problem**: §7.1 concedes direct prompt injection "remains an open problem that cannot be fully addressed within the current architectural framework."
- 2026-04 · [2604.08290v1](https://arxiv.org/html/2604.08290v1) — Farajijobehdar, Köseoğlu Sarı, Üre, Zeydan — Tokalator: A Context Engineering Toolkit for AI Coding Assistants
  - Measurement/visualization toolkit, not authoring methodology.
  - **Token economy**: Instruction files flagged as "invisible budget consumer" — example `.github/copilot-instructions.md` at 4,200 tokens/prompt; average ~500 tokens/instruction file; system-prompt overhead ~2,000 tokens.
  - **Context architecture**: Five-category model (Eq. 2) — open files, system prompt, instruction files, conversation history, output reservation (4,000 tokens reserved). History grows O(T²) under full-history strategy; "context rot" risk after 20+ turns.
  - **Cost**: Prompt-caching break-even at n*=2 reuses across current Anthropic models — skill hot-tier content justifies caching quickly.
  - **Observability**: n=50 user study — 92% agreement (46/50) that flagged distractor tabs should be removed, zero false positives, 21.2% context reduction; 82% cited `/preview` command as most valuable — visibility into loaded context is a primary authoring-feedback mechanism.
  - **Routing heuristic**: Tab relevance scoring weights imports 30%, language match 25%, path similarity 20%, edit recency 15%, diagnostics 10% — explicit dependencies ranked most trustworthy.
- 2026-04 · [2604.11462v1](https://arxiv.org/html/2604.11462v1) — Li et al. — Escaping the Context Bottleneck: Active Context Curation for LLM Agents via Reinforcement Learning
  - RL-trained ContextCurator model for runtime memory, not human-authored skills — two transferable signals:
  - **Context architecture**: Raw DOM contains ">90% structural noise"; DeepSearch achieved 5–8× token reduction (34.4K → 7.3K, 46.7K → 6.6K) while improving success +8–24% relative (§1) — "the problem is not capacity, but signal-to-noise ratio." Supports SKILL.md's pruning/compression guidance.
  - **Retrieval critique**: Similarity retrieval has "retrieval bias; they frequently fail to retrieve implicit reasoning anchors—causally essential information that may be textually dissimilar to the current query" (§2) — argues for always-loaded context over passive similarity retrieval, reinforcing Pollertlam 2603.04814.
- 2026-04 · [2604.14228v1](https://arxiv.org/html/2604.14228v1) — Liu, Zhao, Shang, Shen — Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems
  - Descriptive architecture survey from TypeScript source reverse-engineering.
  - **Context architecture**: Skills are the "Low (descriptions only)" tier — "only frontmatter descriptions (not full content) stay in the prompt"; full skill body injected lazily by `SkillTool` meta-tool "when invoked" (§6.3, Table 2; §6.1).
  - **Cost ordering**: "Zero for hooks, low for skills, medium for plugins, high for MCP" (§6.3). MCP pays "High (tool schemas)" — shifting capability from MCP to a skill is a context-budget win.
  - **Routing/frontmatter**: `parseSkillFrontmatterFields()` in `loadSkillsDir.ts` parses 15+ fields — display name, description, allowed tools, argument hints, **model overrides, execution context (`'fork'` for isolated execution), associated agent definitions, effort levels, shell configuration** (§6.1). Description is the load-bearing field for invocation.
  - **Skill interaction**: "Skills can define their own hooks, which register dynamically on invocation" (§6.1) — skill hooks are lifetime-scoped, not global, avoiding cross-skill leakage.
  - **Boundaries (skill vs subagent vs hook)**: Subagents "create new, isolated context windows rather than extending the current one" (§6.2). Skills "shape *how* the agent thinks"; MCP adds tools; hooks give "cross-cutting lifecycle control (blocking, rewriting, or annotating)" (§6.3). Pick skill for in-context instructions; subagent for a fresh window; hook for automation.
  - **Model binding**: Per-skill `model` and `effort` fields let a skill pin its own model/effort independent of session defaults (§6.1).
