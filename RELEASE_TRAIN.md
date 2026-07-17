# Release Train: building on work that lives in an unreleased newer line

This note covers sequential release lines ("a release train") where each version ships
after the previous one, and you need to build features that **depend on a component that
only exists in a not-yet-released line**.

Real example from this repo:

- Releases ship in order: `release-v1.1.3` → `release-v1.2.0` → `release-v1.3.0` → `main`.
- The chip component (`fc-input-chip`'s foundation) is being added in `release-v1.2.0`,
  which ships **after** `release-v1.1.3`.
- You already need to build `fc-input-chip` and `fc-input-autocomplete` for
  `release-v1.3.0`, and they depend on the chip that is still only in `v1.2.0` (not in
  `main` yet).

This is the same family as `CONFLICT_AVOIDANCE_2.md` (depending on unmerged work on
another line), but here the dependency is a **whole release line ahead of `main`**, not a
single side branch.

Throughout, substitute:

- `release-lower` — the earlier release that owns the dependency (e.g. `release-v1.2.0`).
- `release-upper` — the later release your feature targets (e.g. `release-v1.3.0`).
- `FEATURE` — your branch for the new components (e.g. the input-chip / autocomplete work).

---

## The mental model: forward integration

Think of the release lines as a one-directional train. Changes flow **from older lines
into newer lines**, never backwards:

```
release-v1.1.3  ─▶  release-v1.2.0  ─▶  release-v1.3.0  ─▶  main
    (ships 1st)        (ships 2nd)        (ships 3rd)
```

Two rules follow from this:

1. **`release-upper` must contain `release-lower`.** If `v1.3.0` needs the chip from
   `v1.2.0`, then `v1.2.0` must be merged **forward** into `v1.3.0`. Once it is, the chip
   is simply *there* and you build on it normally.
2. **Never merge a newer line back into an older one.** Merging `v1.3.0` into `v1.2.0`
   would drag unreleased v1.3.0 features into the v1.2.0 release. Only forward merges.

The whole problem "untangles" the moment `release-upper` sits on top of `release-lower`.

---

## The untangling procedure

### 1. Bring the dependency forward: merge `release-lower` into `release-upper`

Do the forward-merge on a **dedicated integration branch** and land it through a **PR**,
never by pushing straight to the release branch:

```bash
git fetch origin
git switch -c integrate/release-lower-into-release-upper origin/release-upper
git merge origin/release-lower       # resolve per MERGE_RUNBOOK.md if needed
git push -u origin integrate/release-lower-into-release-upper
# then open a PR:  integrate/release-lower-into-release-upper  ->  release-upper
```

> **Why not just `git checkout release-upper && git merge && git push`?**
> Release branches are shared and protected. A direct push bypasses review and CI, and (if
> protection is on) will simply be rejected. Routing the merge through an integration
> branch + PR keeps it reviewed, CI-gated, and auditable. This repo already works this way
> — see history like `Merge pull request #242 from wbd-msc/build-release-v1.1.3`.

Once the PR merges, the chip component from `v1.2.0` exists on `v1.3.0`. It's a real,
history-connected merge — not a copy — so when `v1.2.0` later reaches `main` and `v1.3.0`
follows, the lines converge cleanly with nothing duplicated.

> If `release-upper` was branched off `main` (which predates the chip), this forward merge
> is exactly what backfills the missing dependency.

### 2. Base your feature on `release-upper`

```bash
git checkout -b FEATURE origin/release-upper
# build fc-input-chip / fc-input-autocomplete on top of the now-present chip component
```

### 3. Keep the lines in sync while both are in flight

`release-lower` will keep changing during its QA/deploy cycle. Periodically forward-merge
so `release-upper` (and your `FEATURE`) don't drift from the chip's final form. Each
`release-lower -> release-upper` sync goes through the same integration-branch + PR flow
from step 1:

```bash
# Repeat the step-1 flow whenever release-lower moves:
git fetch origin
git switch -c integrate/release-lower-into-release-upper origin/release-upper
git merge origin/release-lower
git push -u origin integrate/release-lower-into-release-upper
# open PR -> release-upper, let CI + review run, then merge

# Your own feature just follows its (now-updated) base:
git switch FEATURE
git merge origin/release-upper     # or rebase while FEATURE is still private/unpushed
```

Pulling the updated `release-upper` into your own `FEATURE` is a private-branch operation,
so a plain `merge`/`rebase` is fine there — the PR gate only matters for the shared release
branches themselves.

### 4. Let the train run its course

When `release-lower` ships and merges to `main`, and then `release-upper` does the same,
the chip is already shared history on both — so the final merges carry no surprise
conflicts about it.

---

## What NOT to do

- **Don't cherry-pick the chip from `v1.2.0` into `v1.3.0`.** A cherry-pick copies the
  changes under a *new* commit SHA. When the two release lines later merge, git sees the
  same content arriving from two unrelated commits and throws phantom conflicts (and can
  even double-apply or drop it). Forward-merge the whole line instead.
- **Don't rebase a shared release line** onto another to "move" the dependency. Release
  branches are public; rewriting them breaks everyone downstream. Merge, don't rebase,
  once a branch is shared.
- **Don't merge `release-upper` back into `release-lower`** to "share" the component —
  that leaks unreleased v1.3.0 work into the v1.2.0 release.
- **Don't hand-copy files between lines.** Copies aren't tracked as the same history and
  reintroduce the divergence you're trying to avoid.
- **Don't push a forward-merge straight to a shared release branch.** Always route it
  through an integration branch + PR so review and CI run before it lands.

---

## Habits that keep the train on the rails

- **Branch a release off the line it builds on**, in train order. `v1.3.0` should descend
  from `v1.2.0` if it depends on it — not from `main`.
- **Forward-merge early and often.** Small, frequent `release-lower → release-upper`
  merges beat one giant catch-up merge at the end.
- **Merge whole lines, not individual commits, across releases.** This keeps history
  connected so eventual convergence at `main` is clean.
- **Confirm the dependency is actually present** before building on it:

  ```bash
  git ls-tree -r --name-only origin/release-upper | grep -i chip
  ```

See `CONFLICT_AVOIDANCE_2.md` for the single-branch cross-line case, `MERGE_RUNBOOK.md`
for mechanical conflict resolution, and `LOST_CHANGES.md` for why reverting merges across
these lines can make content disappear.
