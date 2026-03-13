---
name: split-pr
description: Split a large PR or set of uncommitted changes into multiple small, independent PRs. Each resulting PR is self-contained, can be merged in any order, and is based directly on main. Triggers when the user wants to break up a large PR, decompose changes, or create independent smaller PRs from a big changeset.
argument-hint: [base-branch]
allowed-tools: Bash, Read, Glob, Grep, Edit, Write
---

# Split PR into Independent PRs

Split a large changeset into multiple small, independent PRs — each based on `main` (or `$ARGUMENTS` if provided), self-contained, mergeable in any order, and **verified to build successfully** before pushing.

## Guiding Principles (Google's Small CLs)

- ~100 lines per PR is reasonable; ~1000 lines is too large
- Each PR must be "one self-contained change" — the system must work after it merges
- Keep refactoring separate from features/bug fixes
- Tests travel with the logic they test

---

## Step 0: Understand Context

### 0a. Identify the changeset

The base branch is `$ARGUMENTS` if provided, otherwise `main`. Collect **all** changes relative to it — committed and uncommitted together:

```bash
BASE_BRANCH="${ARGUMENTS:-main}"
git fetch origin $BASE_BRANCH

# All changed files vs base (committed + staged + unstaged combined)
git diff --name-status origin/$BASE_BRANCH

# Full diff for reading content
git diff origin/$BASE_BRANCH

# Stats summary
git diff --stat origin/$BASE_BRANCH
```

> `git diff origin/$BASE_BRANCH` (without `...HEAD`) covers everything: commits ahead of base, staged changes, and unstaged working-tree changes — all in one view. Do not split this into separate committed/uncommitted passes.

If the user referenced a GitHub PR number/URL, use that as the source of truth instead:
```bash
gh pr view <PR_NUMBER> --json files,title,body,baseRefName
gh pr diff <PR_NUMBER>
```

### 0b. Detect build and test commands

Determine what "build success" means for this repo using two sources, in priority order:

**1. Makefile (preferred)** — Read it directly and extract all targets:
```bash
cat Makefile 2>/dev/null
```
Look for targets named `build`, `test`, `lint`, `check`, `vet`, `fmt`, `all`, or any composite target that CI likely calls. Prefer `make <target>` over raw commands — the Makefile already encodes the correct flags, env vars, and ordering.

**2. Language manifest (fallback if no Makefile or Makefile has no relevant targets)**

| Language detected by       | Default BUILD_CMD          | Default TEST_CMD        | Default LINT_CMD           |
|----------------------------|---------------------------|------------------------|---------------------------|
| `go.mod`                   | `go build ./...`          | `go test ./...`        | `golangci-lint run`        |
| `package.json`             | `npm run build`           | `npm test`             | `npm run lint`             |
| `package.json` + `yarn.lock` | `yarn build`            | `yarn test`            | `yarn lint`                |
| `pyproject.toml`/`setup.py`| `python -m py_compile .`  | `pytest`               | `ruff check .`             |
| `Cargo.toml`               | `cargo build`             | `cargo test`           | `cargo clippy`             |
| `pom.xml`                  | `mvn compile`             | `mvn test`             | —                          |
| `build.gradle`             | `./gradlew build`         | `./gradlew test`       | —                          |

Record the final commands as:
- `BUILD_CMD` — compiles / type-checks
- `TEST_CMD` — runs the test suite
- `LINT_CMD` — linter (optional, skip if not present)

### 0c. Read changed files for semantic understanding

For every changed file, read its content (or at least its imports/exports) to understand:
- What does this file do?
- What does it import from other **changed** files?
- What other **changed** files import from it?

This builds a **dependency graph** among the changed files — critical for ensuring each PR builds independently.

```bash
# For Go: find intra-package imports among changed files
git diff --name-only $BASE_BRANCH...HEAD | grep '\.go$' | xargs grep -l "^import" 2>/dev/null

# For JS/TS: find relative imports among changed files
git diff --name-only $BASE_BRANCH...HEAD | grep -E '\.(js|ts|jsx|tsx)$' | xargs grep -l "^import\|require(" 2>/dev/null

# For Python: find relative imports among changed files
git diff --name-only $BASE_BRANCH...HEAD | grep '\.py$' | xargs grep -l "^from \.\|^import " 2>/dev/null
```

---

## Step 1: Analyze & Propose Groups

Using the semantic understanding from Step 0c, group files so that **each group is closed under its dependency graph** — i.e., if file A depends on changed file B, they must be in the same PR (or B must be in an earlier "foundation" PR).

Choose the grouping strategy that best fits the changeset:

### Strategy A — By Change Type (Horizontal)
- Refactoring / renames (no logic change)
- New shared types / interfaces / protos
- Feature implementation
- Tests
- Config / CI / infra
- Documentation

### Strategy B — By Feature (Vertical)
- Each group is a complete working slice (e.g., frontend + backend + tests for one feature)

### Strategy C — By Subsystem/Concern
- Group by domain: auth, database, API, CLI, etc.

### Strategy D — Foundation + Feature layers
- PR 0: Shared utilities/types that others depend on
- PR 1..N: Features that use those utilities (each independent of each other)

**Present the proposed grouping to the user before any git work:**

```
## Proposed Split

| PR # | Branch name          | Files                          | ~Lines | Rationale                        |
|------|----------------------|-------------------------------|--------|----------------------------------|
| 1    | refactor/auth-rename | src/auth/*.go                 | 120    | Pure renames, no logic change    |
| 2    | feat/token-api       | src/api/token.go, tests/...   | 200    | New endpoint + tests (self-cont) |
| 3    | chore/update-docs    | docs/**, README.md            | 50     | Docs only                        |

Base branch for all PRs: main
Build verification: `go build ./...` + `go test ./...`
Dependency notes: token.go imports auth package — auth rename must land first OR both go in same PR.
```

**Wait for user confirmation or adjustments before proceeding.**

---

## Step 2: Resolve Cross-PR File Dependencies

After the user approves the grouping, finalize the dependency plan:

For each pair of proposed PRs (A, B):
- If any file in A imports a **changed** file in B → A cannot merge before B
- Options:
  1. **Merge the two groups** into one PR
  2. **Make B a foundation PR** — keep it standalone, A targets `main` but reviewer knows to merge B first (document ordering in PR body)
  3. **Extract the shared change** into its own tiny foundation PR that both depend on

After resolving, confirm final grouping is dependency-clean (each PR compiles on its own against `main`).

---

## Step 3: Create Branches and Apply Changes

For each group (in dependency order — foundations first):

### 3a. Create branch from base
```bash
git fetch origin $BASE_BRANCH
git checkout -b <branch-name> origin/$BASE_BRANCH
```

Branch naming: `<type>/<short-description>` — e.g., `refactor/auth-rename`, `feat/token-api`

### 3b. Apply the changes for this group

**Option A — Checkout specific files from the source branch** (most common):
```bash
git checkout <big-branch> -- path/to/file1 path/to/file2
git add path/to/file1 path/to/file2
git commit -m "<type>(<scope>): <description>"
```

**Option B — Cherry-pick commits** (when commits map cleanly to groups):
```bash
git cherry-pick <commit-sha>
```

**Option C — Patch specific files**:
```bash
git diff $BASE_BRANCH...HEAD -- path/to/file > /tmp/partial.patch
git apply /tmp/partial.patch
git add .
git commit -m "<message>"
```

### 3c. Verify build SUCCESS — mandatory before pushing

Run the build and test commands identified in Step 0b. **Do not push if any command fails.**

```bash
# Run BUILD_CMD
echo "Running: $BUILD_CMD"
$BUILD_CMD
BUILD_EXIT=$?

# Run TEST_CMD
echo "Running: $TEST_CMD"
$TEST_CMD
TEST_EXIT=$?

# Run LINT_CMD if defined
if [ -n "$LINT_CMD" ]; then
  echo "Running: $LINT_CMD"
  $LINT_CMD
  LINT_EXIT=$?
fi
```

**If build fails:**
1. Read the error output carefully
2. Determine root cause — is it a missing dependency from another group? A compilation error in the applied files? A broken import?
3. Attempt to fix:
   - If a changed file that this group depends on is missing → move it into this group, or move both into a foundation PR
   - If it's a genuine bug introduced by the partial apply → fix it with an additional commit
   - If it cannot be fixed without restructuring the groups → **stop, report to the user**, and ask how to proceed
4. Re-run build verification after the fix
5. Only proceed when all commands exit 0

### 3d. Push only after green build
```bash
git push origin <branch-name>
echo "Branch <branch-name> pushed — build verified green."
```

---

## Step 4: Create the PRs

```bash
gh pr create \
  --base $BASE_BRANCH \
  --title "<type>(<scope>): <short description>" \
  --body "$(cat <<'EOF'
## Summary
- <bullet 1>
- <bullet 2>

## Why
<one sentence on motivation>

## Part of series
This is PR X of Y split from a larger change.
- [ ] PR 1: <!-- will be filled after all PRs are created -->
- [ ] PR 2:
- [ ] PR 3:

## Build verification
- Build: `<BUILD_CMD>` ✅
- Tests: `<TEST_CMD>` ✅

## Test plan
- [ ] <test item>
EOF
)"
```

After **all** PRs are created, edit each PR body to fill in the sibling PR links.

---

## Step 5: Return to Original Branch and Summarize

```bash
git checkout <original-branch>
```

Print the summary:

```
## Split Complete

| PR  | Branch               | URL                     | Build | Tests |
|-----|----------------------|------------------------|-------|-------|
| 1   | refactor/auth-rename | https://github.com/... | ✅    | ✅    |
| 2   | feat/token-api       | https://github.com/... | ✅    | ✅    |
| 3   | chore/update-docs    | https://github.com/... | ✅    | ✅    |

All PRs target: main
Merge order: Any order — all are independent.
```

---

## Edge Cases

### Mix of committed and uncommitted changes
`git diff origin/$BASE_BRANCH` captures everything in one pass — no special handling needed. Committed changes, staged changes, and unstaged working-tree changes are all included automatically.

### Already has a GitHub PR open
Fetch its diff and branch, split from there. Do not close the original PR until the user confirms.

### Build fails due to cross-group dependency
This is the most common failure. Resolution in priority order:
1. Move the dependency file into the failing PR's group
2. Extract the shared dependency into a standalone foundation PR; mark others as "merge after foundation"
3. Merge the two groups into one PR if separation doesn't make sense

### Large binary files
Skip from analysis. Flag to user and ask which group they belong to.

### Merge conflicts during cherry-pick
Stop immediately. Show conflict details. Ask user to resolve before continuing.

### CI runs on push (GitHub Actions, etc.)
After pushing each branch, optionally wait for CI:
```bash
gh run list --branch <branch-name> --limit 1
# Wait and poll until CI completes, then verify status
gh run watch <run-id>
```
If CI fails after push, investigate, fix, push again before creating the PR.

---

## References

- [splitting-strategies.md](references/splitting-strategies.md) — Which strategy to pick, rules for test placement, what to avoid
- [build-verification.md](references/build-verification.md) — Makefile detection, language fallbacks, failure diagnosis, per-branch checklist
- [dependency-resolution.md](references/dependency-resolution.md) — How to detect and resolve cross-PR file dependencies before branching
