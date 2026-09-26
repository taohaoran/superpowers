# diagnosing-superpowers（诊断 Superpowers 会话问题）

> 本文是 `skills-core` 域下的叶子子系统文档。
> 本文只展开"如何对一次出问题的会话做取证与报告"，不重复展开其他技能。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 触发条件 | 会话出问题（重复劳动/忽略计划/技能没触发/太慢/太贵）或要给维护者报 bug | `skills/diagnosing-superpowers/SKILL.md:3` |
| 铁律 | 每条发现须 `path:line`；数字只来自转写或实跑命令 | `SKILL.md:15-18` |
| 七步工作流 | intake→定位→triage→报告→(issue)→(导出)→(相似会话) | `SKILL.md:21-70` |
| 问题 intake | 一次一个问，直到写清问题陈述 | `SKILL.md:23-28` |
| 会话定位 | 用 references 把会话解析成绝对路径并核对首条 prompt | `SKILL.md:29-35` |
| 并行七维分析 | 每维派一个子代理，无 path:line 的发现丢弃 | `SKILL.md:36-43` |
| 报告 | 按 templates/report.md 逐节填写并展示路径 | `SKILL.md:44-47` |
| GitHub issue | 搜重复、按模板起草、人批准才建 | `SKILL.md:48-54` |
| 脱敏导出 | 三档脱敏、scrub+scrub-audit 循环到 CLEAN | `SKILL.md:55-66` |
| 相似会话 | 把发现变签名、并行扫候选 | `SKILL.md:67-70` |
| 硬规则 | 上下文安全/只读/精确路径/人类话术/不诊断 superpowers/审批门 | `SKILL.md:86-109` |

## 2. 核心类型与接口清单

| 结构 | 位置 | 职责 |
|------|------|------|
| frontmatter `description` | `SKILL.md:3` | 触发条件与适用场景（当前/历史会话、任意 harness） |
| prompts/analyst-common.md + 7 维文件 | `skills/diagnosing-superpowers/prompts/` | 每个子代理的系统提示词（skill-timeline/plan-adherence/repeated-work/stumbles/quality-evidence/request-conflicts/cost-and-time） |
| prompts/scrub.md + scrub-audit.md | 同目录 | 脱敏与脱敏审计 |
| references/ | `references/session-discovery.md` 等 4 份 | 会话发现、上下文安全、issue 检索、脱敏策略 |
| templates/ | `case.md`/`report.md`/`issue.md`/`bundle-README.md` | 案例/报告/issue/打包模板 |

## 3. 关键调用链

1. **intake**（`SKILL.md:23-28`）：一次一问，直到能写下"会话+轮次区间+预期+实际+关心的指标"；人不在就把问题写下停下。
2. **定位**（`:29-35`）：按 `references/session-discovery.md` 解析绝对路径，用首条 prompt+时间戳核对，列出被拒候选及原因；建 `~/.superpowers/diagnosing-superpowers/<id>/` 填 `templates/case.md`。
3. **triage**（`:36-43`）：主编自己先读问题区段，再按维度并行派子代理（带 case 路径 + analyst-common + 对应维度文件）；无 `path:line` 的发现一律丢弃。
4. **报告**（`:44-47`）：按 `templates/report.md` 顺序填满、写盘、展示；核对所引内容真能支撑结论。
5. **条件步骤**：§7 判定可能/很可能 bug 时才走 issue 检索与起草（`:48-54`）；人要 bundle 才做脱敏导出并循环到 audit CLEAN（`:55-66`）；人要求才找相似会话（`:67-70`）。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|------|-----------|------|
| 工作目录 | `~/.superpowers/diagnosing-superpowers/<session-id>/` | `SKILL.md:33` |
| 脱敏档位 | skeleton/evidence/full 三档，按需选 | `SKILL.md:58-60` |
| 触发的维度 | 七个分析师始终全跑；只是先读/重点不同 | `SKILL.md:74-84` |

## 5. 错误与重试语义

- 子代理返回无 `path:line` 的发现：丢弃（`:43`）。
- 脱敏审计未达 CLEAN：重复 scrub+audit 直到 CLEAN（`:61`）。
- 归档前必须人看过 scrub log 与文件清单（审批门，`:101-103`）。
- 不修改/移动/删除任何会话文件（只读，`:90`）。

## 6. 并发细节

- triage 阶段按维度**并行**派子代理（`:37`）；长转写按轮次区间再拆分（`:42`）。
- 相似会话也是逐候选并行派发（`:69`）。
- 无共享可变状态；子代理各自读盘取证，主编汇总。

## 7. 系统边界

**In-Scope**：`skills/diagnosing-superpowers/` 全部文件（SKILL.md + prompts/11 + references/4 + templates/4）。
**Out-of-Scope**：磁盘上的会话转写文件（按平台存于用户主目录，**不在本仓库源码内**）；`gh` CLI、GitHub——**不在本仓库源码内**。本技能"只报告不诊断 superpowers 本身"（`:96-100`）。

## 8. 与相邻子系统交互

- 消费 `using-superpowers` 沉淀的会话数据；产物（报告/bundle）面向 superpowers 维护者。
- 与 `runtime-bootstrap` 的关系：本技能诊断的正是那些 bootstrap 注入后会话发生的问题。

## 9. 语言专项适配口径

纯 Markdown 技能包（SKILL.md + 提示词/参考/模板三类支撑文件）。图型：architecture（编排者 + 三类支撑目录 + 外部转写）+ sequence（人类→编排者→转写→分析子代理→报告 的取证时序）。无运行时代码。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 诊断架构图 | `diagnosing-superpowers-architecture.html` | architecture | **standard** |
| 诊断七步时序 | `diagnosing-superpowers-sequence.html` | sequence | showcase |

**架构图降档说明**：初版把 prompts/templates/refs 同列堆叠导致竖向边穿节点；改为左人类→中编排→右三目录横向走线后渲染成功（standard，HTML 800KB）。
省略 lifecycle/dataflow：七步是带条件分支的流程，时序图已表达主路径；条件步骤 5–7 见 MD 第 3 节文字。
