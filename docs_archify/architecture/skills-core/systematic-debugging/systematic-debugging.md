# systematic-debugging（系统化调试）

> 本文是 `skills-core` 域下的叶子子系统文档。
> 本文只展开"根因先行的四阶段调试纪律"，不重复展开 TDD/验证技能。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 触发条件 | 遇到任何 bug/测试失败/异常行为、提修复前 | `skills/systematic-debugging/SKILL.md:3` |
| 铁律 | 未做根因调查不提修复 | `SKILL.md:16-21` |
| 阶段 1 根因调查 | 读错误、稳定复现、查改动、多组件埋点、回溯数据流 | `SKILL.md:48-118` |
| 阶段 2 模式分析 | 找可工作示例、对参考实现、列差异 | `SKILL.md:120-141` |
| 阶段 3 假设检验 | 单假设、最小改动、不叠加修复 | `SKILL.md:143-166` |
| 阶段 4 实现 | 先写复现失败测试、单点修复、验证 | `SKILL.md:168-212` |
| 架构质疑阈值 | 3 次修复失败则停止并质疑架构 | `SKILL.md:191-212` |
| 红旗与借口表 | 识别"再试一次"式赌博 | `SKILL.md:214-255` |
| 支撑技术 | root-cause-tracing / defense-in-depth / condition-based-waiting | `SKILL.md:277-284`；同目录三份 md |

## 2. 核心类型与接口清单

| 结构 | 位置 | 职责 |
|------|------|------|
| frontmatter `description` | `SKILL.md:3` | 触发条件：提任何修复之前 |
| root-cause-tracing.md | `skills/systematic-debugging/root-cause-tracing.md` | 沿调用栈回溯到坏值源头 |
| defense-in-depth.md | 同目录 | 根因找到后多层加校验 |
| condition-based-waiting.md(+.ts) | 同目录 | 用条件轮询替代任意超时 |
| test-pressure-*.md | 同目录 3 份 | 本技能自身的压力测试语料 |

## 3. 关键调用链

1. **阶段 1**：完整读堆栈（`SKILL.md:52-56`）→ 稳定复现（`:58-62`）→ 查 `git diff`/最近改动（`:64-68`）→ 多组件系统在每个边界埋日志定位断在哪一层（`:70-106`）→ 沿调用栈回溯坏值来源（`:108-118`）。
2. **阶段 2**：找相似可工作代码 → 完整读参考实现 → 列出每一处差异（`:124-136`）。
3. **阶段 3**：写下单条假设"我认为 X 是根因因为 Y"→ 做最小改动 → 不奏效则换假设，不在旧修复上叠加（`:147-160`）。
4. **阶段 4**：写复现失败测试（`:172-177`）→ 单点修复（`:179-183`）→ 验证；失败次数 ≥3 则停下质疑架构而非试第 4 次（`:191-212`）。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|------|-----------|------|
| 修复次数阈值 | 3 次失败 → 质疑架构 | `SKILL.md:194-195` |
| 复现要求 | 不可稳定复现则继续收集证据，不猜 | `SKILL.md:62` |
| 环境归因 | 95%"无root cause"实为调查不全 | `SKILL.md:275` |

## 5. 错误与重试语义

- 修复不奏效：<3 次回到阶段 1 带新信息重分析；≥3 次停止，转架构讨论（`SKILL.md:194-196`）。
- 找不到根因：若确属环境/时序/外部问题，记录调查、加重试/超时/监控（`SKILL.md:266-273`）。
- 无代码级重试——本技能是纪律，靠"单变量实验"控制赌博式重试。

## 6. 并发细节

无运行时并发。纪律上要求"一次只改一个变量"（`SKILL.md:153-155`），禁止同时改多处以隔离效果。

## 7. 系统边界

**In-Scope**：`skills/systematic-debugging/` 全部 md + 1 个 ts 示例 + 3 份 pressure 测试语料。
**Out-of-Scope**：被测系统本体、CI/构建/签名等外部环境——**不在本仓库源码内**。

## 8. 与相邻子系统交互

- 阶段 4 调用 `test-driven-development` 写复现测试（`SKILL.md:177`），完成前调用 `verification-before-completion`（`:189`）。
- 支撑技术文件被正文引用；`condition-based-waiting.ts` 是可改编示例。

## 9. 语言专项适配口径

纯 Markdown 技能内容。示例用 shell/python/ts 混合（多组件埋点示例为 bash，`:88-104`；`condition-based-waiting-example.ts` 为 TS）。图型：architecture（主文档 + 三份支撑技术）+ sequence（调试者↔错误现场↔测试 的四阶段往返）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 系统化调试架构图 | `systematic-debugging-architecture.html` | architecture | **standard** |
| 调试四阶段时序 | `systematic-debugging-sequence.html` | sequence | showcase |

**架构图降档说明**：3 份支撑技术文件在右列纵向堆叠、主文档向三者引出三条边，showcase 报 `clean-flow/endpoint-side-direction`（竖向边端点方向）；已把布局改为"左触发→中主文档→右列引用"横向走线，渲染成功（standard，HTML 797KB），未再硬撑 showcase。
省略 lifecycle：四阶段是线性流程，已由 sequence 表达，无单一实体状态机。
