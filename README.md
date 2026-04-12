# Claude-skills

## Skills

### `/research`

Multi-language web research with source verification and critical analysis. Classifies queries by type (consumer, scientific, technical, local), selects optimal search languages, applies source-type-specific quality filters, and cross-references findings before presenting.

### `/java-reviewer`

Deep Java code review that spawns specialist reviewers in parallel (performance, concurrency, security, reliability, maintainability, project-specific), then verifies every finding through a dedicated verification agent. Produces two reports: a clean review with only confirmed findings and a rejection audit trail. Zero config required — works on any Java git repo.

See [java-reviewer/README.md](plugins/java-reviewer/skills/java-reviewer/README.md) for project integration and config options.

### `/context-engineer`

Write, audit, and optimize CLAUDE.md and any file containing AI agent instructions. Covers context engineering principles — token economy, progressive disclosure, right-altitude heuristics.

### `/clean-code`

Proactive code cleanup: simplify, reduce nesting, remove redundancy, enforce clean code principles. Triggers automatically on code changes or on phrases like "clean this up", "simplify", "refactor".

## Installation

Add the marketplace:

```
/plugin marketplace add rootbr/claude-skills
```

Then install individual skills:

```
/plugin install research@rootbr/claude-skills
/plugin install java-reviewer@rootbr/claude-skills
/plugin install context-engineer@rootbr/claude-skills
/plugin install clean-code@rootbr/claude-skills
```

## License

Apache 2.0
