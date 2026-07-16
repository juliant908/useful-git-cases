# Conflict Avoidance 2: depending on a "to-be-merged" branch on an older release line

This note covers the second case: your current work (on `release-vX.Y.Z`) depends on
another branch of yours that is **based on an older release line** (e.g.
`release-vX.(Y-1).0`) and hasn't been merged yet. Because the branch is yours, you *can*
just pull from it and resolve — but doing that naively is the riskiest option, so this
explains the trade-offs.

Throughout, substitute:

- `DEP` — your "to-be-merged" dependency branch, based on the **older** release.
- `WORK` — your current branch, based on the **newer** release `release-vX.Y.Z`.
- `release-old` — the older release `DEP` is based on (e.g. `release-v1.2.0`).
- `release-new` — the newer release `WORK` targets (e.g. `release-v1.3.0`).

---

## Why merging `DEP` straight into `WORK` is risky

`DEP` is anchored to `release-old`. If you `git merge DEP` into `WORK`, you don't just
bring the commits you care about — you bring **the entire delta between `release-old`
and `DEP`**, replayed against a `release-new` base that never saw it. That means:

- conflicts across unrelated files that happened to change between the two release lines,
- possible regressions if `release-new` already fixed or moved something `DEP` still has
  in its old form,
- a merge commit that ties two release lines together in history, which is hard to unpick
  later.

"It's my branch, I can just pull and fix it" works, but you pay for it **every time** you
re-sync, and reviewers inherit a confusing cross-line merge.

The goal: get `DEP`'s changes onto the **same base** as `WORK` before they meet.

---

## Options, best first

### Option 1 — Land `DEP` on its own first (preferred)

If `DEP` is genuinely ready, merge it through its normal PR so it becomes part of a
release, then have `WORK` pick it up from the shared base.

- If `DEP` belongs on the newer line, **retarget/rebase it onto `release-new`** (Option 2)
  and merge it there.
- Once it's landed, `WORK` just merges `release-new` normally (see `MERGE_RUNBOOK.md`) and
  the dependency arrives with no cross-line baggage.

This keeps history linear and means you resolve conflicts **once**, in `DEP`'s own PR,
instead of repeatedly in `WORK`.

### Option 2 — Rebase `DEP` onto the newer release, then base `WORK` on it

Move `DEP` off the old line so it stops carrying `release-old` baggage:

```bash
git fetch origin
git checkout DEP
git rebase --onto origin/release-new origin/release-old DEP
# resolve conflicts once, here, against the correct base
git push --force-with-lease origin DEP
```

`git rebase --onto` replays **only** `DEP`'s own commits (those after `release-old`) onto
`release-new`, dropping the unrelated old-line delta. After that, `WORK` can stack on
`DEP` cleanly:

```bash
git checkout WORK
git rebase DEP        # WORK now sits on top of DEP, on the newer base
```

> Only force-push a branch that's yours and not depended on by others. `--force-with-lease`
> protects you from clobbering work you haven't fetched.

### Option 3 — Cherry-pick just the commits you need

If `WORK` only needs a couple of specific commits from `DEP` (not the whole branch), take
exactly those:

```bash
git checkout WORK
git log --oneline origin/release-old..DEP    # find the commits you actually need
git cherry-pick <sha1> <sha2>                # apply them onto the newer base
```

This is the lightest option and avoids pulling `DEP`'s full history. The downside is the
commits now exist in two places, so if `DEP` later lands too, expect a small dedup at that
merge (git usually handles identical patches fine).

### Option 4 — Merge `DEP` into `WORK` directly (last resort)

Only if `DEP` can't be landed or rebased right now and you need to keep moving:

```bash
git checkout WORK
git merge DEP        # resolve per MERGE_RUNBOOK.md; expect cross-line noise
```

Accept that you'll re-resolve some of this each time either branch moves, and plan to
replace it with Option 1 or 2 before `WORK` opens its real PR — otherwise the cross-line
merge follows `WORK` into the release.

---

## Decision guide

- **Is `DEP` ready to land?** → Option 1.
- **Not ready, but should live on the new line?** → Option 2 (rebase `--onto`).
- **Only need a few commits?** → Option 3 (cherry-pick).
- **Blocked and need to keep working right now?** → Option 4, temporarily, then convert to
  1 or 2.

---

## Habits that keep this from recurring

- **Base a branch on the release line it will actually merge into.** If it's for
  `release-new`, branch it there — don't build on `release-old` and hope to move it later.
- **Land dependencies before dependents.** A branch that others build on should merge
  first, so consumers pick it up from a shared base instead of pulling it sideways.
- **Prefer `rebase --onto` over cross-line merges** for relocating your own unmerged work
  to a newer base — it drops the baggage instead of dragging it along.
- **Keep dependency branches small and short-lived**, so if you ever do have to
  rebase/cherry-pick, there's little to resolve.

See `MERGE_RUNBOOK.md` for the mechanical conflict-resolution steps once a merge is
unavoidable, and `CONFLICT_AVOIDANCE.md` for the sibling-branch case (both branches on the
same release).
