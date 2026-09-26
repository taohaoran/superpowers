# verification-before-completion（完成前验证）

> 本文是 `skills-core` 域下的叶子子系统文档。
> 本文只展开"证据先于声明"的完成门禁，不重复展开 TDD/调试技能。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 触发条件 | 即将宣布完成/修复/通过、提交或建 PR 前 | `skills/verification-before-completion/SKILL.md:3` |
| 铁律 | 无新鲜验证证据不得宣布完成 | `SKILL.md:16-21` |
| 门禁函数 | 识别命令→完整运行→读输出→核对→才声明 | `SKILL.md:24-36` |
| 常见失败表 | 各类"声明"所需的真凭实据 | `SKILL.md:40-48` |
| 红旗清单 | "should/probably/seems"等侥幸措辞 | `SKILL.md:50-59` |
| 借口反驳表 | 8 条常见跳过验证的借口 | `SKILL.md:63-72` |
| 关键范式 | 测试/回归/构建/需求/代理委派的正确姿势 | `SKILL.md:76-104` |
| 适用范围 | 字面、转述、同义、暗示成功的一切表达 | `SKILL.md:106-120` |

## 2. 核心类型与接口清单

| 结构 | 位置 | 职责 |
|------|------|------|
| frontmatter `description` | `SKILL.md:3` | 触发条件：宣布完成前，要求跑验证命令并读输出 |
| 门禁函数（5 步） | `SKILL.md:24-36` | 本技能的核心可执行规则 |

本叶子单文件、无附属脚本。

## 3. 关键调用链

1. **识别**：任何"我做完了/修好了/通过了"的念头出现时，先问"哪条命令能证明它？"（`SKILL.md:27`）。
2. **运行**：完整、新鲜地跑那条命令（而非引用上次结果）（`SKILL.md:28`）。
3. **读**：读完整输出、看退出码、数失败数（`SKILL.md:29`）。
4. **核对**：输出是否真的支持该声明？不支持就按实际状态陈述；支持才附证据声明（`SKILL.md:30-33`）。
5. **回归测试范式**：写测试→跑(过)→改坏→必须失败→恢复→跑(过)，才算"回归测试有效"（`SKILL.md:82-86`）。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|------|-----------|------|
| 验证命令 | 由声明内容决定（test/lint/build/复现症状） | `SKILL.md:40-48` |
| 时间窗 | 必须是"本条消息内"刚跑的，历史结果不算 | `SKILL.md:20` |

## 5. 错误与重试语义

- 输出与声明不符：撤回声明，按实际状态+证据陈述（`SKILL.md:31`）。
- 代理报告"成功"：不信，查 VCS diff 独立验证（`SKILL.md:47,69,101-104`）。
- 无代码级重试——本技能是声明门禁。

## 6. 并发细节

无。纪律上禁止"部分验证=全部通过"的外推（`SKILL.md:71`）。

## 7. 系统边界

**In-Scope**：`skills/verification-before-completion/SKILL.md`（单文件）。
**Out-of-Scope**：实际 test/lint/build 工具链、被验证的产品代码——**不在本仓库源码内**。

## 8. 与相邻子系统交互

- 被几乎所有流程技能在收尾时引用：`systematic-debugging` 阶段 4（`SKILL.md:189`）、TDD 验收清单（`test-driven-development/SKILL.md:293`）。
- 它是"最后一道闸"：在提交/PR/交接前拦截无证据声明。

## 9. 语言专项适配口径

纯 Markdown 单文件纪律文档。图型：architecture（触发→门禁→外部验证命令→人类）+ sequence（声明者↔验证命令↔人类的门禁往返）。无运行时代码。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 完成前验证架构图 | `verification-before-completion-architecture.html` | architecture | showcase |
| 完成前验证门禁时序 | `verification-before-completion-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/`。两张图均一次通过 showcase。
省略 lifecycle/dataflow：本叶子是单次声明门禁，无状态机/管道。
