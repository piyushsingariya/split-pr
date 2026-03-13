# split-pr skill for Claude Code

A Claude Code skill that splits a large PR or set of uncommitted changes into multiple small, independent, build-verified PRs — each based directly on `main` and mergeable in any order.

## Installation

Skills in Claude Code must live in a **named directory** inside `~/.claude/skills/`. The directory name becomes the `/command` name, and `SKILL.md` inside it is the entrypoint. The `references/` folder must also be present because `SKILL.md` links to those files.

**Option A — Copy the whole directory:**

```bash
cp -r /path/to/this/repo ~/.claude/skills/split-pr
```

**Option B — Clone directly into the skills directory:**

```bash
git clone https://github.com/piyushsingariya/split-pr ~/.claude/skills/split-pr
```

**Option C — Symlink (keeps it in sync with your local clone):**

```bash
ln -s "$(pwd)" ~/.claude/skills/split-pr
```

After any of these, your skills directory should look like:

```
~/.claude/skills/
└── split-pr/
    ├── SKILL.md
    └── references/
        ├── build-verification.md
        ├── dependency-resolution.md
        └── splitting-strategies.md
```

Claude Code picks up skills automatically — no restart needed.

**Updating the skill** (if installed via clone or copy):

```bash
cd ~/.claude/skills/split-pr && git pull
```

If installed via symlink, just pull in your local clone — the symlink keeps it live automatically.

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
