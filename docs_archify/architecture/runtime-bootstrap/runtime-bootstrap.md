# runtime-bootstrap（运行时引导）域总览

> 本域包含以下 3 个叶子子系统；各叶子详情见对应文档。
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 域职责

本域解决同一个问题：**Superpowers 的技能是 Markdown 内容，本身不会被任何 agent 自动加载；必须在每次会话启动时，把 `using-superpowers` 这份"元技能"正文注入到 agent 的首条上下文里，agent 才知道"我有超能力、该在回复前先查技能"。**

不同宿主平台（Claude Code、Cursor、Copilot CLI、Muse、OpenCode、Pi）的扩展机制完全不同，本域用三种运行时形态实现同一份引导：

| 平台组 | 实现形态 | 入口 |
|--------|----------|------|
| Claude Code / Cursor / Copilot CLI / Muse | shell SessionStart 钩子 | `hooks/session-start` + `hooks/*.json` + `hooks/run-hook.cmd` |
| OpenCode V1/V2 | 单文件 ESM 插件 | `.opencode/plugins/superpowers.js` |
| Pi | TypeScript 扩展 | `.pi/extensions/superpowers.ts` |

三者产出物是同一段以 `<EXTREMELY_IMPORTANT>` 包裹的引导文本（`using-superpowers/SKILL.md` 正文 + 宿主专属工具名映射），只是注入通道不同。本域不实现任何业务技能逻辑——那是 skills-core 域的职责。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序/数据流 | 职责一句话 |
|------|------|--------|-------------|-----------|
| session-start-hook | [session-start-hook.md](session-start-hook/session-start-hook.md) | [架构图](session-start-hook/session-start-hook-architecture.html) | [时序图](session-start-hook/session-start-hook-sequence.html) | Claude/Cursor 等用 shell 钩子在会话启动输出 additionalContext JSON |
| opencode-bootstrap | [opencode-bootstrap.md](opencode-bootstrap/opencode-bootstrap.md) | [架构图](opencode-bootstrap/opencode-bootstrap-architecture.html) | [时序图](opencode-bootstrap/opencode-bootstrap-sequence.html) | OpenCode V1/V2 单文件插件注册技能并在 messages.transform 注入引导 |
| pi-extension | [pi-extension.md](pi-extension/pi-extension.md) | [架构图](pi-extension/pi-extension-architecture.html) | [时序图](pi-extension/pi-extension-sequence.html) | Pi TS 扩展经 resources_discover 发现技能、context 事件回插引导 |

## 3. 域级机制细节

### 3.1 三份引导实现共享的不变量

1. **单一正文来源**：都读 `skills/using-superpowers/SKILL.md`，本域不复制其内容。
2. **幂等保护**：注入前检查消息/上下文里是否已含 `<EXTREMELY_IMPORTANT>` / bootstrap 标记，避免重复注入（`superpowers.js:251,345`、`superpowers.ts:37,100`）。
3. **fail-open**：读不到 SKILL.md、查不到子会话、钩子找不到 bash 等失败路径都退化为"不注入"而非崩溃（`session-start:11`、`superpowers.js:202-207`、`run-hook.cmd:37-39`）。
4. **零第三方运行时依赖**：shell 用 bash 内建参数替换做 JSON 转义；JS/TS 自写正则剥 frontmatter，不引 YAML 库（符合仓库零依赖设计）。

### 3.2 平台差异矩阵

| 维度 | session-start-hook | opencode-bootstrap | pi-extension |
|------|--------------------|--------------------|--------------|
| 语言 | bash + JSON | Node ESM 纯 JS | TypeScript |
| 触发 | 宿主 SessionStart 事件 | messages.transform / ctx.session.hook | resources_discover + context 事件 |
| 注入通道 | stdout JSON 字段 | 直接改 messages 数组 | context 事件返回新 messages |
| 技能注册 | 无（靠宿主 skills 目录约定） | 主动 draft.add 注册 | 返回 skillPaths 目录 |
| 子会话感知 | 无 | 查 parentID 跳过 | 无（agent_end 开关控制） |
| 缓存 | 无（每次重读） | 模块级 Map 按 mapping 缓存 | 模块级单例缓存 |

### 3.3 为什么不用同一种实现

- Claude Code/Cursor 的插件体系只暴露 shell 钩子，无法加载 JS 运行时，只能用 shell。
- OpenCode 有完整插件 API（V1 config hook / V2 setup），用 JS 才能注册全部技能并做子会话判定。
- Pi 有事件 API 但无 `Skill` 工具，需要在引导里额外注入工具映射段。

## 4. 域级图

![runtime-bootstrap 域架构图](runtime-bootstrap-architecture.html)

![跨平台会话启动引导时序](runtime-bootstrap-sequence.html)

![引导正文数据流](runtime-bootstrap-dataflow.html)

域级图说明：
- **架构图**：三种实现并列，各自对接一组宿主，最终汇聚到同一份 `using-superpowers/SKILL.md` 与 Agent 会话上下文。
- **时序图**：会话启动时三条并行注入路径，分别经 shell stdout / messages.unshift / context 回插，汇入首条上下文。
- **数据流图**：SKILL.md 正文 → 三种加工（shell 转义 / JS 剥 frontmatter / TS 剥 frontmatter）→ 宿主注入层 → Agent 会话上下文。

**省略 lifecycle/workflow**：本域是启动期一次性引导，无跨会话长期状态机、无多泳道审批流程；注入开关（Pi 的 `injectBootstrap`）已在 pi-extension 叶子用文字描述，按资源节省原则不重复出域级状态机图。

## 5. 图表质量档位

| 图 | 文件 | 档位 |
|----|------|------|
| 域架构图 | `runtime-bootstrap-architecture.html` | standard |
| 跨平台启动时序 | `runtime-bootstrap-sequence.html` | showcase |
| 引导正文数据流 | `runtime-bootstrap-dataflow.html` | standard |

架构图与数据流图落 standard：组件/节点较多（域级图允许 standard），渲染退出码 0、HTML 均 >800KB。
