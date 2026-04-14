---
description: Review changes and create atomic commits following Conventional Commits
---

# Commit Changes

Spec: https://www.conventionalcommits.org/v1.0.0/

Review staged and modified files. Split into atomic commits.

## Workflow

1. `git status` and `git diff --cached`
2. Group related changes into logical units
3. One commit per logical change

## Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

## Types

| Type | Purpose |
|--|--|
| feat | New feature |
| fix | Bug fix |
| docs | Documentation only |
| style | Formatting, whitespace |
| refactor | Neither fix nor feature |
| perf | Performance improvement |
| test | Tests |
| build | Build system, dependencies |
| ci | CI configuration |
| chore | Other |

Version bumps: `feat`→MINOR, `fix`→PATCH, `!` or `BREAKING CHANGE:` footer→MAJOR.

## Rules

- MUST: each commit atomic and functional (builds, tests pass)
- MUST: imperative mood, lowercase subject, no trailing period
- SHOULD: subject ≤50 chars, body wrap at 72
- Body explains WHAT and WHY, not HOW
- Footer references issues: `Fixes #123`, `Refs #456`

## Grouping

- Group: same bug, one feature, one module refactor
- Split: unrelated features, fix+feature mixed, different layers

## Examples

```
docs(readme): update installation instructions
```

```
fix(api): handle null user response

Add null check before accessing user.email to prevent NPE.

Fixes #456
```

```
feat(api)!: remove deprecated v1 endpoints

BREAKING CHANGE: All /api/v1/* endpoints removed.
Migrate to /api/v2/* which provides enhanced functionality.
```

## Gotchas

1. Pre-commit hook failure means the commit did NOT happen — fix and create a NEW commit, never `--amend`
2. AVOID `git add -A` / `git add .` — can leak `.env`, credentials, large binaries. Stage by name
3. Merge commits are exempt from format
4. Never `--no-verify` unless user explicitly asks
