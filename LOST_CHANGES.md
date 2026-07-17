# Lost Changes: why merged work disappears (the merge-revert trap)

This note explains a scenario where a feature was merged successfully, the release went
out, and yet **the feature's content was missing afterwards** — even though the diff is
still clearly visible in the original PR.

Real example from this repo:

- `MSC-212947` created the `fc-feedback-message` component (7 files under
  `packages/core/src/components/informative/feedback-message/`).
- It merged into the release line via **PR #223** (merge commit `98064811`). The merge
  commit's tree contains all 7 files — the merge worked.
- The files were later removed from `origin/release-v1.1.3` by:

  ```
  691dff3d  Revert "Merge branch 'MSC-213753' ... into MSC-213753"
  ```

Throughout, substitute:

- `FEATURE` — the branch whose content went missing (e.g. `MSC-212947`).
- `OTHER` — a later branch that was stacked on top of the same line (e.g. `MSC-213753`).
- `release-vX.Y.Z` — the release branch (e.g. `release-v1.1.3`).

---

## Why this happens

Your work was **not** lost by a failed merge. It was undone by a `git revert` of a
**merge commit**.

Here's the mechanism:

1. `FEATURE` merged into the release line correctly (the content was really there).
2. `OTHER` was stacked on top of that line, so its history transitively **included**
   `FEATURE`'s merge.
3. Something went wrong and someone ran `git revert` on the **merge commit** for `OTHER`.
4. Reverting a merge undoes **every file change that merge brought in** — including
   `FEATURE`'s files that were riding along — **but it leaves the merge commit in the
   graph.**

The dangerous part is step 4's side effect:

> Because the merge commits are still ancestors of the branch, git considers those
> commits **already integrated**. Re-merging `FEATURE` (or `OTHER`) later brings in
> **nothing** — git sees no new commits to apply — so the content never comes back on its
> own.

That's why the changes are still visible in the original PR diff (the commits exist) but
absent from the release tree (their effect was reverted, and re-merging is a no-op).

A secondary source of confusion: a **stale local branch**. In this repo the local
`release-v1.1.3` sat far behind `origin/release-v1.1.3`, so inspecting it showed a
different world than what actually shipped. Always compare against `origin/...`.

---

## How to confirm you've hit this

```bash
git fetch origin

# Does the content actually exist on the shipped branch?
git ls-tree -r --name-only origin/release-vX.Y.Z | grep -i <component-name>

# What removed it? (look for a Revert of a merge)
git log --oneline --diff-filter=D origin/release-vX.Y.Z -- <path/to/component>/

# Was the original feature merge real?
git show --stat <feature-merge-sha>     # e.g. the PR merge commit
```

If a `Revert "Merge ..."` shows up as the deleter, this is the merge-revert trap.

---

## How to recover

### Option A — revert the revert (preferred when the merge should stay)

```bash
git checkout release-vX.Y.Z
git revert <revert-sha>        # e.g. git revert 691dff3d
```

This re-applies the file changes the revert removed. Do this when you want the content
back **and** intend to keep the branch merged. (If you ever want to re-merge `OTHER`
properly later, you'll then need to revert *this* revert again — hence: avoid reverting
merges in the first place.)

### Option B — re-introduce just the lost files

If only one component was collateral damage, cherry-pick or re-commit those files:

```bash
git checkout release-vX.Y.Z
git checkout <feature-merge-sha> -- packages/core/src/components/informative/feedback-message/
# regenerate docs, then commit (see MERGE_RUNBOOK.md §4a)
npm exec -w @foundry/core -- stencil build --docs
git add -A && git commit -m "Restore feedback-message lost by merge revert"
```

---

## How to avoid it in the future

- **Do not `git revert` merge commits.** Revert the specific offending commit(s) instead,
  or (on a not-yet-shared branch) roll back with `git reset`. A merge revert undoes *all*
  content that merge carried, and poisons future re-merges.
- **If you must revert a merge, document it and plan the un-revert.** The changes will
  never return via a normal re-merge; you must `git revert` the revert.
- **Avoid deep branch stacking on a shared line.** `OTHER` sitting on top of `FEATURE`
  meant reverting `OTHER`'s merge silently deleted `FEATURE`'s files. Keep branches
  independent where possible so one rollback can't drag another's work out.
- **Verify content, not just green checks.** "The PR merged" ≠ "the code shipped." After a
  release, confirm the files exist in the tree:

  ```bash
  git ls-tree -r --name-only origin/release-vX.Y.Z | grep -i <component>
  git diff origin/release-vX.Y.Z FEATURE -- <path>   # should be empty for shipped work
  ```

- **Keep local release branches fresh.** `git fetch origin` and reset stale locals so what
  you inspect matches what actually shipped.
- **Prefer reverting via a new PR** that reverts only the intended change set, reviewed
  like any other change, rather than an ad-hoc `git revert` of a big merge during release
  crunch.

---

## How to avoid stacking a branch on another feature's unlanded work

The root of this particular incident is topology: `OTHER` (`MSC-213753`) was built on top
of `FEATURE` (`MSC-212947`) *before `FEATURE` had landed*, so reverting `OTHER`'s merge
deleted `FEATURE`'s files. Avoiding that is about two things — noticing when you're about
to stack, and choosing a topology that keeps branches independent.

### 1. Branch from a landed, shared base — not from another feature

```bash
git fetch origin
git switch -c MSC-NEW origin/release-vX.Y.Z     # base = the shared release, NOT a feature branch
```

### 2. Sanity-check what you're actually built on

```bash
# Unlanded commits I'm carrying that the release doesn't have:
git log --oneline origin/release-vX.Y.Z..MSC-NEW

# Am I sitting on top of another feature's un-landed tip?
git merge-base --is-ancestor origin/OTHER MSC-NEW && echo "I depend on OTHER's unlanded work"
```

If that last line prints, you've stacked — the same shape that caused this incident.

### 3. Decide by whether you truly depend on the other work

- **You don't actually need it** → just branch from the release (most common; the overlap
  felt convenient but wasn't required).
- **You need a *shared component*, not the whole feature** → extract it into its own tiny
  branch, land that first, then branch off the updated release (see
  `CONFLICT_AVOIDANCE.md`). The dependency stops being "unlanded."
- **You genuinely must build on unlanded work now** → stack *consciously*: keep it one
  level deep and short-lived, land in dependency order (base merges first), and rebase off
  the temporary base the moment it lands:

  ```bash
  git rebase --onto origin/release-vX.Y.Z origin/MSC-BASE MSC-NEW
  ```

  And **never `git revert` the base merge while something is stacked on it** — that is the
  exact trap documented above.

**Rule of thumb:** branch from something stable and shared. Treat "I'll just branch off my
other feature" as a smell that means either the shared part should be its own landed branch
or you're accepting a deliberate, short, ordered stack — never an accidental one.

See `MERGE_RUNBOOK.md` for mechanical conflict resolution and `CONFLICT_AVOIDANCE.md` /
`RELEASE_TRAIN.md` for branch-topology guidance that keeps you out of this situation.
