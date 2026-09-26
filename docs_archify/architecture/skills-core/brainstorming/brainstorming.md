# brainstorming（brainstorming）

> 本文是 `skills-core` 域下的叶子子系统文档。域级总览见 `../skills-core.md`，本文只展开
> "把想法通过协作对话细化为设计/规格"的方法论，以及本仓库唯一内置的服务端——
> visual companion 浏览器伴侣（Node HTTP + WebSocket）。writing-plans 的衔接见对应叶子。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 三路径分类 | Spike / Bounded / Architectural 三条处理路径，按请求复杂度选择 | `skills/brainstorming/SKILL.md:58-88` |
| 共享理解建立 | Discover intent → Write back understanding → Carry intent into design | `SKILL.md:14-36` |
| 硬闸门（HARD-GATE） | 实现动作前必须完成所选路径的前置审批 | `SKILL.md:38-56` |
| 红旗反合理化 | 8 条"太简单不需要审批"的反驳 | `SKILL.md:90-108` |
| 路径 checklist | Spike 5 步 / Bounded 5 步 / Architectural 9 步 | `SKILL.md:110-138` |
| 规格自审 | Placeholder 扫描 / 内部一致性 / 范围 / 歧义检查 | `SKILL.md:246-254` |
| Visual companion（即时按需） | 浏览器伴侣，用于展示 mockup / 布局对比 / 架构图 | `SKILL.md:268-285`、`visual-companion.md` |
| HTTP + WebSocket 服务端 | 零依赖 Node 实现 RFC 6455，推送屏幕、接收用户事件 | `scripts/server.cjs:1-723` |
| 会话密钥鉴权 | `?key=` + HttpOnly Cookie，timing-safe 比较，防 DNS rebinding | `scripts/server.cjs:115-150`、`319-353` |
| 屏幕文件热更新 | fs.watch 监听 content 目录，防抖 100ms 后广播 reload | `scripts/server.cjs:590-613` |
| 生命周期看门狗 | owner PID 死亡 / 空闲超时（默认 4h）自动退出 | `scripts/server.cjs:634-657` |
| 遥测商标注入 | Prime Radiant 品牌图，可用环境变量关闭 | `scripts/server.cjs:105-112`、`242-256` |
| 启动/停止脚本 | 端口选择、后台化、PID 文件、curl 健康检查 | `scripts/start-server.sh`、`scripts/stop-server.sh` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `OPCODES` / `computeAcceptKey` / `encodeFrame` / `decodeFrame` | `server.cjs:8-81` | 手写 WebSocket 帧编解码（RFC 6455） |
| `preferredPort()` | `server.cjs:89-98` | 端口选择：`BRAINSTORM_PORT` → PORT_FILE → 随机高端口 |
| `initialToken()` / `generateToken()` | `server.cjs:124-146` | 会话密钥：env > TOKEN_FILE > 随机 32 字节 hex |
| `isAuthorized(req)` | `server.cjs:341-353` | `?key=` 或 Cookie 常量时间比较鉴权 |
| `handleRequest(req,res)` | `server.cjs:387-438` | HTTP 路由：`/` 引导页/最新屏幕、`/files/*` 静态资源 |
| `handleUpgrade(req,socket)` | `server.cjs:444-501` | WebSocket 升级 + 帧循环 |
| `handleMessage(text)` | `server.cjs:503-517` | 解析客户端 JSON，`choice` 事件追加到 state/events |
| `broadcast(msg)` | `server.cjs:519-524` | 向所有连接的客户端广播 TEXT 帧 |
| `getNewestScreen()` | `server.cjs:267-278` | 按 mtime 选最新 HTML 屏幕 |
| `isRegularFileInsideContentDir()` | `server.cjs:304-317` | 防路径穿越：拒绝符号链接、非普通文件、nlink≠1 |
| `maybeOpenBrowser()` | `server.cjs:530-548` | 首屏就绪时自动打开浏览器（跨平台 launcher） |
| `shutdown(reason)` / `ownerAlive()` | `server.cjs:616-643` | 优雅关闭 + owner 进程存活检测 |
| `start-server.sh` / `stop-server.sh` | `scripts/` | shell 包装：参数解析、后台化、PID/端口/状态文件 |

## 3. 关键调用链

**链 1：用户消息 → 路径分类 → 审批闸门**

1. agent 收到请求，命中 `SKILL.md:60-63`：先大声说出分类（"this looks bounded..."），允许用户覆盖。
2. 命中 `SKILL.md:65-88`：Spike / Bounded / Architectural 三选一；拿不准取更重路径，中途发现隐藏复杂度只升不降。
3. 命中 `SKILL.md:38-56` HARD-GATE：实现动作前必须完成所选路径的前置审批（Spike 批准 probe；Bounded 批准 chat design；Architectural 批准 spec + plan 选择）。
4. Architectural 路径走到 `SKILL.md:138`：调用 writing-plans 技能衔接实现计划。

**链 2：Visual companion 推送一个屏幕**

1. agent 决定用浏览器展示，命中 `SKILL.md:272-275`：单独发一条 offer 消息，等用户同意。
2. 用户同意后，agent 把渲染好的 HTML 写到 `$BRAINSTORM_DIR/content/` 下某个 `.html` 文件（`server.cjs:103`）。
3. `fs.watch` 触发（`server.cjs:590`），防抖 100ms 后：若是新文件，清空 `state/events`、打日志 `screen-added`、调用 `maybeOpenBrowser()`；广播 `{type:'reload'}`。
4. 浏览器端 `helper.js` 收到 reload，重新拉 `/`，`getNewestScreen()`（`server.cjs:267`）返回最新文件，整屏替换。

**链 3：用户在浏览器里点选 → 事件回传 agent**

1. 浏览器端 JS 发 WebSocket TEXT 帧 `{choice: ...}`。
2. `handleUpgrade` 的 `decodeFrame`（`server.cjs:46-81`）解帧，校验 client 帧必须 masked、payload ≤ 10MB。
3. `handleMessage`（`server.cjs:503-517`）JSON.parse，若含 `choice` 字段，追加一行到 `state/events`，同时 `console.log` 一行 JSON——agent 从子进程 stdout 读到。
4. agent 下一轮根据事件继续对话。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `BRAINSTORM_PORT` | 显式指定端口 | `server.cjs:90` |
| `BRAINSTORM_PORT_FILE` | 记录上次绑定端口，重启复用 | `server.cjs:85`、`672-673` |
| `BRAINSTORM_HOST` | 默认 `127.0.0.1`；远程部署用 `0.0.0.0` | `server.cjs:100` |
| `BRAINSTORM_URL_HOST` | URL 中展示的 host（与 bind host 解耦） | `server.cjs:101` |
| `BRAINSTORM_DIR` | 会话目录，默认 `/tmp/brainstorm`；下分 `content/`、`state/` | `server.cjs:102-104` |
| `BRAINSTORM_TOKEN` / `BRAINSTORM_TOKEN_FILE` | 会话密钥来源 | `server.cjs:123-146` |
| `BRAINSTORM_OWNER_PID` | 看门狗检测的 owner 进程 | `server.cjs:113`、`634-657` |
| `BRAINSTORM_IDLE_TIMEOUT_MS` | 空闲退出，默认 4h | `server.cjs:554-557` |
| `BRAINSTORM_LIFECYCLE_CHECK_MS` | 看门狗周期，默认 60s | `server.cjs:560-563` |
| `BRAINSTORM_OPEN` / `BRAINSTORM_OPEN_CMD` | 首屏自动开浏览器 | `server.cjs:533`、`539` |
| `SUPERPOWERS_DISABLE_TELEMETRY` / `DISABLE_TELEMETRY` / `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 任一为真则不加载品牌图 | `server.cjs:107-112` |
| 规格输出路径 | `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` | `SKILL.md:135`、`241` |

## 5. 错误与重试语义

- **WebSocket 帧解析失败**（`server.cjs:466-471`）：发 CLOSE 帧、删除 socket，不重试——协议错误即断开。
- **未授权请求**（`server.cjs:388-392`）：返回 403 + FORBIDDEN_PAGE，提示用户复制完整 `?key=` URL。
- **路径穿越**（`server.cjs:420-429`、`304-317`）：拒绝 dotfile / 符号链接 / 非普通文件 / nlink≠1，返回 404。
- **端口被占**（`server.cjs:691-708`）：`EADDRINUSE` 时若 token 来自 env 则直接退出（避免用显式 token 跑在随机端口）；否则换随机端口重试一次，token 重新生成。
- **owner PID 启动时已死**（`server.cjs:649-657`）：记录 `owner-pid-invalid`，关闭 owner 监控，只靠空闲超时退出（WSL / Tailscale SSH 常见）。
- **fs.watch error**（`server.cjs:614`）：打日志不崩溃。
- **方法论层**：Architectural 路径中用户要求改 spec → 回到 `Write design doc` 重跑自审循环（`SKILL.md:179`），不视为失败而是正常迭代。

## 6. 并发细节

- **Node 单线程事件循环**：HTTP server + WebSocket upgrade + fs.watch 全部在同一事件循环。
- **`clients: Set<socket>`**（`server.cjs:442`）：所有已升级的 WebSocket 连接，broadcast 时遍历；写失败即从 Set 删除（`server.cjs:522`）。
- **`debounceTimers: Map`**（`server.cjs:572`）：按文件名防抖，避免一次保存触发多次 reload。
- **`lifecycleCheck.unref()`**（`server.cjs:644`）：看门狗定时器不阻止进程退出。
- **buffer 拼接循环**（`server.cjs:461-497`）：TCP 流式到达，逐帧消费，半包保留在 buffer。
- **无锁 / 无 worker**：零依赖单进程设计，不引入线程或子进程 worker。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `skills/brainstorming/SKILL.md` 方法论
- `scripts/server.cjs` 完整 HTTP+WebSocket 服务端
- `scripts/helper.js`（浏览器端注入脚本）、`frame-template.html`（外框模板）
- `scripts/start-server.sh` / `stop-server.sh`（shell 包装）
- `visual-companion.md`、`spec-document-reviewer-prompt.md`

**Out-of-Scope（不在本仓库源码内）**
- 用户浏览器本身（Chrome / Safari 等）
- Prime Radiant 品牌图 CDN（`https://primeradiant.com/brand/...`，`server.cjs:106`）——外部资源
- agent harness 的子进程管理（启动/停止 server 的编排由 agent 完成）
- writing-plans 技能本体——见对应叶子
- 用户项目自身的 `docs/superpowers/specs/` 目录内容

## 8. 与相邻子系统交互

- **上游**：using-superpowers 把"Let's build X"类请求路由到本技能（`using-superpowers/SKILL.md:30`）。
- **下游**：
  - Architectural 路径 → writing-plans 技能（`SKILL.md:138`、`265`）
  - 实现阶段 → TDD / executing-plans / subagent-driven-development
- **横向**：
  - visual companion 服务端被 start-server.sh 拉起，agent 通过 stdout 的 JSON（`server-started` / `screen-added` / `user-event`）与服务端通信
  - 用户浏览器通过 WebSocket 与服务端通信，不直连 agent

## 9. 语言专项适配口径

本叶子混合 Markdown 方法论 + Node CommonJS 服务端，按 TS/Node 口径：

- **capability seam 分组**：
  - Service Definition：SKILL.md 定义方法论契约（三路径、HARD-GATE）
  - Provider：server.cjs 提供"浏览器屏幕推送 + 用户事件回流"能力
  - Consumer：agent 进程（写 content 文件、读 stdout JSON）、用户浏览器
- **图类型**：architecture（服务端组件拓扑）+ sequence（屏幕推送 / 事件回流时序）。
- **外部边界**：标注用户浏览器、Prime Radiant CDN 为 external。
- **无 lifecycle**：会话状态虽有"空闲/活动"，但看门狗逻辑简单，不值得单画状态机。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Visual companion 服务端架构 | `brainstorming-architecture.html` | architecture | showcase |
| 屏幕推送与事件回流时序 | `brainstorming-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 未生成 workflow 图：三路径方法论在 SKILL.md 内嵌 dot 图（`SKILL.md:142-182`）已表达，与 architecture/sequence 信息重复。
- 未生成 dataflow 图：屏幕文件流向已由 sequence 表达。
- 未生成 lifecycle 图：无单一实体状态机（空闲/退出逻辑见 MD 第 5/6 节）。
