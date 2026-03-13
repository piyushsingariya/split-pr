# split-pr skill for Claude Code

A Claude Code skill that splits a large PR or set of uncommitted changes into multiple small, independent, build-verified PRs — each based directly on `main` and mergeable in any order.

## Installation

1. Copy `SKILL.md` into your Claude Code skills directory:

   ```bash
   cp SKILL.md ~/.claude/skills/split-pr.md
   ```

   Or symlink it if you want to keep it in sync with this repo:

   ```bash
   ln -s "$(pwd)/SKILL.md" ~/.claude/skills/split-pr.md
   ```

2. That's it. Claude Code picks up skills automatically from `~/.claude/skills/`.

## Usage

From inside any git repository, invoke the skill in Claude Code:

```
/split-pr
```

To split against a branch other than `main`:

```
/split-pr develop
```

Or reference an open GitHub PR:

```
/split-pr
> split PR #456
```

## What it does

1. **Analyzes** all changed files (committed + staged + unstaged) relative to the base branch and builds a dependency graph.
2. **Proposes** a grouping of files into independent PRs and waits for your approval.
3. **Creates branches** from the base, applies each file group, and **verifies the build passes** before pushing.
4. **Opens PRs** on GitHub with a summary, build status, and cross-links between sibling PRs.
5. **Returns** you to your original branch and prints a summary table.

## Requirements

- `git` and `gh` (GitHub CLI) installed and authenticated
- A GitHub remote configured for the repository

## References

See the [`references/`](references/) directory for detailed guidance on:
- [`splitting-strategies.md`](references/splitting-strategies.md) — which grouping strategy to use
- [`build-verification.md`](references/build-verification.md) — build command detection and failure diagnosis
- [`dependency-resolution.md`](references/dependency-resolution.md) — detecting and resolving cross-PR file dependencies
