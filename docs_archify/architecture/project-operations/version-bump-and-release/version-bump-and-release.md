# 版本号与发布（version-bump-and-release）

> 本文是 `project-operations` 域下的叶子子系统文档。域级总览见 `../project-operations.md`。
> 本文只展开"版本号如何跨 11 处清单同步、发布说明如何维护"；清单本身见
> `../../harness-packaging/plugin-manifests/plugin-manifests.md`。
>
> 源码基准：superpowers main，commit `8ca22db`（v6.4.2）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 跨清单版本 bump | 按 `.version-bump.json` 声明批量改写 11 个文件的版本字段 | `scripts/bump-version.sh:226-256` |
| drift 检测 | `--check` 报告各清单当前版本并比对是否一致 | `scripts/bump-version.sh:116-152` |
| 漏改审计 | `--audit` 全仓 grep 版本字符串，找出未登记的版本引用 | `scripts/bump-version.sh:154-224` |
| JSON 字段读写 | jq 点路径读写，保留格式 | `scripts/bump-version.sh:26-41` |
| YAML 字段读写 | yq 读写 `.hermes-plugin/plugin.yaml` | `scripts/bump-version.sh:50-60` |
| 发布说明 | 按版本倒序记录变更与 PR 编号 | `RELEASE-NOTES.md`（102KB） |

## 2. 核心类型与接口清单

| 函数 | 位置 | 职责 |
|---|---|---|
| `read_json_field/write_json_field` | `bump-version.sh:26-41` | 点路径（含 `plugins.0.version`）↔ jq 路径转换 |
| `read_yaml_field/write_yaml_field` | `:50-60` | yq 读写 |
| `read/write_manifest_field` | `:62-86` | 按文件扩展名分流 json/yaml |
| `declared_files()` | `:90-92` | 从 `.version-bump.json` 读 `path\tfield` 列表 |
| `cmd_check/cmd_audit/cmd_bump` | `:116/154/226` | 三个子命令 |

## 3. 关键调用链

1. `cmd_bump <new>` 先校验 semver 形如 `X.Y.Z`（`:230-233`），再 `preflight_manifests` 确认所有声明清单可读（`:94-107`）。
2. 遍历 `declared_files`，逐个 `read_manifest_field` 取旧值、`write_manifest_field` 写新值（`:240-250`）。
3. bump 完自动跑 `cmd_audit`：以"出现次数最多的版本"为当前版本，全仓 grep，剔除 audit.exclude 后列出未登记文件（`:255, 191-223`）。

## 4. 配置项

| 配置 | 默认/行为 | 位置 |
|---|---|---|
| `.version-bump.json:files` | 11 个 `{path, field}` | `.version-bump.json` |
| `audit.exclude` | CHANGELOG/RELEASE-NOTES/evals/.git 等 | `.version-bump.json` |
| 子命令 | `<new>` / `--check` / `--audit` | `bump-version.sh:260-282` |

## 5. 错误与重试语义

- `set -euo pipefail`；缺工具（jq/yq）即错；drift 时 `--check` 返回非零。
- 无重试。失败即人介入。

## 6. 并发细节

顺序脚本，无并发。

## 7. 系统边界

**In-Scope**：`scripts/bump-version.sh`、`.version-bump.json`、`RELEASE-NOTES.md`、`package.json` 版本字段。

**Out-of-Scope**
- 11 个被改清单的内容本身（见 plugin-manifests 叶子）
- git tag/打包/推送（本脚本只改文件，不发版；Codex 归档见 harness-packaging）

## 8. 与相邻子系统交互

- 改写对象：harness-packaging 域全部平台清单。
- 测试背书：`tests/version-bump/test-bump-version.sh`（testing-evals）。

## 9. 语言专项适配口径

bash 脚本。architecture 图表达配置→脚本→11 清单；workflow 图表达 bump→audit 流程。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 版本 bump 工具与清单 | `version-bump-and-release-architecture.html` | architecture | showcase |
| bump 加 audit 流程 | `version-bump-and-release-workflow.html` | workflow | showcase |

- JSON IR 源：`json/version-bump-and-release-*.json`。
