# project-operations 域总览

> 本域回答：**项目版本如何发布、贡献者如何被治理、文档如何沉淀？**
> 输出根：`docs_archify/architecture/`。源码基准：superpowers main，commit `8ca22db`。

## 域职责

project-operations 是仓库的**运营面**：版本号同步与发布说明、社区治理规则、项目文档树。它不直接参与 agent 运行时，而是让项目可持续演进。

## 叶子索引

| 叶子 | 职责 | 主要产物 |
|---|---|---|
| [version-bump-and-release](version-bump-and-release/version-bump-and-release.md) | 11 处版本同步 + drift/audit | `scripts/bump-version.sh`、`.version-bump.json`、`RELEASE-NOTES.md` |
| [community-governance](community-governance/community-governance.md) | PR/Issue 治理与行为准则 | `AGENTS.md`、`.github/`、`CODE_OF_CONDUCT.md` |
| [project-docs](project-docs/project-docs.md) | README、plans/specs、移植指南、品牌资产 | `README.md`、`docs/`、`assets/` |

## 域级机制

- **版本单点**：`.version-bump.json` 是 11 处版本字段的唯一清单，bump 后自动 audit 回扫。
- **治理硬约束**：94% PR 拒绝率；PR 必须 target dev、披露模型与插件、人工 review 完整 diff；新 harness 必须贴 "Let's make a react todo list" 会话 transcript。
- **文档即决策记录**：`docs/superpowers/specs/` 与 `plans/` 按日期命名，release notes 带 PR 号。

## 图表清单

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 域内组件 | `project-operations-architecture.html` | architecture | showcase |
| 发布到治理流程 | `project-operations-workflow.html` | workflow | showcase |
| 版本号数据流 | `project-operations-dataflow.html` | dataflow | showcase |

- JSON IR：`json/project-operations-*.json`。
