# PyTorch Fork Workflow Skill

Codex skill for working with a local PyTorch fork workflow. It captures the branch synchronization and fork-only development process used in this workspace.

## What It Covers

- Sync the current branch between the team upstream repository, local repository, and personal fork.
- Sync all upstream branches to the personal fork, including missing and diverged branches.
- Keep fork-only branches visible and avoid deleting them unless explicitly confirmed.
- Create development branches from an upstream target branch.
- Apply fixes, validate diffs, commit with the required identity, and push only to the personal fork.
- Use safe force-with-lease updates when amending fork branches that back an open MR.

## Repository Layout

```text
pytorch-fork-workflow/
  SKILL.md
  agents/
    openai.yaml
```

The actual Codex skill lives in `pytorch-fork-workflow/`. This root `README.md` is repository documentation only and is intentionally kept outside the skill folder.

## Installation

Copy the skill folder into your Codex skills directory:

```powershell
Copy-Item -Recurse -Force `
  .\pytorch-fork-workflow `
  "$env:USERPROFILE\.codex\skills\pytorch-fork-workflow"
```

Or keep it in a project-local `.codex/skills` directory:

```powershell
Copy-Item -Recurse -Force `
  .\pytorch-fork-workflow `
  C:\path\to\project\.codex\skills\pytorch-fork-workflow
```

Restart Codex or reload skills if needed.

## Expected PyTorch Remotes

The skill assumes this remote naming convention:

```text
origin   personal fork
upstream team main repository
```

The default local PyTorch repository path documented by the skill is:

```text
C:\Users\zhuyq\Desktop\file\code\fork\pytorch
```

## Commit And Branch Conventions

Default commit identity:

```text
zhuyq <zhuyq@sugon.com>
```

Commit message format:

```text
[das-<module>] <short English description>
```

Development branch naming format:

```text
<target-branch>-<module-or-topic>
```

Example:

```text
2.9.1-dev-aten-fix
```

## Notes

This skill is intentionally workflow-specific. Before using it in another environment, update `SKILL.md` for that repository's path, remote names, branch policy, author identity, and commit message rules.
