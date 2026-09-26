# opencode-bootstrap（OpenCode 插件引导）

> 本文是 `runtime-bootstrap` 域下的叶子子系统文档。域级总览见 `../runtime-bootstrap.md`。
> 本文只展开"OpenCode 平台如何通过单文件 JS 插件注册全部技能并注入引导"，
> 不重复展开 Claude/Cursor 的 shell 钩子（`session-start-hook`）与 Pi 扩展（`pi-extension`）。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| V1 插件入口 | 命名导出 `SuperpowersPlugin`，提供 config hook + messages.transform | `.opencode/plugins/superpowers.js:219-266` |
| V2 插件入口 | 默认导出 `{ id, setup }`，`setup()` 被 PluginSupervisor 调用 | `.opencode/plugins/superpowers.js:286-383` |
| 技能目录注册（V1） | 把 skills 目录追加进 `config.skills.paths` | `superpowers.js:223-233` |
| 技能原生注册（V2） | 遍历 `skills/*/SKILL.md`，`draft.add()` 成 Skill.Info | `superpowers.js:294-330` |
| 极简 frontmatter 解析 | 自写正则剥 YAML frontmatter，无第三方依赖 | `superpowers.js:33-68` |
| 引导正文缓存 | 按 toolMapping 缓存已解析的引导文本，避免每 step 读盘 | `superpowers.js:113-141` |
| 宿主工具映射 | V1/V2 各自一份工具名映射常量（todowrite→task→patch 等） | `superpowers.js:77-107` |
| 子会话判定 | 查 session.parentID，跳过 task 子代理的引导注入 | `superpowers.js:143-210` |
| 注入幂等保护 | 首条 user 消息已含 `EXTREMELY_IMPORTANT` 则跳过 | `superpowers.js:251,345` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `extractAndStripFrontmatter(content)` | `superpowers.js:33-68` | 正则剥 frontmatter，返回 `{frontmatter, content}`；非完整 YAML 解析器 |
| `V1_MAPPING` / `V2_MAPPING` | `superpowers.js:77-107` | 注入到引导里的工具名对照表，按宿主版本区分 |
| `getBootstrapContent(toolMapping)` | `superpowers.js:116-141` | 读 using-superpowers SKILL.md → 剥 frontmatter → 包 `<EXTREMELY_IMPORTANT>`；按 mapping 缓存 |
| `isChildSession(fetchSession, sessionID)` | `superpowers.js:177-210` | 查 session 是否带 parentID；带 LRU 缓存（上限 512，满则淘汰最旧 1/4） |
| `SuperpowersPlugin({client,directory})` | `superpowers.js:219-266` | V1 插件函数，返回 config 与 `experimental.chat.messages.transform` |
| `setup(ctx)` | `superpowers.js:286-370` | V2 入口：注册技能 + `ctx.session.hook('context')` 注入引导 |

## 3. 关键调用链

1. **V2 启动注册**：OpenCode PluginSupervisor 调 `default.setup(ctx)`（`superpowers.js:286`）；若 ctx 缺 `skill`/`session` 域（V1 误调）则静默返回（`superpowers.js:290-292`）。
2. **注册技能**：`fs.readdirSync(superpowersSkillsDir)` 遍历每个技能目录，读 `SKILL.md` → `extractAndStripFrontmatter` → 组装 `{id,name,description,path,content}`，在 `ctx.skill.transform` 里逐个 `draft.add()`（`superpowers.js:298-330`）；单个技能被 host 拒绝只跳过该技能，不拖垮整个插件（`superpowers.js:323-329`）。
3. **每 step 注入引导（V2）**：`ctx.session.hook('context')` 回调（`superpowers.js:339`）→ `getBootstrapContent(V2_MAPPING)` → 找首条 `role==='user'` 消息 → 已含标记则跳过 → `isChildSession` 查 parentID → 子会话跳过 → 否则 `firstUser.content.unshift(bootstrap)`（`superpowers.js:358`）。
4. **V1 注入**：`experimental.chat.messages.transform`（`superpowers.js:244`）→ 从 message 记录自身取 `sessionID` → `client.session.get()` 判子会话 → `firstUser.parts.unshift(bootstrap)`（`superpowers.js:263`）。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|------|-----------|------|
| `superpowersSkillsDir` | `__dirname/../../skills`（包内 skills 目录） | `superpowers.js:25` |
| `CHILD_SESSION_CACHE_MAX` | 512 条，满则淘汰最旧 1/4 | `superpowers.js:163-175` |
| 引导标记 | `<EXTREMELY_IMPORTANT>`（幂等检测关键字） | `superpowers.js:130,251,345` |
| V2 最低宿主版本 | OpenCode ≥ 2.0.4（`path` 字段由 `location` 改名而来） | `superpowers.js:307-309`；INSTALL.md:9 |
| 安装方式 | `opencode.json` 配 `plugin`/`plugins` 数组，git+https 指向仓库 | `.opencode/INSTALL.md:16-29` |

## 5. 错误与重试语义

- **读不到 SKILL.md**：`getBootstrapContent` 把 `null` 写入缓存，后续步骤跳过注入而非崩溃（`superpowers.js:122-125`）。
- **子会话查询失败 fail-open**：`isChildSession` 任何异常（无记录、SDK envelope 错误、身份不匹配）都 `console.error` 后返回 `false`（按顶层会话处理，继续注入），且**不缓存**失败结果，下次 step 可恢复（`superpowers.js:202-207`）。
- **技能注册失败隔离**：`draft.add` 抛错只跳过该技能（`superpowers.js:326-328`）；整个 `skill.transform` 失败只打日志，不中断插件（`superpowers.js:331-335`）。
- **context 钩子回调内错误**：内层 try/catch 吞掉，绝不污染请求管线（`superpowers.js:362-365`）。
- 无重试退避：引导是同步变换，失败即跳过。

## 6. 并发细节

- V2 service 进程长驻；`_bootstrapCache` 是模块级 `Map`，按 toolMapping 做 key，会话内只读一次盘。
- `_childSessionCache` 也是模块级 `Map`，LRU 式淘汰（插入序，满 512 删最旧 1/4），避免长驻进程内存无限涨（`superpowers.js:166-175`）。
- 无锁：单线程 Node 事件循环内 Map 操作原子；异步点仅 `fetchSession`（await），结果在 await 后写缓存。
- 每个 agent step 都触发 transform/context 钩子，故缓存设计是性能关键（注释引用 issue #1202）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `.opencode/plugins/superpowers.js`（383 行，单文件插件）与 `.opencode/INSTALL.md`。
- 被注册/注入的 `skills/*/SKILL.md`（内容分析见 skills-core 各叶子）。

**Out-of-Scope（不在本仓库源码内）**
- OpenCode 宿主本体：PluginSupervisor、Skill.Info schema、session API、messages.transform 调度——**不在本仓库源码内**。
- `@opencode-ai/plugin`、`effect` 等上游包（插件刻意零依赖，未 import）。
- 其他平台的引导（Claude/Cursor 走 session-start-hook，Pi 走 pi-extension）。

## 8. 与相邻子系统交互

- 上游：OpenCode V1/V2 宿主加载本插件（V1 扫命名导出，V2 读 `{id,setup}`）。
- 下游 → `skills-core/`：本插件把 `skills/` 下全部 SKILL.md 注册为宿主原生技能，并把 `using-superpowers` 正文注入首条上下文。
- 并列：`session-start-hook`（shell）与 `pi-extension`（TS）是同职责的其他平台实现。
- 安装/打包关系：`harness-packaging` 域负责多平台插件清单，本文件是 OpenCode 专属实现。

## 9. 语言专项适配口径

本叶子是 **Node ESM 纯 JavaScript 插件**（主仓库 TS/Node 口径）：

- **capability seam 分组**：Provider（`SuperpowersPlugin`/`setup` 向 host 提供插件能力）+ Consumer（消费 host 的 `client`/`ctx` API）；`extractAndStripFrontmatter`、`isChildSession` 是内部纯函数。
- **图型**：architecture（双版本入口 + 缓存 + 子会话判定组件拓扑）+ sequence（每 step 注入时序：钩子→读缓存→查子会话→回注消息）。
- **外部边界**：OpenCode host、session API 标 `external`；skills 目录标 `database`。
- **部署维度**：无 workspace 包；插件经 git-backed 包规范被 OpenCode 包管理器安装（INSTALL.md:72-86），不是单二进制。
- 兼容矩阵：单文件同时服务 V1（1.18.x）与 V2（2.0.4/2.0.7），通过探测 ctx shape 与 session 返回 envelope 形状做运行时分支。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| opencode-bootstrap 架构图 | `opencode-bootstrap-architecture.html` | architecture | **standard** |
| OpenCode 引导注入时序 | `opencode-bootstrap-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。

**架构图降档说明**：该图渲染成功（HTML 806KB），但 showcase 校验报 `composition/ambiguous-corridor`（c4/c5/c8 三条从左列到中列的边在 x≈565 处共享走廊）与 `composition/label-route-clearance`（c4/c5/c6 标签互相 0px 重叠）。已做两轮修复（删低价值边 c3/c9、把 bootcache/child 纵向错行拉开），仍未过 showcase；按出图指南"showcase 反复不过（≥2 轮）降 standard"处理，未再硬撑。

省略 lifecycle/dataflow/workflow：本插件是事件变换钩子，无单一实体状态机、无数据管道、无多泳道审批；注入时序已由 sequence 表达，按资源节省原则省略。
