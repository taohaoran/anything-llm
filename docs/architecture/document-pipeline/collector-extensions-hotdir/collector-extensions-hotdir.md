# Collector 扩展机制与 Hotdir 目录（collector-extensions-hotdir）

> 本文是 `document-pipeline` 域下的叶子子系统文档。域级总览见 `../document-pipeline.md`。
> 本文只展开 collector 的扩展端点（resync/repo/youtube/confluence 等）与 hotdir 监控目录机制。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 扩展挂载入口 | `extensions(app)` 在主 app 上挂载所有 `/ext/*` 路由 | `collector/extensions/index.js:9` |
| 文档重同步 | `/ext/resync-source-document` 按 type 调 `RESYNC_METHODS` | `collector/extensions/index.js:13-31` |
| 仓库加载 | `/ext/:repo_platform-repo` 动态解析 repo loader（GitHub/GitLab 等） | `collector/extensions/index.js:33-55` |
| 仓库分支查询 | `/ext/:repo_platform-repo/branches` | `extensions/index.js:58-77` |
| YouTube 转录 | `/ext/youtube-transcript` | `extensions/index.js:79-103` |
| 网站深度抓取 | `/ext/website-depth` 按 depth/maxLinks 抓整站 | `extensions/index.js:105-123` |
| Confluence | `/ext/confluence` | `extensions/index.js:125-142` |
| Drupal Wiki | `/ext/drupalwiki` | `extensions/index.js:144-161` |
| Obsidian vault | `/ext/obsidian/vault` | `extensions/index.js:163-178` |
| Paperless-ngx | `/ext/paperless-ngx` | `extensions/index.js:180-197` |
| 签名中间件 | `setDataSigner` 为扩展响应签名 | `collector/middleware/setDataSigner.js` |
| Hotdir 目录 | `hotdir/` 为上传文件暂存目录，启动时清空 | `collector/hotdir/__HOTDIR__.md`、`index.js:218` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `extensions(app)` | `extensions/index.js:9` | 扩展路由注册函数 |
| `RESYNC_METHODS` | `extensions/resync/` | 各 source type 的重同步方法表 |
| `resolveRepoLoader(platform)` / `resolveRepoLoaderFunction()` | `utils/extensions/RepoLoader.js` | 按平台名动态 require 对应 repo loader 类 |
| `loadObsidianVault`/`loadConfluence`/`loadYouTubeTranscript`/`websiteDepth` | `utils/extensions/` 各文件 | 各扩展加载器 |

## 3. 关键调用链

1. **扩展请求**：server POST `/ext/github-repo` → `verifyPayloadIntegrity` + `setDataSigner`（`extensions/index.js:34`）→ `resolveRepoLoaderFunction("github")`（`:41`）动态 require → `loadRepo(body, response)` 返回 documents。
2. **重同步**：`/ext/resync-source-document`（`:13`）→ 查 `RESYNC_METHODS[type]`（`:21`）→ 调对应方法重新抓源文档。
3. **Hotdir 生命周期**：server 上传文件写 `hotdir/` → collector `/process` 从 `WATCH_DIRECTORY`（即 hotdir）读文件 → 处理完 trashFile 删除；collector 启动时 `wipeCollectorStorage()` 清空。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `WATCH_DIRECTORY` | hotdir 路径 | `utils/constants.js` |
| 各扩展凭证 | GitHub token/Confluence 用户密码等 | 各扩展加载器读取 |
| website-depth 默认 | depth=1, maxLinks=20 | `extensions/index.js:112` |

## 5. 错误与重试语义

- 未知 resync type → 抛 `"Type ... is not a valid type to sync"`（`extensions/index.js:22`）。
- 所有扩展路由 try/catch 返回 200 + `{success:false, reason:e.message}`（如 `:28-30`）。
- 外部系统（GitHub/Confluence/YouTube）不可达向上抛错。

## 6. 并发细节

- 扩展路由同步挂载；请求 async 处理。
- repo loader 动态 require 按需加载，避免启动期引入全部 SDK。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `collector/extensions/`、`collector/utils/extensions/`、`collector/hotdir/`

**Out-of-Scope（不在本仓库源码内）**
- GitHub/GitLab/Confluence/YouTube/Paperless-ngx 等外部 SaaS API
- `simple-git`/`@octokit` 等第三方 SDK

## 8. 与相邻子系统交互

- 上游 → 本叶子：collector-server-core 的 `extensions(app)` 挂载；server 端 admin endpoints 触发扩展。
- 本叶子 → 下游：各扩展加载器抓外部系统 → 产出 documents → 写 server documents → 向量化。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：扩展点注册表（`RESYNC_METHODS`、`resolveRepoLoader`）+ 动态 require 工厂。
- **图类型侧重**：architecture（扩展路由+外部系统拓扑）+ sequence（repo 加载时序）。
- **外部边界**：多个外部 SaaS API。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 扩展机制与外部源架构 | `collector-extensions-hotdir-architecture.html` | architecture | showcase（一次通过） |
| 仓库加载时序 | `collector-extensions-hotdir-sequence.html` | sequence | showcase（一次通过） |

JSON IR 源文件位于 `json/` 目录。
