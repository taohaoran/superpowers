# requesting-code-review（requesting-code-review）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览见 `../skills-core.md`，本文只展开
> "dispatch 一个 reviewer 子代理按严重度报告问题"的方法论；接收反馈的一侧见 receiving-code-review。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 评审时机 | 强制：SDD 每任务后 / 大特性完成后 / 合并前；可选：卡住、重构前、修复杂 bug 后 | `skills/requesting-code-review/SKILL.md:12-22` |
| 精确上下文构造 | reviewer 只拿到 DESCRIPTION / PLAN_OR_REQUIREMENTS / BASE_SHA / HEAD_SHA，不拿会话历史 | `SKILL.md:24-41` |
| code-reviewer.md 模板 | `general-purpose` 子代理按模板评审 | `code-reviewer.md`、`SKILL.md:34` |
| 严重度处置 | Critical 立即修 / Important 继续前修 / Minor 记录后修 / 错了带理由反驳 | `SKILL.md:42-46` |
| 反合理化 | "自己看 diff 就行"、"reviewer 需要我会话历史" 两条反驳 | `SKILL.md:75-81` |

## 2. 核心类型与接口清单

| 结构 | 位置 | 职责 |
|---|---|---|
| `{DESCRIPTION}` 占位符 | `SKILL.md:37` | 做了什么的简述 |
| `{PLAN_OR_REQUIREMENTS}` | `SKILL.md:38` | 应做什么（plan 路径或需求） |
| `{BASE_SHA}` / `{HEAD_SHA}` | `SKILL.md:39-40` | diff 范围；示例 `git rev-parse HEAD~1` 或 `git merge-base origin/main HEAD` |
| code-reviewer.md 模板 | `code-reviewer.md` | reviewer 的完整指令（Strengths/Issues/Assessment 输出格式） |

## 3. 关键调用链

**链 1：请求评审**

1. agent 命中 `SKILL.md:26-30`：取 `BASE_SHA` 与 `HEAD_SHA`。
2. 命中 `SKILL.md:32-41`：dispatch `general-purpose` 子代理，填 code-reviewer.md 模板。
3. reviewer 返回 Strengths / Issues（Important/Minor）/ Assessment。
4. 命中 `SKILL.md:42-46`：Critical 立即修、Important 继续前修、Minor 记录、错了反驳。

**链 2：与 SDD/executing-plans 的集成**

1. SDD 每任务后跑 `review-package` 打包 diff，dispatch task-reviewer-prompt.md（见 SDD 叶子）。
2. executing-plans 末尾用本技能的 code-reviewer.md dispatch 最强模型做 whole-branch review（`executing-plans/SKILL.md:242-251`）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| BASE_SHA 获取 | `git rev-parse HEAD~1` 或 `git merge-base origin/main HEAD` | `SKILL.md:28-29` |
| reviewer 类型 | `general-purpose` 子代理 | `SKILL.md:34` |

## 5. 错误与重试语义

- reviewer 与代码不符（`SKILL.md:46`）：带技术理由反驳，展示代码/测试。
- 无自动重试：评审是一次性新鲜视角，不通过则按严重度修复后再请求。

## 6. 并发细节

本叶子无并发代码。reviewer 是独立子代理，与主会话上下文隔离。

## 7. 系统边界

**In-Scope**：`skills/requesting-code-review/SKILL.md`、`code-reviewer.md`。
**Out-of-Scope**：reviewer 子代理运行时（harness 提供）、receiving-code-review 的反馈处置流程、SDD 的 task-reviewer 模板。

## 8. 与相邻子系统交互

- **上游**：subagent-driven-development（每任务后）、executing-plans（末尾）。
- **下游**：receiving-code-review（agent 收到反馈后的处置）。

## 9. 语言专项适配口径

纯 Markdown 规范。图类型：architecture（请求方→reviewer→反馈）+ sequence（一次评审请求时序）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 评审请求组件拓扑 | `requesting-code-review-architecture.html` | architecture | 待渲染 |
| 一次评审请求时序 | `requesting-code-review-sequence.html` | sequence | 待渲染 |

- 未生成 workflow/dataflow/lifecycle：无对应语义结构。
