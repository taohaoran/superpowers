# executing-plans（executing-plans）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览见 `../skills-core.md`，本文只展开
> "在当前会话内联执行计划"的方法论；与 subagent-driven-development 共享 workspace/ledger 协议，
> 见对应叶子。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 内联执行模型 | 无 per-task 子代理，一个会话上下文跑完全部任务，末尾一次 fresh reviewer | `skills/executing-plans/SKILL.md:8-18` |
| 连续执行 | 任务间不停下来问"是否继续"；只有四类停止条件 | `SKILL.md:27-43` |
| Rulings 而非 stalls | 计划/规格冲突由 agent 裁决并写入 ledger，不中断会话 | `SKILL.md:32-37` |
| Setup | worktree 校验、ledger 恢复、读 plan+spec、pre-flight 接口冲突扫描 | `SKILL.md:108-162` |
| Task Loop | task-start → 按步骤 TDD → 完成契约 → task-done | `SKILL.md:164-232` |
| 完成契约 | 测试跑过、Expected 比对过、偏离有 Ruling | `SKILL.md:207-220` |
| Final Review | review-package 打包，dispatch 最强模型 reviewer；无 subagent 则自审 | `SKILL.md:234-289` |
| Rulings 汇总 | 最终消息列出所有 Ruling 与 deferred minors | `SKILL.md:291-304` |
| 辅助脚本 | `scripts/task-start`（brief+BASE）、`scripts/task-done`（跑测试+追加 ledger） | `scripts/task-start:1-28`、`scripts/task-done:1-52` |
| 12 条反合理化表 | 防止"我记得 Task N"、"测试应该通过"等借口 | `SKILL.md:306-321` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `task-start PLAN N` | `scripts/task-start:1-28` | 调用 SDD 的 task-brief 抽任务文本，打印 brief 路径与 BASE=`git rev-parse HEAD` |
| `task-done PLAN N BASE -- <test>` | `scripts/task-done` | 跑测试、保留输出到 workspace、通过才追加 `Task N: complete (...)` 到 ledger |
| ledger 行格式 | `SKILL.md:227-229` | `Task <N>: complete (commits <base7>..<head7>, tests: <cmd> → <result>)` |
| Ruling 行格式 | `SKILL.md:35` | `Ruling: <决定> — <原因> — <错了代价>` |
| 四类停止条件 | `SKILL.md:39-43` | 不可逆操作 / 安全敏感 / worktree 外副作用 / 计划全错 |

## 3. 关键调用链

**链 1：Setup → 恢复 ledger**

1. agent 命中 `SKILL.md:110-113`：用 using-git-worktrees 校验隔离工作区。
2. 命中 `SKILL.md:126-130`：调用 `../subagent-driven-development/scripts/sdd-workspace PLAN_FILE` 得到 `<repo>/.superpowers/sdd/<plan-basename>/`。
3. 命中 `SKILL.md:131-137`：读 `<workspace>/progress.md`，首行 plan 名匹配则跳过已 `Task N: complete` 的任务，从第一个未完成任务恢复。
4. 命中 `SKILL.md:149-152`：加载 test-driven-development 子技能。
5. 命中 `SKILL.md:154-162`：pre-flight 扫描任务间接口冲突，写 ledger。

**链 2：单个任务的 TDD 循环**

1. 命中 `SKILL.md:172-176`：跑 `scripts/task-start PLAN N`，读 brief。
2. 命中 `SKILL.md:184-205`：按 plan 步骤 RED→GREEN；每步 `Expected:` 与实际输出比对；不符则判"代码错"（systematic-debugging）或"计划错"（Ruling+ledger）。
3. 命中 `SKILL.md:207-220`：完成契约检查（verification-before-completion 子技能）。
4. 命中 `SKILL.md:224-232`：跑 `scripts/task-done PLAN N BASE -- <test>`，通过才追加 ledger 完成行。

**链 3：Final Review → 收尾**

1. 命中 `SKILL.md:236-251`：跑 `review-package PLAN MERGE_BASE HEAD`，dispatch code-reviewer.md（最强模型）。
2. 命中 `SKILL.md:260-289`：Critical/Important 一轮 fix（TDD 验证）；Minor 记入 deferred；不重派 re-review。
3. 命中 `SKILL.md:293-304`：汇总 Rulings 与 deferred minors，删 workspace，交接 finishing-a-development-branch。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| Workspace 目录 | `<repo>/.superpowers/sdd/<plan-basename>/` | `SKILL.md:126-130` |
| ledger 文件 | `<workspace>/progress.md` | `SKILL.md:131` |
| Spec 引用 | plan header 中的 Spec 字段；无 spec 则 Ruling 为临时 | `SKILL.md:143-147` |
| 测试命令 | 由 brief 指定，task-done 执行 | `SKILL.md:224-232` |

## 5. 错误与重试语义

- **计划与实际不符**（`SKILL.md:197-202`）：写 Ruling 行，继续；后续任务读 ledger 继承。
- **代码错**（`SKILL.md:195-196`）：转 systematic-debugging，不靠 patch symptom。
- **task-done 测试失败**（`SKILL.md:231-232`）：不写 ledger 完成行，任务不算完成。
- **Final review 发现 Critical/Important**（`SKILL.md:277-285`）：一轮 fix，每个 fix 走 RED→GREEN + 全量绿套件；不重派 reviewer。
- **ledger 丢失（git clean -fdx）**（`SKILL.md:140-141`）：从 `git log` 恢复。

## 6. 并发细节

本叶子无并发代码。它是单 agent 会话内联执行。关键"并发"问题是上下文压缩：

- `SKILL.md:115-119`：对话记忆不跨 compaction，ledger 是恢复地图。
- `SKILL.md:166-168`：长测试输出重定向到 workspace 文件，读 tail 即可，不污染上下文。
- Final reviewer 是唯一的 fresh-context dispatch（`SKILL.md:240-251`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `skills/executing-plans/SKILL.md` 全部 373 行
- `scripts/task-start`、`scripts/task-done`

**Out-of-Scope（不在本仓库源码内）**
- SDD 的 workspace/brief/review-package 脚本——subagent-driven-development 叶子
- code-reviewer.md 模板——requesting-code-review 叶子
- test-driven-development / systematic-debugging / verification-before-completion 子技能
- 用户项目的 plan/spec 文件

## 8. 与相邻子系统交互

- **上游**：writing-plans 在 `SKILL.md:203-204` 把 Native 执行路由到本技能。
- **下游**：
  - 调用 SDD 的 `scripts/sdd-workspace`、`scripts/task-brief`、`scripts/review-package`（共享 workspace 协议）
  - 调用 requesting-code-review 的 `code-reviewer.md` 做 final review
  - 交接 finishing-a-development-branch
- **横向**：与 subagent-driven-development 共享同一 ledger/workspace 格式，可中途切换执行器。

## 9. 语言专项适配口径

混合 Markdown 规范 + bash 脚本，按 TS/Node 口径：

- **Service Definition seam**：定义"内联执行计划"的契约（ledger 格式、完成契约、停止条件）。
- **Provider seam**：task-start/task-done 脚本提供 brief 抽取与 ledger 追加能力。
- **Consumer seam**：agent 会话是 consumer。
- 图类型：architecture（组件拓扑）+ workflow（任务循环流程）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 内联执行组件拓扑 | `executing-plans-architecture.html` | architecture | 待渲染 |
| 任务循环与收尾流程 | `executing-plans-workflow.html` | workflow | 待渲染 |

- JSON IR 源文件位于 `json/` 目录。
- 未生成 sequence 图：单线程内联执行无多参与方消息时序。
- 未生成 dataflow 图：ledger 流向已由 workflow 表达。
- 未生成 lifecycle 图：任务状态机（pending→done）过于简单。
