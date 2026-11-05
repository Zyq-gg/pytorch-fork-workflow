---
name: pytorch-fork-workflow
description: "Use for the PyTorch fork workflow: synchronize branches, develop from the team repository, push a work branch to the user's GitHub fork for a PR, or import official PyTorch commits. Covers HYGON-AI/pytorch-das and Zyq-gg/pytorch-das remotes, legacy internal remotes, official PyTorch, branch comparison, safe force-with-lease updates, branch naming, commit conventions, and fork-only pushes."
---

# PyTorch Fork Workflow

## Overview

Use this skill for PyTorch source repositories, not for this skill's own Git repository. New development defaults to the GitHub team repository `hygon-public` and the user's GitHub fork `github-fork`; `official` remains the public PyTorch repository. The older internal `upstream`/`origin` pair remains available only when the user explicitly chooses the internal workflow. The commit convention in `COMMIT_CONVENTION.md` is unchanged.

Default repository path:

```powershell
C:\Users\zhuyq\Desktop\file\code\fork\pytorch
```

GitHub remotes for new development:

```text
hygon-public team repository: https://github.com/HYGON-AI/pytorch-das.git
github-fork  user's fork: https://github.com/Zyq-gg/pytorch-das.git
official public PyTorch repository: https://github.com/pytorch/pytorch.git
```

Legacy internal remotes, retained without changing their URLs:

```text
upstream team repository: http://42.228.13.241:10068/dcutoolkit/deeplearing/pytorch.git
origin   user's fork: http://42.228.13.241:10068/zhuyq/pytorch.git
```

In a PyTorch source checkout, inspect `git remote -v` before adding missing GitHub remotes. Do not repoint an existing internal remote or assume `origin` means the GitHub fork:

```powershell
git remote add hygon-public https://github.com/HYGON-AI/pytorch-das.git
git remote add github-fork https://github.com/Zyq-gg/pytorch-das.git
git remote add official https://github.com/pytorch/pytorch.git
```

Add only missing remotes. If a name already exists, verify both URL and intended role with `git remote get-url <name>`; do not silently overwrite it. For new work, fetch the target branch from `hygon-public`, push the development branch only to `github-fork`, and let the user open a PR from `Zyq-gg/pytorch-das:<dev-branch>` to `HYGON-AI/pytorch-das:<target-branch>`. Never push work branches to `hygon-public`, `upstream`, or `official`. Do not push new work to the legacy `origin` unless the user explicitly requests the internal workflow.

The `origin` of this workflow-skill repository is `https://github.com/Zyq-gg/pytorch-fork-workflow.git`; it is unrelated to the PyTorch code remotes above. Editing or publishing this skill never authorizes a push to either PyTorch repository.

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

- `hygon-public` points to `https://github.com/HYGON-AI/pytorch-das.git`.
- `github-fork` points to `https://github.com/Zyq-gg/pytorch-das.git`.
- `official`, when present or needed, points to `https://github.com/pytorch/pytorch.git`.
- The branch to sync exists on `hygon-public`. Do not assume the internal and GitHub repositories have identical branch sets or commit histories; verify the selected base before creating a work branch.

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
git fetch hygon-public --prune
git fetch github-fork --prune
```

3. Fast-forward the local current branch to the matching hygon-public branch:

```powershell
git merge --ff-only "hygon-public/$branch"
```

If this fails, do not force anything automatically. Explain that local history has diverged from `hygon-public/$branch`, then inspect with:

```powershell
git log --oneline --left-right --graph "hygon-public/$branch...$branch"
```

4. Push the synchronized branch to the user's fork:

```powershell
git push github-fork $branch
```

5. Refresh and verify:

```powershell
git fetch github-fork --prune
git rev-list --left-right --count "hygon-public/$branch...$branch"
git rev-list --left-right --count "github-fork/$branch...$branch"
git status --short --branch
```

Expected verification for both `rev-list` commands is:

```text
0 0
```

## Sync All Branches

Use this when the user asks to synchronize the main repository and fork repository across branches. By default, only same-name branches that exist on both `hygon-public` and `github-fork` are synchronized. In this workflow, "local" normally means the local remote-tracking refs (`hygon-public/*` and `github-fork/*`) plus the currently checked-out local branch; do not create hundreds of local branches unless the user explicitly asks.

This operation may:

- Update fork branches to match `hygon-public`.
- Force-update fork branches that diverged from `hygon-public`.
- Report hygon-public branches missing from `github-fork` without creating them.
- Report fork-only branches without deleting or otherwise changing them.

Only create a missing branch or delete a fork-only branch when the user explicitly names that operation or confirms it after the report. Before force-updating a diverged same-name branch, report the planned changes and ask for confirmation unless the user has clearly requested that synchronization.

1. Refresh both remotes:

```powershell
git fetch hygon-public --prune
git fetch github-fork --prune
```

2. Compare branch sets and commit IDs:

```powershell
$up = git for-each-ref --format='%(refname:strip=3) %(objectname)' refs/remotes/hygon-public | Where-Object { $_ -notmatch '^HEAD ' }
$or = git for-each-ref --format='%(refname:strip=3) %(objectname)' refs/remotes/github-fork | Where-Object { $_ -notmatch '^HEAD ' }

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

"hygon-public branches: $($upMap.Count)"
"github-fork branches: $($orMap.Count)"
"missing on github-fork: $($missing.Count)"
$missing
"different on github-fork: $($different.Count)"
$different
"extra on github-fork: $($extra.Count)"
$extra
```

3. Synchronize only same-name, different fork branches to hygon-public, after confirmation when required:

```powershell
$toSync = @($upMap.Keys | Where-Object { $orMap.ContainsKey($_) -and $orMap[$_] -ne $upMap[$_] } | Sort-Object)

foreach ($b in $toSync) {
  git push --force-with-lease="refs/heads/$b`:$($orMap[$b])" github-fork "refs/remotes/hygon-public/$b`:refs/heads/$b"
}
```

4. Do not create missing branches or delete fork-only branches during the default synchronization. If explicitly requested, handle them as separate, named operations:

```powershell
# Explicitly create a named missing branch only when requested:
# git push github-fork "refs/remotes/hygon-public/<branch>`:refs/heads/<branch>"

# Explicitly delete a named fork-only branch only when requested:
# git push github-fork --delete <branch>
```

5. Refresh and verify final equality:

```powershell
git fetch github-fork --prune

$up = git for-each-ref --format='%(refname:strip=3) %(objectname)' refs/remotes/hygon-public | Where-Object { $_ -notmatch '^HEAD ' }
$or = git for-each-ref --format='%(refname:strip=3) %(objectname)' refs/remotes/github-fork | Where-Object { $_ -notmatch '^HEAD ' }

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

"hygon-public branches: $($upMap.Count)"
"github-fork branches: $($orMap.Count)"
"missing on github-fork: $($missing.Count)"
"different on github-fork: $($different.Count)"
"extra on github-fork: $($extra.Count)"
git status --short --branch
```

Expected verification for synchronized same-name branches:

```text
difference on synchronized same-name branches: 0
```

The `missing on github-fork` and `extra on github-fork` counts are informational and may be nonzero. Always list them in the report and state that no action was taken unless the user explicitly requested branch creation or deletion.

## Local Branches

Do not create local branches for every remote branch during all-branch sync. Local branch creation is only needed when the user wants to work on a specific branch:

```powershell
git checkout -b <local-branch> github-fork/<remote-branch>
```

If a local branch already exists and must be synchronized to hygon-public, use the current-branch workflow after checking it out.

## Import Official PyTorch Commits

Use this when the user asks to bring commits from public PyTorch, for example from `official/release/2.13`, into a team development branch such as `hygon-public/2.13.0-dev` or a fork work branch based on it.

Remote roles:

- `official`: public PyTorch GitHub repository, used only as the source of official PyTorch commits.
- `hygon-public`: team main repository and MR/PR target.
- `github-fork`: user's fork repository and only push target for work branches.

Rules:

- Do not merge `official/release/*` into a team branch with a merge commit unless the user explicitly asks for a merge commit.
- Prefer cherry-picking the specific official commits needed, so GitHub shows the original author and the local committer separately.
- Use `git cherry-pick -x <commit>` when importing official commits directly; this preserves the original commit message and appends the source commit hash.
- If the user wants a batch of official commits and there are no conflicts, cherry-pick the list/range in order instead of creating a merge commit.
- If a cherry-pick conflicts, inspect the target branch's current file layout and adapt the official change to the local code structure. Continue with `git cherry-pick --continue` when preserving the official commit still makes sense.
- If the target branch has diverged enough that the official commit cannot be meaningfully preserved, stop and explain the mismatch. Then create a local `[das] type(scope): subject` adaptation commit only if the user agrees or the requested workflow clearly allows an adapted backport.
- Push only the resulting work branch to `github-fork`. Never push official-import branches to `hygon-public`.

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
git fetch hygon-public --prune
git fetch github-fork --prune
```

4. Create a fork work branch from the team target branch:

```powershell
git checkout -b <target-branch>-official-<topic> hygon-public/<target-branch>
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
git log --oneline hygon-public/<target-branch>..HEAD
```

Run targeted tests when the imported change has clear test coverage.

7. Push only to the fork:

```powershell
git push -u github-fork <work-branch>
```

8. Report:

- Official source branch or commit range.
- Team target branch.
- Fork work branch.
- Whether commits were direct cherry-picks with original authors preserved or adapted `[das]` commits.
- Conflicts and resolution notes.
- Validation results.

## Apply Fix And Push Branch

Use this when the user provides bug analysis, target branch, expected code changes, or a reference commit and asks to modify code, validate it, commit it, and push to the user's fork for a PR.

Team workflow rules:

- Do not push feature branches to `hygon-public`.
- Create a new local development branch from the target hygon-public branch.
- Commit local changes on the new branch.
- Push the new branch only to `github-fork`.
- The user creates the GitHub PR from `github-fork/<dev-branch>` to `hygon-public/<target-branch>`.

Default commit identity:

```text
zhuyq <zyq_bravery@163.com>
```

Use command-level identity for commits rather than changing global config:

```powershell
git -c user.name=zhuyq -c user.email=zyq_bravery@163.com commit --author="zhuyq <zyq_bravery@163.com>" -m "<message>"
```

Commit message format follows Conventional Commits with a mandatory `[das]` prefix that keeps fork commits distinguishable from hygon-public ones:

```text
[das] type(scope): subject
```

- `type`: recommended set is `feat` / `fix` / `perf` / `refactor` / `docs` / `test` / `build` / `ci` / `chore`.
- `scope`: module or topic, e.g. `aten`, `inductor`, `runtime`, `build`, `readme`, `license`.
- `subject`: lowercase, verb-first, aim to keep under 72 characters.
- Add a `Why` / `What` / `Impact` body for significant commits. `fix` must state cause, affected scope, and the fix point; `perf` must include a metric (throughput, latency, benchmark before/after).
- For locally authored commits, write each explanatory body paragraph in English and immediately follow it with an equivalent Chinese translation. Pair the `Why` / `What` / `Impact` sections individually; do not replace the English text with a Chinese-only summary. Preserve the original message when cherry-picking official commits.
- Do not add AI-assistance attribution, generation disclosures, or similar signatures to locally authored commit messages, MR/PR descriptions, or new repository documentation. Use the configured Git author and committer identities.
- Use a `BREAKING CHANGE:` body line for breaking changes.

For example, a body section should use `Why: ...` followed by `原因：...`, then `What: ...` followed by `修改：...`, and `Impact: ...` followed by `影响：...`.

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

Prefer a branch name that includes the original target branch prefix, so it remains obvious which hygon-public branch the MR/PR targets.

1. Inspect current state before switching:

```powershell
git status --short --branch
git remote -v
git branch --show-current
```

If there are uncommitted changes on the wrong branch, do not checkout another branch until those changes are either intentionally moved, stashed, or restored. If the changes are the exact requested patch but landed on the wrong branch, record the diff, restore the wrong branch to clean, then recreate the patch on the correct development branch.

2. Refresh remotes and confirm the target branch:

```powershell
git fetch hygon-public --prune
git fetch github-fork --prune
git rev-parse --short hygon-public/<target-branch>
```

3. Check that the development branch name is free locally and on the fork:

```powershell
git branch --list <dev-branch>
git ls-remote --heads github-fork <dev-branch>
```

If either command shows an existing branch, ask before reusing, deleting, or force-updating it.

4. Create the development branch from hygon-public:

```powershell
git checkout -b <dev-branch> hygon-public/<target-branch>
```

5. Apply the requested code changes using the repository's existing style. Keep edits scoped to the requested fix and nearby official compatibility updates when the user provides a reference official commit.

For patches based on an official PyTorch commit, copy only the relevant changes for the target branch. Do not blindly cherry-pick if the branch history or surrounding code differs; inspect the local file first and adapt the minimal patch.

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
git push -u github-fork <dev-branch>
```

Do not run `git push hygon-public <dev-branch>`.

10. Final verification and handoff:

```powershell
git status --short --branch
git log -1 --format="commit %H%nAuthor: %an <%ae>%nCommitter: %cn <%ce>%nSubject: %s"
git log --oneline hygon-public/<target-branch>..HEAD
```

Report:

- Target branch.
- Development branch pushed to `github-fork`.
- Commit hash and subject.
- Author/committer.
- Validation commands and results.
- MR/PR source branch and target branch.

## Extension Points

Keep future additions in this same skill:

- Creating MR/PR from fork branch to hygon-public target branch.
- Collaborating with a coworker's fork remote.
