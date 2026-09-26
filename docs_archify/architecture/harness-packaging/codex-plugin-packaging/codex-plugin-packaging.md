# Codex 插件打包（codex-plugin-packaging）

> 本文是 `harness-packaging` 域下的叶子子系统文档。域级总览见 `../harness-packaging.md`，
> 本文只展开"把本仓库源码打包成可上传 Codex 门户的 rootless 归档 + 同步到 OpenAI 官方插件仓库"
> 这一条产线；各平台插件清单本身见相邻叶子 `../plugin-manifests/plugin-manifests.md`，
> 版本号统一管理见 `../../project-operations/version-bump-and-release/version-bump-and-release.md`。
>
> 源码基准：superpowers main，commit `8ca22db`（v6.4.2）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Codex 门户归档打包 | 用 `git archive` 从指定 git ref 取出白名单文件，打成 rootless zip/tar.gz 供门户上传 | `scripts/package-codex-plugin.sh:235-242` |
| OpenAI 元数据播种 | 从历史官方包中恢复 `skills/*/agents/openai.yaml`（OpenAI 私有元数据，不在本仓库源码内） | `scripts/package-codex-plugin.sh:260-282` |
| 确定性归档 | 固定 tar.umask=0022、统一 mtime（zip 1980/tar 1970）、ustar+uid/gid 0，保证同输入产出同字节 | `scripts/package-codex-plugin.sh:233-242, 292-319` |
| 出站路径守卫 | 打包后回读归档清单，若混入 hooks/tests/docs/scripts 等源码-only 路径则失败退出 | `scripts/package-codex-plugin.sh:334-341` |
| 同步到官方插件仓 | 把上游 checkout rsync 到 `prime-radiant-inc/openai-codex-plugins` 的 `plugins/superpowers/`，保留目的地元数据，开同步分支并提 PR | `scripts/sync-to-codex-plugin.sh:342-355, 402-461` |
| 忽略路径自动排除 | 把 git 未跟踪/已忽略路径自动转成 rsync exclude，避免本地临时文件进插件仓 | `scripts/sync-to-codex-plugin.sh:112-134` |
| Dry-run 预览 | 总是先打印 rsync `--itemize-changes` 预览，确认后才真正 apply | `scripts/sync-to-codex-plugin.sh:371-380` |
| 引导模式 | `--bootstrap` 在目的地插件目录缺失时首次创建，跳过"插件必须已存在"校验 | `scripts/sync-to-codex-plugin.sh:23-25, 285-294` |

## 2. 核心类型与接口清单

本叶子是两个独立可执行 shell 脚本，无自定义类型；核心"接口"是它们的命令行参数与约定产物。

| 入口/函数 | 位置 | 职责 |
|---|---|---|
| `usage()` | `package-codex-plugin.sh:22-46` | 打印打包脚本用法与归档内容白名单说明 |
| `prepare_metadata_root()` | `package-codex-plugin.sh:202-229` | 接受目录/zip/tar.gz 三种元数据源，解压并定位其中的 `skills/` 根 |
| `die()` | 两脚本均有 | 统一错误退出（`set -euo pipefail` 下的显式失败点） |
| `confirm()` | `sync-to-codex-plugin.sh:189-193` | 交互确认（`-y` 跳过），脏工作树/非 main 分支都要二次确认 |
| `prepare_sync_source()` | `sync-to-codex-plugin.sh:342-353` | 先把上游 rsync 到临时 overlay，再把目的地已有的 `openai.yaml` 覆盖回去（保留 OpenAI 元数据） |
| `copy_preserved_destination_metadata()` | `sync-to-codex-plugin.sh:327-340` | 递归复制目的地 `skills/**/agents/openai.yaml` 到 overlay 源 |
| `EXCLUDES` 数组 | `sync-to-codex-plugin.sh:45-83` | 锚定到根的 rsync 排除清单（防误伤 `skills/*/scripts/`） |

## 3. 关键调用链

**链路一：门户归档打包（`package-codex-plugin.sh`）**

1. 解析参数并推断格式；预检 `git/jq/tar/gzip/shasum/zip/unzip` 全部在 PATH（`package-codex-plugin.sh:133-141`）。
2. 校验 `--ref` 能解析到 commit；非 `--allow-dirty` 时要求工作树干净（`:144-154`）。
3. 定位元数据源：默认按 `../_tmp/sup-codex-packaging/` 下目录→zip→tar.gz 顺序找，找不到就 `--metadata-source` 报错（`:156-166`）。
4. `git archive -c tar.umask=0022` 白名单导出 `.codex-plugin/`、`assets/`、`skills/`、README、LICENSE、CODE_OF_CONDUCT 到暂存区（`:235-242`）。
5. 从暂存区 `.codex-plugin/plugin.json` 读 `version`，据此拼默认输出路径（`:244-256`）。
6. 遍历暂存 `skills/*`，把元数据源对应 `openai.yaml` 复制进去；缺元数据累计 `missing_metadata`，最后非零即失败（`:260-277`），并校验 skill 数 == 元数据文件数（`:279-282`）。
7. 生成归档清单，按格式打包：zip 统一 mtime=1980、tar.gz 用 ustar+uid/gid 0（GNU/BSD tar 标志分流）（`:284-319`）。
8. 回读归档路径，正则匹配源码-only 黑名单（hooks/scripts/tests/docs/.claude 等）命中即 `die`（`:334-341`），最后打印条目数与 SHA-256。

**链路二：同步到 OpenAI 插件仓（`sync-to-codex-plugin.sh`）**

1. 预检 `rsync/git/gh/python3` 且 `gh auth status` 成功；从 `.codex-plugin/plugin.json` 读 `UPSTREAM_VERSION`（`:171-187`）。
2. 非 main 分支或脏工作树时警告并要求确认（`:195-206`）。
3. `gh repo clone prime-radiant-inc/openai-codex-plugins` 到临时目录（或 `--local` 复用）；为 `--local` 再克隆一份 preview 副本并 overlay 本地未提交改动（`:220-288`）。
4. 拼 rsync 参数：固定 `-av --delete --delete-excluded` + `EXCLUDES` + git 忽略路径自动排除（`:322-325`）。
5. `prepare_sync_source`：上游 → overlay 源，再把目的地 `openai.yaml` 回贴（`:342-355`）。
6. 打印 `rsync --dry-run --itemize-changes` 预览；`-n` 到此退出（`:371-380`）。
7. 确认后切 `base` 分支、建新分支 `sync/superpowers-<shortsha>-<timestamp>`，正式 rsync；无变化则提前退出（`:402-416`）。
8. `git add` → `git commit` → `git push -u origin` → `gh pr create`，打印 PR 链接（`:422-468`）。

## 4. 配置项

| 参数/配置 | 默认/行为 | 位置 |
|---|---|---|
| `--ref` | HEAD | `package-codex-plugin.sh:15,80-84` |
| `--format` | zip（或由 `--output` 扩展名推断） | `:17,60-74,105-131` |
| `--metadata-source` | `../_tmp/sup-codex-packaging/` 下目录/zip/tar.gz 按序回退 | `:18,156-166` |
| `--allow-dirty` | 0（默认拒绝脏树打包） | `:19,147-154` |
| `FORK` | `prime-radiant-inc/openai-codex-plugins` | `sync-to-codex-plugin.sh:35` |
| `DEFAULT_BASE` | `main` | `:36` |
| `DEST_REL` | `plugins/superpowers` | `:37` |
| `-n / -y / --local / --base / --bootstrap` | 干跑 / 跳过确认 / 复用本地 checkout / 指定基分支 / 首次建目录 | `:155-159` |
| 归档白名单 | 仅 `.codex-plugin assets skills README LICENSE CODE_OF_CONDUCT` | `package-codex-plugin.sh:236-241` |

## 5. 错误与重试语义

- 两个脚本都 `set -euo pipefail`，任何命令非零即终止；没有自动重试——这是一次性发布脚本，失败留给人处理。
- 打包脚本的失败点：工具缺失（`:133-141`）、ref 不可解析（`:144-145`）、脏树（`:147-154`）、元数据源缺失（`:163-165`）、元数据不全（`:275-277`）、skill/元数据数不符（`:281-282`）、归档混入源码路径（`:338-341`）。
- 同步脚本的失败点：`gh` 未认证（`:176`）、缺 `.codex-plugin/plugin.json`（`:179`）、base 分支不存在（`:281,291`）、非 bootstrap 下插件目录缺失（`:286,293`）、本地 checkout 有未提交目的地改动（`:391-393`）。
- 幂等性：同步脚本注释明确"对同一上游 SHA 连跑两次应产出 diff 完全一致的 PR"（`:12-13,443`），作为脚本自身正确性的验证手段。
- 临时目录用 `trap cleanup EXIT` 清理；`--keep-stage` 保留暂存目录供排查。

## 6. 并发细节

- 纯顺序 shell 流程，无后台进程、无并行 rsync、无锁。
- 唯一"并发"语义是同步脚本的双 checkout 模型：preview checkout（干跑预览）与 apply checkout（真正提交）分离，避免预览改动污染真实提交；`--local` 模式下 preview 是 `--no-local` 克隆出的独立副本（`:273-279`）。
- rsync `--delete --delete-excluded` 保证目的地与源严格一致；`--exclude` 锚定 `/` 防止误伤嵌套 `skills/*/scripts/`（`:41-44` 注释）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `scripts/package-codex-plugin.sh`、`scripts/sync-to-codex-plugin.sh` 两个打包/同步脚本
- `.codex-plugin/plugin.json`（Codex 平台清单，被打包与同步）
- `assets/`、`skills/`、README/LICENSE/CODE_OF_CONDUCT 等被打包内容

**Out-of-Scope（不在本仓库源码内）**
- `prime-radiant-inc/openai-codex-plugins`（OpenAI 官方插件仓，被同步目的地，外部 GitHub 仓库）
- `skills/*/agents/openai.yaml`（OpenAI 私有元数据，需从历史官方包播种）
- Codex 门户（接收归档的上传站点，外部系统）
- `../_tmp/sup-codex-packaging/`（本地打包工作目录，不入库）
- 版本号本身的批量修改见 `version-bump-and-release` 叶子，本脚本只读版本

## 8. 与相邻子系统交互

- 上游依赖：本仓库源码树（`skills/`、`.codex-plugin/`、`assets/`）——内容来自 `skills-core` 域。
- 版本来源：`.codex-plugin/plugin.json` 的 `version` 字段，由 `version-bump-and-release` 叶子统一维护。
- 测试背书：`tests/codex/test-package-codex-plugin.sh`、`tests/codex/test-marketplace-manifest.sh`、`tests/codex-plugin-sync/test-sync-to-codex-plugin.sh`（属 `testing-evals` 域）。
- 下游产物：门户归档（zip/tar.gz）与同步 PR（指向 OpenAI 插件仓），最终用户通过 Codex 门户/插件市场安装。

## 9. 语言专项适配口径

主语言按 TS/Node 口径，但本叶子实际是 **bash 工具脚本**（无 npm 依赖、零运行时库）：
- 不涉及 capability seam/服务定义；"组件"是脚本、git、rsync、gh CLI 这一命令行工具链。
- 图型选择：静态工具链拓扑用 architecture；"打包/同步分步流程 + 分支/失败回退"用 workflow（比 sequence 更贴切，因为核心是步骤流转而非多对象互发消息）。
- 外部边界：OpenAI 插件仓、Codex 门户、gh CLI 认证会话均标 external。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 打包工具链架构 | `codex-plugin-packaging-architecture.html` | architecture | showcase |
| 同步到官方仓流程 | `codex-plugin-packaging-workflow.html` | workflow | showcase |

- JSON IR 源：`json/codex-plugin-packaging-architecture.json`、`json/codex-plugin-packaging-workflow.json`。
- 省略说明：本叶子不补 sequence 图——打包/同步是单脚本顺序步骤，已由 workflow 表达；不补 dataflow——无"数据加工管道"语义；不补 lifecycle——无单一实体状态机。
