# writing-skills（编写技能）

> 本文是 `skills-core` 域下的叶子子系统文档。
> 本文只展开"如何用 TDD 方法创作技能文档"及本叶子自带的图渲染工具，不重复展开其他技能。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 触发条件 | 新建/编辑/部署前验证技能 | `skills/writing-skills/SKILL.md:3` |
| 核心命题 | 写技能=把 TDD 用到流程文档上 | `SKILL.md:10-18` |
| TDD 映射表 | 测试用例→压力场景；生产代码→SKILL.md | `SKILL.md:32-45` |
| SKILL.md 结构规范 | frontmatter/overview/when-to-use/quick ref 等 | `SKILL.md:93-137` |
| 技能发现优化 SDO | description 只写"何时用"不写"做什么"、关键词、命名、token 预算 | `SKILL.md:140-276` |
| 形式匹配失败类型 | 禁令 vs 处方 vs 结构槽 vs 条件 | `SKILL.md:461-476` |
| 反合理化加固 | 堵漏洞表/红旗/借口表 | `SKILL.md:478-552` |
| 部署清单 | RED/GREEN/REFACTOR 三阶段逐项核对 | `SKILL.md:629-668` |
| dot→SVG 渲染工具 | 抽 SKILL.md 里 ```dot 块渲染成 SVG | `skills/writing-skills/render-graphs.js` |
| graphviz 样式约定 | 节点形状/边标签/命名规范 | `skills/writing-skills/graphviz-conventions.dot` |
| 配套参考 | anthropic 最佳实践、说服原理、子代理测试法 | `anthropic-best-practices.md`、`persuasion-principles.md`、`testing-skills-with-subagents.md` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `extractDotBlocks(markdown)` | `render-graphs.js:20-36` | 正则抽 ```dot 代码块并取 digraph 名 |
| `extractGraphBody(dot)` | `render-graphs.js:38-49` | 取 digraph 体、去掉重复 rankdir |
| `combineGraphs(blocks, name)` | `render-graphs.js:51-68` | 把多个图包进 cluster 合并成一张 |
| `renderToSvg(dot)` | `render-graphs.js:70-82` | `execFileSync('dot',['-Tsvg'])`，maxBuffer 10MB |
| `main()` | `render-graphs.js:84-167` | 解析 `--combine` 参数、检查 dot 存在、写 `diagrams/` 目录 |

## 3. 关键调用链

1. **写技能的 TDD 循环**（`SKILL.md:554-587`）：RED=无技能让子代理跑压力场景、记录违规借口原话 → GREEN=写最小技能针对这些借口 → REFACTOR=发现新借口再补、再验。
2. **微测试措辞**（`:577-587`）：正式压力场景前先做小样本（每变体 5+ 次、必含无指导对照），验证措辞是否真的绑定行为。
3. **render-graphs.js 流程**（`render-graphs.js:121-166`）：读 `SKILL.md` → `extractDotBlocks` → 逐个 `renderToSvg`（或 `--combine` 合并）→ 写 `<skill>/diagrams/*.svg`。
4. **dot 工具可用性探测**（`render-graphs.js:112-119`）：直接 `dot -V` 而非 `which`（Windows 无 which）；找不到则提示 brew/apt 安装并退出 1。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|------|-----------|------|
| `--combine` | 多图合并成一张带 cluster 的 SVG | `render-graphs.js:86,136-151` |
| dot 输入 | SKILL.md 内 ```dot 块 | `render-graphs.js:22` |
| 输出目录 | `<skill-dir>/diagrams/` | `render-graphs.js:131` |
| frontmatter 上限 | name+description ≤1024 字符 | `SKILL.md:97,640` |
| token 预算 | 入门流程 <150 词、常载 <200、其他 <500 | `SKILL.md:216-220` |

## 5. 错误与重试语义

- `render-graphs.js` 读不到 SKILL.md：打印错误并 `process.exit(1)`（`:105-108`）。
- `dot` 命令失败：`renderToSvg` 捕获、打印 stderr、返回 null，主循环只跳过该图不中断（`:77-81,155-162`）。
- graphviz 未安装：提前探测退出并给出安装提示（`:112-119`）。
- 写技能的纪律错误（无测试先写文档）：铁律要求"删掉重来"（`SKILL.md:376-395`）。

## 6. 并发细节

- render-graphs.js 是同步 CLI（`execFileSync`），单次渲染一张图，无并发。
- 写技能的"并发"是纪律：禁止批量不测试就连写多个技能（`SKILL.md:620-623`）。

## 7. 系统边界

**In-Scope**：`skills/writing-skills/`（SKILL.md + render-graphs.js + graphviz-conventions.dot + 3 份参考 md + examples/）。
**Out-of-Scope**：系统 graphviz `dot` 二进制（外部依赖，`render-graphs.js:13` 注释要求用户自装）；被创作的目标技能本身——**不在本仓库源码内**（是产物）。

## 8. 与相邻子系统交互

- 依赖 `test-driven-development` 的 RED-GREEN-REFACTOR 作为方法基础（`SKILL.md:18,395`）。
- `render-graphs.js` 被其他技能的 SKILL.md 引用以可视化自己的 dot 流程图（`SKILL.md:318-322`）。
- 产出的技能经 `runtime-bootstrap` 注册/加载。

## 9. 语言专项适配口径

本叶子混合 Markdown 技能 + Node ESM CLI 脚本（仓库主语言 TS/Node 口径）：
- **capability seam**：render-graphs.js 是独立工具脚本（Provider：对外提供 dot→SVG），其余是 Markdown 内容。
- **图型**：architecture（SKILL.md 主文档 + 渲染工具 + dot 约定 + 外部 dot 二进制）+ sequence（技能创作的红绿重构：作者↔压力子代理↔SKILL.md）。
- **外部边界**：graphviz dot 标 `external`；无第三方 npm 依赖（只用 node:fs/path/child_process 内建）。
- **部署维度**：脚本通过解释器调用（`node render-graphs.js`），文档提醒不要裸路径执行（插件打包会丢可执行位，`SKILL.md:374`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| writing-skills 架构图 | `writing-skills-architecture.html` | architecture | **standard** |
| 技能创作红绿重构时序 | `writing-skills-sequence.html` | sequence | showcase |

**架构图降档说明**：初版右列纵向堆叠导致竖向边端点方向报错；改为左触发→中主文档→右列（工具/约定/参考）横向走线后渲染成功（standard，HTML 798KB）。
省略 lifecycle/dataflow：创作循环已由 sequence 表达，无数据管道。
