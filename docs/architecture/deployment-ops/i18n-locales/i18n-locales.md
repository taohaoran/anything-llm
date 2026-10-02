# i18n-locales 国际化语言包（i18n-locales）

> 本文是 `deployment-ops` 域下的叶子子系统文档。域级总览见 `../deployment-ops.md`。
> 本文只展开**前端国际化机制与多语言包**：i18next 初始化、语言检测、en 基准字典与 30+ 语言 common.js、翻译校验脚本；
> 镜像构建见 `docker-build`，云部署见 `cloud-deployments`。
>
> 源码基准：`frontend/src/locales/` 与根 `locales/`，commit `128a015`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| i18next 初始化 | `initReactI18next` + `LanguageDetector`，fallbackLng=en，lowerCaseLng | `frontend/src/i18n.js:6-19` |
| 语言资源聚合 | `resources.js` import 30+ 语言的 `common.js` | `frontend/src/locales/resources.js:22-50` |
| 英文基准字典 | `en/common.js` 为键名 ground-truth | `frontend/src/locales/en/common.js` |
| 多语言包 | ko/es/zh/de/ru/it/pt_BR/he/nl/vn/ja/... 各 `common.js` | `frontend/src/locales/<lang>/common.js` |
| 缺失键回退 | 某语言未定义/为 null 的键回退 `en/common.js` | `resources.js:1-11` 注释、`i18n.js:11` |
| 语言切换选项 | `useLanguageOptions` 提供语言列表 | `frontend/src/hooks/useLanguageOptions.js` |
| 翻译校验脚本 | 对比各语言与 en 结构一致性 | `frontend/src/locales/verifyTranslations.mjs:1-20` |
| 未用键/归一脚本 | findUnusedTranslations、normalizeEn | `frontend/src/locales/findUnusedTranslations.mjs`、`normalizeEn.mjs` |
| README 翻译 | 根 `locales/` 下 fa-IR/ja-JP/tr-TR/zh-CN README | `locales/README.*.md` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `i18next.init({...})` | `frontend/src/i18n.js:10` | i18n 运行时配置 |
| `resources` / `defaultNS` | `locales/resources.js` | 语言包注册表与默认命名空间 |
| `useTranslation()` | `react-i18next`（外部） | 组件取 `t()` 函数 |
| `verifyTranslations.mjs` | `locales/verifyTranslations.mjs` | CI 翻译结构校验 |
| `Intl.DisplayNames` | `verifyTranslations.mjs:3` | 语言名本地化展示 |

## 3. 关键调用链

**链 1：页面加载语言检测**
1. `main.jsx` 引入 `i18n.js`，i18next `.use(initReactI18next).use(LanguageDetector).init(...)`（`i18n.js:6-19`）。
2. `LanguageDetector` 读浏览器 `navigator.language`（`request.js:16` 同时把 `X-Language` 带给后端）。
3. 按 `lowerCaseLng` 归一后取对应语言字典；缺失键回退 `en`（`i18n.js:11/15`）。
4. 组件经 `useTranslation()` 的 `t(key)` 渲染文案。

**链 2：贡献新语言**
1. 复制 `en/common.js` 为模板，在 `<lang>/common.js` 翻译（`resources.js:1-9` 注释）。
2. 在 `resources.js` import 并注册。
3. 本地跑 `yarn verify:translations` 对比结构（`verifyTranslations.mjs`、`resources.js:13-18` 贡献者须知）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `fallbackLng` | `"en"` | `i18n.js:11` |
| `defaultNS` | 来自 resources | `i18n.js:13` |
| `lowerCaseLng` | true，语言码小写归一 | `i18n.js:15` |
| `interpolation.escapeValue` | false（React 已转义） | `i18n.js:16-18` |
| `debug` | DEV 开启 | `i18n.js:12` |

## 5. 错误与重试语义

- **缺失键**：不报错，回退英文（设计如此）。
- **结构不一致**：`verifyTranslations.mjs` 在 CI/本地校验失败，阻断不合规 PR；运行时不校验。
- 无网络重试；语言包随前端打包，无运行时远程拉取。

## 6. 并发细节

- i18next 单例；语言切换触发订阅组件重渲染（react-i18next 机制）。
- 无后台轮询；语言包静态打包进 Vite bundle。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `frontend/src/i18n.js`、`frontend/src/locales/`（resources.js + 各语言 common.js + 校验脚本）、`hooks/useLanguageOptions.js`、根 `locales/README.*.md`

**Out-of-Scope（不在本仓库源码内）**
- `i18next`/`react-i18next`/`i18next-browser-languagedetector` 为第三方 npm 包
- 浏览器 `navigator.language`/`Intl` 为浏览器 API
- 翻译内容由社区贡献者维护（外部协作流程）

## 8. 与相邻子系统交互

- **app-shell → i18n-locales**：`I18nextProvider` 包裹全局（`App.jsx:33`）；`X-Language` 请求头带语种给后端。
- **i18n-locales → 全部前端叶子**：所有页面经 `useTranslation` 消费文案。
- **docker-build → i18n-locales**：语言包随前端 `dist` 打包进镜像。

## 9. 语言专项适配口径（JavaScript / TS-JS）

- **capability seam**：Provider（`resources.js` 语言包注册表）/ Consumer（组件 `useTranslation`）。
- **图型**：architecture（检测→实例→字典流）+ sequence（语言检测渲染 Promise/事件链）；无 ETL 管道与单实体状态机。
- **外部边界**：第三方 i18n npm 包、浏览器 API；不涉及 etcd/CRD。
- **部署维度**：语言包静态打包进 Vite SPA（见 docker-build 叶子）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 国际化架构图 | `i18n-locales-architecture.html` | architecture | **showcase** |
| 语言检测与渲染时序 | `i18n-locales-sequence.html` | sequence | **showcase** |

- JSON IR 源文件：`json/i18n-locales-architecture.json`、`json/i18n-locales-sequence.json`。
- 省略说明：本叶子未生成 dataflow/lifecycle/workflow 图——语言包是静态资源注册（已用 architecture/sequence 表达），无 ETL 管道、单实体状态机或多角色审批泳道语义，按资源节省原则省略。
