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

## Step -1: Sync Target Branch

Before doing anything else, sync the local target (base) branch with remote and merge it into the current development branch:

```bash
BASE_BRANCH="${ARGUMENTS:-main}"
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)

# 1. Fetch latest from remote
git fetch origin

# 2. Sync local base branch with remote
git checkout $BASE_BRANCH
git pull origin $BASE_BRANCH

# 3. Return to development branch and merge in the updated base
git checkout $CURRENT_BRANCH
git merge $BASE_BRANCH
```

**If the merge produces conflicts:** stop immediately, show the conflicting files, and ask the user to resolve them before continuing.

After a clean merge, proceed to Step 0.

---

## Step 0: Understand Context

### 0a. Identify the changeset

The base branch is `$ARGUMENTS` if provided, otherwise `main` (already set as `BASE_BRANCH` in Step -1). Collect **all** changes relative to it — committed and uncommitted together:

```bash
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

### Prime directive: every PR must be independently mergeable

**Avoid chaining PRs at all costs.** A chained PR (one that cannot merge until another merges first) is a failure of grouping, not a feature. Reviewers should be able to merge any PR in any order without breaking the repo.

Techniques to break a dependency and make a PR stand alone:
- **Inline the dependency** — copy or duplicate the small shared piece into each PR rather than sharing it; the duplication is temporary and harmless
- **Use the existing (unmodified) API** — if feature A needs a new method on package X, check whether a slightly different existing method satisfies A; if so, skip the package change and ship A alone
- **Stub or no-op** — add the new function/type as an empty stub that compiles but does nothing; the feature PR fleshes it out without needing the real implementation first
- **Bundle when forced** — if two pieces genuinely cannot compile without each other, put them in the same PR rather than chaining

Chaining (PR B must merge before PR A) is only acceptable when all of the above are truly impossible. If you propose a chain, explicitly call it out and explain why independence could not be achieved.

### Default grouping: By Feature (Vertical)

Group by **feature / logical change** — a feature may span multiple folders and packages, and that is correct. Each PR is a complete working slice of one feature.

If a feature must modify an existing shared package, prefer **bundling that package change into the same feature PR** over creating a separate foundation PR that the feature depends on — unless that package change is also needed by a completely separate, unrelated PR.

### Tests and docs always travel with their PR

**Never** split tests or docs into a separate PR. They belong to the PR that introduces the code they cover:
- Unit/integration tests for feature X → same PR as feature X
- Inline code comments or doc updates for feature X → same PR as feature X
- A standalone docs-only PR is only valid when it documents something that already exists in `main` (e.g. a README catch-up, architecture notes) — not new work being split off

### When to use other strategies

| Situation | Strategy |
|---|---|
| Pure refactor / rename, no logic change | Standalone PR — no deps possible |
| Shared change needed by 2+ independent features | Extract to one foundation PR; make feature PRs compile against the stub on `main` until it lands |
| Completely orthogonal subsystems | Split by subsystem — each is independent |
| Docs for already-existing functionality | Standalone PR, any order |

**Present the proposed grouping to the user before any git work:**

```
## Proposed Split

| PR # | Branch name         | Folders / packages touched                              | ~Lines | Independent? | Rationale                                      |
|------|---------------------|---------------------------------------------------------|--------|--------------|------------------------------------------------|
| 1    | feat/login-flow     | pkg/auth/, pkg/login/, api/login.go, tests/, docs/login | 320    | YES          | Auth change + tests + docs bundled with feature|

Base branch for all PRs: main
Build verification: `go build ./...` + `go test ./...`
Chained PRs: none
```

**Wait for user confirmation or adjustments before proceeding.**

---

## Step 2: Eliminate Remaining Cross-PR Dependencies

After the user approves the grouping, audit every pair of proposed PRs for hidden dependencies:

For each pair (A, B) where a file in A imports a changed file in B:
1. **Bundle** — move the dependency into A (preferred)
2. **Stub** — add a no-op version of the dependency to `main` via a tiny standalone PR that has zero dependencies itself, then both A and B build against the stub independently
3. **Inline** — duplicate the small shared piece in each PR
4. **Chain as last resort** — only if 1–3 are genuinely impossible; document exactly why

Goal: the "Chained PRs" line in the summary table above should read **none**. If any chain remains after exhausting options 1–3, flag it clearly to the user before proceeding.

After resolving, confirm every PR compiles independently against `main`.

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

**Commit message rules:**
- Write a contextual message that describes what the user built and why
- Use the format `<type>(<scope>): <description>` reflecting the actual change
- **Never** append `Co-Authored-By` or any Claude attribution — the user is the author

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
git commit -m "<type>(<scope>): <description>"
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

### 4a. Detect the repository's PR template

Before writing the PR body, check for an existing PR description template:

```bash
# Common template locations
cat .github/pull_request_template.md 2>/dev/null \
  || cat .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null \
  || cat docs/pull_request_template.md 2>/dev/null \
  || cat PULL_REQUEST_TEMPLATE.md 2>/dev/null
```

**If a template is found:** use it as the base structure for every PR body. Fill in all its sections with content appropriate to each PR — do not skip or remove any sections the template defines.

**If no template is found:** fall back to the default structure below.

### 4b. Raise the PR

**PR title rules:**
- Max 72 characters
- Format: `<type>(<scope>): <what changed>` — no filler words, no trailing punctuation
- Describe the change, not the process (e.g. `feat(auth): add token refresh` not `Split PR 2 of 3 - implement token refresh logic for auth module`)

```bash
gh pr create \
  --base $BASE_BRANCH \
  --title "<type>(<scope>): <what changed>" \
  --body "$(cat <<'EOF'
<PR body — either the repo template filled in, or the default below>

--- DEFAULT (use only when no repo template exists) ---
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
