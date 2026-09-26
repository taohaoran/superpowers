# session-start-hook（会话启动钩子）

> 本文是 `runtime-bootstrap` 域下的叶子子系统文档。域级总览见 `../runtime-bootstrap.md`。
> 本文只展开"Claude Code / Cursor 等平台如何在会话启动时通过 shell 钩子注入 `using-superpowers` 引导"，
> 不重复展开 OpenCode 插件（`opencode-bootstrap`）与 Pi 扩展（`pi-extension`）的同类机制。
>
> 源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 会话启动触发注册 | Claude Code 在 `startup\|clear\|compact` 时触发 SessionStart | `hooks/hooks.json:3-12` |
| Cursor 会话启动注册 | Cursor 启动时调用同一脚本 | `hooks/hooks-cursor.json:3-9` |
| Windows 跨平台包装 | 多语言 polyglot：Windows 下找 Git Bash，Unix 下直接 exec | `hooks/run-hook.cmd:1-46` |
| 引导正文读取 | 读取 `skills/using-superpowers/SKILL.md` 全文 | `hooks/session-start:10-11` |
| JSON 转义 | 用 bash 参数替换一次性转义反斜杠/引号/换行/回车/制表符 | `hooks/session-start:16-24` |
| 平台字段分叉 | 按 `CURSOR_PLUGIN_ROOT`/`CLAUDE_PLUGIN_ROOT`/`MUSE_PLUGIN_ROOT`/`COPILOT_CLI` 选择输出字段名 | `hooks/session-start:39-51` |
| 引导包裹 | 用 `<EXTREMELY_IMPORTANT>` 标记包裹正文后输出 | `hooks/session-start:27` |

## 2. 核心类型与接口清单

本叶子是 shell + JSON 配置，无 class/struct。核心"接口"是钩子脚本的 stdout JSON 契约：

| 接口/函数 | 位置 | 职责 |
|-----------|------|------|
| `escape_for_json()` | `hooks/session-start:16-24` | 把 SKILL.md 正文转义为可嵌入 JSON 字符串字面量 |
| SessionStart 命令 | `hooks/hooks.json:9` | `"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd" session-start`，`shell: bash`、`async: false` |
| Cursor sessionStart 命令 | `hooks/hooks-cursor.json:6` | `./hooks/run-hook.cmd session-start` |
| stdout 契约 | `hooks/session-start:39-51` | Cursor→`additional_context`；Claude/Muse→`hookSpecificOutput.additionalContext`；Copilot→顶层 `additionalContext` |

## 3. 关键调用链

1. **会话启动**：宿主在 `startup`/`clear`/`compact` 时按 `hooks/hooks.json:5` 的 matcher 触发 SessionStart → 执行 `run-hook.cmd session-start`（`hooks/hooks.json:9`）。
2. **定位 bash**（Windows）：`run-hook.cmd` 依次探测 `C:\Program Files\Git\bin\bash.exe`、`Program Files (x86)`、PATH 上的 `bash`；都找不到则静默 `exit /b 0`（`hooks/run-hook.cmd:21-39`），退化为"无引导注入"。
3. **执行脚本**：Unix 分支 `exec bash "${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"`（`hooks/run-hook.cmd:46`）。
4. **读正文**：`session-start` 由 `SCRIPT_DIR/..` 推出 `PLUGIN_ROOT`，`cat` 读 `skills/using-superpowers/SKILL.md`（`hooks/session-start:7-11`）。
5. **转义与分叉**：`escape_for_json` 做 5 次参数替换（`hooks/session-start:18-22`），随后按环境变量选择 3 种 stdout JSON 形态之一（`hooks/session-start:39-51`），用 `printf` 输出（规避 bash 5.3+ heredoc 挂起，见 issue #571）。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|------|-----------|------|
| `matcher` | `startup\|clear\|compact` 三类会话事件均触发 | `hooks/hooks.json:5` |
| `async` | `false`（同步等待钩子完成再进会话） | `hooks/hooks.json:11` |
| `CURSOR_PLUGIN_ROOT` | 非空 → 输出 `additional_context` | `hooks/session-start:39-41` |
| `CLAUDE_PLUGIN_ROOT`（且无 COPILOT/MUSE） | 输出 `hookSpecificOutput.additionalContext` | `hooks/session-start:42-44` |
| `MUSE_PLUGIN_ROOT` | 同样走嵌套 `hookSpecificOutput` | `hooks/session-start:45-47` |
| `COPILOT_CLI` / 未知平台 | 输出 SDK 标准顶层 `additionalContext` | `hooks/session-start:48-50` |

## 5. 错误与重试语义

- 读 SKILL.md 失败时不中断：`cat ... 2>&1 || echo "Error reading using-superpowers skill"`（`hooks/session-start:11`），把错误字符串塞进引导，宿主仍能启动。
- `set -euo pipefail`（`hooks/session-start:4`）：任何未预期错误直接非零退出；宿主一般仅告警，不阻塞会话。
- Windows 找不到 bash 时**静默成功**（`run-hook.cmd:37-39` 注释明确：插件仍工作，只是没有启动上下文注入）——这是有意的 fail-open。
- 无重试：钩子是一次性同步执行，宿主不负责重跑。

## 6. 并发细节

- 单次会话启动只跑一次钩子，`async: false` 意味着宿主等待其 stdout。
- 无后台进程、无锁、无长驻状态；脚本读完即退。
- Cursor 与 Claude Code 读取字段不做去重，故脚本**只输出当前平台消费的那一个字段**（`hooks/session-start:33-34` 注释），避免双重注入。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `hooks/` 目录下 4 个文件：session-start 脚本、hooks.json、hooks-cursor.json、run-hook.cmd。
- 被注入的正文来源 `skills/using-superpowers/SKILL.md`（其内容分析见 `skills-core/using-superpowers/`）。

**Out-of-Scope（不在本仓库源码内）**
- Claude Code / Cursor / Copilot CLI / Muse 等宿主本体的钩子调度器与上下文注入管线——**不在本仓库源码内**。
- OpenCode、Pi 的引导注入不走本脚本，分别见 `opencode-bootstrap`、`pi-extension`。

## 8. 与相邻子系统交互

- 上游：宿主平台（Claude Code/Cursor/Copilot/Muse）在会话启动事件触发钩子。
- 下游 → `skills-core/using-superpowers/`：本叶子只是把该技能正文原样塞进首条上下文；技能本身的行为由 using-superpowers 定义。
- 并列：`opencode-bootstrap`（JS 插件）与 `pi-extension`（TS 扩展）是同一"会话启动注入引导"职责在其他平台上的另两种实现，机制不同但产出相同（一段 `<EXTREMELY_IMPORTANT>` 包裹的引导文本）。

## 9. 语言专项适配口径

本叶子主语言为 **bash shell + JSON 配置**（非 TS/Node）。按 TS/Node 主仓库口径补充说明：

- 无 Node 事件循环/异步 promise；时序是同步 stdout 契约。
- 图型：架构图（组件拓扑：宿主→清单→包装器→脚本→SKILL.md）+ 时序图（启动→读文件→转义→回注）。
- 外部边界：宿主钩子调度器标注为 `external`，不画进仓库边界。
- Windows polyglot 脚本（`run-hook.cmd`）利用 `: << 'CMDBLOCK'` 在 Unix 下把 batch 段注释掉、batch 段又能被 cmd.exe 解析——属于跨平台兼容技巧，不引入第三方依赖（符合仓库零依赖设计）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| session-start-hook 架构图 | `session-start-hook-architecture.html` | architecture | showcase |
| 会话启动引导注入时序 | `session-start-hook-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。两张图均一次通过 showcase（0 错误）。
省略 lifecycle/dataflow/workflow：本叶子是无状态的单次启动脚本，无实体状态机、无数据管道、无多泳道审批流程，与已有图信息重复，按资源节省原则省略。
