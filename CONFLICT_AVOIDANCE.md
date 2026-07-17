# Conflict Avoidance: sibling branches sharing an internal component

This note covers the case where **two (or more) feature branches off the same
release branch both depend on the same new internal component**, and they collide
badly when you try to merge them back.

Concrete example from this repo:

- `MSC-217412` — input-chip
- `MSC-217413` — input-autocomplete
- both branched off `release-v1.3.0`
- both created / edited the shared internal `_token-input` component

Throughout, substitute:

- `SHARED` — a branch holding only the shared dependency (e.g. `MSC-token-input`).
- `FEATURE_A`, `FEATURE_B` — the feature branches (e.g. `MSC-217412`, `MSC-217413`).
- `release-vX.Y.Z` — the release branch they target (e.g. `release-v1.3.0`).

---

## Why this conflicts

The problem is **branch topology**, not git being difficult.

Each feature branched off the release independently and each one wrote its own copy
of `_token-input` (and everything derived from it). Git therefore has **two unrelated
histories for the same files**. When the second branch merges into `release-vX.Y.Z`,
every shared file collides:

- the `_token-input` source itself
- every Stencil-generated artifact that references it:
  `packages/core/docs/docs.json`, `packages/core/src/components.d.ts`,
  `packages/core/docs/vscode-data.json`, and the per-component `readme.md` files

The generated files make it look worse than it is — but the root cause is that the
shared code was written twice.

**Rule of thumb:** any code that two branches will both touch should live in exactly
one place in history. If it's written twice, it will conflict twice.

---

## Preferred approach (going forward): extract the shared dependency first

Give `_token-input` a single source of truth *before* building the features on top.

### 1. Create a branch with only the shared component

```bash
git fetch origin
git checkout -b SHARED origin/release-vX.Y.Z
# implement ONLY _token-input here (plus its generated docs), commit, push
git push -u origin SHARED
```

### 2. Base both features on the shared branch (stacking)

```bash
git checkout -b FEATURE_A SHARED   # input-chip
git checkout -b FEATURE_B SHARED   # input-autocomplete
```

Now both features inherit the **same commits** for `_token-input`. Git sees one shared
history, so the shared files no longer conflict.

### 3. Land in dependency order

1. Merge `SHARED` into `release-vX.Y.Z` first (small, low-risk PR).
2. Update each feature onto the new release tip:

```bash
git checkout FEATURE_A
git rebase origin/release-vX.Y.Z   # or: git merge origin/release-vX.Y.Z
```

Because the shared files already match what's now on the release, there is nothing to
resolve for them. Repeat for `FEATURE_B`.

> Prefer `rebase` while the branch is still private/unpushed for a clean linear history;
> use `merge` once it's shared with others or already has an open PR.

---

## Recovery (your current situation): the shared code is already split across two branches

You can't retro-extract easily, so **sequence the merges** instead of merging both in
parallel. You resolve the shared code **once** rather than fighting it from both sides.

### 1. Merge the first feature normally

```bash
git checkout FEATURE_A
git merge origin/release-vX.Y.Z    # resolve per MERGE_RUNBOOK.md, then push PR / land it
```

Once `FEATURE_A` is on `release-vX.Y.Z`, that release now contains a canonical
`_token-input`.

### 2. Pull the updated release into the second feature and resolve ONCE

```bash
git fetch origin
git checkout FEATURE_B
git merge origin/release-vX.Y.Z
```

Now resolve the `_token-input` conflicts deliberately:

- If both branches evolved the component, **keep the union** — merge the real behaviour
  by hand so `FEATURE_B` keeps what it needs and doesn't regress `FEATURE_A`.
- For the Stencil-generated files, **do not hand-merge** — resolve then regenerate
  (see `MERGE_RUNBOOK.md` §4a):

```bash
git checkout --theirs -- packages/core/docs/docs.json packages/core/src/components.d.ts
npm exec -w @foundry/core -- stencil build --docs
git add packages/core/docs/docs.json packages/core/src/components.d.ts
```

### 3. Verify and commit

```bash
git diff --name-only --diff-filter=U        # empty
git grep -n '^<<<<<<<\|^>>>>>>>' -- .        # no markers left
git commit -m "Merge branch 'release-vX.Y.Z' into FEATURE_B"
```

After this, `FEATURE_B` shares the same `_token-input` history as the release, so its own
PR merges cleanly.

---

## Habits that keep this from recurring

- **One source of truth for shared code.** Extract it to its own branch/PR and stack on
  top; never let two branches originate the same component independently.
- **Land in dependency order, smallest first.** Merge the shared piece before the things
  that consume it.
- **Merge the release into your branch early and often**, not once at the end. Small,
  frequent conflict resolutions beat one giant one.
- **Treat generated files as build output, not source.** Resolve by taking one side and
  regenerating (`stencil build --docs`) — never hand-edit `docs.json`,
  `components.d.ts`, `vscode-data.json`, or the generated readmes.
- **Keep feature branches short-lived.** The longer two siblings live in parallel, the
  more the shared component drifts apart.

---

## How to avoid stacking a branch on another feature's unlanded work

Stacking (branching off a feature that hasn't merged yet) is convenient but dangerous: if
the base is reverted or rebased, your branch inherits the damage — that's how the
`fc-feedback-message` component was lost (see `LOST_CHANGES.md`). Keep branches independent
unless you deliberately choose otherwise.

### 1. Branch from a landed, shared base — not from another feature

```bash
git fetch origin
git switch -c FEATURE origin/release-vX.Y.Z     # base = the shared release, NOT a feature branch
```

### 2. Sanity-check your base

```bash
# Unlanded commits I'm carrying beyond the release:
git log --oneline origin/release-vX.Y.Z..FEATURE

# Am I sitting on top of another feature's un-landed tip?
git merge-base --is-ancestor origin/OTHER_FEATURE FEATURE && echo "I depend on OTHER_FEATURE's unlanded work"
```

### 3. Decide by whether the dependency is real

- **Don't actually need it** → branch from the release. Done.
- **Need only a *shared component*** → extract it to its own branch and land it first (the
  "single source of truth" pattern at the top of this file). Both features then branch off
  the updated release and the shared code no longer conflicts *or* stacks.
- **Genuinely must build on unlanded work now** → stack *consciously*: one level deep,
  short-lived, land the base first, and rebase off it the moment it lands:

  ```bash
  git rebase --onto origin/release-vX.Y.Z origin/SHARED FEATURE
  ```

  Never `git revert` the base merge while something is stacked on it.

**Rule of thumb:** branch from something stable and shared. "I'll just branch off my other
feature" is a smell — either the shared part should be its own landed branch, or you're
accepting a deliberate, short, ordered stack, never an accidental one.

See `MERGE_RUNBOOK.md` for the mechanical, step-by-step conflict resolution once a merge
is unavoidable, and `LOST_CHANGES.md` for what happens when a stacked base gets reverted.
