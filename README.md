# Rooted — Claude Code plugins rooted in evidence

[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/rootbr/rooted?style=social)](https://github.com/rootbr/rooted/stargazers)
[![Rules cited from](https://img.shields.io/badge/rules%20cited%20from-28%20papers%20%C2%B7%202%20RFCs-success)](#references)

Context engineering and agent-management skills for Claude Code — give your agents grounded instructions, verified sources, and disciplined reasoning instead of vibes. Built on top: software-development and code-review skills that boost productivity through context management and critical analysis. Every rule in this marketplace traces to a peer-reviewed paper or an RFC — never to "ask the model to write a skill." [Research shows](https://arxiv.org/abs/2602.11988) LLM-generated context files *decrease* agent task success and raise inference cost by ~20%. Rooted structurally refuses them.

- **Evidence-based, not LLM-generated.** Every rule traces to a peer-reviewed paper, RFC/spec, or a hands-on test — not to "ask the model to write a skill."
- **Lean context over static dumps.** For example, instead of an `/init`-style project-structure blob in `CLAUDE.md` (shown to *decrease* agent success and raise cost by ~20% [[15]](#references)), orientation happens on demand via [`overview.sh`](overview.sh) with an explicit stop-instruction. The agent doesn't preemptively slurp the whole tree into context, so it doesn't fill its working memory with project content it doesn't need and later lose focus on the actual task.

## Methodology

Every rule, checklist item, and review criterion traces to one of:

- **Academic research** — arxiv.org, peer-reviewed papers, Google Scholar, PubMed
- **Technical standards** — RFCs, official specifications, language/framework documentation
- **Human-tested** — double-checked and reviewed by a human before commit

## Plugins

### code-quality

| Skill | Description |
|--|--|
| `/reviewing-java` | Deep Java code review with parallel specialist agents and verification. [Details](plugins/code-quality/skills/reviewing-java/README.md) |
| `/cleaning-code` | Improve code readability and maintainability |
| `/committing-changes` | Review changes and create atomic commits following Conventional Commits |

### evidence-based-authoring

| Skill | Description |
|--|--|
| `/researching-topics` | Research any topic on the web with source verification and critical analysis |
| `/auditing-ai-context` | Audit and optimize any AI agent context: CLAUDE.md, SKILL.md, prompts, instructions. Adapt project docs for AI consumption |

## Installation

Add the marketplace:

```
/plugin marketplace add rootbr/rooted
```

Install plugins:

```
/plugin install code-quality@rooted
/plugin install evidence-based-authoring@rooted
```

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). Every new rule must cite a source — an arXiv ID, an RFC number, or a linked hands-on test. PRs without citations are rejected by design, not by oversight.

## References

For the full cross-check of each paper against the research questions this marketplace tries to answer, see the [research audit](research/2026-04-18_context-engineering-research.md) — findings extracted per paper, including tangential results that inform but do not yet back a specific skill rule.

### Technical standards

- Bradner (1997). *Key words for use in RFCs to Indicate Requirement Levels.* [BCP 14 / RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) — MUST / SHOULD / MAY priority keywords used in [`/auditing-ai-context`](plugins/evidence-based-authoring/skills/auditing-ai-context/SKILL.md).
- Leiba (2017). *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words.* [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) — Clarifies that BCP 14 keywords only apply in ALL CAPITALS.

### Research papers

1. Liang et al. (2024). *Can Language Models Replace Programmers for Coding? RepoCod Says 'Not Yet'.* [arXiv:2410.21647](https://arxiv.org/abs/2410.21647) — Repo-level benchmarks: no tested LLM exceeds 30% pass@1; RAG retrieval beats full-file context.
2. Yang et al. (2025). *What Prompts Don't Say: Understanding and Managing Underspecification in LLM Prompts.* [arXiv:2505.13360](https://arxiv.org/abs/2505.13360) — Rule bundling tax: 37.5% of rules lose >5% compliance when combined; per-rule validators hit 95.6% human agreement.
3. Nair et al. (2025). *Tournament of Prompts: Evolving LLM Instructions Through Structured Debates and Elo Ratings.* [arXiv:2506.00178](https://arxiv.org/abs/2506.00178) — Multi-round adversarial LLM debate (d=3, two advocates + judge) beats single-pass LLM-as-judge 80.3–95% of the time — evidence for adversarial eval over single-judge scoring.
4. Murthy et al. (2025). *Promptomatix: An Automatic Prompt Optimization Framework for Large Language Models.* [arXiv:2507.14241](https://arxiv.org/abs/2507.14241) — Positive-over-negative framing; 2–5 diverse examples, best-fit placed last.
5. Lindenbauer et al. (2025). *The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization for Agent Context Management.* [arXiv:2508.21433](https://arxiv.org/abs/2508.21433) — Tool outputs consume ~84% of agent tokens; masking beats summarization.
6. Chatlatanagulchai et al. (2025). *On the Use of Agentic Coding Manifests: An Empirical Study of Claude Code.* [arXiv:2509.14744](https://arxiv.org/abs/2509.14744) — Empirical analysis of 253 `CLAUDE.md` files; shallow 2–3 level hierarchies dominate.
7. Zhang et al. (ICLR 2026). *Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models.* [arXiv:2510.04618](https://arxiv.org/abs/2510.04618) — ACE: context-collapse evidence against aggressive compression; bullets with metadata beat prose.
8. Gao & Peng (2025). *More with Less: An Empirical Study of Turn-Control Strategies for Efficient Coding Agents.* [arXiv:2510.16786](https://arxiv.org/abs/2510.16786) — Concrete cost/accuracy across Claude / Gemini / GPT; eval repetition (10× per task) required to reach statistical significance.
9. Mohsenimofidi et al. (2025). *Context Engineering for AI Agents in Open-Source Software.* [arXiv:2510.21413](https://arxiv.org/abs/2510.21413) — `AGENTS.md` across 466 OSS projects; five instruction styles observed in the wild.
10. Khan (2025). *You Don't Need Prompt Engineering Anymore: The Prompting Inversion.* [arXiv:2510.22251](https://arxiv.org/abs/2510.22251) — Constrained prompts help <90%-accuracy models but degrade >95%-accuracy frontier models.
11. Santos et al. (2025). *Decoding the Configuration of AI Coding Agents: Insights from Claude Code Projects.* [arXiv:2511.09268](https://arxiv.org/abs/2511.09268) — Empirical study of 328 `CLAUDE.md` files; Architecture dominates top-5 patterns.
12. Bulle Labate et al. (2025). *Solving Context Window Overflow in AI Agents via Memory Pointers.* [arXiv:2511.22729](https://arxiv.org/abs/2511.22729) — Replacing tool outputs with pointer IDs achieved 16,900× and 7× token reductions; supports handle-returning tools over inline blobs.
13. Dong et al. (ACL 2026). *Revisiting the Reliability of Language Models in Instruction-Following.* [arXiv:2512.14754](https://arxiv.org/abs/2512.14754) — IFEval++ and `reliable@k` metric; up to 61.8% drop under paraphrased instructions.
14. Lulla, Mohsenimofidi, Galster, Zhang, Baltes, Treude (2026). *On the Impact of AGENTS.md Files on the Efficiency of AI Coding Agents.* [arXiv:2601.20404](https://arxiv.org/abs/2601.20404) — 124 PRs × 10 repos; qualifying AGENTS.md yields −28.64% median runtime, −16.58% output tokens.
15. Gloaguen et al. (2026). *Evaluating AGENTS.md: Are Repository-Level Context Files Helpful for Coding Agents?* [arXiv:2602.11988](https://arxiv.org/abs/2602.11988) — **Primary citation.** LLM-generated context files *decrease* task success and raise inference cost by ~20%; even developer-written files help only marginally.
16. Xu & Yan (2026). *Agent Skills for Large Language Models: Architecture, Acquisition, Security, and the Path Forward.* [arXiv:2602.12430](https://arxiv.org/abs/2602.12430) — Three-level progressive disclosure; 26.1% of 42,447 community skills contain vulnerabilities.
17. Li et al. (2026). *SkillsBench: Benchmarking How Well Agent Skills Work Across Diverse Tasks.* [arXiv:2602.12670](https://arxiv.org/abs/2602.12670) — 2–3 skills optimal (+18.6pp); self-generated skills regress −1.3pp avg; Comprehensive skills hurt −2.9pp.
18. Galster et al. (2026). *Configuring Agentic AI Coding Tools: An Exploratory Study.* [arXiv:2602.14690](https://arxiv.org/abs/2602.14690) — 2,631 repos; 95% of Skills within 500-line budget; reference-over-duplication pattern.
19. Vasilopoulos (2026). *Codified Context: Infrastructure for AI Agents in a Complex Codebase.* [arXiv:2602.20478](https://arxiv.org/abs/2602.20478) — Three-tier context system (~660/~9,300/~16,250 lines); drift detector against Git commits; "explained twice → write it down".
20. Pollertlam & Kornsuwannawit (2026). *Beyond the Context Window: Fact-Based Memory vs. Long-Context LLMs for Persistent Agents.* [arXiv:2603.04814](https://arxiv.org/abs/2603.04814) — Long-context beats fact-extracted memory by 33–35pp; retrieval wins only after ~10 reuses.
21. Liu et al. (2026). *A Scalable Benchmark for Repository-Oriented Long-Horizon Conversational Context Management.* [arXiv:2603.06358](https://arxiv.org/abs/2603.06358) — LoCoEval benchmark; composite text+path memory (Mem0R 62.22%) beats pure-text (Vanilla RAG 52.22%) on multi-hop repo tasks.
22. Vishnyakova (2026). *Context Engineering: From Prompts to Corporate Multi-Agent Architecture.* [arXiv:2603.09619](https://arxiv.org/abs/2603.09619) — Four context-rot modes (poisoning, distraction, confusion, clash); provenance and privilege-attenuation criteria.
23. Zheng et al. (2026). *SkillRouter: Skill Routing for LLM Agents at Scale.* [arXiv:2603.22455](https://arxiv.org/abs/2603.22455) — Hiding skill body from routing at 80K-skill scale costs 31–44 pp accuracy; name the function, not the surface topic.
24. Li, Wu, Ling, Cui, Luo (2026). *Towards Secure Agent Skills: Architecture, Threat Taxonomy, and Security Analysis.* [arXiv:2604.02837](https://arxiv.org/abs/2604.02837) — 7 threat categories × 17 scenarios; ClawHavoc compromised 1,184 skills; YAML frontmatter is not a contract.
25. Farajijobehdar, Köseoğlu Sarı, Üre, Zeydan (2026). *Tokalator: A Context Engineering Toolkit for AI Coding Assistants.* [arXiv:2604.08290](https://arxiv.org/abs/2604.08290) — Instruction-file token accounting; prompt-caching break-even at n*=2 reuses; O(T²) history growth.
26. Li et al. (2026). *Escaping the Context Bottleneck: Active Context Curation for LLM Agents via Reinforcement Learning.* [arXiv:2604.11462](https://arxiv.org/abs/2604.11462) — DOM contains >90% structural noise; similarity-retrieval critique — "fails to retrieve implicit reasoning anchors" — reinforces always-loaded over retrieval for causally essential context.
27. Liu, Zhao, Shang, Shen (2026). *Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems.* [arXiv:2604.14228](https://arxiv.org/abs/2604.14228) — Peer-academic analysis of Claude Code internals; cost ordering `hooks < skills < plugins < MCP`; `parseSkillFrontmatterFields` parses 15+ fields including model/effort overrides.
28. Hong, Troynikov, Huber (2025). *Context Rot: How Increasing Input Tokens Impacts LLM Performance.* [trychroma.com/research/context-rot](https://www.trychroma.com/research/context-rot) — 18-model study; non-uniform degradation with input length; single distractor reduces accuracy.

## License

Apache 2.0
