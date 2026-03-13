# Splitting Strategies for Independent PRs

Adapted from Google's Small CLs guide: https://google.github.io/eng-practices/review/developer/small-cls.html

## What Makes a Good Split?

Each PR must be:
- **Self-contained** — addresses one aspect, one feature component, or one concern
- **Build-passing** — the system compiles and tests pass after merging it alone
- **Independently mergeable** — no dependency on another unmerged PR in the series
- **Reviewable in isolation** — a reviewer can understand it without reading sibling PRs

Size guidelines:
- ~100 lines changed: ideal
- ~300 lines changed: acceptable with good rationale
- ~1000 lines changed: too large, split further

## Strategy A — By Change Type (Horizontal)

Best when the big PR mixes different *kinds* of work.

| Group          | What goes in it                                         | Example files                        |
|----------------|---------------------------------------------------------|--------------------------------------|
| Refactor       | Renames, moves, restructuring — zero logic change       | Renamed packages, moved files        |
| Shared types   | New interfaces, protos, shared structs used by others   | `types.go`, `schema.proto`           |
| Feature        | New logic, new behaviour                                | `handler.go`, `service.go`           |
| Tests          | Only if tests can't be co-located with the feature      | `*_test.go`, `spec/`                 |
| Config / CI    | Env vars, Dockerfile, GitHub Actions, Makefile changes  | `.github/`, `Makefile`, `Dockerfile` |
| Docs           | README, docs/, comments-only changes                   | `README.md`, `docs/`                 |

Rule: **Refactor PR must merge before Feature PR** if the feature touches the refactored code. All others can merge in any order.

## Strategy B — By Feature (Vertical)

Best when the big PR implements several distinct features.

Each group is a **complete vertical slice**: frontend + backend + tests + docs for one feature. The feature works end-to-end after its PR merges — no half-implemented states.

Example: A PR that adds user auth and payment support → split into:
- PR 1: User authentication (all layers)
- PR 2: Payment support (all layers)

## Strategy C — By Subsystem / Concern

Best for large codebases with clear domain boundaries.

Group files by the subsystem they belong to:
- `auth/` → one PR
- `database/` → one PR
- `api/` → one PR

Each subsystem team can review their own PR independently.

## Strategy D — Foundation + Feature Layers

Best when several features share new common utilities.

- **PR 0 (Foundation)**: Shared helpers, base types, utilities — no feature logic
- **PR 1..N (Features)**: Each feature uses the foundation; independent of each other

PR 1..N all target `main` but note in their body that PR 0 should merge first (or is already merged).

## Choosing the Right Strategy

Ask these questions in order:

1. **Does the PR mix refactoring with feature work?** → Strategy A (separate refactor first)
2. **Are there multiple distinct features?** → Strategy B (vertical slices)
3. **Are changes spread across clear domain boundaries?** → Strategy C (by subsystem)
4. **Do multiple changes depend on new shared code?** → Strategy D (foundation first)
5. **None of the above?** → Strategy A by default (horizontal by change type)

## Handling Refactoring

Always keep in its own PR, separate from features and bug fixes:
- Moving or renaming a package → separate PR
- Minor cleanups *within* a feature PR are acceptable if they are trivially small
- Refactoring-only PRs still need test coverage (existing tests passing is sufficient)

## Test Placement

- Tests **always travel with the logic they cover** in the same PR
- Exception: if tests for feature X depend on infrastructure introduced in a separate foundation PR, split them only if the feature PR already has meaningful test coverage
- Refactoring PRs must keep existing tests; add new ones only if coverage gaps are exposed

## What to Avoid

- **Don't** create a PR that only passes tests when another unmerged PR is also present
- **Don't** split a single atomic change (e.g., renaming a type used everywhere must touch all call sites in one PR)
- **Don't** leave the system in a broken intermediate state after any PR in the series merges
