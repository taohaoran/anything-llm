# 多 LLM 提供商适配（llm-providers）

> 本文是 `ai-integrations` 域下的叶子子系统文档。域级总览见 `../ai-integrations.md`。
> 本文只展开"多 LLM 提供商的统一调用接口与工厂路由"，不展开嵌入引擎、向量库、多模态等相邻叶子（分别见各自文档）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM，Node >= 18。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| LLM 提供商工厂路由 | 按 `LLM_PROVIDER` 环境变量或入参 switch-case 实例化对应提供商类，注入当前 embedder | `server/utils/helpers/index.js:136` `getLLMProvider()` |
| 40+ 提供商适配 | OpenAI / Azure / Anthropic / Gemini / Ollama / LM Studio / LocalAi / TogetherAi / FireworksAi / Perplexity / OpenRouter / Mistral / Groq / KoboldCPP / TextGenWebUI / Cohere / LiteLLM / GenericOpenAi / Bedrock / DeepSeek / ApiPie / Novita / XAi / NvidiaNim / PPIO / MoonshotAi / CometApi / Foundry / ZAi / GiteeAI / Llmman / Privatemode / SambaNova / Lemonade / OMLX / Minimax / Cerebras / Vertex 等 | `server/utils/AiProviders/<provider>/index.js`（每个子目录一个） |
| 统一聊天接口 | 每个提供商类实现 `constructPrompt()` / `getChatCompletion()` / `streamGetChatCompletion()` / `handleStream()` / `embedTextInput()` / `embedChunks()` / `compressMessages()` | 以 `server/utils/AiProviders/openAi/index.js:14` `OpenAiLLM` 为代表实现 |
| 流式响应处理 | `streamGetChatCompletion()` 返回测量过的流，`handleStream()` 把 SSE chunk 写入 HTTP response，并统计 token 用量 | `server/utils/AiProviders/openAi/index.js:187`、`:210` |
| 动态模型路由（Model Router） | `anythingllm-router` 提供商按工作区规则（calculated/LLM/sticky）动态选择底层 delegate LLM | `server/utils/AiProviders/modelRouter/index.js:4` `AnythingLLMModelRouter` |
| 上下文窗口模型表 | 维护各 provider 模型的 prompt window 上限，远程拉取 LiteLLM 价格与上下文窗口 JSON（3 天缓存） | `server/utils/AiProviders/modelMap/index.js:5` `ContextWindowFinder` |
| 性能监控 | 测量异步函数/流的 duration、token 用量、outputTps，输出 metrics | `server/utils/helpers/chat/LLMPerformanceMonitor.js` |
| 多模态附件 | `#generateContent()` 把用户文本 + 图片附件组装成多模态 content 数组 | `server/utils/AiProviders/openAi/index.js:89` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `getLLMProvider({provider, model})` | `server/utils/helpers/index.js:136` | 工厂函数：按 provider 名 require 并 new 对应类，注入 embedder；`anythingllm-router` 直接抛错要求走 ModelRouter |
| `OpenAiLLM`（及 40+ 同类类） | `server/utils/AiProviders/openAi/index.js:14` | 无显式基类（JS duck-typing）；统一约定方法：`constructor(embedder, modelPreference)`、`promptWindowLimit()`、`isValidChatCompletionModel()`、`constructPrompt()`、`getChatCompletion()`、`streamGetChatCompletion()`、`handleStream()`、`embedTextInput()`、`embedChunks()`、`compressMessages()` |
| `AnythingLLMModelRouter` | `server/utils/AiProviders/modelRouter/index.js:4` | 动态路由包装类：`resolve(context, opts)` 解析规则后 `#finalize()` 实例化 delegate provider，再把聊天方法委托给 delegate |
| `ContextWindowFinder` | `server/utils/AiProviders/modelMap/index.js:5` | 单例；`static get(provider, modelName)` 返回上下文窗口 token 数；远程拉取 LiteLLM model_prices JSON |
| `LLMPerformanceMonitor` | `server/utils/helpers/chat/LLMPerformanceMonitor.js` | `measureAsyncFunction()` / `measureStream()` 包裹 LLM 调用，记录 duration、tokens、tps |
| `writeResponseChunk()` / `clientAbortedHandler()` | `server/utils/helpers/chat/responses.js` | 把流式 token 写成 SSE chunk；处理客户端断开 |

## 3. 关键调用链

1. **聊天请求 → 工厂实例化 → 同步补全**：调用方（`server/utils/chats/stream.js`）调用 `getLLMProvider({provider, model})`（`helpers/index.js:136`）→ switch-case `require("../AiProviders/openAi")` 并 `new OpenAiLLM(embedder, model)`（`:142-143`）→ 调用方 `llm.constructPrompt({systemPrompt, contextTexts, chatHistory, userPrompt, attachments})`（`openAi/index.js:109`）组装 messages 数组 → `llm.getChatCompletion(messages, {temperature})`（`openAi/index.js:148`）经 `LLMPerformanceMonitor.measureAsyncFunction` 调 `openai.responses.create()` → 返回 `{textResponse, metrics}`。
2. **流式聊天主路径**：调用方 `llm.streamGetChatCompletion(messages, {temperature})`（`openAi/index.js:187`）返回测量流 → `llm.handleStream(response, stream, {uuid, sources})`（`openAi/index.js:210`）内 `for await (const chunk of stream)` 遍历，遇 `response.output_text.delta` 用 `writeResponseChunk(response, {...})` 推 SSE，遇 `response.completed` 读取 usage 并 `resolve(fullText)`。
3. **动态模型路由解析**：`AnythingLLMModelRouter.resolve(context, {user, thread})`（`modelRouter/index.js:33`）→ 先算 calculated 规则（`:51`）→ 再查 LLM 规则缓存（`:63`）→ 再查 sticky route（`:78`）→ 都未命中走 default model → `#finalize()` 实例化 delegate LLM provider。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `LLM_PROVIDER` | 默认 `"openai"`；决定工厂走哪个 case | `helpers/index.js:137` |
| `OPEN_AI_KEY` / `OPEN_MODEL_PREF` | OpenAI 必需 key；模型默认 `gpt-4.1-nano` | `AiProviders/openAi/index.js:16,24` |
| 各 provider 专属 key | 如 `ANTHROPIC_API_KEY`、`GEMINI_API_KEY`、`OLLAMA_BASE_PATH` 等，构造函数内读取 | 各 `AiProviders/<provider>/index.js` 构造函数 |
| `LLM_PROVIDER` 之外的 modelPreference | 可在 `getLLMProvider({model})` 入参覆盖 | `helpers/index.js:136` |
| Model Router `cooldown_seconds` | sticky 路由冷却，默认 300s | `modelRouter/index.js:41` |

## 5. 错误与重试语义

- **构造期错误**：缺 API key 直接 `throw new Error("No OpenAI API key was set.")`（`openAi/index.js:16`），进程启动即暴露。
- **模型校验**：`isValidChatCompletionModel()`（`openAi/index.js:71`）对非 gpt/o 前缀模型调 `openai.models.retrieve()`，失败 catch 返回 `null`，进而 `getChatCompletion` 抛 `"is not valid for chat completion!"`。
- **流式错误**：`handleStream` 内 catch（`openAi/index.js:270`）先用 `isAbortError(e)` 判断客户端断开（不报错，走 `clientAbortedHandler`）；否则写 `type: "abort"` 的 SSE chunk 并 `resolve(fullText)`，不 reject Promise。
- **工厂 default 分支**：未知 `LLM_PROVIDER` 抛 `ENV: No valid LLM_PROVIDER value found`（`helpers/index.js:262`）。
- 本层不做自动重试；重试由上层（chats/stream、agents）决定。

## 6. 并发细节

- 每个请求 `getLLMProvider` 新建一次 provider 实例（无池化），embedder 由 `getEmbeddingEngineSelection()` 共享。
- 流式处理基于 `for await` 异步迭代器；`response.on("close", handleAbort)`（`openAi/index.js:225`）注册客户端断开监听，结束时 `response.removeListener` 防止泄漏。
- `ContextWindowFinder` 用静态单例 + 3 天远程缓存（`modelMap/index.js:16`），避免每次请求拉远程 JSON。
- NativeEmbedder 用静态 `#pipelines` Map 复用 ONNX session（见 embedding 叶子）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `server/utils/helpers/index.js` 的工厂函数
- `server/utils/AiProviders/` 下全部 provider 适配类、modelRouter、modelMap
- `server/utils/helpers/chat/LLMPerformanceMonitor.js`、`responses.js` 的流式工具

**Out-of-Scope（不在本仓库源码内）**
- OpenAI / Anthropic / Gemini / Ollama 等外部 LLM 云 API 服务端
- `openai` / `@anthropic-ai/sdk` / `google-generativeai` 等第三方 npm SDK（在 node_modules，不在本仓库源码）
- LiteLLM 远程 model_prices JSON（`raw.githubusercontent.com/BerriAI/litellm/...`）
- 嵌入引擎实现（见 embedding-engines-rerankers 叶子）、向量库（见 vector-db-providers 叶子）不在本叶子

## 8. 与相邻子系统交互

- 上游 → 本叶子：`server/utils/chats/stream.js`（聊天主链路）、`server/utils/agents/`（agent 运行时）、`server/endpoints/`（API）调用 `getLLMProvider()` 获取实例。
- 本叶子 → 下游：每个 provider 类内部 `require("openai")` 等 SDK → 外部 LLM API；同时持有 `this.embedder`（来自 embedding 叶子）用于 `embedTextInput/embedChunks` 委托。
- Model Router 反向依赖 `getLLMProvider`（`modelRouter/index.js:2`）实例化 delegate。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：本叶子是典型 capability seam——Service Definition 为 `getLLMProvider()` 工厂 + 统一方法约定（无显式 interface，靠 JSDoc `@returns {BaseLLMProvider}` 与 duck-typing）；Provider 为 `AiProviders/<provider>/index.js` 40+ 实现；Consumer 为 chats/agents/endpoints。
- **图类型侧重**：以 architecture（工厂+多 provider 静态拓扑）+ sequence（流式聊天调用链）为主；不画 dataflow（无数据管道语义）、不画 lifecycle（无单实体状态机）。
- **外部边界**：各 provider SDK 调用标注为外部 LLM API（OpenAI/Anthropic/…），Node 内置模块（fs/path/http）为依赖。
- **部署维度**：server 为单一 Node Express 进程包（`server/package.json` name=anything-llm-server），provider 通过懒加载 `require()` 在 switch 分支内按需引入，避免启动期加载全部 SDK。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| LLM 提供商工厂与调用架构 | `llm-providers-architecture.html` | architecture | standard（showcase 校验因边标签与组件重叠失败，去掉边标签后 standard 通过；render 退出码 0） |
| 流式聊天调用链时序 | `llm-providers-sequence.html` | sequence | showcase（一次通过；participant 标签缩短后达标） |

JSON IR 源文件位于 `json/` 目录。
