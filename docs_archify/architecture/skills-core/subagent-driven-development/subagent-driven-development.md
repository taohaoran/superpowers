# subagent-driven-development（subagent-driven-development）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览见 `../skills-core.md`，本文只展开
> "每任务 dispatch 新鲜实现者 + 任务级 reviewer + 末尾全分支 review"的方法论，以及
> 三个无扩展名可执行 shell 脚本（sdd-workspace / task-brief / review-package）。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 模型分级 dispatch | 机械任务用便宜模型；集成用标准；评审/架构用最强；fix loop R4-5 升级一档 | `skills/subagent-driven-development/SKILL.md:184-220` |
| 任务循环 | dispatch implementer → 处理报告（DONE/CONCERNS/NEEDS_CONTEXT/BLOCKED）→ task review → fix loop → 完成 | `SKILL.md:246-443` |
| Fix loop 5 轮上限 | R1-3 恢复原 implementer；R4-5 换新鲜更强模型；R5 仍有问题则裁决 | `SKILL.md:373-429` |
| 整批同类小任务 | 同形状小编辑合并成一次 dispatch，不每任务一个子代理 | `SKILL.md:223-229` |
| 等待子代理 | 不短轮询不空等；本地有活就干，闲时 5-10 分钟分段等待 | `SKILL.md:235-245` |
| Workspace 隔离 | 每计划一个 `<repo>/.superpowers/sdd/<plan-basename>/` 目录，git-ignored | `scripts/sdd-workspace:1-82` |
| Plan 路径标记防串台 | workspace 内 `plan-path` 文件记录归属计划；冲突时用父目录名消歧 | `scripts/sdd-workspace:58-79` |
| Task brief 抽取 | awk 抽 plan 中 `## Task N` 块到独立文件，子代理只读 brief 不读全 plan | `scripts/task-brief:30-41` |
| Review package 打包 | git log + diff --stat + diff -U10 写到一文件；BASE 用 dispatch 前记录的 SHA，不用 HEAD~1 | `scripts/review-package:39-50` |
| 范围守卫 | BASE 必须是 HEAD 祖先、范围非空，否则退出 3 | `scripts/review-package:27-28` |
| 三个 prompt 模板 | implementer-prompt.md / task-reviewer-prompt.md / re-review-prompt.md | 同目录 |
| Final review | 全分支 review-package dispatch 最强模型；一轮 fix + 一次 scoped re-review | `SKILL.md:445-469` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `sdd-workspace PLAN_FILE` | `scripts/sdd-workspace:31-82` | 解析/创建计划 workspace 目录，打印绝对路径 |
| `task-brief PLAN_FILE N [OUT]` | `scripts/task-brief:10-43` | awk 抽 Task N 文本到 `task-N-brief.md` |
| `review-package PLAN BASE HEAD [OUT]` | `scripts/review-package:10-53` | 打包 diff 到 `review-<base7>..<head7>.diff` |
| 报告四态 | `SKILL.md:288-302` | DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED |
| Ledger 行格式 | `SKILL.md:437-439` | `Task <N>: complete (commits <base7>..<head7>, review clean)` |
| Fix round 行 | `SKILL.md:405-406` | `Task <N>: fix round <R>/5 (<X> addressed, <Y> open; commits a7..b7)` |
| Breaker 裁决 | `SKILL.md:411-429` | R5 仍有 open finding 时 controller 裁决：park / ruling |

## 3. 关键调用链

**链 1：Setup → Workspace 解析**

1. agent 命中 `SKILL.md:126-130`：用 using-git-worktrees 校验隔离工作区。
2. 命中 `SKILL.md:136-139`：跑 `bash scripts/sdd-workspace PLAN_FILE`。
3. `sdd-workspace`（`scripts/sdd-workspace:45-56`）：取 repo root，拼 `<root>/.superpowers/sdd/<slug>`。
4. 命中 `scripts/sdd-workspace:61-68` `owns()`：检查 `plan-path` 标记；不匹配则用父目录名消歧，再冲突加 `-2`、`-3`。
5. 命中 `scripts/sdd-workspace:81`：写自忽略 `.gitignore`（`*`），打印目录。

**链 2：单任务完整循环**

1. 命中 `SKILL.md:251-262`：跑 `task-brief PLAN N`，dispatch implementer（带 brief 路径、report 路径、上下文）。
2. implementer 返回四态之一（`SKILL.md:288-302`）：DONE → 进 review；BLOCKED → 换模型/拆任务/ruling。
3. 命中 `SKILL.md:290`：跑 `review-package PLAN BASE HEAD`，dispatch task-reviewer。
4. 命中 `SKILL.md:354-372`：若有 spec ❌ / Critical / Important，进入 fix loop。
5. 命中 `SKILL.md:373-394`：R1-3 恢复原 implementer；每轮 fix 后跑 `review-package PLAN FIX_BASE HEAD` 做 scoped re-review。
6. 命中 `SKILL.md:411-429`：R5 仍 open → controller 裁决：park（带 ruling）或 load-bearing 则 ruling 并传给下一任务。
7. 命中 `SKILL.md:431-443`：追加 ledger 完成行，标记 todo 完成，下一任务。

**链 3：Final review → 收尾**

1. 全部任务完成后，命中 `SKILL.md:447-456`：`review-package PLAN MERGE_BASE HEAD`，dispatch 最强模型 code-reviewer.md。
2. 命中 `SKILL.md:458-469`：有 finding → 一次 fix dispatch + 一次 scoped re-review；残余裁决。
3. 命中 `SKILL.md:482-487`：删 workspace，交接 finishing-a-development-branch。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| Workspace 根 | `<repo-root>/.superpowers/sdd/` | `scripts/sdd-workspace:46` |
| Plan 标记文件 | `<workspace>/plan-path` | `scripts/sdd-workspace:62-67` |
| Brief 命名 | `task-<N>-brief.md` | `scripts/task-brief:27` |
| Review 包命名 | `review-<base7>..<head7>.diff` | `scripts/review-package:36` |
| Ledger | `<workspace>/progress.md` | `SKILL.md:141` |
| Fix 轮上限 | 5 | `SKILL.md:373` |

## 5. 错误与重试语义

- **implementer BLOCKED**（`SKILL.md:296-302`）：上下文问题→补上下文重派；推理不够→更强模型；任务太大→拆分；plan 错→ruling 后重派。
- **Fix loop R5 仍 open**（`SKILL.md:411-429`）：breaker 触发，controller 裁决，不再自动重试。
- **Review 包范围错误**（`scripts/review-package:27-28`）：BASE 非 HEAD 祖先或空范围 → exit 3，拒绝生成误导性包。
- **Workspace 被 git clean 销毁**（`SKILL.md:153-154`）：从 `git log` 恢复。
- **EADDRINUSE 类不适用**：本技能不启服务。

## 6. 并发细节

- **无并行 implementer**（`SKILL.md:282`）：never dispatch multiple implementation subagents in parallel（冲突）。
- **等待子代理**（`SKILL.md:235-245`）：分段等待 5-10 分钟，期间做本地活（ledger、打包下一个 review），不靠短轮询。
- **上下文隔离**：每个子代理只拿 brief + report 路径 + 接口摘要，不继承会话历史（`SKILL.md:267-271`）。
- **Ledger 是恢复地图**（`SKILL.md:150-154`）：抗 compaction。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `skills/subagent-driven-development/SKILL.md` 全部 568 行
- `scripts/sdd-workspace`、`scripts/task-brief`、`scripts/review-package`
- `implementer-prompt.md`、`task-reviewer-prompt.md`、`re-review-prompt.md`

**Out-of-Scope（不在本仓库源码内）**
- 子代理运行时（harness 提供 dispatch 能力）
- code-reviewer.md 模板——requesting-code-review 叶子
- test-driven-development / systematic-debugging / verification-before-completion
- 用户项目的 plan/spec 文件

## 8. 与相邻子系统交互

- **上游**：writing-plans 在 `SKILL.md:200-201` 把 Subagent-driven 执行路由到本技能。
- **下游**：
  - 调用 using-git-worktrees 建隔离工作区
  - 调用 requesting-code-review 的 code-reviewer.md 做 final review
  - 交接 finishing-a-development-branch
- **横向**：与 executing-plans 共享 workspace/ledger 格式（`executing-plans/SKILL.md:121-123`），可中途切换执行器。

## 9. 语言专项适配口径

混合 Markdown 规范 + bash 脚本，按 TS/Node 口径：

- **Service Definition seam**：定义"每任务新鲜上下文 + fix loop"契约。
- **Provider seam**：三个 shell 脚本提供 workspace/brief/review 打包能力。
- **Consumer seam**：agent 会话（controller）与 implementer/task-reviewer 子代理。
- 图类型：architecture（组件拓扑）+ workflow（任务循环状态流）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| SDD 组件拓扑 | `subagent-driven-development-architecture.html` | architecture | 待渲染 |
| 任务循环与 fix loop 流程 | `subagent-driven-development-workflow.html` | workflow | 待渲染 |

- JSON IR 源文件位于 `json/` 目录。
- 未生成 sequence 图：任务循环已由 workflow 表达，多参与方消息与 workflow 信息重复。
- 未生成 dataflow 图：brief/review 包流向已由 architecture 表达。
- 未生成 lifecycle 图：fix loop 状态机虽存在，但 workflow 已覆盖其分支语义。
