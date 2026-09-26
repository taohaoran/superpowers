# receiving-code-review（receiving-code-review）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览见 `../skills-core.md`，本文只展开
> "收到评审反馈后如何技术核验而非表演性同意"的方法论。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 响应模式 | READ→UNDERSTAND→VERIFY→EVALUATE→RESPOND→IMPLEMENT 六步 | `skills/receiving-code-review/SKILL.md:14-25` |
| 禁止表演性同意 | 禁止 "You're absolutely right!" / "Great point!" / "Thanks" | `SKILL.md:27-48`、`131-148` |
| 不清晰项先问 | 部分理解=错误实现；先澄清再动手 | `SKILL.md:40-57` |
| 来源分级 | human partner 信任但仍问范围；外部 reviewer 先做 5 项技术核验 | `SKILL.md:59-86` |
| YAGNI 检查 | "implement properly" 类建议先 grep 实际使用 | `SKILL.md:88-98` |
| 实现顺序 | 阻塞问题→简单修→复杂修；逐项测试 | `SKILL.md:100-111` |
| 何时反驳 | 破坏现有功能 / reviewer 缺上下文 / 违反 YAGNI / 技术错误 / 遗留兼容 / 冲突架构决策 | `SKILL.md:113-129` |
| 反驳后认错 | 简洁事实陈述，不长篇道歉 | `SKILL.md:150-162` |
| GitHub 线程回复 | 用 gh api 在评论线程内回复，不发顶层 PR 评论 | `SKILL.md:203-205` |

## 2. 核心类型与接口清单

| 结构 | 位置 | 职责 |
|---|---|---|
| 响应模式六步 | `SKILL.md:16-25` | 收到反馈的固定处理序列 |
| 外部 reviewer 5 项核验 | `SKILL.md:69-75` | 技术正确？破坏功能？为何现状？跨平台？reviewer 全上下文？ |
| 实现顺序三级 | `SKILL.md:104-109` | Blocking → Simple → Complex |

## 3. 关键调用链

**链 1：收到外部 reviewer 反馈**

1. 命中 `SKILL.md:17-25`：READ 完整反馈不反应 → 用自己的话复述 → 对代码库核实。
2. 命中 `SKILL.md:69-75`：5 项技术核验。
3. 命中 `SKILL.md:76-84`：错了带技术理由反驳；不能验证就明说；与人类伙伴先前决策冲突则停下讨论。
4. 命中 `SKILL.md:100-111`：按 Blocking→Simple→Complex 顺序实现，每项单独测试。

**链 2：YAGNI 建议处置**

1. 命中 `SKILL.md:91-95`：reviewer 说 "implement properly" → grep 代码库实际调用。
2. 无调用 → 反问 "Remove it (YAGNI)?"；有调用 → 实现。

## 4. 配置项

无外部配置。行为由 Markdown 规范驱动。

## 5. 错误与重试语义

- 部分理解（`SKILL.md:42-48`）：停下澄清，不猜。
- 反驳后发现错了（`SKILL.md:152-160`）：简洁陈述 "You were right, I checked X..."，不长篇道歉。
- 不能验证（`SKILL.md:79-80`）：明说 "I can't verify this without X"，问方向。

## 6. 并发细节

无。单线程 agent 同步处理反馈。

## 7. 系统边界

**In-Scope**：`skills/receiving-code-review/SKILL.md` 全部 205 行。
**Out-of-Scope**：reviewer 如何产生反馈（requesting-code-review）；GitHub API 本身；SDD 的 fix loop。

## 8. 与相邻子系统交互

- **上游**：requesting-code-review 收到反馈后由本技能处置。
- **下游**：修复走 TDD；冲突走人类伙伴裁决。

## 9. 语言专项适配口径

纯 Markdown 规范。图类型：architecture（反馈源→核验→实现）+ workflow（六步响应流程）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 反馈处置组件拓扑 | `receiving-code-review-architecture.html` | architecture | 待渲染 |
| 六步响应流程 | `receiving-code-review-workflow.html` | workflow | 待渲染 |

- 未生成 sequence/dataflow/lifecycle：无对应语义结构。
