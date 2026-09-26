# 项目文档（project-docs）

> 本文是 `project-operations` 域下的叶子子系统文档。域级总览见 `../project-operations.md`。
>
> 源码基准：superpowers main，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 项目主 README | 定位、15 平台安装命令 | `README.md`（14KB） |
| 内部计划/设计文档 | 按日期命名的 plans 与 specs | `docs/superpowers/plans/`、`docs/superpowers/specs/` |
| 早期 plans | opencode 支持、视觉头脑风暴等 | `docs/plans/` |
| 平台专属 README | kimi/opencode 安装说明 | `docs/README.kimi.md`、`docs/README.opencode.md` |
| 新 harness 移植指南 | 移植到新编码助手 | `docs/porting-to-a-new-harness.md` |
| Windows 钩子说明 | polyglot hooks | `docs/windows/polyglot-hooks.md` |
| 测试说明 | 如何跑测试 | `docs/testing.md` |
| 品牌资产 | 应用图标与小 logo | `assets/app-icon.png`、`assets/superpowers-small.svg` |

## 2. 核心类型与接口清单

无代码类型；文档组织即接口：`docs/superpowers/plans/<date>-<topic>.md` 与 `specs/<date>-<topic>-design.md` 配对。

## 3. 关键调用链

无运行时调用链。文档流：需求 → design spec → plan → 实现 → release notes。

## 4. 配置项

无运行配置。`docs/` 是用户自有文档目录，因此本架构分析产物落在 `docs_archify/`。

## 5. 错误与重试语义

不适用。

## 6. 并发细节

不适用。

## 7. 系统边界

**In-Scope**：`README.md`、`docs/`、`assets/`。

**Out-of-Scope**：`docs_archify/`（本分析产物，非项目文档）；外部托管文档站点。

## 8. 与相邻子系统交互

- plans/specs 记录了 harness-packaging（opencode/codex 兼容）、testing-evals（lift drill into evals）等历史决策。
- README 安装章节被 plugin-manifests 引用。

## 9. 语言专项适配口径

Markdown 文档树。architecture 图表达文档目录组成；dataflow 图表达"设计→计划→实现→发布"的文档流。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 项目文档组成 | `project-docs-architecture.html` | architecture | showcase |
| 设计到发布文档流 | `project-docs-dataflow.html` | dataflow | showcase |

- JSON IR 源：`json/project-docs-*.json`。
