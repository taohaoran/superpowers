# 技能行为与 shell 静态检查测试（skill-behavior-tests）

> 本文是 `testing-evals` 域下的叶子子系统文档。域级总览见 `../testing-evals.md`。
> 本文只展开"不依赖具体平台宿主、对技能脚本与仓库 shell 代码做静态/行为检查"的测试；
> 平台矩阵测试见 `../harness-integration-tests/harness-integration-tests.md`。
>
> 源码基准：superpowers main，commit `8ca22db`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| shell 脚本 lint | ShellCheck + `bash -n`/`sh -n` 语法检查，支持 `--all/--format/--strict` | `scripts/lint-shell.sh`（211 行） |
| lint-shell 自测 | 对 `lint-shell.sh` 自身行为做断言 | `tests/shell-lint/test-lint-shell.sh` |
| bump-version 行为测试 | 构造 fixture 仓，断言版本 bump/check/audit | `tests/version-bump/test-bump-version.sh` |
| 技能结构测试 | diagnosing-superpowers 技能文件结构 | `tests/diagnosing-superpowers/test-skill-structure.sh` |
| TDD/调试技能脚本 | systematic-debugging 找污染源、writing-skills 图渲染 | `tests/systematic-debugging/`、`tests/writing-skills/` |
| 钩子测试 | `hooks/session-start` 行为 | `tests/hooks/test-session-start.sh` |

## 2. 核心类型与接口清单

| 入口 | 位置 | 职责 |
|---|---|---|
| `collect_changed_shell_files()` | `lint-shell.sh:77-97` | 默认只 lint 相对 HEAD 改动 + 暂存 + 未跟踪的 shell 文件 |
| `collect_all_shell_files()` | `lint-shell.sh:67-75` | `--all` 时 lint 全部 `git ls-files` |
| `is_shell_file()` | `lint-shell.sh:26-40` | 按扩展名或 shebang 识别 shell 脚本 |
| `run_syntax_checks()` | `lint-shell.sh:123-141` | 按 shebang 选 `bash -n` 或 `sh -n` |
| `shellcheck_args` | `lint-shell.sh:205-208` | 默认 warning 级；`--strict` 额外开 5 个严规则 |

## 3. 关键调用链

**链路一：默认 lint（提交前）**

1. 解析参数：默认 lint 改动文件；`--format` 先 shfmt；`--strict` 加严规则（`lint-shell.sh:148-176`）。
2. 预检 `shellcheck` 在 PATH（`:178`）。
3. `collect_changed_shell_files`：`git diff HEAD` + `git diff --cached` + `git ls-files --others` 三路收集（`:82-96`）。
4. 若 `--format`，先 `shfmt -i 2 -ci -bn -w`（`:197-201`）。
5. 跑 `shellcheck --severity=warning --external-sources`，再跑语法检查（`:205-211`）。

**链路二：bump-version fixture 测试**

1. `test-bump-version.sh:24-40` 在 mktemp 里造 fixture 仓，复制 `bump-version.sh` 与最小 `.version-bump.json`。
2. 跑 bump/audit，断言版本字段被改写、drift 被检出。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|---|---|---|
| `--all` | lint 全部跟踪文件 | `lint-shell.sh:150-152` |
| `--format` | 先 shfmt `-i 2 -ci -bn` | `:197-201` |
| `--strict` | 额外 5 个 ShellCheck 检查 | `:206-208` |
| ShellCheck 严重度 | warning | `:205` |

## 5. 错误与重试语义

- `require_tool` 缺工具即 `die`；`set -euo pipefail`。
- 无重试。lint 是 CI 性质，失败留给人修。

## 6. 并发细节

- 顺序执行 shellcheck；无并行。

## 7. 系统边界

**In-Scope**：`scripts/lint-shell.sh`、`tests/shell-lint/`、`tests/version-bump/`、`tests/hooks/`、`tests/diagnosing-superpowers/`、`tests/systematic-debugging/`、`tests/writing-skills/`。

**Out-of-Scope**
- ShellCheck / shfmt / pytest 等外部工具本身
- `evals/`（行为评测外部仓）
- 被测脚本业务逻辑见各自叶子

## 8. 与相邻子系统交互

- 被测对象：project-operations（bump-version）、runtime-bootstrap（hooks/session-start）、skills-core（debugging/writing-skills 脚本）。
- 是 harness-integration-tests 之外的"仓库内静态质量门"。

## 9. 语言专项适配口径

本叶子是 **bash 工具脚本 + 对它们的行为测试**。
- architecture 图表达 lint 收集器与被测 shell 文件的关系。
- workflow 图表达"收集改动 → format → shellcheck → 语法检查"的 lint 流水线。
- 不补 sequence/dataflow/lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| lint 工具与被测脚本 | `skill-behavior-tests-architecture.html` | architecture | showcase |
| shell lint 流水线 | `skill-behavior-tests-workflow.html` | workflow | showcase |

- JSON IR 源：`json/skill-behavior-tests-*.json`。
