# Build Verification for Split PRs

Each PR branch must pass build verification **before** it is pushed or a PR is created. This guarantees reviewers and CI see a green branch from the start.

## Detection Order

### 1. Read the Makefile first

```bash
cat Makefile
```

Look for targets that map to build, test, and lint. Prefer `make <target>` — it captures the correct flags, env vars, and dependencies already defined by the project.

Common target names to look for:

| Purpose | Common target names                                  |
|---------|------------------------------------------------------|
| Build   | `build`, `compile`, `all`, `install`                 |
| Test    | `test`, `tests`, `check`, `unit-test`, `integration` |
| Lint    | `lint`, `vet`, `fmt-check`, `check`, `staticcheck`   |
| Combined| `ci`, `verify`, `validate`, `pre-commit`             |

If a combined target exists (e.g., `make ci`), prefer it — it runs everything in the correct order.

### 2. Language manifest fallback

Use only if no Makefile exists or it has no relevant targets:

| Detected by                  | BUILD_CMD                  | TEST_CMD              | LINT_CMD               |
|------------------------------|----------------------------|-----------------------|------------------------|
| `go.mod`                     | `go build ./...`           | `go test ./...`       | `golangci-lint run`    |
| `package.json`               | `npm run build`            | `npm test`            | `npm run lint`         |
| `package.json` + `yarn.lock` | `yarn build`               | `yarn test`           | `yarn lint`            |
| `pyproject.toml` / `setup.py`| `python -m py_compile .`   | `pytest`              | `ruff check .`         |
| `Cargo.toml`                 | `cargo build`              | `cargo test`          | `cargo clippy`         |
| `pom.xml`                    | `mvn compile`              | `mvn test`            | —                      |
| `build.gradle`               | `./gradlew build`          | `./gradlew test`      | —                      |

## Running Verification

After applying a group's changes to its branch, run in this order:

```bash
# 1. Build — catch compilation / type errors first
$BUILD_CMD
# Must exit 0 before continuing

# 2. Tests — catch logic regressions
$TEST_CMD
# Must exit 0 before continuing

# 3. Lint — catch style/static issues (skip if LINT_CMD not defined)
$LINT_CMD
# Must exit 0 before continuing
```

**Do not push the branch until all three exit 0.**

## Diagnosing Failures

### Compilation error — missing symbol / undefined reference
**Cause**: A file in this group imports a symbol defined in a changed file assigned to a *different* group.
**Fix**: Move the depended-on file into this group, or create a foundation PR that introduces the shared symbol.

### Test failure — function not found / nil pointer
**Cause**: Test relies on a helper or fixture that lives in another group's files.
**Fix**: Move the helper into this group alongside the test.

### Lint error — in a file this PR didn't change
**Cause**: The base branch already had lint issues; unrelated to this split.
**Fix**: Either fix the pre-existing lint issue in this PR (acceptable if trivial) or document it and skip lint for this branch with a comment in the PR body.

### Build passes but CI fails after push
CI may run additional checks not covered by local commands (integration tests, Docker builds, etc.). After push:
```bash
gh run list --branch <branch-name> --limit 1
gh run watch <run-id>
```
Investigate the CI failure, fix, and push again before creating the PR.

## Verification Checklist per Branch

Before pushing each branch, confirm:

- [ ] `$BUILD_CMD` exits 0
- [ ] `$TEST_CMD` exits 0
- [ ] `$LINT_CMD` exits 0 (or is not applicable)
- [ ] No files from other groups' changes are present in `git diff origin/$BASE_BRANCH`
- [ ] The branch targets `origin/$BASE_BRANCH` (not another feature branch)
