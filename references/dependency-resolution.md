# Cross-PR Dependency Resolution

The hardest part of splitting a PR is ensuring each sub-PR compiles and passes tests **on its own**, without relying on files from sibling PRs that haven't merged yet.

## Detecting Dependencies Among Changed Files

Before finalising groups, build a dependency map of the changed files:

```bash
# Get all changed files
CHANGED=$(git diff --name-only origin/$BASE_BRANCH)

# Go: which changed files import other changed files?
echo "$CHANGED" | grep '\.go$' | while read f; do
  grep -n '"' "$f" 2>/dev/null | grep -oP '(?<=")[^"]+(?=")' | while read pkg; do
    echo "$CHANGED" | grep -q "$(echo $pkg | tr '/' '/')" && echo "$f -> $pkg"
  done
done

# JS/TS: relative imports among changed files
echo "$CHANGED" | grep -E '\.(js|ts|jsx|tsx)$' | xargs grep -n "from '\." 2>/dev/null

# Python: relative imports among changed files
echo "$CHANGED" | grep '\.py$' | xargs grep -n "^from \." 2>/dev/null
```

For each changed file, read its imports and check if they point to other changed files. That edge in the import graph is a **cross-file dependency**.

## Dependency Scenarios and Fixes

### Scenario 1: File A imports from File B — both changed

```
FileA (Group 1) ──imports──> FileB (Group 2)
```

Group 1 will fail to build without Group 2's changes. Options:

| Option | When to use | How |
|--------|------------|-----|
| **Merge groups** | Changes are small and cohesive | Put both files in one PR |
| **Foundation PR** | FileB is a shared utility used by multiple groups | FileB → PR 0; FileA → PR 1 targeting main |
| **Assign to owner** | One group owns most of FileB's changes | Move FileB entirely into that group |

### Scenario 2: Shared file modified by multiple groups

```
FileC changed in Group 1 (lines 10-30)
FileC changed in Group 2 (lines 60-80)
```

The changes are in disjoint sections but the file is listed in both. Options:

| Option | When to use |
|--------|------------|
| Assign to the group with more changes | Changes don't interact at all |
| Extract FileC into a foundation PR | Changes are both needed by others |
| Merge the two groups | Groups are small and related enough |

### Scenario 3: New type/interface used across groups

```
types.go (new struct) ──used by──> handler.go (Group 1)
                      ──used by──> service.go (Group 2)
```

`types.go` must land before anything that uses it. Fix:
- Create PR 0 with only `types.go`
- PR 1 and PR 2 both target `main`, note "merge PR 0 first" in their bodies

### Scenario 4: No cross-group dependencies

All groups are fully independent. Proceed directly — any merge order works.

## Validation Before Branching

After assigning all files to groups, do a final check for each group:

1. List all imports in the group's files that point to other **changed** files
2. Verify every such import's target file is **in the same group**
3. If not → apply one of the fixes above

Only proceed to branch creation once every group is closed under its own dependency graph.

## Communicating Dependencies in PRs

If a merge ordering recommendation exists (e.g., PR 0 must land first), make it explicit in every affected PR body:

```markdown
## Merge order
> Merge [PR 0: shared types](url) before this PR. All other PRs in this series are independent.
```

Never silently depend on another unmerged PR — always document it.
