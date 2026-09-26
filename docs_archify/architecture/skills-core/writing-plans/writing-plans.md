# writing-plans（writing-plans）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览见 `../skills-core.md`，本文只展开
> "把规格转成可执行的、2-5 分钟粒度的任务计划"的方法论；上游 brainstorming 见对应叶子，
> 下游 executing-plans / subagent-driven-development 见对应叶子。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 计划文档规范 | 强制 header：Goal / Architecture / Tech Stack / Spec / Global Constraints / Review Focus | `skills/writing-plans/SKILL.md:52-90` |
| 范围检查 | 多子系统规格应拆成多份计划，每份独立可测 | `SKILL.md:19-22` |
| 文件结构映射 | 先定文件职责，再拆任务；一文件一职责 | `SKILL.md:23-32` |
| 任务右 sizing | 任务 = 自带测试周期、值得独立 reviewer 闸门的最小单元 | `SKILL.md:34-42` |
| 步骤粒度 | 每步一个可验证动作（写失败测试 / 跑 / 实现 / 跑 / 提交） | `SKILL.md:44-50` |
| 任务结构模板 | Files / Interfaces（Consumes+Produces）/ Steps | `SKILL.md:92-138` |
| 步骤内容契约 | 步骤只承载"实现者不能独自决定的事"，不写代码转录 | `SKILL.md:140-161` |
| 自审清单 | Spec 覆盖 / 步骤扫描 / 类型一致 / Review Focus / 比例 | `SKILL.md:163-177` |
| 执行方式交接 | 用户选 Subagent-driven 或 Native，分别路由到对应技能 | `SKILL.md:179-204` |
| 计划落盘 | `docs/superpowers/plans/YYYY-MM-DD-<feature>.md` | `SKILL.md:16` |

## 2. 核心类型与接口清单

本叶子是纯 Markdown 规范，"核心类型"即计划文档的数据结构：

| 结构 | 位置 | 职责 |
|---|---|---|
| Plan Header | `SKILL.md:56-90` | 元信息 + Global Constraints + Review Focus |
| Task 块 | `SKILL.md:94-138` | Files / Interfaces / Steps |
| Interfaces.Consumes | `SKILL.md:103-105` | 声明本任务依赖前序任务的哪些签名 |
| Interfaces.Produces | `SKILL.md:105-107` | 声明后续任务依赖本任务的哪些签名 |
| Step（checkbox） | `SKILL.md:108-137` | RED-GREEN-COMMIT 序列 |
| Review Focus | `SKILL.md:78-88` | 规格暗示但测试未覆盖的 5 类输入/失败模式 |

## 3. 关键调用链

**链 1：从规格到计划文档**

1. agent 命中 `SKILL.md:12`：宣布 "I'm using the writing-plans skill..."。
2. 命中 `SKILL.md:19-22`：范围检查——多子系统则建议拆计划。
3. 命中 `SKILL.md:23-32`：先画文件结构（创建/修改/测试路径）。
4. 命中 `SKILL.md:34-42`：按"独立 reviewer 闸门"切任务边界。
5. 命中 `SKILL.md:94-138`：每任务按 Files / Interfaces / Steps 模板写。
6. 命中 `SKILL.md:163-177`：自审 5 项，就地修复。
7. 命中 `SKILL.md:179-204`：请用户审阅并选执行方式。

**链 2：执行方式路由**

1. 命中 `SKILL.md:187-194`：用户未指定执行方式时，列出 Subagent-driven 与 Native 两选项并给推荐。
2. 选 Subagent-driven → 命中 `SKILL.md:200-201`：REQUIRED SUB-SKILL = subagent-driven-development。
3. 选 Native → 命中 `SKILL.md:203-204`：REQUIRED SUB-SKILL = executing-plans。

**链 3：步骤粒度仲裁**

1. 命中 `SKILL.md:142-144`：步骤完成标准 = 实现者能从中写出"恰好一件合理的事"。
2. 命中 `SKILL.md:157-161`：计划是实现者不能独自决定的决策集合；计划比代码长就是写了代码。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| 计划输出路径 | `docs/superpowers/plans/YYYY-MM-DD-<feature>.md` | `SKILL.md:16` |
| 用户偏好覆盖 | 用户可指定其他计划位置 | `SKILL.md:17` |
| Spec 引用 | Header 中 Spec 字段指向规格文件，随计划一起传递 | `SKILL.md:67-69` |
| 执行方式 | Subagent-driven（默认推荐）/ Native | `SKILL.md:191-194` |

## 5. 错误与重试语义

本叶子无执行命令。其"失败"模式由自审清单拦截：

- **Spec 覆盖缺口**（`SKILL.md:167`）：规格某节找不到对应任务 → 补任务。
- **步骤无信息量**（`SKILL.md:169`）："handle edge cases" 类废话 → 重写或删除。
- **类型不一致**（`SKILL.md:171`）：Task 3 叫 `clearLayers()`、Task 7 叫 `clearFullLayers()` → 统一。
- **计划过长**（`SKILL.md:175`）：代码块占比过高 → 把函数体换成签名+测试断言。
- 修复后不重审，直接继续（`SKILL.md:177`）。

## 6. 并发细节

本叶子无并发代码。它在 agent 单线程会话中同步执行：读规格 → 写计划 → 自审 → 交接。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `skills/writing-plans/SKILL.md` 全部 204 行规范

**Out-of-Scope（不在本仓库源码内）**
- 计划文档的实际内容（用户项目的 `docs/superpowers/plans/`）
- 规格文档本身（brainstorming 产出）
- 执行计划的两个技能（executing-plans / subagent-driven-development）——见各自叶子
- 隔离工作区创建（using-git-worktrees）

## 8. 与相邻子系统交互

- **上游**：brainstorming 的 Architectural 路径在 `SKILL.md:138` 调用本技能；using-superpowers 路由。
- **下游**：本技能在 `SKILL.md:200-204` 把执行交接给：
  - subagent-driven-development（默认推荐）
  - executing-plans（Native 内联）
- **横向**：计划文档通过 `Spec:` 字段回引 brainstorming 产出的规格文件。

## 9. 语言专项适配口径

纯 Markdown 规范，按 TS/Node 口径：

- **Service Definition seam**：定义"计划文档长什么样"的契约。
- **Provider seam**：无代码 provider。
- **Consumer seam**：executing-plans / subagent-driven-development 按本技能定义的 header/task/step 格式消费计划。
- 图类型：architecture（计划文档组件拓扑）+ workflow（从规格到交接的流程）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 计划文档结构拓扑 | `writing-plans-architecture.html` | architecture | 待渲染 |
| 从规格到执行交接流程 | `writing-plans-workflow.html` | workflow | 待渲染 |

- JSON IR 源文件位于 `json/` 目录。
- 未生成 sequence 图：无多参与方消息时序。
- 未生成 dataflow 图：计划文档不是数据管道。
- 未生成 lifecycle 图：无单一实体状态机。
