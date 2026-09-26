# testing-evals 域总览

> 本域回答：**superpowers 如何在 17 个平台/集成面上验证自己没坏？**
> 输出根：`docs_archify/architecture/`。源码基准：superpowers main，commit `8ca22db`。

## 域职责

testing-evals 是仓库内的**测试与静态质量层**：`tests/` 下 17 个目录，覆盖多平台集成、brainstorming 服务端、shell/技能脚本行为。它不做线上行为评测（那是外部 `evals/` 仓库的事），只做可在本机 bash/node/pytest 跑起来的确定性测试。

## 叶子索引

| 叶子 | 职责 | 主要产物 |
|---|---|---|
| [harness-integration-tests](harness-integration-tests/harness-integration-tests.md) | 17 个平台/集成目录的测试矩阵 | `tests/{claude-code,codex,opencode,hermes,...}/` |
| [brainstorm-server-tests](brainstorm-server-tests/brainstorm-server-tests.md) | brainstorming visual companion 服务端集成测试 | `tests/brainstorm-server/`（10 文件） |
| [skill-behavior-tests](skill-behavior-tests/skill-behavior-tests.md) | shell lint 与技能脚本行为测试 | `scripts/lint-shell.sh`、`tests/{shell-lint,version-bump,hooks,...}/` |

## 域级机制

### 1. 三种测试语言并存
- **bash 断言框架**：`pass/fail/assert_equals` + mktemp 沙箱，占绝大多数平台测试。
- **Node `assert` + `ws`**：专测 brainstorm 服务端，spawn 真实子进程。
- **pytest**：唯一用于 `.hermes-plugin` Python 引导，用 mock 复刻历史 bug 契约。

### 2. 测试分层
- 平台矩阵测试（harness-integration）断言脚本/清单行为，不依赖真宿主。
- 服务端测试（brainstorm-server）起真实子进程做 HTTP/WS 集成。
- 静态行为测试（skill-behavior）在提交前跑 ShellCheck/syntax。

### 3. 外部边界
`evals/`（superpowers-evals，Quorum/GAUNTLET 行为评测）**不在本仓库源码内**，是独立克隆目录；本域不覆盖它。

## 图表清单

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 域内测试分层 | `testing-evals-architecture.html` | architecture | showcase |
| 测试执行总流程 | `testing-evals-workflow.html` | workflow | showcase |
| 测试数据沙箱隔离 | `testing-evals-dataflow.html` | dataflow | showcase |

- JSON IR：`json/testing-evals-*.json`。
