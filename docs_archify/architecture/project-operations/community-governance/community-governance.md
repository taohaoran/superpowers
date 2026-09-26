# 社区治理与 PR 规范（community-governance）

> 本文是 `project-operations` 域下的叶子子系统文档。域级总览见 `../project-operations.md`。
> 本文只展开"项目如何通过文档约束贡献者行为"——94% PR 拒绝率的治理机制。
>
> 源码基准：superpowers main，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 贡献者指南 | AI agent 贡献前必读：PR 拒绝率、必须披露模型/插件、目标 dev 分支 | `AGENTS.md` |
| PR 模板 | 强制填写：提交者身份、问题、变更、是否核心适配、已有 PR、环境、评测、人工 review | `.github/PULL_REQUEST_TEMPLATE.md` |
| Issue 模板 | bug/feature/diagnosis/platform support 四类 | `.github/ISSUE_TEMPLATE/` |
| 行为准则 | Contributor Covenant | `CODE_OF_CONDUCT.md` |
| 赞助 | GitHub Sponsors 指向 obra | `.github/FUNDING.yml` |
| Issue 分流 | 关闭 blank issue，问题引导到 Discord | `.github/ISSUE_TEMPLATE/config.yml` |

## 2. 核心类型与接口清单

| 文件 | 作用 |
|---|---|
| `AGENTS.md` | 项目级规则（被 AGENTS.md 机制注入 agent 上下文）：零第三方依赖、不做技能"合规化"改写、批量/投机 PR 拒绝、新 harness 必须贴 "Let's make a react todo list" 会话 transcript |
| PR 模板 | 把"谁在提交、解决什么真实问题、是否适合核心、是否人工 review 过完整 diff"做成必填 checkbox |

## 3. 关键调用链

治理没有运行时调用链，是**文档强制流程**：贡献者开 PR → 必须完整填模板 → 维护者按模板逐条审 → 缺项/无人工 review 证据/投机改动即关。

## 4. 配置项

| 配置 | 值 | 位置 |
|---|---|---|
| 目标分支 | `dev`（不是 main） | PR 模板首段 |
| blank_issues_enabled | false | `config.yml` |
| 赞助渠道 | github: [obra] | `FUNDING.yml` |

## 5. 错误与重试语义

治理"失败"= PR 被关；无重试机制。模板要求重复 PR 先搜 open+closed。

## 6. 并发细节

不适用（文档流程）。

## 7. 系统边界

**In-Scope**：`AGENTS.md`、`.github/`、`CODE_OF_CONDUCT.md`。

**Out-of-Scope**：GitHub 平台本身、Discord、CI（仓库无 `.github/workflows/`）。

## 8. 与相邻子系统交互

- 约束本仓库所有贡献，包括 harness-packaging/testing-evals/project-operations。
- 与 version-bump：PR 模板要求 target dev 分支，发布走 bump-version。

## 9. 语言专项适配口径

Markdown 治理文档。architecture 图表达治理文件与贡献流程的关系；workflow 图表达"开 PR→填模板→审查"流程。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 治理文件组成 | `community-governance-architecture.html` | architecture | showcase |
| PR 提交流程 | `community-governance-workflow.html` | workflow | showcase |

- JSON IR 源：`json/community-governance-*.json`。
