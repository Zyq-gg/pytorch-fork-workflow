---
name: pytorch-fork-workflow
description: Use when working in the local PyTorch fork workflow under this project, especially to synchronize the current branch or all branches among upstream main repository, the user's fork remote, and local Git refs, apply a fix to a specified target branch and push a development branch to the user's fork for MR/PR, or import commits from the official PyTorch repository. Covers upstream/origin/official remotes, branch comparison, safe force-with-lease updates, pruning fork-only branches, official release cherry-picks without merge commits, branch naming, author identity, commit message format, validation, and fork-only push rules.
---

# PyTorch Fork Workflow

## Overview

Use this skill for the PyTorch repository fork workflow in this workspace. Treat `upstream` as the team main repository, `origin` as the user's personal fork, and `official` as the public PyTorch GitHub repository.

Default repository path:

```powershell
C:\Users\zhuyq\Desktop\file\code\fork\pytorch
```

Default remotes:

```text
origin   user's fork: http://42.228.13.241:10068/zhuyq/pytorch.git
upstream team main repository: http://42.228.13.241:10068/dcutoolkit/deeplearing/pytorch.git
official public PyTorch repository: https://github.com/pytorch/pytorch.git
```

These URLs are the source of truth for this workflow. When initializing a new
local PyTorch repository, or when a remote is missing or incorrect, configure
the remotes as follows:

```powershell
git remote add origin http://42.228.13.241:10068/zhuyq/pytorch.git
git remote add upstream http://42.228.13.241:10068/dcutoolkit/deeplearing/pytorch.git
git remote add official https://github.com/pytorch/pytorch.git
```

If a named remote already exists, verify it with `git remote get-url <name>` and
use `git remote set-url <name> <url>` when it does not match. Only `origin` is a
push target; never push work branches to `upstream` or `official`.

In Codex managed sandboxes, run Git commands that fetch, push, update refs, or write lock files with escalated permissions.

## Preflight

Before any synchronization, inspect the repository and remotes:

```powershell
git status --short --branch
git remote -v
git branch --show-current
```

Stop and ask before syncing if the worktree has uncommitted changes, unless the user explicitly confirms those changes can be ignored, stashed, or committed.

Confirm that:

- `origin` points to `http://42.228.13.241:10068/zhuyq/pytorch.git`.
- `upstream` points to `http://42.228.13.241:10068/dcutoolkit/deeplearing/pytorch.git`.
- `official`, when present or needed, points to `https://github.com/pytorch/pytorch.git`.
- The branch to sync exists on `upstream`.

If official PyTorch commits or release branches are needed and `official` is missing, add it once:

```powershell
git remote add official https://github.com/pytorch/pytorch.git
git fetch official --prune
```

## Sync Current Branch

Use this when the user asks to sync the current branch among main repo, local repo, and fork.

1. Identify the current branch:

```powershell
$branch = git branch --show-current
```

2. Refresh both remotes:

```powershell
git fetch upstream --prune
git fetch origin --prune
```

3. Fast-forward the local current branch to the matching upstream branch:

```powershell
git merge --ff-only "upstream/$branch"
```

If this fails, do not force anything automatically. Explain that local history has diverged from `upstream/$branch`, then inspect with:

```powershell
git log --oneline --left-right --graph "upstream/$branch...$branch"
```

4. Push the synchronized branch to the user's fork:

```powershell
git push origin $branch
```

5. Refresh and verify:

```powershell
git fetch origin --prune
git rev-list --left-right --count "upstream/$branch...$branch"
git rev-list --left-right --count "origin/$branch...$branch"
git status --short --branch
```

Expected verification for both `rev-list` commands is:

```text
0 0
```

## Sync All Branches

Use this when the user asks to synchronize the main repository and fork repository across branches. By default, only same-name branches that exist on both `upstream` and `origin` are synchronized. In this workflow, "local" normally means the local remote-tracking refs (`upstream/*` and `origin/*`) plus the currently checked-out local branch; do not create hundreds of local branches unless the user explicitly asks.

This operation may:

- Update fork branches to match `upstream`.
- Force-update fork branches that diverged from `upstream`.
- Report upstream branches missing from `origin` without creating them.
- Report fork-only branches without deleting or otherwise changing them.

Only create a missing branch or delete a fork-only branch when the user explicitly names that operation or confirms it after the report. Before force-updating a diverged same-name branch, report the planned changes and ask for confirmation unless the user has clearly requested that synchronization.

1. Refresh both remotes:

```powershell
git fetch upstream --prune
git fetch origin --prune
```

2. Compare branch sets and commit IDs:

```powershell
$up = git for-each-ref --format='%(refname:strip=3) %(objectname)' refs/remotes/upstream | Where-Object { $_ -notmatch '^HEAD ' }
$or = git for-each-ref --format='%(refname:strip=3) %(objectname)' refs/remotes/origin | Where-Object { $_ -notmatch '^HEAD ' }

$upMap = @{}
foreach ($line in $up) {
  $parts = $line -split ' ',2
  $upMap[$parts[0]] = $parts[1]
}

$orMap = @{}
foreach ($line in $or) {
  $parts = $line -split ' ',2
  $orMap[$parts[0]] = $parts[1]
}

$missing = @($upMap.Keys | Where-Object { -not $orMap.ContainsKey($_) } | Sort-Object)
$different = @($upMap.Keys | Where-Object { $orMap.ContainsKey($_) -and $orMap[$_] -ne $upMap[$_] } | Sort-Object)
$extra = @($orMap.Keys | Where-Object { -not $upMap.ContainsKey($_) } | Sort-Object)

"upstream branches: $($upMap.Count)"
"origin branches: $($orMap.Count)"
"missing on origin: $($missing.Count)"
$missing
"different on origin: $($different.Count)"
$different
"extra on origin: $($extra.Count)"
$extra
```

3. Synchronize only same-name, different fork branches to upstream, after confirmation when required:

```powershell
$toSync = @($upMap.Keys | Where-Object { $orMap.ContainsKey($_) -and $orMap[$_] -ne $upMap[$_] } | Sort-Object)

foreach ($b in $toSync) {
  git push --force-with-lease="refs/heads/$b`:$($orMap[$b])" origin "refs/remotes/upstream/$b`:refs/heads/$b"
}
```

4. Do not create missing branches or delete fork-only branches during the default synchronization. If explicitly requested, handle them as separate, named operations:

```powershell
# Explicitly create a named missing branch only when requested:
# git push origin "refs/remotes/upstream/<branch>`:refs/heads/<branch>"

# Explicitly delete a named fork-only branch only when requested:
# git push origin --delete <branch>
```

5. Refresh and verify final equality:

```powershell
git fetch origin --prune

$up = git for-each-ref --format='%(refname:strip=3) %(objectname)' refs/remotes/upstream | Where-Object { $_ -notmatch '^HEAD ' }
$or = git for-each-ref --format='%(refname:strip=3) %(objectname)' refs/remotes/origin | Where-Object { $_ -notmatch '^HEAD ' }

$upMap = @{}
foreach ($line in $up) {
  $parts = $line -split ' ',2
  $upMap[$parts[0]] = $parts[1]
}

$orMap = @{}
foreach ($line in $or) {
  $parts = $line -split ' ',2
  $orMap[$parts[0]] = $parts[1]
}

$missing = @($upMap.Keys | Where-Object { -not $orMap.ContainsKey($_) } | Sort-Object)
$different = @($upMap.Keys | Where-Object { $orMap.ContainsKey($_) -and $orMap[$_] -ne $upMap[$_] } | Sort-Object)
$extra = @($orMap.Keys | Where-Object { -not $upMap.ContainsKey($_) } | Sort-Object)

"upstream branches: $($upMap.Count)"
"origin branches: $($orMap.Count)"
"missing on origin: $($missing.Count)"
"different on origin: $($different.Count)"
"extra on origin: $($extra.Count)"
git status --short --branch
```

Expected verification for synchronized same-name branches:

```text
difference on synchronized same-name branches: 0
```

The `missing on origin` and `extra on origin` counts are informational and may be nonzero. Always list them in the report and state that no action was taken unless the user explicitly requested branch creation or deletion.

## Local Branches

Do not create local branches for every remote branch during all-branch sync. Local branch creation is only needed when the user wants to work on a specific branch:

```powershell
git checkout -b <local-branch> origin/<remote-branch>
```

If a local branch already exists and must be synchronized to upstream, use the current-branch workflow after checking it out.

## Import Official PyTorch Commits

Use this when the user asks to bring commits from public PyTorch, for example from `official/release/2.13`, into a team development branch such as `upstream/2.13.0-dev` or a fork work branch based on it.

Remote roles:

- `official`: public PyTorch GitHub repository, used only as the source of upstream PyTorch commits.
- `upstream`: team main repository and MR/PR target.
- `origin`: user's fork repository and only push target for work branches.

Rules:

- Do not merge `official/release/*` into a team branch with a merge commit unless the user explicitly asks for a merge commit.
- Prefer cherry-picking the specific official commits needed, so GitLab/GitHub shows the original author and the local committer separately.
- Use `git cherry-pick -x <commit>` when importing official commits directly; this preserves the original commit message and appends the source commit hash.
- If the user wants a batch of official commits and there are no conflicts, cherry-pick the list/range in order instead of creating a merge commit.
- If a cherry-pick conflicts, inspect the target branch's current file layout and adapt the official change to the local code structure. Continue with `git cherry-pick --continue` when preserving the official commit still makes sense.
- If the target branch has diverged enough that the official commit cannot be meaningfully preserved, stop and explain the mismatch. Then create a local `[das] type(scope): subject` adaptation commit only if the user agrees or the requested workflow clearly allows an adapted backport.
- Push only the resulting work branch to `origin`. Never push official-import branches to `upstream`.

Recommended workflow:

1. Inspect repository state:

```powershell
git status --short --branch
git remote -v
```

2. Ensure `official` exists and fetch only what is needed:

```powershell
git remote get-url official
git fetch official --prune
```

If `official` is absent:

```powershell
git remote add official https://github.com/pytorch/pytorch.git
git fetch official --prune
```

3. Refresh team and fork remotes:

```powershell
git fetch upstream --prune
git fetch origin --prune
```

4. Create a fork work branch from the team target branch:

```powershell
git checkout -b <target-branch>-official-<topic> upstream/<target-branch>
```

Examples:

```text
2.13.0-dev-official-release213
2.13.0-dev-official-release213-picks
2.10.0-dev-official-fix-topk
```

5. Cherry-pick the official commits in chronological order:

```powershell
git cherry-pick -x <official-commit>
```

For multiple commits, list them explicitly in the intended order:

```powershell
git cherry-pick -x <oldest-commit> <next-commit> <newest-commit>
```

6. Validate:

```powershell
git diff --check
git log --oneline upstream/<target-branch>..HEAD
```

Run targeted tests when the imported change has clear test coverage.

7. Push only to the fork:

```powershell
git push -u origin <work-branch>
```

8. Report:

- Official source branch or commit range.
- Team target branch.
- Fork work branch.
- Whether commits were direct cherry-picks with original authors preserved or adapted `[das]` commits.
- Conflicts and resolution notes.
- Validation results.

## Apply Fix And Push Branch

Use this when the user provides bug analysis, target branch, expected code changes, or a reference upstream commit and asks to modify code, validate it, commit it, and push to the user's fork for MR/PR.

Team workflow rules:

- Do not push feature branches to `upstream`.
- Create a new local development branch from the target upstream branch.
- Commit local changes on the new branch.
- Push the new branch only to `origin`.
- The user creates the MR/PR from `origin/<dev-branch>` to `upstream/<target-branch>`.

Default commit identity:

```text
zhuyq <zyq_bravery@163.com>
```

Use command-level identity for commits rather than changing global config:

```powershell
git -c user.name=zhuyq -c user.email=zyq_bravery@163.com commit --author="zhuyq <zyq_bravery@163.com>" -m "<message>"
```

Commit message format follows Conventional Commits with a mandatory `[das]` prefix that keeps fork commits distinguishable from upstream ones:

```text
[das] type(scope): subject
```

- `type`: recommended set is `feat` / `fix` / `perf` / `refactor` / `docs` / `test` / `build` / `ci` / `chore`.
- `scope`: module or topic, e.g. `aten`, `inductor`, `runtime`, `build`, `readme`, `license`.
- `subject`: lowercase, verb-first, aim to keep under 72 characters.
- Add a `Why` / `What` / `Impact` body for significant commits. `fix` must state cause, affected scope, and the fix point; `perf` must include a metric (throughput, latency, benchmark before/after).
- Use a `BREAKING CHANGE:` body line for breaking changes.

Examples:

```text
[das] feat(aten): add rmsnorm kernel
[das] fix(inductor): handle missing rocm combo kernel grid metadata
[das] perf(aten): optimize simd memory access in swap2d
[das] build(runtime): enable aicc via USE_AICC
```

See `COMMIT_CONVENTION.md` for the full type-by-type definitions and rewrite examples.

Branch naming format:

```text
<target-branch>-<module-or-topic>
```

Examples:

```text
2.12.0-dev-dynamo
2.9.1-dev-aten
2.7.1-dev-distributed
```

Prefer a branch name that includes the original target branch prefix, so it remains obvious which upstream branch the MR/PR targets.

1. Inspect current state before switching:

```powershell
git status --short --branch
git remote -v
git branch --show-current
```

If there are uncommitted changes on the wrong branch, do not checkout another branch until those changes are either intentionally moved, stashed, or restored. If the changes are the exact requested patch but landed on the wrong branch, record the diff, restore the wrong branch to clean, then recreate the patch on the correct development branch.

2. Refresh remotes and confirm the target branch:

```powershell
git fetch upstream --prune
git fetch origin --prune
git rev-parse --short upstream/<target-branch>
```

3. Check that the development branch name is free locally and on the fork:

```powershell
git branch --list <dev-branch>
git ls-remote --heads origin <dev-branch>
```

If either command shows an existing branch, ask before reusing, deleting, or force-updating it.

4. Create the development branch from upstream:

```powershell
git checkout -b <dev-branch> upstream/<target-branch>
```

5. Apply the requested code changes using the repository's existing style. Keep edits scoped to the requested fix and nearby official compatibility updates when the user provides a reference upstream commit.

For patches based on an official upstream commit, copy only the relevant changes for the target branch. Do not blindly cherry-pick if the branch history or surrounding code differs; inspect the local file first and adapt the minimal patch.

6. Validate before committing:

```powershell
python -m py_compile <changed-python-file>
git diff --check
```

Run the targeted test when the local environment supports it:

```powershell
python -m pytest -q <test-file>::<TestClass>::<test_case>
```

If the target pytest cannot run because dependencies are missing, report the exact environment failure and keep the successful lightweight checks separate from the skipped full test.

7. Review the diff:

```powershell
git diff -- <changed-files>
git status --short --branch
```

8. Stage and commit:

```powershell
git add <changed-files>
git -c user.name=zhuyq -c user.email=zyq_bravery@163.com commit --author="zhuyq <zyq_bravery@163.com>" -m "[das] type(scope): subject"
```

9. Push only to the fork:

```powershell
git push -u origin <dev-branch>
```

Do not run `git push upstream <dev-branch>`.

10. Final verification and handoff:

```powershell
git status --short --branch
git log -1 --format="commit %H%nAuthor: %an <%ae>%nCommitter: %cn <%ce>%nSubject: %s"
git log --oneline upstream/<target-branch>..HEAD
```

Report:

- Target branch.
- Development branch pushed to `origin`.
- Commit hash and subject.
- Author/committer.
- Validation commands and results.
- MR/PR source branch and target branch.

## Extension Points

Keep future additions in this same skill:

- Creating MR/PR from fork branch to upstream target branch.
- Collaborating with a coworker's fork remote.
