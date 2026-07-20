# PyTorch Fork Workflow Skill

这是一个用于 Codex 的 PyTorch fork 工作流技能。它把本地 PyTorch 仓库、团队主仓和个人 fork 仓之间常用的同步、开发分支、提交和推送流程固化成可复用操作规范。

## 适用场景

当你在本地维护 PyTorch fork，并且需要遵守以下团队流程时使用：

- 从团队主仓拉取代码。
- 基于目标分支创建新的开发分支。
- 本地修改后提交到开发分支。
- 只把开发分支推送到个人 fork。
- 由个人 fork 向团队主仓目标分支创建 MR/PR。
- 不向团队主仓直接 push 新功能分支。

默认约定：

```text
origin   个人 fork 仓库
upstream 团队主仓
```

默认 PyTorch 仓库路径：

```text
C:\Users\zhuyq\Desktop\file\code\fork\pytorch
```

## Skill 能力

### 1. 同步当前分支

适用于“当前分支在主仓有同名分支，需要让本地和 fork 跟主仓一致”的场景。

技能会指导 Codex：

- 检查工作区是否干净。
- 确认当前分支名。
- 拉取 `upstream` 和 `origin` 最新状态。
- 使用 `git merge --ff-only upstream/<branch>` 快进本地分支。
- 将同步后的分支推送到 `origin`。
- 使用 `git rev-list --left-right --count` 验证三方一致。

如果本地分支和主仓分叉，技能要求先停止并展示差异，不自动强推。

### 2. 同步所有分支

适用于“希望个人 fork 尽量跟团队主仓一致”的场景。

技能会指导 Codex：

- 比较 `upstream/*` 和 `origin/*` 的分支集合。
- 找出 fork 缺失的主仓分支。
- 找出 fork 与主仓 commit 不一致的分支。
- 找出只存在于 fork 的工作分支。
- 对缺失或落后的 fork 分支执行同步。
- 对分叉分支使用 `--force-with-lease`，避免覆盖别人刚推的新内容。
- 删除 fork-only 分支前必须明确确认。

默认不会在本地创建大量同名分支，只维护 remote-tracking refs。

### 3. 基于指定分支修改代码并推送 fork 工作分支

适用于“给定目标分支、patch、官方 commit、错误分析或修改要求，让 Codex 完成代码修改并推送到 fork”的场景。

技能会指导 Codex：

- 从 `upstream/<target-branch>` 创建新的开发分支。
- 分支名带上目标分支信息，例如：

```text
2.9.1-dev-aten-fix
2.12.0-dev-dynamo
2.13.0-dev-hipblaslt-groupmm
```

- 按仓库现有风格修改代码。
- 对官方 commit 只移植目标分支需要的最小改动，不盲目 cherry-pick。
- 做轻量校验，例如：

```text
git diff --check
python -m py_compile <changed-python-file>
pytest <target-test>
```

- 使用指定作者身份提交。
- 只推送到 `origin/<dev-branch>`。
- 最后给出 MR/PR 源分支和目标分支信息。

### 4. 保护打开中的 MR 分支

当一个 fork 分支已经创建 MR，但还需要修改提交时，技能会优先使用：

```text
git push --force-with-lease
```

这样可以更新自己的 MR 分支，同时避免无保护地覆盖远端新提交。

## 提交规范

默认提交作者：

```text
zhuyq <zhuyq@sugon.com>
```

提交信息格式采用 Conventional Commits 风格，并统一加 `[das]` 前缀以区别官方提交：

```text
[das] type(scope): subject
```

- `type`：建议使用 `feat` / `fix` / `perf` / `refactor` / `docs` / `test` / `build` / `ci` / `chore`。
- `scope`：模块名或主题，如 `aten`、`inductor`、`runtime`、`build`、`readme`、`license`。
- `subject`：小写、动词开头，尽量不超过 72 字符。
- 重要提交补 `Why` / `What` / `Impact` body；`fix` 必须写原因、影响范围和修复点；`perf` 必须写性能指标。
- 破坏性变更必须写 `BREAKING CHANGE:`。

示例：

```text
[das] feat(aten): add rmsnorm kernel
[das] fix(inductor): handle missing rocm combo kernel grid metadata
[das] perf(aten): optimize simd memory access in swap2d
[das] build(runtime): enable aicc via USE_AICC
```

完整的 type 逐项定义和改写示例见 `COMMIT_CONVENTION.md`。

## 仓库结构

```text
SKILL.md
COMMIT_CONVENTION.md
agents/
  openai.yaml
README.md
```

仓库根目录就是 Codex skill 根目录。`SKILL.md` 和 `agents/openai.yaml` 都放在根目录下，因此在其它环境中直接 clone 到 Codex skills 目录后即可被解析。

## 安装方式

推荐直接 clone 到用户级 Codex skills 目录：

```powershell
git clone https://github.com/Zyq-gg/pytorch-fork-workflow.git `
  "$env:USERPROFILE\.codex\skills\pytorch-fork-workflow"
```

也可以 clone 到项目级 `.codex/skills` 目录：

```powershell
git clone https://github.com/Zyq-gg/pytorch-fork-workflow.git `
  C:\path\to\project\.codex\skills\pytorch-fork-workflow
```

安装后重启 Codex，或让 Codex 重新加载 skills。

## 典型请求示例

```text
同步一下当前 PyTorch 分支到 fork。
```

```text
根据主仓同步一下 fork 仓库所有分支。
```

```text
把官方 commit <hash> 移植到 2.9.1-dev，按团队流程提交并推送到 fork。
```

```text
当前 MR 分支还没合并，把最后一个 commit amend 后 force-with-lease 更新 fork。
```

## GitHub 授权和提交身份

GitHub 授权和 Git 提交身份是两件不同的事：

- GitHub 授权决定这台机器能不能创建仓库、push 代码。
- Git 提交身份决定 GitHub 页面上 commit 显示成哪个用户名和头像。

多环境复用时，推荐先用 GitHub CLI 做一次设备登录：

```powershell
gh auth login --hostname github.com --web
gh auth status
```

如果没有安装 `gh`，先从 GitHub CLI 官方 release 或系统包管理器安装。

公开 GitHub skill 仓库建议使用仓库级身份配置，避免把公司邮箱身份带到公开仓库：

```powershell
git config user.name "Zyq-gg"
git config user.email "70554563+Zyq-gg@users.noreply.github.com"
```

如果已经用错误身份提交，可以 amend 后重新推送：

```powershell
git commit --amend --reset-author --no-edit
git push --force-with-lease
```

## 注意事项

这个 skill 带有明确的本地路径、远端命名、作者身份和团队提交规范。如果迁移到其它环境，需要先修改 `SKILL.md` 中的默认仓库路径、remote 名称、分支策略和提交身份。
