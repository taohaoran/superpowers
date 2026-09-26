# 多平台插件清单（plugin-manifests）

> 本文是 `harness-packaging` 域下的叶子子系统文档。域级总览见 `../harness-packaging.md`。
> 本文只展开"同一个 superpowers 包如何为 10+ 个编码助手平台各自声明一份插件清单"；
> 面向 Codex 门户的归档打包见 `../codex-plugin-packaging/codex-plugin-packaging.md`，
> 版本号如何跨清单同步见 `../../project-operations/version-bump-and-release/version-bump-and-release.md`。
>
> 源码基准：superpowers main，commit `8ca22db`（v6.4.2）。

## 1. 功能清单

| 平台/产物 | 清单文件 | 形态 | 源码路径 |
|---|---|---|---|
| Claude Code | `.claude-plugin/plugin.json` + `marketplace.json` | JSON 插件清单 + 市场清单 | `.claude-plugin/plugin.json`、`.claude-plugin/marketplace.json` |
| Codex CLI/App | `.codex-plugin/plugin.json` | JSON，含完整 `interface`（图标/品牌色/默认提示词） | `.codex-plugin/plugin.json` |
| Cursor | `.cursor-plugin/plugin.json` | JSON，`skills` 与 `hooks` 指针 | `.cursor-plugin/plugin.json` |
| Devin CLI | `.devin-plugin/plugin.json` | JSON 精简清单 | `.devin-plugin/plugin.json` |
| Kimi Code | `.kimi-plugin/plugin.json` | JSON，含 `skillInstructions` 工具映射长文 | `.kimi-plugin/plugin.json` |
| Muse | `.muse-plugin/plugin.json` + `marketplace.json` | JSON，显式枚举 15 个 skills + SessionStart hook | `.muse-plugin/plugin.json` |
| Hermes Agent | `.hermes-plugin/plugin.yaml` + `__init__.py` | YAML 清单 + Python 运行时引导 | `.hermes-plugin/plugin.yaml`、`.hermes-plugin/__init__.py` |
| 通用 Agent 市场 | `.agents/plugins/marketplace.json` | JSON 市场清单（url source） | `.agents/plugins/marketplace.json` |
| OpenCode | `.opencode/plugins/superpowers.js` + `INSTALL.md` | JS 运行时插件（非清单） | `.opencode/plugins/superpowers.js` |
| Pi | `.pi/extensions/superpowers.ts` | TS 扩展（非清单） | `.pi/extensions/superpowers.ts` |
| Gemini CLI | `gemini-extension.json` + `GEMINI.md` | 扩展清单，`contextFileName` 指向 GEMINI.md | `gemini-extension.json`、`GEMINI.md` |
| npm 包根 | `package.json` | ESM 包，`main` 指向 opencode 插件，`pi` 字段声明扩展与 skills | `package.json` |

> OpenCode/Pi 是**代码式运行时引导**（被 runtime-bootstrap 域覆盖），这里仅登记其清单入口；
> 本表其余为纯声明式清单。

## 2. 核心类型与接口清单

| 类型/结构 | 位置 | 职责 |
|---|---|---|
| `plugin.json`（各平台） | `.*/plugin.json` | 各平台插件元信息：name/version/author/license/keywords |
| `interface` 对象 | `.codex-plugin/plugin.json:24-46`、`.kimi-plugin/plugin.json:52-66` | 门户展示层：displayName/shortDescription/longDescription/capabilities/brandColor/icon |
| `marketplace.json` | `.claude-plugin/`、`.muse-plugin/`、`.agents/plugins/` | 市场索引：owner + plugins 数组（name/version/source） |
| `provides_hooks` | `.hermes-plugin/plugin.yaml` | 声明 Hermes 钩子点 `pre_llm_call` |
| `register(ctx)` | `.hermes-plugin/__init__.py:64-90` | Hermes 插件入口：注册全部 skills + 挂 `pre_llm_call` 钩子 |
| `_build_bootstrap()` | `.hermes-plugin/__init__.py:48-62` | 拼出 `<EXTREMELY_IMPORTANT>` 引导文本（using-superpowers 正文 + hermes-tools 映射） |
| `pi.extensions / pi.skills` | `package.json:14-18` | Pi 包发现扩展与 skills 的入口字段 |

## 3. 关键调用链

**链路一：Hermes 会话引导（唯一带代码的清单）**

1. Hermes 加载插件时调用 `register(ctx)`（`.hermes-plugin/__init__.py:64`）。
2. `_skills_dir()` 按两种安装布局候选 `../skills` 或 `./skills` 解析，找不到 `using-superpowers/SKILL.md` 就响亮抛错（`:14-46`）。
3. `_build_bootstrap()` 读 `using-superpowers/SKILL.md`，`_strip_frontmatter` 去掉 YAML frontmatter，再拼上 `references/hermes-tools.md` 工具映射（`:48-62`）。
4. 遍历 skills 目录，每个含 `SKILL.md` 的子目录都 `ctx.register_skill(name, Path(...))`——**必须传 `pathlib.Path`，传 str 会让 hermes 静默禁用整个插件**（`:71-77` 注释记录 2026-07-23 实测）。
5. 注册 `pre_llm_call` 钩子；仅当 `is_first_turn` 时返回 `{"context": bootstrap}` 注入首轮用户消息（`:79-90`）。

**链路二：版本号跨清单广播**

- 11 处版本位集中登记在 `.version-bump.json`（11 个文件的 `path + field`，见 version-bump-and-release 叶子）。
- `package.json`、`.hermes-plugin/plugin.yaml`、7 个 `plugin.json`、2 个 `marketplace.json` 的 `plugins.0.version`、`gemini-extension.json` 全部同源更新。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|---|---|---|
| `version` | 6.4.2（全部清单一致） | 各 manifest |
| `skills` 指针 | 多数为 `"./skills/"`；Muse 显式枚举 15 个 SKILL.md 路径 | `.cursor-plugin`、`.codex-plugin`、`.kimi-plugin`、`.muse-plugin` |
| `sessionStart.skill` | Kimi 指向 `using-superpowers` | `.kimi-plugin/plugin.json` |
| `hooks` | Cursor 指 `./hooks/hooks-cursor.json`；Muse 声明 SessionStart 调 `hooks/session-start`（5s 超时） | `.cursor-plugin`、`.muse-plugin` |
| `brandColor` | `#F59E0B`（琥珀色） | `.codex-plugin/plugin.json` |
| `policy.installation/authentication` | `.agents` 市场标 `AVAILABLE` / `ON_INSTALL` | `.agents/plugins/marketplace.json` |

## 5. 错误与重试语义

- 清单本身是声明式数据，无运行时重试。
- Hermes 是唯一有显式错误语义的：`_skills_dir()` 找不到 skills 树时 `raise RuntimeError`，而不是静默跳过——注释明确"静默跳过是坏安装伪装成好安装的方式"（`:42-46`）。
- `register_skill` 传 str 会 `AttributeError` 并被 hermes 静默禁用插件；`tests/hermes/conftest.py` 用 mock 复刻了这个"必须 Path"的契约，防止回归。

## 6. 并发细节

- 无并发。清单是静态文件；Hermes 钩子 `pre_llm_call` 同步返回 context 字符串，不涉及异步。
- 唯一"注册"是顺序遍历 `os.listdir(skills_dir)` 注册 skills。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 全部 `.claude-plugin/ .codex-plugin/ .cursor-plugin/ .devin-plugin/ .hermes-plugin/ .kimi-plugin/ .muse-plugin/ .agents/` 清单
- `.opencode/plugins/superpowers.js`、`.pi/extensions/superpowers.ts`（运行时入口，详见 runtime-bootstrap 域）
- `package.json`、`gemini-extension.json`、`GEMINI.md`

**Out-of-Scope（不在本仓库源码内）**
- 各平台宿主程序本身（Claude Code、Codex、Cursor、Kimi、Muse、Hermes、OpenCode、Pi、Gemini CLI 等的运行时）——外部产品
- `skills/*/references/hermes-tools.md` 等平台适配文件（facts.md 记录：部分 references 已在 v6.2.0 compression sweep 中移除，当前仓库不存在）
- 各平台官方市场/门户后端

## 8. 与相邻子系统交互

- 上游：`skills/`（15 个技能，skills-core 域）是所有清单共同指向的内容主体。
- 版本：`.version-bump.json` + `bump-version.sh`（version-bump-and-release）是所有清单 `version` 字段的唯一同步入口。
- 打包：Codex 清单被 `package-codex-plugin.sh` / `sync-to-codex-plugin.sh` 打入归档或同步到官方仓。
- 测试背书：`tests/kimi/test-plugin-manifest.sh`、`tests/codex/test-marketplace-manifest.sh`、`tests/hermes/`（testing-evals 域）。

## 9. 语言专项适配口径

主语言 TS/Node 口径下，本叶子是**清单矩阵**（capability seam = "同一份 skills 内容面向 N 个宿主的声明适配"）：
- architecture 图表达"一个 skills 内容源 → N 份宿主清单"的拓扑。
- dataflow 图表达"版本号 bump 一次 → 11 处清单字段同步"的数据广播。
- 不补 sequence：无多对象按时间序互发消息；不补 lifecycle：无单一实体状态机；不补 workflow：无分步审批流程。
- `.hermes-plugin/__init__.py` 是仓库内唯一 Python 代码，在 §3 单列其引导调用链。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 多平台清单拓扑 | `plugin-manifests-architecture.html` | architecture | showcase |
| 版本号跨清单广播 | `plugin-manifests-dataflow.html` | dataflow | showcase |

> dataflow 图曾因 writer 节点三条出边标签重叠（`composition/label-route-clearance`）失败，
> 修复动作：将 writer 节点移到中间行（row 1）并对上下两条出边加 `labelDy: ±28`，使三个标签分居高中低三路。

- JSON IR 源：`json/plugin-manifests-architecture.json`、`json/plugin-manifests-dataflow.json`。
- 省略说明：本叶子不补 sequence/workflow/lifecycle——清单矩阵是静态声明，无时序交互、无审批流、无状态机，按资源节省原则省略。
