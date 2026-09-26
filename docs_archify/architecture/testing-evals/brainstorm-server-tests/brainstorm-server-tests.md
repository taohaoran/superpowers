# brainstorming 视觉伴侣服务端测试（brainstorm-server-tests）

> 本文是 `testing-evals` 域下的叶子子系统文档。域级总览见 `../testing-evals.md`。
> 本文只展开"对 brainstorming 技能内置 visual companion HTTP/WebSocket 服务端的测试"；
> 服务端实现本身见 skills-core 域 brainstorming 叶子。
>
> 源码基准：superpowers main，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| HTTP 服务集成测试 | 启动真实 server.cjs，断言路由/静态文件 | `tests/brainstorm-server/server.test.js`（592 行） |
| WebSocket 协议测试 | 用 `ws` 客户端测帧协议 | `tests/brainstorm-server/ws-protocol.test.js`（407 行） |
| 鉴权测试 | 测试 session key cookie 鉴权与未授权拒绝 | `tests/brainstorm-server/auth.test.js`（312 行） |
| 品牌/商标测试 | Prime Radiant 商标加载与遥测关闭开关 | `tests/brainstorm-server/branding.test.js`（344 行） |
| 生命周期测试 | start/stop 脚本、PID 管理、文件监听 | `tests/brainstorm-server/lifecycle.test.js`（515 行） |
| helper 单测 | `helper.js` 工具函数 | `tests/brainstorm-server/helper.test.js`（197 行） |
| 浏览器启动器测试 | 自动开浏览器逻辑 | `tests/brainstorm-server/browser-launcher.test.js`（66 行） |
| start/stop 脚本测试 | shell 启动/停止脚本行为 | `tests/brainstorm-server/start-server.test.sh`、`stop-server.test.sh` |
| Windows 生命周期 | Windows 下进程清理 | `tests/brainstorm-server/windows-lifecycle.test.sh`（394 行） |

测试聚合入口 `tests/brainstorm-server/package.json` 的 `test` 脚本按顺序串跑 7 个 node 测试 + 2 个 shell 测试。

## 2. 核心类型与接口清单

| 设施 | 位置 | 职责 |
|---|---|---|
| `SERVER_PATH` | `server.test.js:19` | 指向被测 `skills/brainstorming/scripts/server.cjs` |
| `TEST_PORT=3334` / `TEST_DIR=/tmp/brainstorm-test` | `server.test.js:21-24` | 固定测试端口与隔离目录 |
| `TOKEN` | `server.test.js:26` | 测试用鉴权 cookie `brainstorm-key-<port>` |
| `spawn(server)` | `server.test.js:14` | 起子进程跑真实服务端 |
| `ws` 依赖 | `package.json:dependencies` | 唯一测试期 npm 依赖（不随产物分发） |

## 3. 关键调用链

**链路：一次 server 集成测试**

1. `cleanup()` 清掉 `/tmp/brainstorm-test`（`server.test.js:30-35`）。
2. `spawn` 启动 `server.cjs`，环境变量注入测试 token（`server.test.js:14`）。
3. 测试客户端带 `Cookie: brainstorm-key-3334=<TOKEN>` 发 HTTP GET（`server.test.js:42-44`）。
4. 断言响应码、静态文件内容、WebSocket 帧；结束后杀掉子进程。
5. `package.json` 的 `test` 脚本把上述 7 个 node 文件与 2 个 shell 文件串行跑。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|---|---|---|
| `TEST_PORT` | 3334（避开开发常用端口） | `server.test.js:22` |
| 鉴权 cookie 名 | `brainstorm-key-<port>` | `server.test.js:43` |
| `SUPERPOWERS_DISABLE_TELEMETRY` | 测遥测开关时使用（见 branding 测试） | 服务端环境变量 |

## 5. 错误与重试语义

- node 测试用 `assert` 原生断言，失败即抛错退出；shell 测试 `set -e`。
- 服务端子进程必须在测试结束时杀干净，否则端口占用污染下一轮——lifecycle 测试专门覆盖此路径。

## 6. 并发细节

- 测试是串行脚本；被测 server 是单进程。WebSocket 测试在单连接上交互，无并发会话断言。

## 7. 系统边界

**In-Scope**：`tests/brainstorm-server/` 全部测试文件。

**Out-of-Scope**
- 被测服务端实现 `skills/brainstorming/scripts/server.cjs`（723 行）、`helper.js`、`start/stop-server.sh`——属 skills-core 域 brainstorming 叶子
- `ws` npm 包本身——第三方依赖（仅测试期）
- 真实浏览器——browser-launcher 测试只断言调用逻辑，不真开浏览器

## 8. 与相邻子系统交互

- 被测对象：skills-core 域 brainstorming 叶子的 visual companion 服务端。
- 与 harness-integration-tests 平级：本叶子专测服务端，其余平台测试不碰它。

## 9. 语言专项适配口径

主语言 TS/Node：本叶子是 Node `assert` + `ws` 客户端的**真实子进程集成测试**。
- sequence 图表达"测试进程 spawn 服务端 → HTTP/WS 交互 → 杀进程"的单次调用时序。
- architecture 图表达测试套件组成。
- 不补 dataflow/lifecycle/workflow——无数据管道、无状态机、无审批流程。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 测试套件组成 | `brainstorm-server-tests-architecture.html` | architecture | showcase |
| 一次集成测试时序 | `brainstorm-server-tests-sequence.html` | sequence | showcase |

- JSON IR 源：`json/brainstorm-server-tests-*.json`。
