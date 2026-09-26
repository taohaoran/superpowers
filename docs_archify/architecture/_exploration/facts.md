# superpowers 仓库探查事实（共享只读探查产出）

> 本文档由 MainAgent 在分析任务开始时一次性探查产出，供组织者与全部分片共享，
> 分片不要再各自重扫全仓库。路径一律以仓库根 `/Users/thr/Documents/AllProjects/OpenSource/superpowers` 为基准。

## 1. 身份

- 项目名：superpowers
- 定位（README 原文）："Superpowers is a complete software development methodology for your coding agents, built on top of a set of composable skills and some initial instructions that make sure your agent uses them."
- 版本：6.4.2（`package.json`，commit `8ca22db` "Release v6.4.2: leaner plans from writing-plans"）
- 包类型：Node ESM（`package.json` `"type": "module"`）；`main` 指向 `.opencode/plugins/superpowers.js`
- git 无 tag；LICENSE：MIT
- 构建/包管理：无 npm 运行时依赖（零依赖设计），无 go.mod/pom.xml

## 2. 规模

- 总文件数（排除 .git）：229
- 按扩展名分布：md 117、sh 45、json 16、js 11、py 6、mjs 4、ts 2、yml/yaml 4、txt 9、html 1、dot 1、cjs 1、svg/png 各 1
- 技能内容：`skills/` 74 个文件；15 个 SKILL.md 合计 3888 行（最大 writing-skills 681、subagent-driven-development 568、executing-plans 373）
- 运行时/工具脚本行数：hooks/session-start 53、.opencode/plugins/superpowers.js 383、.pi/extensions/superpowers.ts 121、scripts/ 4 个 sh 合计 1312（sync-to-codex-plugin.sh 468、package-codex-plugin.sh 351、bump-version.sh 282、lint-shell.sh 211）
- brainstorming 技能内置 visual companion 服务端：skills/brainstorming/scripts/server.cjs 723、helper.js 167、start/stop-server.sh 209/120、frame-template.html
- subagent-driven-development 内置脚本：scripts/task-brief、scripts/sdd-workspace、scripts/review-package（无扩展名可执行脚本）
- writing-skills 内置：render-graphs.js、graphviz-conventions.dot

## 3. 结构

```
superpowers/
├── skills/                  # 15 个技能（核心内容域）
├── hooks/                   # session-start、hooks.json、hooks-cursor.json、run-hook.cmd
├── scripts/                 # bump-version / lint-shell / package-codex-plugin / sync-to-codex-plugin
├── tests/                   # 17 个测试目录（claude-code/codex/cursor… 多平台矩阵）
├── .opencode/               # OpenCode 插件（plugins/superpowers.js + INSTALL.md）
├── .pi/                     # Pi 扩展（extensions/superpowers.ts）
├── .claude-plugin/ .codex-plugin/ .cursor-plugin/ .devin-plugin/
│   .hermes-plugin/ .kimi-plugin/ .muse-plugin/ .agents/   # 多平台插件清单
├── .github/                 # PR 模板、ISSUE_TEMPLATE、FUNDING（无 CI workflows）
├── docs/                    # 用户自有文档（plans/、superpowers/plans+specs、README.kimi/opencode、testing.md、porting、windows）
├── AGENTS.md / README.md / RELEASE-NOTES.md / CODE_OF_CONDUCT.md
└── assets/                  # app-icon.png、superpowers-small.svg
```

- `.hermes-plugin/` 含 `__init__.py` + `plugin.yaml`（Python 侧）；tests/hermes 为 pytest
- README 提及 evals/（superpowers-evals 克隆目录）——**不在本仓库源码内**（未出现在根目录）

## 4. 功能清单（README Feature List 摘录）

1. 基础工作流：brainstorming（设计细化）→ using-git-worktrees（隔离工作区）→ writing-plans（2-5 分钟粒度任务计划）→ subagent-driven-development 或 executing-plans（并行/内联执行）→ test-driven-development（RED-GREEN-REFACTOR）→ requesting-code-review（按严重度报告）→ finishing-a-development-branch（合并/PR 决策）
2. 技能自动触发：`using-superpowers` 规定"任何回复前必须先检查并调用技能"（Skill Priority：流程技能先于实现技能）
3. 故障诊断：`diagnosing-superpowers` 读会话转写、行级证据、打包脱敏报告
4. 技能库：Testing（test-driven-development）/ Debugging（systematic-debugging、verification-before-completion、diagnosing-superpowers）/ Collaboration（brainstorming、writing-plans、executing-plans、dispatching-parallel-agents、requesting-code-review、receiving-code-review、using-git-worktrees、finishing-a-development-branch、subagent-driven-development）/ Meta（writing-skills、using-superpowers）
5. 多平台支持：Claude Code、Antigravity、Codex App/CLI、Cursor、Devin CLI、Factory Droid、Gemini CLI、GitHub Copilot CLI、Grok Build CLI、Kimi Code、OpenCode、Pi、Qwen Code、Hermes Agent、Muse（README 安装章节逐平台命令）
6. 遥测：brainstorming visual companion 加载 Prime Radiant 商标（含版本号），可用 `SUPERPOWERS_DISABLE_TELEMETRY` 等环境变量关闭
7. 哲学：TDD、系统化而非临时、复杂度削减、证据而非声称

## 5. 约束

- 仓库根已存在 `docs/` → **输出根 = `docs_archify/architecture/`**
- `docs/architecture`、`docs_archify` 均不存在 → 首次分析，无基线，不产出 improve-comparison.md
- 输出物全部简体中文；文件名/目录名一律英文短横线（-），禁止中文短横线（用户级规则）
- 用户偏好：产出必须含顶层"系统级文档区"目录 + 各域目录，禁止扁平化输出
- AGENTS.md（项目级）：PR 必须完整填模板、先搜重复 PR、披露模型与插件、目标 dev 分支、零第三方依赖、不做技能"合规化"改写、批量/投机性 PR 拒绝——属于治理域事实，不在本分析中修改仓库
- using-superpowers 技能引用的 `references/claude-code-tools.md` 等平台适配文件**在当前仓库中不存在**（已被压缩清扫移除，v6.2.0 release note 提及 "skills compression sweep"）——分析时可作为事实记录

## 6. 主语言判定

混合仓库：Node/TS 运行时引导（.opencode/plugins/superpowers.js、.pi/extensions/superpowers.ts、index.js）+ shell（hooks、scripts、多数测试）+ Markdown 技能内容（核心价值）+ Python（hermes 插件与测试）。
**主语言按 TS/Node 口径**读取 `references/language-ts.md`（capability seam 分组、architecture+sequence+dataflow 为主、外部边界标注、包发布+依赖图表达）；shell 与 Markdown 内容在对应叶子中按各自语言专项说明。交付说明须披露该口径。

## 7. 域与叶子盘点（26 叶子 / 5 域）

| 域 | 叶子 |
|---|---|
| skills-core（技能内容，skills/） | using-superpowers、brainstorming（含 visual companion 服务端）、writing-plans、executing-plans、test-driven-development、systematic-debugging、verification-before-completion、requesting-code-review、receiving-code-review、dispatching-parallel-agents、subagent-driven-development（含 3 个可执行脚本）、using-git-worktrees、finishing-a-development-branch、writing-skills、diagnosing-superpowers（15） |
| runtime-bootstrap（运行时引导） | session-start-hook、opencode-bootstrap、pi-extension（3） |
| harness-packaging（多平台打包） | codex-plugin-packaging、plugin-manifests（2） |
| testing-evals（测试验证体系） | harness-integration-tests、brainstorm-server-tests、skill-behavior-tests（3） |
| project-operations（发布/治理/文档） | version-bump-and-release、community-governance、project-docs（3） |
