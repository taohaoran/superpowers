# finishing-a-development-branch（finishing-a-development-branch）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览见 `../skills-core.md`，本文只展开
> "实现完成后如何集成：验证测试→检测环境→给选项→执行→清理"的方法论。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Step 1 验证测试 | 跑全量测试；失败则停下报告 | `skills/finishing-a-development-branch/SKILL.md:14-26` |
| Step 2 环境检测 | 比较 GIT_DIR/GIT_COMMON，记录 WORKTREE_PATH；决定菜单与清理方式 | `SKILL.md:28-44` |
| Step 3 定 base 分支 | 从 plan/对话/upstream 推断；不确定则问 | `SKILL.md:46-51` |
| Step 4 给选项 | 普通/命名 worktree：3 选项（本地合并/推 PR/保留）；detached HEAD：2 选项 | `SKILL.md:53-82` |
| Step 5 执行选择 | 合并（先拉 base、合并、跑测试、删分支）/ 推 PR / 保留 | `SKILL.md:84-130` |
| Discard 显式请求 | 只有用户明确说 "discard" 才强制删分支；先列影响 | `SKILL.md:132-157` |
| Step 6 清理 worktree | 只清理 `.worktrees/` 或 `worktrees/` 下本技能创建的；删除被拒则问用户 | `SKILL.md:159-201` |
| 快速参考表 | 4 选项矩阵 | `SKILL.md:203-210` |

## 2. 核心类型与接口清单

| 结构 | 位置 | 职责 |
|---|---|---|
| 环境状态表 | `SKILL.md:40-44` | normal repo / named worktree / detached HEAD 三态决定菜单与清理 |
| 3 选项菜单 | `SKILL.md:57-65` | 本地合并 / 推 PR / 保留 |
| 2 选项菜单 | `SKILL.md:69-76` | detached HEAD：推为新分支 PR / 保留 |
| 清理规则 | `SKILL.md:167-201` | 只清理 `.worktrees/`、`worktrees/` 下的；删除被拒不 --force |

## 3. 关键调用链

**链 1：验证 → 菜单 → 执行**

1. agent 命中 `SKILL.md:16-26`：跑全量测试；失败停下。
2. 命中 `SKILL.md:30-36`：检测 GIT_DIR/GIT_COMMON/WORKTREE_PATH。
3. 命中 `SKILL.md:48-51`：确认 base 分支。
4. 命中 `SKILL.md:55-82`：按环境给菜单，等用户选。
5. 命中 `SKILL.md:86-130`：执行选择；合并路径先 `git checkout base && git pull && git merge`，跑测试，绿了再删分支。

**链 2：清理 worktree**

1. 命中 `SKILL.md:167-168`：normal repo 无 worktree 清理。
2. 命中 `SKILL.md:169-175`：WORKTREE_PATH 在 `.worktrees/` 或 `worktrees/` 下 → `git worktree remove` + `prune`。
3. 命中 `SKILL.md:177-198`：删除被拒（有未提交文件）→ 不 --force，列文件给用户三选一。
4. 命中 `SKILL.md:200-201`：其他位置的 worktree 属宿主环境，不动。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| Worktree 识别目录 | `.worktrees/` 或 `worktrees/` | `SKILL.md:169` |
| Discard 确认词 | 必须是字面 `discard` | `SKILL.md:143-146` |
| PR 创建 | 用 forge CLI 或 push 输出的 URL | `SKILL.md:121-124` |

## 5. 错误与重试语义

- **合并后测试失败**（`SKILL.md:102-104`）：停下，worktree 和分支保留，调查；不 push 所以可恢复。
- **push 被拒**（`SKILL.md:225`）：不 force-push；报告用户远端动了。
- **worktree 删除被拒**（`SKILL.md:177-198`）：不 --force；列未提交文件给用户三选一。
- **无自动重试**：集成决策由用户做。

## 6. 并发细节

无。单线程 git 命令序列。

## 7. 系统边界

**In-Scope**：`skills/finishing-a-development-branch/SKILL.md` 全部 225 行。
**Out-of-Scope**：forge 的 PR 创建 API；CI 检查；宿主环境的 worktree 管理。

## 8. 与相邻子系统交互

- **上游**：executing-plans / subagent-driven-development 在末尾调用本技能。
- **下游**：完成后回到主工作区；PR 反馈通过 worktree 继续迭代。

## 9. 语言专项适配口径

纯 Markdown + git 命令。图类型：architecture（环境→选项→执行）+ workflow（完成流程）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 分支收尾组件拓扑 | `finishing-a-development-branch-architecture.html` | architecture | 待渲染 |
| 完成与清理流程 | `finishing-a-development-branch-workflow.html` | workflow | 待渲染 |

- 未生成 sequence/dataflow/lifecycle：无对应语义结构。
