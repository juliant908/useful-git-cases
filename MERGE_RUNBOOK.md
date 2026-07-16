# Merge Runbook: resolving release-branch conflicts by hand

This runbook describes how to merge a feature branch (`FEATURE`) into its target
release branch (`release-vX.Y.Z`) and resolve the recurring conflicts, while keeping
your local developer files intact.

Throughout, substitute:

- `FEATURE` — the feature branch you're working on (e.g. `MSC-217412`).
- `release-vX.Y.Z` — the release branch it targets (e.g. `release-v1.3.0`).

The flow is: pull the release branch **into** the feature branch, resolve conflicts,
then push the feature branch so the PR can proceed.

## Protected files

These must stay as your local working-tree changes and must **never** be committed
into a merge:

- `.gitignore` (local tweaks, e.g. `@foundry/local`, `generator/`)
- `package.json` (root — local `build.local` scripts)
- `local.mk` (untracked)
- `generator/` (untracked directory)

---

## 1. Preserve your protected files

Back up the tracked ones outside the repo. `local.mk` and `generator/` are untracked,
so `git reset --hard` won't touch them, but backing up `local.mk` is cheap insurance:

```bash
mkdir -p /tmp/protected
cp .gitignore package.json local.mk /tmp/protected/
```

## 2. Get a clean, current base

```bash
git fetch origin
git checkout FEATURE

# Make sure you have no local-only commits you care about (should print 0):
git rev-list --count origin/FEATURE..FEATURE

# Reset to the published tip. This discards tracked working-tree edits
# (backed up above); untracked files like local.mk / generator/ survive.
git reset --hard origin/FEATURE
```

## 3. Merge the release branch

```bash
git merge origin/release-vX.Y.Z
```

If it completes with no conflicts, jump to step 6.

## 4. Resolve conflicts by category

See what conflicted:

```bash
git diff --name-only --diff-filter=U
```

### a) Stencil-generated files -> regenerate, never hand-edit

Files: `packages/core/docs/docs.json`, `packages/core/docs/vscode-data.json`,
`packages/core/docs/**/readme.md`, `packages/core/src/components.d.ts`.

`fc-demo.tsx` imports `docs.json`, so the build fails if it's missing or contains
conflict markers. Restore a valid copy first, then let the build overwrite it from the
merged source:

```bash
git checkout --theirs -- packages/core/docs/docs.json packages/core/src/components.d.ts
npm exec -w @foundry/core -- stencil build --docs
git add packages/core/docs/docs.json packages/core/src/components.d.ts
```

### b) Go test file (only on some release lines)

`packages/go/components_test.go` is generated from `docs.json` by the `generate:go`
script. Some release lines have it (e.g. v1.2.0), some don't (e.g. v1.3.0). If it
conflicts, regenerate it **after** the stencil build:

```bash
git checkout --theirs -- packages/go/components_test.go
npm run generate:go -w @foundry/core
git add packages/go/components_test.go
```

If `generate:go` doesn't exist on that branch, this file won't conflict — skip it.

### c) Version-only conflicts

`package-lock.json` and the sub-package `package.json` files
(`packages/core`, `packages/react`, `packages/angular/libs/core`) usually conflict only
on the version string (`1.1.2` vs a CI `1.1.2-beta.<n>`). Confirm nothing else differs,
then take the release side:

```bash
# Verify ONLY the version line differs:
git diff HEAD origin/release-vX.Y.Z -- packages/core/package.json

git checkout --theirs -- \
  package-lock.json \
  packages/core/package.json \
  packages/react/package.json \
  packages/angular/libs/core/package.json
git add \
  package-lock.json \
  packages/core/package.json \
  packages/react/package.json \
  packages/angular/libs/core/package.json
```

### d) Everything else

Real source conflicts: resolve by hand as normal, then `git add` them.

## 5. Verify the resolution

```bash
git diff --name-only --diff-filter=U           # should be empty
git grep -n '^<<<<<<<\|^>>>>>>>' -- .           # no conflict markers left
node -e "JSON.parse(require('fs').readFileSync('packages/core/docs/docs.json','utf8'))"  # valid JSON
git status --porcelain package.json .gitignore local.mk  # protected files not staged
```

Note: `stencil build --docs` may also update other generated files (e.g.
`packages/angular/libs/core/src/libs/components.ts`) if they were stale relative to the
source. Decide per case:

- `git add <file>` — stage it so generated output stays internally consistent (preferred
  when the change reflects the branch's real source, e.g. a renamed prop).
- `git checkout -- <file>` — discard it to keep the commit scope minimal.

## 6. Restore protected files (unstaged), then commit

Copy your local versions back into the working tree **without staging them**:

```bash
cp /tmp/protected/.gitignore /tmp/protected/package.json /tmp/protected/local.mk .
```

Commit **exactly like this** — the merge is already fully staged:

```bash
git commit -m "Merge branch 'release-vX.Y.Z' into FEATURE"
```

> Do NOT use `git add .` or `git commit -a`. Either would drag your `.gitignore`,
> `package.json`, `local.mk`, and `generator/` into the merge commit. Plain `git commit`
> records only the staged merge files and leaves your protected files as local
> working-tree changes.

## 7. Push

```bash
git push origin FEATURE
```

---

## Key gotchas

- **Never delete `docs.json` before building.** The build imports it and will fail with
  `Cannot find module '../../../../docs/docs.json'`. Always restore a valid copy
  (`--theirs` or `--ours`) first, then let `stencil build --docs` overwrite it.
- **`--theirs` = the release branch** being merged in; **`--ours` = your feature branch.**
  For version bumps you want `--theirs`.
- **`git reset --hard`** discards tracked working-tree edits (hence the backup in step 1)
  but leaves untracked files (`local.mk`, `generator/`) alone.
- **Prerequisites:** `node_modules` must be installed and the `@foundry/core` workspace
  must build, since conflict resolution relies on regenerating docs.
- **If the push is rejected or the PR still shows conflicts,** `release-vX.Y.Z` moved
  after your fetch — run `git fetch origin` and redo from step 2.
- **If commits hang**, it's likely GPG signing waiting for a passphrase
  (`git config --get commit.gpgsign` -> `true`). Run the commit in an interactive
  terminal so you can enter the passphrase.
