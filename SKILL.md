---
name: pytorch-fork-workflow
description: Use when working in the local PyTorch fork workflow under this project, especially to synchronize the current branch or all branches among upstream main repository, the user's fork remote, and local Git refs, or to apply a fix to a specified target branch and push a development branch to the user's fork for MR/PR. Covers upstream/origin remotes, branch comparison, safe force-with-lease updates, pruning fork-only branches, branch naming, author identity, commit message format, validation, and fork-only push rules.
---

# PyTorch Fork Workflow

## Overview

Use this skill for the PyTorch repository fork workflow in this workspace. Treat `upstream` as the team main repository and `origin` as the user's personal fork.

Default repository path:

```powershell
C:\Users\zhuyq\Desktop\file\code\fork\pytorch
```

Default remotes:

```text
origin   user's fork
upstream team main repository
```

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

- `origin` points to the user's fork.
- `upstream` points to the team main repository.
- The branch to sync exists on `upstream`.

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

Use this when the user asks to make the fork remote match the main repository across all branches. In this workflow, "local" normally means the local remote-tracking refs (`upstream/*` and `origin/*`) plus the currently checked-out local branch; do not create hundreds of local branches unless the user explicitly asks.

This operation may:

- Update fork branches to match `upstream`.
- Force-update fork branches that diverged from `upstream`.
- Create fork branches missing from `origin`.
- Delete fork branches that do not exist on `upstream`.

Before deleting fork-only branches or force-updating diverged branches, report the planned changes and ask for confirmation unless the user has already clearly said the fork is only theirs and should be fully synchronized to `upstream`.

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

3. Synchronize missing and different fork branches to upstream:

```powershell
$toSync = @($upMap.Keys | Where-Object { (-not $orMap.ContainsKey($_)) -or ($orMap[$_] -ne $upMap[$_]) } | Sort-Object)

foreach ($b in $toSync) {
  if ($orMap.ContainsKey($b)) {
    git push --force-with-lease="refs/heads/$b`:$($orMap[$b])" origin "refs/remotes/upstream/$b`:refs/heads/$b"
  } else {
    git push origin "refs/remotes/upstream/$b`:refs/heads/$b"
  }
}
```

4. Delete fork branches that do not exist on upstream, only after confirmation:

```powershell
foreach ($b in $extra) {
  git push origin --delete $b
}
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

Expected verification:

```text
missing on origin: 0
different on origin: 0
extra on origin: 0
```

## Local Branches

Do not create local branches for every remote branch during all-branch sync. Local branch creation is only needed when the user wants to work on a specific branch:

```powershell
git checkout -b <local-branch> origin/<remote-branch>
```

If a local branch already exists and must be synchronized to upstream, use the current-branch workflow after checking it out.

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
zhuyq <zhuyq@sugon.com>
```

Use command-level identity for commits rather than changing global config:

```powershell
git -c user.name=zhuyq -c user.email=zhuyq@sugon.com commit --author="zhuyq <zhuyq@sugon.com>" -m "<message>"
```

Commit message format:

```text
[das-<module>] <short English description>
```

Examples:

```text
[das-dynamo] Update HF pretrained test for transformers v5
[das-aten] Fix matmul dtype fallback
[das-distributed] Avoid uneven allgather device mismatch
```

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
git -c user.name=zhuyq -c user.email=zhuyq@sugon.com commit --author="zhuyq <zhuyq@sugon.com>" -m "[das-<module>] <short English description>"
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
