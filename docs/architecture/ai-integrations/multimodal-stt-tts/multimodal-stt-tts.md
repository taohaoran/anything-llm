# 多模态：图像生成与语音 STT/TTS（multimodal-stt-tts）

> 本文是 `ai-integrations` 域下的叶子子系统文档。域级总览见 `../ai-integrations.md`。
> 本文只展开图像生成、语音转文字（STT）、文字转语音（TTS）三类多模态能力的统一工厂与适配。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 图像生成工厂 | 按 `IMAGE_GEN_PROVIDER` switch-case 实例化图像生成器 | `server/utils/helpers/index.js:332` `getImageGeneratorProvider()` |
| 5 个图像生成适配 | openai / ollama / lemonade / localai / openrouter | `server/utils/ImageGenerators/<name>/index.js` |
| STT 工厂 | 按 `STT_PROVIDER`（默认 native）实例化语音转文字器 | `server/utils/SpeechToText/index.js:1` `getSTTProvider()` |
| 6 个 STT 适配 | openai / lemonade / deepgram / generic-openai / groq（default native 为浏览器端） | `SpeechToText/<name>/index.js` |
| TTS 工厂 | 按 `TTS_PROVIDER`（默认 openai）实例化文字转语音器 | `server/utils/TextToSpeech/index.js:1` `getTTSProvider()` |
| 4 个 TTS 适配 | openai / elevenlabs / generic-openai / kokoro | `TextToSpeech/<name>/index.js` |
| 音频格式工具 | 音频格式转换辅助 | `server/utils/TextToSpeech/audioFormat.js` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `getImageGeneratorProvider()` | `helpers/index.js:332` | 图像生成工厂；无匹配 provider 抛错 |
| `getSTTProvider()` | `SpeechToText/index.js:1` | STT 工厂；default 抛错提示 native 为浏览器端能力 |
| `getTTSProvider()` | `TextToSpeech/index.js:1` | TTS 工厂；default 抛错 |
| `OpenAiImageGenerator` / `OpenAiSTT` / `OpenAiTTS`（及各同类） | 各子目录 `index.js` | duck-typing 实现：图像生成 `generateImage()`、STT `transcribe()`、TTS `synthesize()` |
| `BaseImageGenerator` | `ImageGenerators/base.js` | 图像生成抽象基类 |

## 3. 关键调用链

1. **图像生成**：管理端请求 → `getImageGeneratorProvider()`（`helpers/index.js:332`）→ `new OpenAiImageGenerator()` → 调 OpenAI images API 生成图片 → 返回 URL/base64。
2. **语音转文字（STT）**：用户上传音频 → server 端 `getSTTProvider()`（`SpeechToText/index.js:1`）→ `new DeepgramSTT()` 或 `OpenAiSTT()` → `transcribe(audioBuffer)` → 返回文本写入文档聊天。
3. **文字转语音（TTS）**：聊天回复文本 → `getTTSProvider()`（`TextToSpeech/index.js:1`）→ `new ElevenLabsTTS()` 等 → `synthesize(text)` → 返回音频流播放。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `IMAGE_GEN_PROVIDER` | 未设置抛错 | `helpers/index.js:333` |
| `STT_PROVIDER` | 默认 `native`（浏览器端 Web Speech API，server 端无此实现故 default 抛错） | `SpeechToText/index.js:2` |
| `TTS_PROVIDER` | 默认 `openai`；未设置抛错 | `TextToSpeech/index.js:2` |
| 各 provider 专属 key | `DEEPGRAM_API_KEY`、`ELEVENLABS_API_KEY` 等 | 各子目录构造函数 |

## 5. 错误与重试语义

- **工厂无匹配分支**：图像/TTS 直接 `throw new Error("No valid ... provider was set")`（`helpers/index.js:353`、`TextToSpeech/index.js:26`）；STT default 抛 `"STT_PROVIDER ... is not a server-side provider"`（`SpeechToText/index.js:22`），提示 native 是浏览器能力。
- 外部 API 错误向上抛出；本层不做重试。

## 6. 并发细节

- 与其他 provider 工厂一致，每请求新建实例，无池化。
- TTS/STT 涉及音频二进制流处理；图像生成返回 URL 或 base64。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `helpers/index.js` 的 `getImageGeneratorProvider()`
- `server/utils/ImageGenerators/`、`SpeechToText/`、`TextToSpeech/` 全部适配

**Out-of-Scope（不在本仓库源码内）**
- OpenAI Images / Deepgram / ElevenLabs / Groq 等外部多模态云 API
- 浏览器端 Web Speech API（native STT，在前端运行，不在 server 仓库）
- 浏览器内置 TTS（前端能力）

## 8. 与相邻子系统交互

- 上游 → 本叶子：admin/chat endpoints、前端设置页调用三个工厂。
- 本叶子 → 下游：各适配类调外部多模态 API。
- 与 llm-providers 并列（同为 capability seam），互不依赖。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：三个独立 seam（图像/STT/TTS），各有工厂 + duck-typing Provider 实现。
- **图类型侧重**：architecture（三类工厂并列静态拓扑）+ sequence（一次 STT 转写调用链）。
- **外部边界**：外部多模态云 API、浏览器 Web Speech API。
- **部署维度**：与 server 同一 Node 进程；native STT 为浏览器端能力。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 多模态三类工厂架构 | `multimodal-stt-tts-architecture.html` | architecture | showcase（一次通过） |
| STT 语音转写时序 | `multimodal-stt-tts-sequence.html` | sequence | showcase（一次通过） |

JSON IR 源文件位于 `json/` 目录。
