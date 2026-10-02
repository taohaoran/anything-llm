# app-shell 应用外壳（app-shell）

> 本文是 `frontend` 域下的叶子子系统文档。域级总览见 `../frontend.md`。
> 本文只展开**应用外壳**：路由表、Provider 上下文树、主题/会话等全局状态、路由权限守卫与侧边导航布局；
> 具体业务页面（聊天、工作区管理、设置、登录引导）分别见 `chat-interface`、`workspace-admin-ui`、
> `settings-console-models`、`auth-onboarding-ui` 等叶子。
>
> 源码基准：`frontend/`（Vite + React SPA），commit `128a015`，主语言 JavaScript（ESM，并入 TS-JS 口径）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 路由表与懒加载 | `createBrowserRouter` 定义全部路由，页面用 `lazy()` 动态 import 分包；开发态用 `React.Fragment`、生产态用 `React.StrictMode` | `frontend/src/main.jsx:18`、`main.jsx:15-16` |
| Provider 上下文树 | 根组件按 `ErrorBoundary → Theme → PWAMode → Auth → Logo → Pfp → I18next` 顺序包裹全局状态 | `frontend/src/App.jsx:19-46` |
| 路由权限守卫 | 四种守卫组件：`PrivateRoute`（登录即可）、`AdminRoute`（admin 或单机）、`ManagerRoute`（manager/admin）、`SingleUserRoute`（仅单机模式） | `frontend/src/components/PrivateRoute/index.jsx:79/108/130/148` |
| 会话校验 | 进入受保护路由前调用 `/onboarding`、`/setup-complete`、会话 token 校验，决定放行/跳 onboarding/跳登录 | `PrivateRoute/index.jsx:21-72` |
| 会话刷新 | AuthContext 在 token 变化后拉取当前用户；失败时清 localStorage 会话并跳登录 | `frontend/src/AuthContext.jsx:50-72` |
| 主题切换 | 三态主题 system/light/dark，跟随系统 `prefers-color-scheme`，写 `data-theme` 属性与 body class | `frontend/src/hooks/useTheme.js:27-86`、`frontend/src/ThemeContext.jsx` |
| 统一请求头 | 所有 API 请求注入 `Authorization: Bearer`、`X-Timezone`、`X-Language` | `frontend/src/utils/request.js:11-18` |
| 路径常量集中管理 | 所有内部/外部链接收敛到 `paths` 对象，避免硬编码 | `frontend/src/utils/paths.js:22-261` |
| 侧边导航布局 | 桌面端渲染可折叠 Sidebar（Logo、搜索、活跃工作区、设置入口），移动端隐藏 | `frontend/src/components/Sidebar/index.jsx:19-70` |
| 全局弹层/快捷键 | Toast、快捷键帮助、图片灯箱作为全局节点挂在 Provider 树内 | `frontend/src/App.jsx:35-37` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `createBrowserRouter([...])` | `main.jsx:18` | 浏览器路由表根，children 为懒加载路由 |
| `App()` | `App.jsx:19` | 根布局，组合全部 Provider 与 `<Outlet />` |
| `AuthProvider` / `AuthContext` | `AuthContext.jsx:12-13` | 全局会话 `{user, authToken}`，localStorage 持久化 |
| `ThemeProvider` / `useThemeContext` | `ThemeContext.jsx:6/14` | 主题状态下发；`useTheme()` 实现三态与系统监听 |
| `useIsAuthenticated()` | `PrivateRoute/index.jsx:14` | Hook：拉 onboarding/setup 状态，返回 `{isAuthd, shouldRedirectToOnboarding, multiUserMode}` |
| `AdminRoute/ManagerRoute/SingleUserRoute` | `PrivateRoute/index.jsx:79/108/130` | 角色守卫组件，未授权 `<Navigate>` 回首页 |
| `baseHeaders(providedToken)` | `utils/request.js:11` | 构造鉴权与时区/语言请求头 |
| `paths`（默认导出对象） | `utils/paths.js:22` | 路由/外链路径工厂，含 `workspace`、`settings`、`onboarding` 命名空间 |

## 3. 关键调用链

**链 1：用户访问受保护路由的鉴权放行链**
1. 浏览器请求 URL → `main.jsx:18` 的 router 匹配到某懒加载路由，该路由 `element` 是 `<PrivateRoute Component={Page} />`（`main.jsx:24-29`）。
2. `PrivateRoute` 挂载 `useIsAuthenticated()`（`PrivateRoute/index.jsx:148` → `:14`），`useEffect` 触发 `validateSession()`（`:21`）。
3. 先 `System.isOnboardingComplete()`（`PrivateRoute/index.jsx:22`，实现见 `models/system.js:40`，GET `/onboarding`）；未完成则 `setShouldRedirectToOnboarding(true)`（`:27-30`）。
4. 再 `System.keys()`（`:23`，GET `/setup-complete`）取 `MultiUserMode/RequiresAuth`（实现见 `models/system.js:61`）。
5. 按模式分支：单机无密码直接放行（`:34-37`）；单机密码模式校验 localStorage token（`:40-50`）；多用户模式校验 user+token 并 `validateSessionTokenForUser()`（`:53-67`，失败清三类 localStorage key）。
6. `isAuthd===null` 时渲染 `<FullScreenLoader />`（`:150`）；`shouldRedirectToOnboarding` 时 `<Navigate to="/onboarding">`（`:152-154`）；通过则包裹 `KeyboardShortcutWrapper > UserMenu > Component`（`:156-161`），否则 `<Navigate to={paths.login(true)}>`（`:163`）。

**链 2：token 变化后的会话刷新链**
1. `AuthProvider` 初始从 localStorage 读 `AUTH_USER/AUTH_TOKEN` 建 `store`（`AuthContext.jsx:14-19`）。
2. `useEffect` 监听 `store.authToken`（`:72`），存在 token 则 `refreshUser()`（`:51`）。
3. 调 `System.refreshUser()`（`:52`）；`success && user===null`（单机模式）直接返回（`:53`）；`!success` 时清 `AUTH_USER/AUTH_TOKEN/AUTH_TIMESTAMP/USER_PROMPT_INPUT_MAP` 并 `navigate("/login")`（`:55-63`）；成功则写回 localStorage 并 `setStore`（`:65-69`）。

**链 3：主题切换链**
1. `useTheme()` 初始从 localStorage 读 `theme`，旧值 `"default"` 迁移为 `"dark"`（`useTheme.js:28-32`），并读 `prefers-color-scheme`（`:34-38`）。
2. 监听系统主题变化 `mql.addEventListener("change")`（`:41-47`）。
3. `resolvedTheme = theme==="system" ? systemTheme : theme`（`:49`），effect 写 `document.documentElement[data-theme]`、toggle body `.light` class、持久化 localStorage 并广播 `REFETCH_LOGO_EVENT`（`:51-56`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `import.meta.env.DEV` | 开发态关闭 StrictMode、开启 i18n debug、注册 `Cmd+.` 主题切换快捷键 | `main.jsx:15`、`useTheme.js:60`、`i18n.js:12` |
| localStorage `theme` | `"system"`（旧值 `"default"`→`"dark"`） | `useTheme.js:29-31` |
| localStorage `AUTH_TOKEN/AUTH_USER/AUTH_TIMESTAMP` | 会话凭据，守卫与请求头共用 | `utils/constants.js`、`request.js:1-2` |
| `MultiUserMode` / `RequiresAuth` | 来自 `/setup-complete`，决定守卫分支 | `PrivateRoute/index.jsx:23` |
| `fallbackLng: "en"`、`lowerCaseLng: true` | i18n 回退语言与大小写归一 | `frontend/src/i18n.js:11-15` |

## 5. 错误与重试语义

- **onboarding/setup 接口失败**：`System.isOnboardingComplete()`/`keys()` 内部 `.catch()` 回退（`models/system.js:47`、`:68`），守卫据此走"未完成/单机"保守分支，不会因接口异常白屏。
- **会话刷新失败**：`refreshUser()` 返回 `!success` 时**不重试**，直接清空本地会话并跳登录（`AuthContext.jsx:55-63`），避免悬挂 token。
- **token 校验失败**：多用户模式下校验不通过即清会话跳登录（`PrivateRoute/index.jsx:61-67`），无自动重登。
- **路由级错误**：`<ErrorBoundary FallbackComponent>` 捕获渲染异常，`resetKeys=[location.pathname]` 使切路由自动复位（`App.jsx:22-26`）。
- 外壳层**不实现重试/退避**；重试语义归业务页面与 models 层。

## 6. 并发细节

- React 并发模型：`useEffect` + 状态驱动重渲染；懒加载路由用 Suspense（`App.jsx:29` `<Suspense fallback={<FullScreenLoader/>}>`）实现代码分块加载态。
- 鉴权判定为单次异步 `validateSession()`（`PrivateRoute/index.jsx:21`），用 `useState(null)` 三态（加载中/通过/拒绝）避免竞态闪烁。
- `ResizeObserver`/`matchMedia`/自定义事件（`PROMPT_INPUT_EVENT`、`REFETCH_LOGO_EVENT`）为外壳下发的副作用通道，卸载时 `disconnect/removeEventListener` 清理（`useTheme.js:46`）。
- 无后台轮询 worker；会话相关轮询在 `hooks/usePolling.js`（归业务叶子）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `frontend/src/main.jsx`、`App.jsx`、`AuthContext.jsx`、`ThemeContext.jsx`、`PWAContext.jsx`、`LogoContext.jsx`、`PfpContext.jsx`
- `frontend/src/components/PrivateRoute/`、`Sidebar/`、`UserMenu`、`SettingsButton`、`Preloader`
- `frontend/src/utils/request.js`、`paths.js`、`constants.js`、`session.js`

**Out-of-Scope（不在本仓库源码内）**
- 后端鉴权/onboarding/setup 接口实现 → `server/`（见 server-api 域叶子）
- 浏览器本身、`matchMedia`、`localStorage`、`fetch` 为浏览器 API（外部运行时）
- 具体页面业务逻辑（聊天渲染、文档管理、设置表单）→ 见对应业务叶子
- `browser-extension/`、`embed/` 为未 checkout 的 git 子模块，不在本仓库源码内

## 8. 与相邻子系统交互

- **上游（浏览器 → 外壳）**：用户输入 URL，浏览器加载 SPA；外壳通过 `baseHeaders()` 为下游 models 层注入鉴权头。
- **外壳 → chat-interface**：路由 `/workspace/:slug` 经 `PrivateRoute` 放行后渲染 `WorkspaceChat`，Sidebar 提供工作区切换入口（`Sidebar/index.jsx:66` `<ActiveWorkspaces/>`）。
- **外壳 → settings-console-models**：`/settings/*` 路由经 `AdminRoute/ManagerRoute` 按角色放行（`main.jsx:69-339`）。
- **外壳 → auth-onboarding-ui**：未完成 onboarding 时守卫 `<Navigate to="/onboarding">`（`PrivateRoute/index.jsx:153`）；会话失败跳 `/login`。
- **外壳 → 后端 API**：所有 models 经 `utils/request.js` 统一出口，外部 REST API 不在本仓库前端源码内。

## 9. 语言专项适配口径（JavaScript / TS-JS）

- **capability seam 分组**：本叶子按 Service Definition（路由表 `main.jsx`、`paths.js`）/ Provider（`App.jsx` 的 Context 树）/ Consumer（各业务页面消费守卫与上下文）三层归组，而非 controller/reconciler。
- **图型**：以 architecture（组件分层与边界）+ sequence（启动鉴权时序，事件/Promise 链）为主；外壳无数据管道与单实体状态机，故不画 dataflow/lifecycle。
- **外部边界**：标注浏览器 API（`matchMedia`/`localStorage`/`fetch`/DOM）与外部 REST API；不涉及 etcd/CRD。
- **部署维度**：不适用单二进制；本叶子属 Vite SPA，构建产物由 server/ 静态托管（见 docker-build 叶子），依赖图以 npm 包（react-router-dom、react-i18next）表达。
- JS 无类型标注，接口线索来自 `AuthContext.createContext`、JSDoc `@typedef`（`useTheme.js:10-20`）与 Context 导出。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 应用外壳架构图 | `app-shell-architecture.html` | architecture | **standard**（showcase 未过：纵向对齐列节点间 `orthogonal-v` 边的走线清档检查未达 showcase 阈值；已改用 `orthogonal-v` 显式路由并精简为单条 dashed 边，render 退出码 0） |
| 启动与路由鉴权时序 | `app-shell-sequence.html` | sequence | **showcase** |

- JSON IR 源文件：`json/app-shell-architecture.json`、`json/app-shell-sequence.json`。
- 省略说明：本叶子未生成 dataflow/lifecycle/workflow 图——外壳是静态组件树 + 单次鉴权调用链，无数据管道、单实体状态机或多角色审批泳道语义，与已有图信息重复，按资源节省原则省略。
