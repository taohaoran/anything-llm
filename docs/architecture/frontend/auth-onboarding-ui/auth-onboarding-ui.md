# auth-onboarding-ui 认证与引导界面（auth-onboarding-ui）

> 本文是 `frontend` 域下的叶子子系统文档。域级总览见 `../frontend.md`。
> 本文只展开**登录/注册/首次设置向导 UI**：登录页与 SSO 分支、单机/多用户密码表单、首次 onboarding 五步向导；
> 路由守卫与会话上下文见 `app-shell`。
>
> 源码基准：`frontend/`，commit `128a015`，JavaScript（ESM，TS-JS 口径）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 登录页分支 | 处理单机/多用户登录；Simple SSO 开启且 noLogin 时跳 SSO 或 noLoginRedirect | `frontend/src/pages/Login/index.jsx:13-43` |
| Simple SSO 检测 | `useSimpleSSO()` 取 SSO 配置 | `pages/Login/index.jsx:15`、`hooks/useSimpleSSO.js` |
| 密码门禁 | `usePasswordModal(noTry)` 决定是否要求认证；已认证跳首页 | `Login/index.jsx:21-40`、`components/Modals/Password/index.jsx:35` |
| 单机认证表单 | 单用户密码登录 | `components/Modals/Password/SingleUserAuth.jsx` |
| 多用户认证表单 | 用户名+密码登录/注册 | `components/Modals/Password/MultiUserAuth.jsx` |
| 认证预检 | `System.needsAuthCheck()`/`keys()`/`checkAuth(token)` | `Password/index.jsx:40-53/95` |
| 首次引导向导 | 按 URL `:step` 表驱动渲染步骤页 | `pages/OnboardingFlow/index.jsx:11-22` |
| 向导步骤注册 | home/llm-preference/user-setup/data-handling/survey 五步 | `pages/OnboardingFlow/Steps/index.jsx:12-18` |
| 向导布局与前后导航 | `OnboardingLayout` 统一 header/back/forward 按钮 | `Steps/index.jsx:21-60` |
| 完成后自动跳首页 | `useRedirectToHomeOnOnboardingComplete` | `Steps/index.jsx:22`、`hooks/useOnboardingComplete.js` |
| 标记完成 | `System.markOnboardingComplete()` POST `/onboarding` | `models/system.js:53-60` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Login()` | `Login/index.jsx:13` | 登录页入口，SSO/认证分支决策 |
| `PasswordModal` / `usePasswordModal` | `components/Modals/Password/index.jsx` | 认证门禁状态与表单容器 |
| `OnboardingFlow()` | `OnboardingFlow/index.jsx:11` | 向导步骤路由 |
| `OnboardingSteps`（映射） | `Steps/index.jsx:12` | step key → 步骤组件 |
| `OnboardingLayout` | `Steps/index.jsx:21` | 向导外壳与导航按钮状态 |
| `System.checkAuth/needsAuthCheck/markOnboardingComplete` | `models/system.js` | 认证与引导完成 API |
| `useSimpleSSO()` | `hooks/useSimpleSSO.js` | SSO 配置读取 |

## 3. 关键调用链

**链 1：访问 /login 的分支决策**
1. `Login()` 挂载后 `useQuery()` 取 `?nt=1`（no-try）与 `useSimpleSSO()`（`Login/index.jsx:14-15`）。
2. `usePasswordModal(!!query.get("nt"))` 判定认证要求（`:21`）。
3. 若 SSO 开启且 `noLogin`：无 token 时 `window.location.replace(noLoginRedirect)`，否则 `<Navigate to={paths.sso.login()}>`（`:24-31`）。
4. 若 `requiresAuth === false` 直接 `<Navigate to={paths.home()}>`（`:33`）；否则渲染 `<PasswordModal mode={mode}/>`（`:35`）。

**链 2：密码认证预检**
1. `PasswordModal` 内 `checkAuthReq`（`Password/index.jsx:35`）：若 `System.needsAuthCheck()` 为 false 且非 no-try 则放行（`:40`）。
2. 否则 `System.keys()` 取设置（`:49`），有本地 token 时 `System.checkAuth(currentToken)` 校验（`:53/95`）。
3. 校验通过 → 写 localStorage 会话；失败 → 显示登录表单。

**链 3：首次引导完成**
1. 守卫检测 `onboardingComplete===false` 时 `<Navigate to="/onboarding">`（见 app-shell）。
2. `OnboardingFlow` 按 `:step` 渲染对应步骤（`OnboardingFlow/index.jsx:13-18`）。
3. 末步调 `System.markOnboardingComplete()` POST `/onboarding`（`models/system.js:54`），`useRedirectToHomeOnOnboardingComplete` 轮询完成后跳首页（`Steps/index.jsx:22`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `?nt=1` | no-try，跳过已有 token 自动放行 | `Login/index.jsx:14/21` |
| SSO `enabled/noLogin/noLoginRedirect` | Simple SSO 开关与跳转地址 | `Login/index.jsx:24-31` |
| MultiUserMode / RequiresAuth | 决定渲染哪种表单与门禁 | `Password/index.jsx:49` |
| `onboardingComplete` | `/onboarding` 返回，false 触发向导 | `models/system.js:46` |

## 5. 错误与重试语义

- **checkAuth 失败**：不自动重试，显示登录表单让用户重输（`Password/index.jsx:53/95`）。
- **markOnboardingComplete 失败**：`.catch(() => false)`，向导停留在当前步（`models/system.js:59`）。
- **SSO 跳转**：硬跳转 `window.location.replace`，前端不处理回调错误（`Login/index.jsx:28`）。
- 无指数退避；登录失败 toast 由表单组件处理。

## 6. 并发细节

- React 状态驱动：`loading/requiresAuth/mode` 三态避免闪烁；`useEffect` 内单次预检。
- 向导 `OnboardingLayout` 用 `useState` 管 header/back/forward 按钮状态，子步骤通过回调设置（`Steps/index.jsx:23-30`）。
- 完成检测用 `useOnboardingComplete` 轮询 hook（归 app-shell 轮询机制）。
- 无后台 worker；移动端向导单列布局（`Steps/index.jsx:33` `isMobile` 分支）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `frontend/src/pages/Login/`、`pages/OnboardingFlow/`、`components/Modals/Password/`、`hooks/useSimpleSSO.js`、`hooks/useOnboardingComplete.js`、`models/system.js`（认证/引导 API）

**Out-of-Scope（不在本仓库源码内）**
- 服务端认证签发、SSO 回调校验、onboarding 状态存储 → `server/`（server-api 域）
- SSO 身份提供商（外部 IdP，不在本仓库源码内）
- 路由守卫实现 → app-shell 叶子

## 8. 与相邻子系统交互

- **app-shell → auth-onboarding-ui**：`PrivateRoute` 检测未完成 onboarding 跳 `/onboarding`；会话失败跳 `/login`。
- **auth-onboarding-ui → app-shell**：登录成功写 localStorage，`AuthContext` 据此刷新用户。
- **auth-onboarding-ui → 后端**：经 `models/system.js` 调 REST，外部 API 不在前端源码内。
- **auth-onboarding-ui → settings-console-models**：向导中选定的 LLM/向量库偏好落到系统设置。

## 9. 语言专项适配口径（JavaScript / TS-JS）

- **capability seam**：Consumer（Login/OnboardingFlow 页面消费配置）/ Provider（`models/system.js` API、`useSimpleSSO` hook）/ Service（后端认证，外部）。
- **图型**：architecture（水平认证流）+ sequence（登录 Promise 链）；向导是表驱动步骤而非状态机，不画 lifecycle。
- **外部边界**：外部 REST API、外部 SSO IdP、浏览器 location/localStorage；不涉及 etcd/CRD。
- **部署维度**：Vite SPA 静态托管（见 docker-build 叶子）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 认证引导架构图 | `auth-onboarding-ui-architecture.html` | architecture | **showcase** |
| 登录与引导完成时序 | `auth-onboarding-ui-sequence.html` | sequence | **showcase** |

- JSON IR 源文件：`json/auth-onboarding-ui-architecture.json`、`json/auth-onboarding-ui-sequence.json`。
- 省略说明：本叶子未生成 dataflow/lifecycle/workflow 图——向导是表驱动步骤渲染（非带泳道审批流），认证是单次请求链（已用 sequence 表达），无 ETL 管道或单实体状态机语义，按资源节省原则省略。
