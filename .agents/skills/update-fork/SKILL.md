---
name: update-fork
description: 将 myfanhua/turb-gpt-free-register 的 main 分支安全同步到当前 fork，并在验证后推送 origin。仅在用户要求更新、同步或重新 fork 当前仓库时使用。
---

# 更新 Fork

此流程同步 Git 分支内容，不会删除或重新创建 GitHub 上的 fork 关系。

## 固定目标

- 上游 remote：`upstream`，仓库为 `myfanhua/turb-gpt-free-register`。
- fork remote：`origin`，仓库为 `zhdgzs/turb-gpt-free-register`。
- 同步分支：`main`。
- 合并方向：`upstream/main` -> 本地 `main` -> `origin/main`。

不要改用 rebase、`reset --hard`、强制推送或全局 `ours`/`theirs` 合并策略。不要自动 stash、丢弃或覆盖用户改动。

## 执行流程

### 1. 预检

先执行只读检查：

```bash
git status --short --branch
git branch --show-current
git remote -v
```

必须同时满足：

- 当前分支是 `main`；
- 工作区和暂存区为空；
- 不存在进行中的 merge、rebase、cherry-pick 或 revert；
- `origin` 和 `upstream` 指向上述仓库。

任一条件不满足时停止并说明具体状态。不要自行切分支、清理文件、stash 或改写 remote。

### 2. 获取并比较远程状态

```bash
git fetch origin
git fetch upstream
git rev-list --left-right --count main...origin/main
git rev-list --left-right --count main...upstream/main
git log --oneline --decorate main..upstream/main
```

如果本地 `main` 领先或分叉于 `origin/main`，停止，列出差异提交，不要把本地提交顺带推送。仅当本地 `main` 等于 `origin/main`，或只是落后于 `origin/main` 时继续。

汇报将从 `origin/main` 快进的数量、将从 `upstream/main` 引入的提交，以及 upstream 与 main 是快进关系还是分叉关系。

### 3. 合并前确认

在任何会修改工作区的操作前，使用以下格式说明实际提交数、文件影响和预计采用的合并方式，并等待明确确认：

```text
⚠️ 危险操作检测！
操作类型：更新本地 main 并合并 upstream/main
影响范围：[提交数、文件范围、快进或 merge commit]
风险评估：将批量修改当前分支；分叉历史可能产生冲突

请确认是否继续？[需要明确的"是"、"确认"、"继续"]
```

确认后，如果本地 `main` 只落后于 `origin/main`，先执行：

```bash
git merge --ff-only origin/main
```

快进后重新计算 `main` 与 `upstream/main` 的祖先关系，然后选择一种方式：

- `upstream/main` 已包含在 `main` 中：不执行合并，也不推送。
- `main` 是 `upstream/main` 的祖先：执行 `git merge --ff-only upstream/main`。
- 两者已经分叉：执行 `git merge --no-ff --no-commit upstream/main`，让提交动作保持可审查。

### 4. 冲突处理

合并返回非零时，先确认是否存在 `MERGE_HEAD`，再列出冲突：

```bash
git status --short
git diff --name-only --diff-filter=U
git diff --cc
```

不要立即使用 `--ours`、`--theirs`，不要提交或推送。逐个冲突文件检查 `HEAD`、`upstream/main`、共同祖先及相关调用方，区分以下意图：

- upstream 的修复、接口变化和数据结构变化；
- 当前 fork 的定制功能与兼容性要求；
- 生成文件或锁文件，应先解决源文件，再用项目既有工具重新生成。

给出逐文件解决方案和行为影响。只有完全机械、不会改变语义的冲突才可建议直接处理；凡涉及功能取舍、删除/修改冲突、配置默认值、安全行为或不确定意图，必须询问用户。未获得确认前不编辑冲突文件。

用户确认解决方案后，编辑具体文件并仅暂存已解决文件。确认以下命令不再输出冲突文件：

```bash
git diff --name-only --diff-filter=U
git diff --check
git diff --cached --check
```

如果用户选择放弃合并，`git merge --abort` 会撤销合并现场；执行前仍需按危险操作格式说明影响并获得明确确认。

### 5. 验证与发布

合并内容完整后执行：

```bash
python -m pytest
git diff --check
git diff --cached --check
git diff --cached --stat
git status --short --branch
```

测试失败时停止，不提交、不推送，并报告失败原因。测试通过后展示提交摘要和差异统计。

分叉合并需要创建 merge commit；在执行 `git commit` 和 `git push` 前，再次用危险操作格式明确说明 commit 信息、目标 `origin/main` 和外部影响，并等待确认。确认后执行：

```bash
git commit -m "Merge upstream/main into main"
git push origin main
```

快进合并不创建 commit，但执行 `git push origin main` 前同样必须单独确认。推送被拒绝时重新 fetch 并停止分析，绝不强推。

最后验证并报告：

```bash
git status --short --branch
git rev-parse main origin/main upstream/main
git merge-base --is-ancestor upstream/main origin/main
```

报告合并方式、引入的提交、测试结果、推送结果以及三个分支的最终关系。
