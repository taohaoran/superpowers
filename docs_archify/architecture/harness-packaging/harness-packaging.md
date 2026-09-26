# harness-packaging 域总览

> 本域回答：**同一份 superpowers 源码，如何被打包、声明、分发到 10+ 个编码助手平台？**
> 输出根：`docs_archify/architecture/`（项目根已有用户自有 `docs/`，故架构分析产物落在 `docs_archify/`）。
> 源码基准：superpowers main，commit `8ca22db`（v6.4.2）。

## 域职责

harness-packaging 是"一份内容，N 个宿主"的**适配与出包层**：上游是 skills-core 域产出的 15 个技能（`skills/`），本域不生产技能内容，只负责：

1. 为每个编码助手平台（Claude Code、Codex、Cursor、Devin、Kimi、Muse、Hermes、OpenCode、Pi、Gemini、通用 Agent 市场）写一份**插件清单/运行时入口**，告诉宿主"去哪加载 skills、会话开始挂什么钩子、如何展示插件"。
2. 把仓库内容**打成可分发产物**：面向 Codex 门户的 rootless 归档，以及同步到 OpenAI 官方插件仓的 PR。

本域是纯构建/适配层，零运行时依赖（打包脚本仅依赖 git/rsync/gh/jq 等系统工具）。

## 叶子索引

| 叶子 | 职责 | 主要产物 |
|---|---|---|
| [codex-plugin-packaging](codex-plugin-packaging/codex-plugin-packaging.md) | Codex 门户归档打包 + 同步到 OpenAI 官方插件仓 | `scripts/package-codex-plugin.sh`、`scripts/sync-to-codex-plugin.sh` |
| [plugin-manifests](plugin-manifests/plugin-manifests.md) | 10+ 平台的插件清单与 Hermes Python 引导 | `.claude-plugin/` 等 10+ 清单目录、`.hermes-plugin/__init__.py` |

## 域级机制

### 1. 双轨分发：归档 vs 同步

- **归档轨**（`package-codex-plugin.sh`）：`git archive` 白名单导出 → 播种 OpenAI 私有 `openai.yaml` → 确定性 zip/tar.gz → 上传 Codex 门户。
- **同步轨**（`sync-to-codex-plugin.sh`）：rsync 上游跟踪文件 → 保留目的地元数据 → 同步分支 → `gh pr create`。两轨都以 `.codex-plugin/plugin.json` 为版本与清单源头。

### 2. 清单矩阵：同源一份 skills

所有平台清单共同指向 `./skills/`，差异只在"宿主如何加载"：
- 多数平台（claude/codex/cursor/devin/kimi/muse）用 JSON 声明 `skills` 指针；
- Hermes 是唯一**带 Python 代码**的清单：`register(ctx)` 在首轮 LLM 调用注入 using-superpowers 引导文本；
- OpenCode/Pi 是 JS/TS 运行时插件（见 runtime-bootstrap 域），本域只登记其入口；
- Kimi/Muse 在清单里额外写了**工具映射长文**（把 Skill 里的 TodoWrite/Task 翻译成宿主原生工具）。

### 3. 版本同源

11 处 `version` 字段由 `scripts/bump-version.sh`（project-operations 域）按 `.version-bump.json` 集中维护，本域清单只读不改。

## 与相邻域的边界

- **上游 skills-core**：本域打包/清单指向的内容主体（15 个 SKILL.md），不在本域修改。
- **runtime-bootstrap**：OpenCode/Pi 的 JS/TS 运行时引导代码归属该域；本域只登记清单入口。
- **project-operations**：版本号 bump、发布说明、治理模板归属该域。
- **testing-evals**：`tests/codex/`、`tests/kimi/`、`tests/hermes/` 等对本域产物的测试背书归属该域。

## 图表清单

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 域内组件与边界 | `harness-packaging-architecture.html` | architecture | showcase |
| 一次 Codex 发布流程 | `harness-packaging-workflow.html` | workflow | showcase |
| skills 内容到多平台产物数据流 | `harness-packaging-dataflow.html` | dataflow | showcase |

- JSON IR：`json/harness-packaging-*.json`。
- 省略说明：本域不补 lifecycle——无单一实体状态机（归档/同步是一次性脚本流程，非常驻对象）。
