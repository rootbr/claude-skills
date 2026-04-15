# Claude Skills — evidence-backed AI agent skills

## How this differs from other skill marketplaces

- **Evidence-based, not LLM-generated.** Every rule traces to a peer-reviewed paper, RFC/spec, or a hands-on test — not to "ask the model to write a skill."
- **Lean context over static dumps.** For example, instead of an `/init`-style project-structure blob in `CLAUDE.md` (shown to *decrease* agent success and raise cost by ~20% [[11]](#references)), orientation happens on demand via [`overview.sh`](overview.sh) with an explicit stop-instruction. The agent doesn't preemptively slurp the whole tree into context, so it doesn't fill its working memory with project content it doesn't need and later lose focus on the actual task.

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
| `/researching-topics` | Web research with source verification and critical analysis |
| `/auditing-ai-context` | Audit and optimize any AI agent context: CLAUDE.md, SKILL.md, prompts, instructions. Adapt project docs for AI consumption |

## Installation

Add the marketplace:

```
/plugin marketplace add rootbr/claude-skills
```

Install plugins:

```
/plugin install code-quality@claude-skills
/plugin install evidence-based-authoring@claude-skills
```

## References

Research papers and technical standards informing the rules, checklists, and design choices in this repository.

### Technical standards

- Bradner (1997). *Key words for use in RFCs to Indicate Requirement Levels.* [BCP 14 / RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) — MUST / SHOULD / MAY priority keywords used in [`/auditing-ai-context`](plugins/evidence-based-authoring/skills/auditing-ai-context/SKILL.md).
- Leiba (2017). *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words.* [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) — Clarifies that BCP 14 keywords only apply in ALL CAPITALS.

### Research papers

1. Liang et al. (2024). *Can Language Models Replace Programmers for Coding? RepoCod Says 'Not Yet'.* [arXiv:2410.21647](https://arxiv.org/abs/2410.21647) — Repo-level benchmarks: no tested LLM exceeds 30% pass@1.
2. Luo et al. (2025). *Large Language Model Agent: A Survey on Methodology, Applications and Challenges.* [arXiv:2503.21460](https://arxiv.org/abs/2503.21460) — Taxonomy of agent architectures.
3. Nair et al. (2025). *Tournament of Prompts: Evolving LLM Instructions Through Structured Debates and Elo Ratings.* [arXiv:2506.00178](https://arxiv.org/abs/2506.00178) — Debate- and Elo-driven prompt evolution beats manual prompt engineering.
4. Murthy et al. (2025). *Promptomatix: An Automatic Prompt Optimization Framework for Large Language Models.* [arXiv:2507.14241](https://arxiv.org/abs/2507.14241) — Auto-transforms task descriptions into optimized prompts.
5. Dong et al. (2025). *A Survey on Code Generation with LLM-based Agents.* [arXiv:2508.00083](https://arxiv.org/abs/2508.00083) — SDLC-aligned taxonomy of code-generation agents.
6. Chatlatanagulchai et al. (2025). *On the Use of Agentic Coding Manifests: An Empirical Study of Claude Code.* [arXiv:2509.14744](https://arxiv.org/abs/2509.14744) — Empirical analysis of 253 `CLAUDE.md` files.
7. Mohsenimofidi et al. (2025). *Context Engineering for AI Agents in Open-Source Software.* [arXiv:2510.21413](https://arxiv.org/abs/2510.21413) — `AGENTS.md` across 466 OSS projects; structure drives agent output quality.
8. Khan (2025). *You Don't Need Prompt Engineering Anymore: The Prompting Inversion.* [arXiv:2510.22251](https://arxiv.org/abs/2510.22251) — Constrained prompts help mid-tier models but hurt frontier models.
9. Santos et al. (2025). *Decoding the Configuration of AI Coding Agents: Insights from Claude Code Projects.* [arXiv:2511.09268](https://arxiv.org/abs/2511.09268) — Empirical study of 328 `CLAUDE.md` files.
10. Zehle et al. (2025). *promptolution: A Unified, Modular Framework for Prompt Optimization.* [arXiv:2512.02840](https://arxiv.org/abs/2512.02840) — Modular prompt-optimization framework.
11. Gloaguen et al. (2026). *Evaluating AGENTS.md: Are Repository-Level Context Files Helpful for Coding Agents?* [arXiv:2602.11988](https://arxiv.org/abs/2602.11988) — **Primary citation.** LLM-generated context files *decrease* task success and raise inference cost by ~20%; even developer-written files help only marginally.
12. Galster et al. (2026). *Configuring Agentic AI Coding Tools: An Exploratory Study.* [arXiv:2602.14690](https://arxiv.org/abs/2602.14690) — 2,923 repos; Context Files dominate, Skills and Subagents remain shallowly adopted.
13. Liu et al. (2026). *A Scalable Benchmark for Repository-Oriented Long-Horizon Conversational Context Management.* [arXiv:2603.06358](https://arxiv.org/abs/2603.06358) — Repo-oriented context benchmark up to 64K–256K tokens.

## License

Apache 2.0
