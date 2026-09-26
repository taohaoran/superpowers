# dispatching-parallel-agents（dispatching-parallel-agents）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览见 `../skills-core.md`，本文只展开
> "对 2+ 个独立问题域并行 dispatch 子代理"的方法论。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 并行判定 | 多失败且独立、可并行、无共享状态 → 并行 dispatch | `skills/dispatching-parallel-agents/SKILL.md:16-45` |
| 独立域分组 | 按"坏的是什么"分组：不同测试文件/子系统/bug | `SKILL.md:49-56` |
| 聚焦任务构造 | 每代理：特定范围 / 清晰目标 / 约束 / 期望输出 | `SKILL.md:58-64` |
| 同响应内多 dispatch | 一个响应里发多个 dispatch = 并行；一个响应一个 = 串行 | `SKILL.md:66-78` |
| 整合验证 | 读摘要→查冲突→跑全量测试→抽查 | `SKILL.md:80-85`、`161-167` |
| 反模式 | 太宽 / 无上下文 / 无约束 / 输出模糊 | `SKILL.md:115-128` |
| 不适用场景 | 相关失败 / 需全系统上下文 / 探索性调试 / 共享状态 | `SKILL.md:129-134` |

## 2. 核心类型与接口清单

| 结构 | 位置 | 职责 |
|---|---|---|
| Agent Prompt 四要素 | `SKILL.md:60-64` | 范围 / 目标 / 约束 / 输出 |
| 决策图（dot） | `SKILL.md:18-34` | 多失败？→ 独立？→ 可并行？的分支 |

## 3. 关键调用链

**链 1：并行 dispatch**

1. agent 命中 `SKILL.md:28-33`：判定多失败是否独立且可并行。
2. 命中 `SKILL.md:49-56`：按问题域分组。
3. 命中 `SKILL.md:66-78`：在同一响应内发 N 个 `general-purpose` 子代理 dispatch。
4. 命中 `SKILL.md:80-85`：全部返回后整合、跑全量测试。

## 4. 配置项

无外部配置。harness 提供子代理 dispatch 能力（不在本仓库源码内）。

## 5. 错误与重试语义

- 代理间冲突（`SKILL.md:83`）：检查是否改了同一代码，手动整合。
- 代理系统性错误（`SKILL.md:167`）：抽查。
- 无自动重试：失败则串行调查。

## 6. 并发细节

本技能本身不实现并发；它依赖 harness 的子代理并行执行。关键约束：

- `SKILL.md:70-77`：同响应多 dispatch = 并行；不同响应 = 串行。
- `SKILL.md:45`：共享状态（同文件）时不并行。

## 7. 系统边界

**In-Scope**：`skills/dispatching-parallel-agents/SKILL.md` 全部 167 行。
**Out-of-Scope**：子代理运行时（harness）、subagent-driven-development 的 per-task 串行模型。

## 8. 与相邻子系统交互

- **上游**：systematic-debugging 发现多独立失败时路由到本技能。
- **下游**：修复结果合入主会话；冲突时人工整合。

## 9. 语言专项适配口径

纯 Markdown 规范。图类型：architecture（协调者→N 个并行子代理）+ sequence（并行 dispatch 时序）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 并行 dispatch 组件拓扑 | `dispatching-parallel-agents-architecture.html` | architecture | 待渲染 |
| 并行 dispatch 时序 | `dispatching-parallel-agents-sequence.html` | sequence | 待渲染 |

- 未生成 workflow/dataflow/lifecycle：无对应语义结构。
