# 多平台 harness 集成测试（harness-integration-tests）

> 本文是 `testing-evals` 域下的叶子子系统文档。域级总览见 `../testing-evals.md`。
> 本文只展开"针对 17 个平台/集成目录的测试矩阵"；brainstorming 服务端测试见
> `../brainstorm-server-tests/brainstorm-server-tests.md`，shell/lint 等技能行为测试见
> `../skill-behavior-tests/skill-behavior-tests.md`。
>
> 源码基准：superpowers main，commit `8ca22db`。

## 1. 功能清单

`tests/` 下 17 个目录，每个对应一个平台或集成面，各自带 `run-tests.sh` 聚合器：

| 测试目录 | 被测对象 | 语言 | 源码路径 |
|---|---|---|---|
| claude-code | SDD/worktree/executing-plans 脚本在 Claude Code 下的行为 | bash + python | `tests/claude-code/` |
| codex | `package-codex-plugin.sh` 与 marketplace 清单断言 | bash | `tests/codex/` |
| codex-plugin-sync | `sync-to-codex-plugin.sh` 行为 | bash | `tests/codex-plugin-sync/` |
| cursor / devin / kimi / antigravity | 各平台插件加载与工具映射 | bash | `tests/{cursor,devin,kimi,antigravity}/` |
| opencode | 插件加载、会话引导缓存、技能注册、优先级 | bash + mjs | `tests/opencode/` |
| pi | `.pi/extensions/superpowers.ts` | mjs | `tests/pi/` |
| hermes | Hermes 插件 `register`/bootstrap | pytest | `tests/hermes/` |
| hooks | `hooks/session-start` | bash | `tests/hooks/` |
| explicit-skill-requests | 多轮对话里技能是否被正确显式触发 | bash + prompts | `tests/explicit-skill-requests/` |
| diagnosing-superpowers / systematic-debugging / writing-skills | 对应技能结构与脚本 | bash | `tests/{diagnosing-superpowers,systematic-debugging,writing-skills}/` |
| version-bump | `bump-version.sh` | bash | `tests/version-bump/` |
| shell-lint | `lint-shell.sh` | bash | `tests/shell-lint/` |

## 2. 核心类型与接口清单

本叶子是测试脚本集合，无自定义类型。共性测试设施：

| 设施 | 位置 | 职责 |
|---|---|---|
| `pass/fail/FAILURES` 计数 | 各 `test-*.sh` 头部（如 `tests/codex/test-package-codex-plugin.sh:11-19`） | 轻量断言框架，`trap cleanup EXIT` 用 mktemp 隔离 |
| `assert_equals/assert_contains/assert_not_matches` | `tests/codex/test-package-codex-plugin.sh:22-58` | shell 断言原语，被多数平台测试复用模式 |
| `run-tests.sh` 聚合器 | `tests/antigravity/run-tests.sh`、`tests/opencode/run-tests.sh` | 遍历 `test-*.sh` 逐个 bash 执行，任一失败即非零退出 |
| pytest `mock_ctx` fixture | `tests/hermes/conftest.py` | 复刻 hermes `register_skill` 必须收 `Path` 的契约 |

## 3. 关键调用链

**链路一：一个平台测试如何跑起来（以 antigravity 为例）**

1. `tests/antigravity/run-tests.sh` 遍历同目录 `test-*.sh`，逐个 `bash` 执行（`run-tests.sh:13-16`）。
2. 每个 `test-*.sh` 自举 `SCRIPT_DIR`/`REPO_ROOT`，建 `mktemp -d` 作测试沙箱，注册 `trap cleanup EXIT`。
3. 调用被测脚本（如 `scripts/package-codex-plugin.sh`），用 `assert_*` 断言产物/退出码。

**链路二：hermes pytest 复刻历史 bug**

1. `conftest.py:11-26` 构造 `MagicMock` ctx，`register_skill` 对非 `Path` 参数 `raise AttributeError`。
2. `test_bootstrap.py:19-22` 断言注入 context 必须小于 Hermes 的 10000 字符溢出阈值。
3. `test_bootstrap.py:24-46` 测 `_strip_frontmatter` 与 `_skills_dir()` 两种布局解析。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|---|---|---|
| `TEST_PORT=3334` | brainstorm 测试服务端口（见 brainstorm-server-tests） | `tests/brainstorm-server/server.test.js:22` |
| `--integration/-i` | opencode 测试里需要真实 OpenCode 的子集才跑 | `tests/opencode/run-tests.sh:24` |
| pytest | hermes 目录唯一用 Python 测试框架 | `tests/hermes/` |

## 5. 错误与重试语义

- 所有 shell 测试 `set -euo pipefail`，断言失败即 `FAILURES++` 并最终非零退出；无重试。
- 测试隔离靠 `mktemp -d` + `trap cleanup EXIT`，不污染仓库工作树。
- hermes pytest 故意把"传 str 给 register_skill"做成硬失败，防止 hermes 静默禁用插件的历史 bug 回归。

## 6. 并发细节

- 顺序执行，无并行测试。`run-tests.sh` 串行遍历。
- 无锁、无后台服务（除 brainstorm-server-tests 子叶子会 spawn server）。

## 7. 系统边界

**In-Scope**：`tests/` 下 17 个平台测试目录及其脚本。

**Out-of-Scope**
- 真实各平台宿主程序（Claude Code/Codex/Cursor 等运行时）——外部产品，测试只做静态断言或本地脚本行为
- `evals/`（superpowers-evals 仓库，Quorum/GAUNTLET 行为评测）——不在本仓库源码内，是外部克隆目录
- 被测脚本本身的实现见对应叶子（package-codex-plugin、bump-version、lint-shell 等）

## 8. 与相邻子系统交互

- 被测对象：harness-packaging（codex/hermes/kimi）、project-operations（version-bump）、testing-evals 内部（brainstorm-server、shell-lint）。
- 不直接驱动 skills-core 内容；`explicit-skill-requests` 通过真实 prompt 间接验证技能触发。

## 9. 语言专项适配口径

主语言 TS/Node，但本叶子测试语言是 **bash 为主 + mjs/pytest 点缀**：
- architecture 图表达"测试矩阵 → 被测对象"的多对多映射。
- workflow 图表达"run-tests.sh 聚合 → 逐脚本断言 → 汇总"的执行流程。
- 不补 sequence/dataflow/lifecycle——测试是顺序脚本，无消息时序、无数据管道、无状态机。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 测试矩阵与被测对象 | `harness-integration-tests-architecture.html` | architecture | showcase |
| 平台测试执行流程 | `harness-integration-tests-workflow.html` | workflow | showcase |

- JSON IR 源：`json/harness-integration-tests-*.json`。
