# pi-extension（Pi 平台扩展）

> 本文是 `runtime-bootstrap` 域下的叶子子系统文档。域级总览见 `../runtime-bootstrap.md`。
> 本文只展开"Pi 宿主如何通过 TypeScript 扩展注册技能目录并在 context 事件里回插引导"，
> 不重复展开 shell 钩子（`session-start-hook`）与 OpenCode 插件（`opencode-bootstrap`）。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 技能目录发现 | `resources_discover` 事件返回 skills 目录供 Pi 原生技能系统加载 | `.pi/extensions/superpowers.ts:19-21` |
| 注入开关复位 | `session_start`/`session_compact` 时把 `injectBootstrap` 置 true | `superpowers.ts:23-29` |
| 注入开关关闭 | `agent_end` 时置 false（一轮 agent 结束后不再重复注入） | `superpowers.ts:31-33` |
| context 事件回插 | 在消息数组首个非 compactionSummary 位置插入引导 user 消息 | `superpowers.ts:35-56` |
| 引导正文懒加载缓存 | 首次读 SKILL.md 后缓存，失败缓存 null | `superpowers.ts:59-81` |
| frontmatter 剥离 | 正则去掉 YAML frontmatter 只留正文 | `superpowers.ts:83-86` |
| Pi 工具映射段 | 把 Claude 风格指令映射到 Pi 小写工具（read/write/edit/bash…） | `superpowers.ts:88-98` |
| 幂等检测 | 消息已含 bootstrap 标记则跳过 | `superpowers.ts:37,100-113` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `superpowersPiExtension(pi: ExtensionAPI)` | `superpowers.ts:16-57` | 默认导出扩展主函数，注册全部事件处理器 |
| `getBootstrapContent()` | `superpowers.ts:59-81` | 读+剥 frontmatter+包标记+拼 Pi 工具映射；模块级 `cachedBootstrap` 缓存 |
| `stripFrontmatter(content)` | `superpowers.ts:83-86` | 正则 `^---\n...\n---\n` 剥离 |
| `piToolMapping()` | `superpowers.ts:88-98` | 生成 Pi 专属工具映射段落（无 Skill 工具时用 read 加载 SKILL.md） |
| `messageContainsBootstrap(message)` | `superpowers.ts:100-113` | 递归检查消息文本是否已含 `BOOTSTRAP_MARKER` |
| `firstNonCompactionSummaryIndex(messages)` | `superpowers.ts:115-121` | 找到首个非 `compactionSummary` 消息的插入下标 |

## 3. 关键调用链

1. **启动发现**：Pi 派发 `resources_discover` → 扩展返回 `{ skillPaths: [skillsDir] }`（`superpowers.ts:19-21`），Pi 据此发现全部技能。
2. **会话复位**：`session_start` 或 `session_compact` 事件把 `injectBootstrap=true`（`superpowers.ts:23-29`）。
3. **每轮 context 注入**：Pi 派发 `context` 事件（`superpowers.ts:35`）→ 若 `injectBootstrap=false` 直接返回 → 若消息已含 bootstrap 标记则返回 → `getBootstrapContent()` 取缓存正文 → 在 `firstNonCompactionSummaryIndex` 处插入一条 `role:user` 的文本消息（`superpowers.ts:48-55`）。
4. **一轮结束**：`agent_end` 把 `injectBootstrap=false`（`superpowers.ts:31-33`），后续 context 事件不再注入，直到下次 start/compact。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|------|-----------|------|
| `EXTREMELY_IMPORTANT_MARKER` | 包裹引导文本的标记串 | `superpowers.ts:6` |
| `BOOTSTRAP_MARKER` | 幂等检测用的唯一标记 "superpowers:using-superpowers bootstrap for pi" | `superpowers.ts:7` |
| `skillsDir` | `<packageRoot>/skills` | `superpowers.ts:11` |
| `bootstrapSkillPath` | `skills/using-superpowers/SKILL.md` | `superpowers.ts:12` |
| 注入位置 | 首个非 compactionSummary 消息之前 | `superpowers.ts:48,115-121` |

## 5. 错误与重试语义

- 读 SKILL.md 失败：`catch` 里把 `cachedBootstrap=null` 并返回 null（`superpowers.ts:77-80`），context 事件静默跳过注入，不抛错。
- `cachedBootstrap === null` 时 `getBootstrapContent` 直接返回 null（`superpowers.ts:60`）——失败被永久缓存，本会话不再重试（文件本会话内不变，可接受）。
- 无重试/退避：扩展是事件回调，异常由 Pi 宿主兜底；本文件未捕获的异常会在 `console` 留痕但不崩会话。

## 6. 并发细节

- 单文件 TS 扩展，无显式线程；Pi 事件按序派发。
- `cachedBootstrap` 是模块级单例（`let`，`undefined`=未缓存、`null`=文件缺失、字符串=已缓存），惰性初始化（`superpowers.ts:14,60`）。
- `injectBootstrap` 闭包变量随扩展实例存活；事件处理器闭包共享它。
- 无锁：context 事件与 start/end 事件在 Pi 单线程事件循环内串行。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `.pi/extensions/superpowers.ts`（121 行 TS 扩展）。
- 被发现/注入的 `skills/` 目录。

**Out-of-Scope（不在本仓库源码内）**
- `@earendil-works/pi-coding-agent` 的 `ExtensionAPI`、事件调度器、context 事件管线——**不在本仓库源码内**（仅 `import type`，类型导入不产生运行时依赖）。
- Pi 宿主本体、其原生 `read/write/edit/bash` 工具。
- 其他平台引导（session-start-hook、opencode-bootstrap）。

## 8. 与相邻子系统交互

- 上游：Pi 宿主加载本扩展并派发事件。
- 下游 → `skills-core/using-superpowers/`：把该技能正文作为 user 消息注入会话上下文。
- 并列：`session-start-hook`（Claude/Cursor shell）、`opencode-bootstrap`（OpenCode JS 插件）是同职责的其他平台实现。
- 与 opencode-bootstrap 的差异：Pi 不暴露 Claude Code 的 `Skill` 工具，故引导里额外注入 `piToolMapping`，指示模型用 `read` 加载 SKILL.md 或由人显式 `/skill:name`。

## 9. 语言专项适配口径

本叶子是 **TypeScript ESM 扩展**（主仓库 TS/Node 口径）：

- **capability seam**：Provider（`superpowersPiExtension` 向 Pi host 暴露事件订阅）+ 内部纯函数（`stripFrontmatter`/`messageContainsBootstrap`/`firstNonCompactionSummaryIndex`）。
- **图型**：architecture（扩展组件与缓存/工具映射/skills 目录拓扑）+ sequence（resources_discover→start→context→read→回插 的事件时序）。
- **外部边界**：Pi host 标 `external`，`@earendil-works/pi-coding-agent` 仅类型依赖。
- **部署维度**：TS 源文件经 Pi 扩展加载机制加载（不打包进二进制），与 opencode 插件同属"包内插件文件"模式。
- 语言专项：本文件是仓库内仅有的 2 个 `.ts` 文件之一（facts §2），类型严格（`role: "user" as const`、`type: "text" as const`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| pi-extension 架构图 | `pi-extension-architecture.html` | architecture | **standard** |
| Pi 扩展事件处理时序 | `pi-extension-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。

**架构图降档说明**：架构图渲染成功（HTML 803KB），showcase 报 `label overlap`（c2 边标签与 ext 组件重叠）。已缩短标签为"返回技能路径"后降 standard 通过；未再硬撑 showcase。

**未生成 lifecycle 图**：本叶子核心实体 `injectBootstrap` 虽是布尔状态机，但该状态机仅 3 态、1 条回迁边，archify lifecycle 渲染对跨列回边的走线校验较严（回迁边穿中间态），两轮调整未过；其状态变迁已在本 MD 第 3 节用文字描述（start/compact→true、agent_end→false、context 消费），与已有时序图信息互补，按资源节省原则不强行出图。
