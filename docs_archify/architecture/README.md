# superpowers 系统架构文档

> 基于 superpowers 源码（v6.4.2，commit `8ca22db`，229 个文件，15 个技能合计 3888 行）深度分析产出，
> 覆盖系统级、5 个域、26 个叶子子系统的功能、问题域、系统边界、架构图、时序图与数据流图。
> 所有图表由 archify 渲染为自包含交互式 HTML。
>
> 输出目录说明：项目根已存在 `docs/`（用户自有文档），故本次分析输出至 `docs_archify/architecture/`。

## 文档导航

### 系统级

| 文档 | 说明 | 图表 |
|------|------|------|
| [system-overview.md](system-overview.md) | 功能总览、解决的问题、系统边界、核心代码映射 | [系统架构图](system-architecture.html) · [核心时序图](system-sequence.html) · [数据流图](system-dataflow.html) |

### skills-core（技能内容核心，15 叶子）

| 子系统 | 文档 | 架构图 | 第二图 | 职责 |
|--------|------|--------|--------|------|
| 域总览 | [skills-core.md](skills-core/skills-core.md) | [架构图](skills-core/skills-core-architecture.html) | [工作流](skills-core/skills-core-workflow.html) · [时序](skills-core/skills-core-sequence.html) | 15 技能调度与方法论 |
| using-superpowers | [MD](skills-core/using-superpowers/using-superpowers.md) | [架构图](skills-core/using-superpowers/using-superpowers-architecture.html) | [工作流](skills-core/using-superpowers/using-superpowers-workflow.html) | 技能调度入口 |
| brainstorming | [MD](skills-core/brainstorming/brainstorming.md) | [架构图](skills-core/brainstorming/brainstorming-architecture.html) | [时序图](skills-core/brainstorming/brainstorming-sequence.html) | 设计细化+visual companion |
| writing-plans | [MD](skills-core/writing-plans/writing-plans.md) | [架构图](skills-core/writing-plans/writing-plans-architecture.html) | [工作流](skills-core/writing-plans/writing-plans-workflow.html) | 编写粒度计划 |
| executing-plans | [MD](skills-core/executing-plans/executing-plans.md) | [架构图](skills-core/executing-plans/executing-plans-architecture.html) | [工作流](skills-core/executing-plans/executing-plans-workflow.html) | 内联执行计划 |
| test-driven-development | [MD](skills-core/test-driven-development/test-driven-development.md) | [架构图](skills-core/test-driven-development/test-driven-development-architecture.html) | [时序图](skills-core/test-driven-development/test-driven-development-sequence.html) | TDD 循环 |
| systematic-debugging | [MD](skills-core/systematic-debugging/systematic-debugging.md) | [架构图](skills-core/systematic-debugging/systematic-debugging-architecture.html) | [时序图](skills-core/systematic-debugging/systematic-debugging-sequence.html) | 系统化调试 |
| verification-before-completion | [MD](skills-core/verification-before-completion/verification-before-completion.md) | [架构图](skills-core/verification-before-completion/verification-before-completion-architecture.html) | [时序图](skills-core/verification-before-completion/verification-before-completion-sequence.html) | 完成前验证 |
| requesting-code-review | [MD](skills-core/requesting-code-review/requesting-code-review.md) | [架构图](skills-core/requesting-code-review/requesting-code-review-architecture.html) | [时序图](skills-core/requesting-code-review/requesting-code-review-sequence.html) | 请求代码审查 |
| receiving-code-review | [MD](skills-core/receiving-code-review/receiving-code-review.md) | [架构图](skills-core/receiving-code-review/receiving-code-review-architecture.html) | [工作流](skills-core/receiving-code-review/receiving-code-review-workflow.html) | 接收审查反馈 |
| dispatching-parallel-agents | [MD](skills-core/dispatching-parallel-agents/dispatching-parallel-agents.md) | [架构图](skills-core/dispatching-parallel-agents/dispatching-parallel-agents-architecture.html) | [时序图](skills-core/dispatching-parallel-agents/dispatching-parallel-agents-sequence.html) | 并行代理分派 |
| subagent-driven-development | [MD](skills-core/subagent-driven-development/subagent-driven-development.md) | [架构图](skills-core/subagent-driven-development/subagent-driven-development-architecture.html) | [工作流](skills-core/subagent-driven-development/subagent-driven-development-workflow.html) | 子代理驱动开发 |
| using-git-worktrees | [MD](skills-core/using-git-worktrees/using-git-worktrees.md) | [架构图](skills-core/using-git-worktrees/using-git-worktrees-architecture.html) | [工作流](skills-core/using-git-worktrees/using-git-worktrees-workflow.html) | worktree 隔离 |
| finishing-a-development-branch | [MD](skills-core/finishing-a-development-branch/finishing-a-development-branch.md) | [架构图](skills-core/finishing-a-development-branch/finishing-a-development-branch-architecture.html) | [工作流](skills-core/finishing-a-development-branch/finishing-a-development-branch-workflow.html) | 分支收尾 |
| writing-skills | [MD](skills-core/writing-skills/writing-skills.md) | [架构图](skills-core/writing-skills/writing-skills-architecture.html) | [时序图](skills-core/writing-skills/writing-skills-sequence.html) | 元技能开发 |
| diagnosing-superpowers | [MD](skills-core/diagnosing-superpowers/diagnosing-superpowers.md) | [架构图](skills-core/diagnosing-superpowers/diagnosing-superpowers-architecture.html) | [时序图](skills-core/diagnosing-superpowers/diagnosing-superpowers-sequence.html) | 故障诊断 |

### runtime-bootstrap（运行时引导，3 叶子）

| 子系统 | 文档 | 架构图 | 第二图 | 职责 |
|--------|------|--------|--------|------|
| 域总览 | [runtime-bootstrap.md](runtime-bootstrap/runtime-bootstrap.md) | [架构图](runtime-bootstrap/runtime-bootstrap-architecture.html) | [时序](runtime-bootstrap/runtime-bootstrap-sequence.html) · [数据流](runtime-bootstrap/runtime-bootstrap-dataflow.html) | 跨平台引导注入 |
| session-start-hook | [MD](runtime-bootstrap/session-start-hook/session-start-hook.md) | [架构图](runtime-bootstrap/session-start-hook/session-start-hook-architecture.html) | [时序图](runtime-bootstrap/session-start-hook/session-start-hook-sequence.html) | 会话启动钩子 |
| opencode-bootstrap | [MD](runtime-bootstrap/opencode-bootstrap/opencode-bootstrap.md) | [架构图](runtime-bootstrap/opencode-bootstrap/opencode-bootstrap-architecture.html) | [时序图](runtime-bootstrap/opencode-bootstrap/opencode-bootstrap-sequence.html) | OpenCode 插件入口 |
| pi-extension | [MD](runtime-bootstrap/pi-extension/pi-extension.md) | [架构图](runtime-bootstrap/pi-extension/pi-extension-architecture.html) | [时序图](runtime-bootstrap/pi-extension/pi-extension-sequence.html) | Pi 平台扩展 |

### harness-packaging（多平台打包，2 叶子）

| 子系统 | 文档 | 架构图 | 第二图 | 职责 |
|--------|------|--------|--------|------|
| 域总览 | [harness-packaging.md](harness-packaging/harness-packaging.md) | [架构图](harness-packaging/harness-packaging-architecture.html) | [工作流](harness-packaging/harness-packaging-workflow.html) · [数据流](harness-packaging/harness-packaging-dataflow.html) | 多平台插件打包 |
| codex-plugin-packaging | [MD](harness-packaging/codex-plugin-packaging/codex-plugin-packaging.md) | [架构图](harness-packaging/codex-plugin-packaging/codex-plugin-packaging-architecture.html) | [工作流](harness-packaging/codex-plugin-packaging/codex-plugin-packaging-workflow.html) | Codex 插件打包同步 |
| plugin-manifests | [MD](harness-packaging/plugin-manifests/plugin-manifests.md) | [架构图](harness-packaging/plugin-manifests/plugin-manifests-architecture.html) | [数据流](harness-packaging/plugin-manifests/plugin-manifests-dataflow.html) | 10+ 平台清单 |

### testing-evals（测试验证体系，3 叶子）

| 子系统 | 文档 | 架构图 | 第二图 | 职责 |
|--------|------|--------|--------|------|
| 域总览 | [testing-evals.md](testing-evals/testing-evals.md) | [架构图](testing-evals/testing-evals-architecture.html) | [工作流](testing-evals/testing-evals-workflow.html) · [数据流](testing-evals/testing-evals-dataflow.html) | 测试与评测体系 |
| harness-integration-tests | [MD](testing-evals/harness-integration-tests/harness-integration-tests.md) | [架构图](testing-evals/harness-integration-tests/harness-integration-tests-architecture.html) | [工作流](testing-evals/harness-integration-tests/harness-integration-tests-workflow.html) | 17 平台集成测试 |
| brainstorm-server-tests | [MD](testing-evals/brainstorm-server-tests/brainstorm-server-tests.md) | [架构图](testing-evals/brainstorm-server-tests/brainstorm-server-tests-architecture.html) | [时序图](testing-evals/brainstorm-server-tests/brainstorm-server-tests-sequence.html) | visual companion 测试 |
| skill-behavior-tests | [MD](testing-evals/skill-behavior-tests/skill-behavior-tests.md) | [架构图](testing-evals/skill-behavior-tests/skill-behavior-tests-architecture.html) | [工作流](testing-evals/skill-behavior-tests/skill-behavior-tests-workflow.html) | 技能行为与 lint |

### project-operations（发布与治理，3 叶子）

| 子系统 | 文档 | 架构图 | 第二图 | 职责 |
|--------|------|--------|--------|------|
| 域总览 | [project-operations.md](project-operations/project-operations.md) | [架构图](project-operations/project-operations-architecture.html) | [工作流](project-operations/project-operations-workflow.html) · [数据流](project-operations/project-operations-dataflow.html) | 版本发布与社区治理 |
| version-bump-and-release | [MD](project-operations/version-bump-and-release/version-bump-and-release.md) | [架构图](project-operations/version-bump-and-release/version-bump-and-release-architecture.html) | [工作流](project-operations/version-bump-and-release/version-bump-and-release-workflow.html) | 版本号升级发布 |
| community-governance | [MD](project-operations/community-governance/community-governance.md) | [架构图](project-operations/community-governance/community-governance-architecture.html) | [工作流](project-operations/community-governance/community-governance-workflow.html) | PR 模板与贡献指南 |
| project-docs | [MD](project-operations/project-docs/project-docs.md) | [架构图](project-operations/project-docs/project-docs-architecture.html) | [数据流](project-operations/project-docs/project-docs-dataflow.html) | 项目文档体系 |

## 产出统计

| 层级 | MD 文档 | HTML 图 | JSON IR |
|------|---------|---------|---------|
| 系统级 | 1（system-overview） | 3 | 3 |
| skills-core 域 | 1（域总览）+ 15（叶子） | 3（域级）+ 30（叶子级） | 33 |
| runtime-bootstrap 域 | 1 + 3 | 3 + 6 | 9 |
| harness-packaging 域 | 1 + 2 | 3 + 4 | 7 |
| testing-evals 域 | 1 + 3 | 3 + 6 | 9 |
| project-operations 域 | 1 + 3 | 3 + 6 | 9 |
| **合计** | **33** | **70** | **70** |

## 覆盖范围与说明

- **叶子数**：26 个，与规划一致
- **语言适配口径**：主语言按 TS/Node（capability seam 分组、architecture+sequence+dataflow 为主）；Markdown 技能内容按图型语义补 workflow/sequence；shell 脚本在对应叶子说明；hermes 插件为 Python
- **质量档位**：系统级架构图 standard（组件多为预期内），时序图/数据流图 showcase；域级图多数 showcase，部分 standard 已披露；叶子级图以 showcase 为主，降档均在对应叶子 MD 第 10 节披露失败检查名与修复动作
- **外部系统标注**：编码代理平台、Git、Node.js、superpowers-evals 仓库、graphviz 均标注"不在本仓库源码内"
- **事实校正**：`using-superpowers` 引用的 `references/*-tools.md` 平台适配文件在当前仓库不存在（v6.2.0 压缩清扫移除），已在对应叶子 MD 记录
- **未修改仓库任何源码**，仅在 `docs_archify/` 下新增产物
