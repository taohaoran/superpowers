# test-driven-development（测试驱动开发）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览由组织者产出。
> 本文只展开"TDD 纪律技能"这一流程文档的结构与行为约束，不重复展开其他技能。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 触发条件 | 实现任何功能/bugfix/重构前先调用 | `skills/test-driven-development/SKILL.md:3`（frontmatter description） |
| 铁律宣告 | "无失败测试不写生产代码" | `SKILL.md:33-45` |
| 红绿重构循环 | RED→验证红→GREEN→验证绿→REFACTOR | `SKILL.md:49-68`（dot 图）、`71-206` |
| 好测试准则 | 单一行为、清晰命名、真实代码 | `SKILL.md:208-220` |
| 借口反驳表 | 10 条常见借口逐条驳斥 | `SKILL.md:222-236` |
| 红旗清单 | 识别"想抄近路"的念头 | `SKILL.md:238-254` |
| 调试集成 | bug 先写复现失败测试 | `SKILL.md:317-321` |
| 验收清单 | 完成前 8 项核对 | `SKILL.md:293-306` |
| 好测试细则引用 | 链接到 writing-good-tests.md | `SKILL.md:216` |

## 2. 核心类型与接口清单

本叶子是 Markdown 流程文档，无运行时类型。核心"接口"是 frontmatter 的 `description`（触发条件）与 dot 状态图（循环规则）：

| 接口/结构 | 位置 | 职责 |
|-----------|------|------|
| frontmatter `description` | `SKILL.md:3` | "Use when implementing any feature or bugfix, before writing implementation code"——决定 agent 何时加载本技能 |
| `tdd_cycle` dot 图 | `SKILL.md:49-68` | RED/verify_red/GREEN/verify_green/REFACTOR 的状态与回边 |
| writing-good-tests.md | `skills/test-driven-development/writing-good-tests.md` | 命名生产变更、断言真实行为等细则 |

## 3. 关键调用链

1. **触发**：agent 准备写实现代码前，读到 description 触发条件 → 加载 SKILL.md。
2. **RED**：写一个最小失败测试（`SKILL.md:71-111`），运行 `npm test`（`SKILL.md:117-119`），确认"测试失败且失败原因是功能缺失"（`SKILL.md:121-128`）。
3. **GREEN**：写刚好通过测试的最小实现（`SKILL.md:130-166`），再跑测试确认本测试与全套都绿（`SKILL.md:168-193`）——明确要求跑项目全套而非单文件。
4. **REFACTOR**：绿之后才去重/改名/提函数，保持绿（`SKILL.md:195-202`），进入下一个失败测试。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|------|-----------|------|
| 测试命令 | 由仓库决定（`npm test`/`pytest`/`cargo test`） | `SKILL.md:187-189` |
| 例外（需问 partner） | 一次性原型、生成代码、配置文件 | `SKILL.md:24-28` |
| 全套验证 | 即使任务只点了一个测试文件，完成前仍跑项目测试命令 | `SKILL.md:185-193` |

## 5. 错误与重试语义

- 测试"一跑就过"：说明在测已有行为，要求改测试（`SKILL.md:126`）。
- 测试"报错而非失败"：先修错误，重跑到正确地失败（`SKILL.md:128`）。
- GREEN 后其他测试挂：立刻修，不带病前进（`SKILL.md:183`）。
- 本文件无代码级错误处理——它是行为约束文档，"失败"靠红旗清单让 agent 自我拦截。

## 6. 并发细节

无运行时并发。本技能规定的"并发"是纪律：不要在 GREEN 阶段同时改多处、不要 bundle 重构（`SKILL.md:166,181`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `skills/test-driven-development/SKILL.md` 与 `writing-good-tests.md`。

**Out-of-Scope（不在本仓库源码内）**
- 实际测试框架（Jest/pytest/cargo 等）、被测项目代码——**不在本仓库源码内**。
- 技能如何被加载/触发：见 `runtime-bootstrap` 域与 `using-superpowers` 叶子。

## 8. 与相邻子系统交互

- 被 `using-superpowers` 在流程中调度；`systematic-debugging` 第 4 阶段明确"用本技能写复现测试"（`systematic-debugging/SKILL.md:177`）；`writing-skills` 把本技能的 RED-GREEN-REFACTOR 映射到技能文档创作。
- 下游：agent 按本技能约束写产品代码。

## 9. 语言专项适配口径

本叶子是纯 Markdown 技能内容（仓库主语言 TS/Node，但本叶子无代码）。按 TS 口径：
- 无 capability seam、无运行时；"组件"即文档文件与外部测试命令。
- 图型：architecture（文件结构 + 外部测试运行器）+ sequence（红绿重构循环中开发者与测试运行器的消息往返）。
- 示例代码用 TypeScript（`SKILL.md:76-163`），与仓库主语言一致。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| TDD 架构图 | `test-driven-development-architecture.html` | architecture | showcase |
| TDD 红绿重构循环 | `test-driven-development-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/`。两张图均一次通过 showcase。
省略 lifecycle/dataflow/workflow：RED-GREEN-REFACTOR 本是状态机语义，但其循环已由时序图表达"开发者↔测试运行器"往返；状态迁移规则在 SKILL.md dot 图与第 3 节文字中完整给出，不重复出图。
