# superpowers 系统级总览

> 本文档是 superpowers 仓库（v6.4.2，commit `8ca22db`）的系统级架构总览。
> 源码基准：`package.json` 声明 `"type": "module"`，零 npm 运行时依赖。
> 输出目录规则：项目根已存在 `docs/`，故本次分析输出至 `docs_archify/architecture/`。

## 1. 项目概述

**定位**（README 原文）："Superpowers is a complete software development methodology for your coding agents, built on top of a set of composable skills and some initial instructions that make sure your agent uses them."

superpowers 是一套面向编码代理（Coding Agent）的完整软件开发方法论插件，核心由 15 个可组合技能（skills）和一组初始引导指令构成，确保代理在会话中自动触发并使用正确的技能。它不是一个独立运行的应用程序，而是一个**零依赖插件包**，通过各平台的钩子（hook）或插件入口注入到编码代理的运行时中。

| 维度 | 数据 |
|---|---|
| 版本 | 6.4.2 |
| 总文件数 | 229（排除 .git） |
| 技能文件 | `skills/` 74 个文件，15 个 SKILL.md 合计 3888 行 |
| 运行时代码 | `.opencode/plugins/superpowers.js` 383 行、`.pi/extensions/superpowers.ts` 121 行 |
| 脚本 | `scripts/` 4 个 sh 合计 1312 行、`hooks/` 53 行 |
| 支持平台 | 15+（Claude Code、Codex、Cursor、Devin、Gemini CLI、Kimi、OpenCode、Pi 等） |
| 许可证 | MIT |

**架构范式**：技能驱动（Skill-Driven）+ 多平台引导注入。核心价值在于 Markdown 格式的技能内容（行为塑造指令），运行时代码仅负责"在正确的时机加载正确的技能"。

## 2. 功能总览

按 5 个域组织功能模块：

| 域 | 核心功能 | 叶子子系统 |
|---|---|---|
| **skills-core**（技能内容核心） | 15 个技能覆盖完整开发方法论：设计细化、工作区隔离、计划编写与执行、并行子代理、TDD、系统化调试、代码审查、分支收尾、技能编写与故障诊断 | using-superpowers、brainstorming、writing-plans、executing-plans、test-driven-development、systematic-debugging、verification-before-completion、requesting-code-review、receiving-code-review、dispatching-parallel-agents、subagent-driven-development、using-git-worktrees、finishing-a-development-branch、writing-skills、diagnosing-superpowers |
| **runtime-bootstrap**（运行时引导） | 会话启动时注入 using-superpowers 引导指令，确保技能自动触发；OpenCode 插件与 Pi 扩展为平台特定入口 | session-start-hook、opencode-bootstrap、pi-extension |
| **harness-packaging**（多平台打包） | 将技能集打包为各平台插件格式，同步到 Codex 插件目录，维护 10+ 平台的插件清单 | codex-plugin-packaging、plugin-manifests |
| **testing-evals**（测试验证体系） | 17 个平台集成测试目录、brainstorming 服务端测试、shell 语法检查、技能行为评测（外部 superpowers-evals） | harness-integration-tests、brainstorm-server-tests、skill-behavior-tests |
| **project-operations**（发布与治理） | 版本号自动升级与发布、严格的 PR 贡献指南（94% 拒绝率）、项目文档与多平台 README | version-bump-and-release、community-governance、project-docs |

## 3. 解决的问题

| 用户痛点 | superpowers 解法 |
|---|---|
| 编码代理行为不可预测，每次会话质量波动大 | `using-superpowers` 规定"任何回复前必须先检查并调用技能"，通过 Skill Priority 机制（流程技能先于实现技能）强制结构化工作流 |
| 代理直接写代码，缺乏设计与计划环节 | `brainstorming` 技能在编码前强制设计细化（含 visual companion 可视化服务端），`writing-plans` 要求 2-5 分钟粒度任务计划 |
| 多任务并行时代理上下文混乱 | `dispatching-parallel-agents` 和 `subagent-driven-development` 提供子代理分派协议，含 task-brief/sdd-workspace/review-package 三个可执行脚本 |
| 调试随意、缺乏系统性 | `systematic-debugging` 提供结构化调试流程，`verification-before-completion` 要求完成前验证 |
| 代码审查流于形式 | `requesting-code-review` 按严重度分级报告，`receiving-code-review` 规范审查反馈处理 |
| 多平台适配成本高 | 统一技能内容 + 平台特定引导层（hook/插件/扩展），一套技能包适配 15+ 编码代理平台 |
| 技能本身质量难以保证 | `writing-skills` 元技能指导技能开发，`diagnosing-superpowers` 提供故障诊断与脱敏报告打包 |

## 4. 系统边界

### 上边界（用户/第三方接入）
- 编码代理平台（Claude Code、Codex App/CLI、Cursor、Devin CLI、Gemini CLI、Kimi Code、OpenCode、Pi 等）通过安装插件或配置 hook 接入 superpowers
- 开发者通过自然语言与代理交互，superpowers 技能在代理回复过程中自动触发
- **不在本仓库源码内**：各编码代理平台本身

### 下边界（基础设施）
- Git 仓库：`using-git-worktrees` 技能调用 `git worktree` 命令管理隔离工作区
- Node.js 运行时：brainstorming 的 visual companion 服务端（`server.cjs`）需要 Node ≥ 18
- **不在本仓库源码内**：Git、Node.js 运行时本身

### 内边界（本仓库 vs 扩展/外部）
- 本仓库包含：技能内容（`skills/`）、运行时引导（`hooks/`、`.opencode/`、`.pi/`）、打包脚本（`scripts/`）、测试（`tests/`）、平台插件清单（`.claude-plugin/` 等 10+ 目录）
- **不在本仓库源码内**：`evals/`（superpowers-evals 外部仓库，需单独克隆）、`using-superpowers` 引用的 `references/*-tools.md` 平台适配文件（v6.2.0 压缩清扫后已移除）

### 侧边界
- 多平台支持通过"统一技能内容 + 平台特定清单目录"实现，各平台插件清单为独立目录（`.codex-plugin/`、`.cursor-plugin/` 等）
- `.hermes-plugin/` 是唯一含 Python 代码（`__init__.py`）的平台插件

### 不做什么
- 不提供独立的 GUI 或 CLI 应用（仅作为插件运行）
- 不包含任何第三方 npm 运行时依赖（零依赖设计）
- 不实现具体的业务逻辑或领域功能（技能是通用方法论，非领域特定）
- 不包含 CI/CD 工作流（`.github/` 仅有 PR 模板与 Issue 模板，无 workflows）

## 5. 系统架构图说明

![系统架构图](system-architecture.html)

架构图按从左到右的四层布局：

1. **外部系统层**（最左）：编码代理平台、Git 仓库、superpowers-evals 外部评测仓库
2. **域服务层**（左中）：5 个域的后端组件——运行时引导层、技能内容核心、多平台打包、测试验证体系、发布与治理
3. **技能调度层**（右中）：`using-superpowers` 作为技能调度入口，按优先级触发流程技能集、实现技能集、元技能集
4. **扩展组件层**（最右）：visual companion 服务端（brainstorming 内置 Node HTTP 服务）、平台插件清单（10+ 平台目录）

关键交互路径：
- **主路径**（emphasis）：编码代理平台 → 运行时引导层（会话启动注入）→ 技能内容核心（加载技能集）→ 技能调度入口 → 流程/实现技能集
- **次要路径**（dashed）：测试验证体系 → superpowers-evals（行为评测）、多平台打包 → 平台插件清单（生成清单）

## 6. 核心时序图说明

![核心工作流时序图](system-sequence.html)

时序图描述一次完整的开发任务从发起到收尾的 11 步交互：

1. **发起任务**：开发者向编码代理平台发送开发请求
2. **会话启动触发**：平台触发 session-start hook，运行时引导层介入
3. **注入调度规则**：引导层向 `using-superpowers` 注入技能调度规则与优先级
4. **触发设计细化**：`using-superpowers` 按优先级首先触发 `brainstorming` 技能
5. **输出设计方案**：brainstorming 向开发者返回设计方案与技术权衡（含 visual companion 可视化）
6. **创建隔离 worktree**：触发 `using-git-worktrees` 创建隔离工作区
7. **git 操作**：worktree 技能调用 git 命令（异步，dashed）
8. **编写粒度计划**：触发 `writing-plans` 编写 2-5 分钟粒度任务计划
9. **并行/内联执行**：`executing-plans` 或 `subagent-driven-development` 执行计划任务
10. **TDD 后请求审查**：执行过程中遵循 TDD（RED-GREEN-REFACTOR），完成后请求代码审查
11. **报告审查结果**：`requesting-code-review` 按严重度向开发者报告审查结果，随后 `finishing-a-development-branch` 做分支收尾

## 7. 系统数据流图说明

![系统数据流图](system-dataflow.html)

数据流图按 5 个阶段描述引导注入与技能触发的数据流向：

1. **平台接入**：编码代理平台提供会话启动事件，插件清单目录提供各平台元数据
2. **运行时引导**：session-start hook 接收启动事件，平台插件入口加载技能路径
3. **技能调度**：`using-superpowers` 作为统一调度入口，接收引导指令与技能路径，按优先级排序
4. **技能执行**：调度入口分别触发流程技能集（brainstorming/plan/worktree）和实现技能集（SDD/TDD/debug/review）
5. **产出反馈**：流程技能集产出设计文档与计划，实现技能集产出代码与测试、审查报告

**关键数据特征**：
- 技能内容为 Markdown 格式，在运行时由代理解析执行，无需编译
- brainstorming 的 visual companion 服务端是唯一有状态的运行时组件（Node HTTP 服务，加载 Prime Radiant 商标，可用 `SUPERPOWERS_DISABLE_TELEMETRY` 环境变量关闭遥测）
- 所有产出物（设计文档、计划、代码、测试、审查报告）均写入文件系统，由 git 管理版本
