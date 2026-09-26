# skills-core（技能内容核心）域总览

> 本域是 superpowers 的核心价值所在，包含 15 个技能（SKILL.md），覆盖完整的软件开发方法论。
> 各叶子详情见对应文档。源码基准：superpowers v6.4.2，commit `8ca22db`。

## 1. 域职责

skills-core 域定义了编码代理在软件开发全流程中应遵循的行为规范与方法论。技能以 Markdown 格式编写，在运行时由代理根据 `using-superpowers` 的调度规则自动触发。核心设计哲学：

- **Skill Priority**：流程技能（设计、计划、隔离）优先于实现技能（编码、测试、审查）
- **证据而非声称**：所有结论必须有源码或测试证据支撑
- **系统化而非临时**：调试、审查、验证均有结构化流程
- **复杂度削减**：YAGNI 原则、最小可行设计

域内源码路径：`skills/` 目录，共 74 个文件，15 个 SKILL.md 合计 3888 行。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 第二图 | 职责一句话 |
|---|---|---|---|---|
| using-superpowers | [using-superpowers.md](using-superpowers/using-superpowers.md) | [架构图](using-superpowers/using-superpowers-architecture.html) | [工作流](using-superpowers/using-superpowers-workflow.html) | 技能调度入口，规定回复前必须检查并调用技能 |
| brainstorming | [brainstorming.md](brainstorming/brainstorming.md) | [架构图](brainstorming/brainstorming-architecture.html) | [时序图](brainstorming/brainstorming-sequence.html) | 编码前设计细化，含 visual companion 可视化服务端 |
| writing-plans | [writing-plans.md](writing-plans/writing-plans.md) | [架构图](writing-plans/writing-plans-architecture.html) | [工作流](writing-plans/writing-plans-workflow.html) | 编写 2-5 分钟粒度任务计划 |
| executing-plans | [executing-plans.md](executing-plans/executing-plans.md) | [架构图](executing-plans/executing-plans-architecture.html) | [工作流](executing-plans/executing-plans-workflow.html) | 内联执行计划任务，含 task-start/task-done 脚本 |
| test-driven-development | [test-driven-development.md](test-driven-development/test-driven-development.md) | [架构图](test-driven-development/test-driven-development-architecture.html) | [时序图](test-driven-development/test-driven-development-sequence.html) | TDD RED-GREEN-REFACTOR 循环 |
| systematic-debugging | [systematic-debugging.md](systematic-debugging/systematic-debugging.md) | [架构图](systematic-debugging/systematic-debugging-architecture.html) | [时序图](systematic-debugging/systematic-debugging-sequence.html) | 四阶段系统化调试流程 |
| verification-before-completion | [verification-before-completion.md](verification-before-completion/verification-before-completion.md) | [架构图](verification-before-completion/verification-before-completion-architecture.html) | [时序图](verification-before-completion/verification-before-completion-sequence.html) | 完成前验证门禁 |
| requesting-code-review | [requesting-code-review.md](requesting-code-review/requesting-code-review.md) | [架构图](requesting-code-review/requesting-code-review-architecture.html) | [时序图](requesting-code-review/requesting-code-review-sequence.html) | 按严重度分级请求代码审查 |
| receiving-code-review | [receiving-code-review.md](receiving-code-review/receiving-code-review.md) | [架构图](receiving-code-review/receiving-code-review-architecture.html) | [工作流](receiving-code-review/receiving-code-review-workflow.html) | 六步响应审查反馈，含 YAGNI 检查 |
| dispatching-parallel-agents | [dispatching-parallel-agents.md](dispatching-parallel-agents/dispatching-parallel-agents.md) | [架构图](dispatching-parallel-agents/dispatching-parallel-agents-architecture.html) | [时序图](dispatching-parallel-agents/dispatching-parallel-agents-sequence.html) | 并行子代理分派协议 |
| subagent-driven-development | [subagent-driven-development.md](subagent-driven-development/subagent-driven-development.md) | [架构图](subagent-driven-development/subagent-driven-development-architecture.html) | [工作流](subagent-driven-development/subagent-driven-development-workflow.html) | 子代理驱动开发，含 task-brief/sdd-workspace/review-package 脚本 |
| using-git-worktrees | [using-git-worktrees.md](using-git-worktrees/using-git-worktrees.md) | [架构图](using-git-worktrees/using-git-worktrees-architecture.html) | [工作流](using-git-worktrees/using-git-worktrees-workflow.html) | git worktree 隔离工作区管理 |
| finishing-a-development-branch | [finishing-a-development-branch.md](finishing-a-development-branch/finishing-a-development-branch.md) | [架构图](finishing-a-development-branch/finishing-a-development-branch-architecture.html) | [工作流](finishing-a-development-branch/finishing-a-development-branch-workflow.html) | 绿测试闸门、分支合并/PR 决策、worktree 清理 |
| writing-skills | [writing-skills.md](writing-skills/writing-skills.md) | [架构图](writing-skills/writing-skills-architecture.html) | [时序图](writing-skills/writing-skills-sequence.html) | 元技能：指导技能开发与测试，含 render-graphs.js |
| diagnosing-superpowers | [diagnosing-superpowers.md](diagnosing-superpowers/diagnosing-superpowers.md) | [架构图](diagnosing-superpowers/diagnosing-superpowers-architecture.html) | [时序图](diagnosing-superpowers/diagnosing-superpowers-sequence.html) | 读会话转写、行级证据、打包脱敏故障报告 |

## 3. 域级机制细节

### 3.1 Skill Priority 调度机制

`using-superpowers`（`skills/using-superpowers/SKILL.md:10-16`）定义了强制规则：代理在任何回复前必须先检查当前情境是否匹配某个技能的触发条件，并按优先级调用。优先级顺序为：

1. **流程技能**（先于实现技能）：brainstorming → using-git-worktrees → writing-plans → executing-plans → finishing-a-development-branch
2. **实现技能**：test-driven-development → systematic-debugging → verification-before-completion → requesting/receiving-code-review → dispatching-parallel-agents → subagent-driven-development
3. **元技能**：writing-skills → diagnosing-superpowers（仅在技能开发或故障时触发）

### 3.2 技能间数据传递

技能之间通过文件系统传递上下文：
- `writing-plans` 产出计划文件（含 Plan Header + Task 列表），`executing-plans` 和 `subagent-driven-development` 读取该文件执行
- `subagent-driven-development` 的 `scripts/sdd-workspace`（`:61-79`）通过 plan-path 参数防止多任务串台
- `requesting-code-review` 取当前 SHA 并 dispatch 给 reviewer，`receiving-code-review` 处理返回的审查结果
- `finishing-a-development-branch` 读取测试状态（三环境：本地/CI/预发）决定是否可合并

### 3.3 内置可执行脚本

本域包含两类内置脚本：
- **brainstorming visual companion**：`skills/brainstorming/scripts/server.cjs`（723 行 Node HTTP 服务）、`helper.js`、`start-server.sh`、`stop-server.sh`、`frame-template.html`。服务端通过 WebSocket 帧编解码（`server.cjs:40-81`）与代理通信，含鉴权（`:341-353`）、fs.watch 文件监听（`:590-613`）、看门狗（`:634-657`）
- **SDD 脚本**：`skills/subagent-driven-development/scripts/task-brief`（awk 抽取 Task N，`:30-41`）、`sdd-workspace`（plan-path 防串台，`:61-79`）、`review-package`（范围守卫，`:27-28`）
- **writing-skills 脚本**：`skills/writing-skills/render-graphs.js`（graphviz 渲染）、`graphviz-conventions.dot`

### 3.4 Red Flags 与防护机制

多个技能内置 Red Flags 表，用于检测代理行为偏差：
- `using-superpowers`（`:33-51`）：检测跳过技能、直接编码等行为
- `writing-plans`（`:163-177`）：计划自审 5 项
- `subagent-driven-development`（`:373-429`）：fix loop 5 轮上限与 breaker 机制
- `finishing-a-development-branch`（`:132-157`）：discard 操作需字面确认

## 4. 域级图

![skills-core 域架构图](skills-core-architecture.html)

![skills-core 核心开发工作流](skills-core-workflow.html)

![skills-core 技能触发优先级时序](skills-core-sequence.html)

**质量档位**：架构图 standard（15 组件跨 3 分组，连线密度较高为预期内）；工作流图 showcase；时序图 showcase。
