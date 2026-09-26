# using-superpowers（using-superpowers）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览见 `../skills-core.md`，本文只展开
> "会话启动时的技能路由与强制调用规则"，不重复展开 brainstorming / writing-plans 等被路由到的
> 具体技能（分别见各自叶子）。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 会话入口强制规则 | 规定"任何回复前必须先检查并调用技能"，含 1% 概率即必须调用的强硬约束 | `skills/using-superpowers/SKILL.md:10-16` |
| 子代理短路 | 被 dispatch 为子代理执行具体任务时，忽略本技能，避免子代理再触发技能路由 | `skills/using-superpowers/SKILL.md:6-8` |
| 技能优先级裁决 | 多技能同时命中时，流程技能（brainstorming / systematic-debugging）先于实现技能 | `skills/using-superpowers/SKILL.md:26-32` |
| 红旗清单（反合理化） | 11 条"觉得可以跳过技能"的念头及其反驳，防止 agent 自我合理化 | `skills/using-superpowers/SKILL.md:33-51` |
| 平台适配索引 | 列出各 harness 对应的 references/*-tools.md 文件路径 | `skills/using-superpowers/SKILL.md:52-61` |
| 用户指令优先级 | 用户指令（CLAUDE.md / AGENTS.md 等）> 技能 > 默认行为 | `skills/using-superpowers/SKILL.md:63-65` |
| 进入 plan mode 前钩子 | 进入 plan mode 前若未 brainstorm，先调用 brainstorming | `skills/using-superpowers/SKILL.md:22` |

## 2. 核心类型与接口清单

本叶子是纯 Markdown 行为规范（无代码），"核心类型"即规范中定义的规则实体：

| 规则实体 | 位置 | 职责 |
|---|---|---|
| `<SUBAGENT-STOP>` 块 | `SKILL.md:6-8` | 子代理短路开关 |
| `<EXTREMELY-IMPORTANT>` 块 | `SKILL.md:10-16` | 强制调用声明 |
| Skill Priority 规则 | `SKILL.md:26-32` | 流程技能 vs 实现技能的排序 |
| Red Flags 表 | `SKILL.md:33-51` | 反合理化检查表 |
| Platform Adaptation 列表 | `SKILL.md:52-61` | 6 个 harness 的 references 文件映射 |

## 3. 关键调用链

**链 1：用户消息到达 → 技能路由决策**

1. 用户消息进入会话，agent 首先读取 `skills/using-superpowers/SKILL.md:10-16` 的强制规则。
2. 命中 `SKILL.md:6-8` 的 `<SUBAGENT-STOP>`：若自身是被 dispatch 的子代理，立即忽略本技能，直接执行任务。
3. 命中 `SKILL.md:20`：在给出任何回复（含澄清问题、探索代码库）前，判断是否有技能适用。
4. 命中 `SKILL.md:22`：若即将进入 plan mode 且未 brainstorm，先调用 brainstorming。
5. 命中 `SKILL.md:24`：宣布 "Using [skill] to [purpose]"，按技能 checklist 建 todo。

**链 2：多技能命中时的优先级裁决**

1. 命中 `SKILL.md:28`：流程技能先于实现技能。
2. 例如 `SKILL.md:30-31`："Let's build X" → brainstorming 先；"Fix this bug" → systematic-debugging 先。
3. 实现技能（frontend-design 等）在流程技能设定方法后再执行。

**链 3：平台适配文件加载**

1. 命中 `SKILL.md:54-61`：按 harness 名称（Claude Code / Codex / Pi / Antigravity / Hermes / Muse）读取对应 `references/<harness>-tools.md`。
2. 事实记录：这些 `references/*-tools.md` 文件在 v6.4.2 当前仓库中**不存在**（已被 skills compression sweep 移除，见 facts.md §5）。

## 4. 配置项

本叶子无可执行配置项。行为由 Markdown 文本本身驱动。唯一的"配置"是用户指令优先级：

| 规则 | 默认行为 | 位置 |
|---|---|---|
| 用户指令 > 技能 > 默认行为 | 用户在 CLAUDE.md / AGENTS.md / GEMINI.md 中的明确指示可覆盖技能流程 | `SKILL.md:63-65` |
| 平台 references 文件 | 6 个 harness 各有对应文件（当前仓库不存在） | `SKILL.md:56-61` |

## 5. 错误与重试语义

本叶子是规范层，不执行命令，无重试逻辑。其"失败"表现为 agent 跳过技能调用，由 Red Flags 表（`SKILL.md:33-51`）作为运行时自查清单拦截：

- "This is just a simple question" → 反驳：Questions are tasks（`SKILL.md:39`）
- "I need more context first" → 反驳：Skill check comes BEFORE clarifying questions（`SKILL.md:40`）
- "The skill is overkill" → 反驳：Simple things become complex（`SKILL.md:47`）

## 6. 并发细节

本叶子无并发代码。它是单线程 agent 循环中的"每轮前置检查"。harness 的 hooks/session-start（见 runtime-bootstrap 域）会在会话启动时把本技能注入上下文，之后每轮用户消息都触发一次路由判断。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `skills/using-superpowers/SKILL.md` 全部 65 行规范文本
- Red Flags 表、Skill Priority 规则、Platform Adaptation 列表

**Out-of-Scope（不在本仓库源码内）**
- `references/claude-code-tools.md` / `references/codex-tools.md` 等 6 个平台适配文件——**当前仓库不存在**（已压缩移除）
- 各被路由到的技能本体（brainstorming、systematic-debugging 等）——见各自叶子
- harness 本身的子代理 dispatch 机制——不在本仓库源码内
- hooks/session-start 的注入实现——属 runtime-bootstrap 域

## 8. 与相邻子系统交互

- **上游**：harness 会话启动（runtime-bootstrap 域的 session-start-hook）把本技能注入上下文；之后每轮用户消息触发路由。
- **下游**：本技能是"路由器"，把会话分发到：
  - 流程技能：brainstorming、systematic-debugging、writing-plans、executing-plans、subagent-driven-development、using-git-worktrees、requesting-code-review、receiving-code-review、finishing-a-development-branch 等
  - 实现技能：frontend-design 等（本仓库外）
- **横向**：用户指令（CLAUDE.md / AGENTS.md）优先级高于本技能，可在 `SKILL.md:63-65` 覆盖其规则。

## 9. 语言专项适配口径

本叶子主语言为 Markdown（行为规范），按 TS/Node 口径中的"capability seam"视角归类：

- **Service Definition seam**：本叶子定义"技能何时被调用"的路由契约，是整个 skills/ 域的入口 seam。
- **Provider seam**：各具体技能（brainstorming 等）是 provider，本叶子不实现它们。
- **Consumer seam**：harness 的 agent loop 是 consumer，按本叶子规则消费技能。
- 图类型选择：architecture（组件拓扑：会话 → 路由器 → 流程/实现技能）+ workflow（路由决策流程：消息 → 子代理短路 → 技能命中 → 优先级裁决 → 宣布执行）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 技能路由拓扑图 | `using-superpowers-architecture.html` | architecture | showcase |
| 技能调用决策流程 | `using-superpowers-workflow.html` | workflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 未生成 sequence 图：本叶子无多参与方消息时序，路由是单线程 agent 内部决策，与 workflow 信息重复，按资源节省原则省略。
- 未生成 dataflow 图：本叶子不处理数据管道。
- 未生成 lifecycle 图：本叶子无单一实体状态机。
