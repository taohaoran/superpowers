# using-git-worktrees（using-git-worktrees）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览见 `../skills-core.md`，本文只展开
> "确保工作发生在隔离工作区"的方法论：先检测既有隔离，再用平台原生工具，最后才手动 git worktree。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Step 0 检测既有隔离 | 比较 `git rev-parse --git-dir` 与 `--git-common-dir`；子模块守卫 | `skills/using-git-worktrees/SKILL.md:16-45` |
| 原生工具优先 | EnterWorktree / WorktreeCreate / `/worktree` 命令优先，避免 phantom state | `SKILL.md:51-57` |
| Git worktree 回退 | 无原生工具时手动 `git worktree add` | `SKILL.md:59-100` |
| 目录选择优先级 | 用户声明 > `.worktrees/` > `worktrees/` > 默认 `.worktrees/` | `SKILL.md:64-77` |
| .gitignore 守卫 | 创建前 `git check-ignore`，未忽略则加入并提交 | `SKILL.md:78-88` |
| 沙箱回退 | 权限错误时在原目录工作 | `SKILL.md:100` |
| Step 2 项目 setup | 自动检测 npm/cargo/pip/poetry/go 并安装依赖 | `SKILL.md:102-119` |
| Step 3 基线测试 | 跑测试确认干净基线；失败则报告并问 | `SKILL.md:121-132` |
| 快速参考表 | 12 种情形的处置动作 | `SKILL.md:142-157` |

## 2. 核心类型与接口清单

本叶子是纯 Markdown 规范，无代码。关键检测命令：

| 命令 | 位置 | 职责 |
|---|---|---|
| `git rev-parse --git-dir` / `--git-common-dir` | `SKILL.md:21-22` | 判定是否已在 linked worktree |
| `git rev-parse --show-superproject-working-tree` | `SKILL.md:30` | 子模块守卫 |
| `git check-ignore -q .worktrees` | `SKILL.md:83` | 确认目录已被 git 忽略 |
| `git worktree add "$path" -b "$BRANCH"` | `SKILL.md:96` | 回退创建 |

## 3. 关键调用链

**链 1：检测 → 创建**

1. agent 命中 `SKILL.md:20-24`：取 GIT_DIR 与 GIT_COMMON。
2. 命中 `SKILL.md:26-32`：子模块守卫——`GIT_DIR != GIT_COMMON` 也可能是 submodule。
3. 命中 `SKILL.md:33-45`：已在 worktree 则跳到 Step 2；在普通 checkout 则征求用户同意。
4. 命中 `SKILL.md:51-57`：先找平台原生 worktree 工具，有则用。
5. 命中 `SKILL.md:61-98`：无原生工具则手动 `git worktree add`。

**链 2：项目 setup 与基线**

1. 命中 `SKILL.md:106-119`：按 package.json / Cargo.toml / requirements.txt / pyproject.toml / go.mod 选安装命令。
2. 命中 `SKILL.md:125-130`：跑测试；失败报告并问。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| Worktree 目录优先级 | 用户声明 > `.worktrees/` > `worktrees/` | `SKILL.md:65-77` |
| 默认目录 | `.worktrees/` | `SKILL.md:76` |
| 沙箱回退 | 权限错误时原地工作 | `SKILL.md:100` |

## 5. 错误与重试语义

- **权限错误**（`SKILL.md:100`）：告诉用户沙箱阻止，原地工作。
- **目录未忽略**（`SKILL.md:86-88`）：加入 .gitignore 并提交，再继续。
- **基线测试失败**（`SKILL.md:130`）：报告失败，问用户继续还是调查。

## 6. 并发细节

无。单线程 git 命令序列。

## 7. 系统边界

**In-Scope**：`skills/using-git-worktrees/SKILL.md` 全部 167 行。
**Out-of-Scope**：平台原生 worktree 工具实现（harness 提供）；git worktree 本身；依赖安装工具链。

## 8. 与相邻子系统交互

- **上游**：writing-plans / executing-plans / subagent-driven-development 在 setup 阶段调用本技能。
- **下游**：创建后进入开发流程；完成时 finishing-a-development-branch 清理。

## 9. 语言专项适配口径

纯 Markdown + git 命令。图类型：architecture（检测→创建分支）+ workflow（决策流程）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| worktree 决策组件拓扑 | `using-git-worktrees-architecture.html` | architecture | 待渲染 |
| 隔离工作区决策流程 | `using-git-worktrees-workflow.html` | workflow | 待渲染 |

- 未生成 sequence/dataflow/lifecycle：无对应语义结构。
