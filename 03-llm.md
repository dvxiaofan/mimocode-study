# 03 · LLM 调用层

## 一句话定位
MiMo-Code 的 LLM 层把 Vercel `ai-sdk` 包成一个"加载即用 + 失败可重试 + 跨 provider 消息归一化"的统一接口——**22 个官方 provider + 1 个 MiMo 匿名免费通道**通过同一个 `Provider.Service` 暴露，运行时按 `provider.npm` 动态 import、按 model 特性（vision/responses-api/region prefix）动态调整消息结构；上游 SSE 连接断了/429/5xx/网络抖动能**指数退避 10 次**（最坏 ~97 分钟），由单一来源 `isRetryableTransientError` 判定可重试错误。

## 涉及文件
- `packages/opencode/src/provider/provider.ts` (1500+ 行，已读头 400 行) — **所有 provider 的统一入口**
- `packages/opencode/src/provider/models.ts` — 远端 `models.dev` 拉取的 model catalog（含 cost / limit / modalities）
- `packages/opencode/src/provider/transform.ts` — 跨 provider 消息归一化（vision 支持、tool call 转换、anthropic 空消息过滤）
- `packages/opencode/src/provider/schema.ts` — `ProviderID` / `ModelID` 带 brand 的 zod schema
- `packages/opencode/src/provider/index.ts` — 5 行 re-export
- `packages/opencode/src/provider/auth.ts` — OAuth / API key 鉴权
- `packages/opencode/src/provider/error.ts` — 错误类型
- `packages/opencode/src/provider/sdk/copilot/` — **GitHub Copilot 自定义适配**（用 `chat/` + `responses/` 双形态）
- `packages/opencode/src/session/llm.ts` (已读前 100 行) — 流式 LLM 调用 + `persistentRetrySchedule`
- `packages/opencode/src/session/retry.ts` — `isRetryableTransientError` + `delay` 计算
- `packages/opencode/src/plugin/mimo-free.ts` — **MiMo 匿名免费通道 plugin**

## 核心数据流

### Provider 加载流程
```
用户运行 `mimo run "fix bug"`
  ↓
session/llm.ts:streamText({ ..., model: language })
  ↓
Provider.Service.getLanguage({ providerID, modelID })
  ↓
1. 查 models.dev catalog（cost/limit/modalities）
2. 查 auth（OAuth/API key）
3. 查 provider registry，定位 @ai-sdk/<name> 包的 dynamic import
4. 实例化 sdk = createAnthropic / createOpenAI / createVertex / ...
5. 按 model 特性选 chat() / responses() / languageModel()
6. 调 ProviderTransform.normalizeMessages() 规整消息
7. 调 ProviderTransform.providerOptions() 加 provider-specific options
  ↓
返回 LanguageModelV3 给 ai-sdk
  ↓
ai-sdk 内置 streamText 走真实 HTTP
  ↓
wrapSSE() 包装 chunk 8 分钟超时
  ↓
返回 AsyncIterable<chunk> 给 caller
```

### Persistent-Retry 决策
```
streamText throws error
  ↓
llm.ts:persistentRetrySchedule (Schedule.exponential("500ms").pipe(Schedule.recurs(10)))
  ↓
isRetryableTransientError(error)?
  ├─ HTTP 429/500/502/503/504/529 → true
  ├─ code in {ECONNRESET, EPIPE, ETIMEDOUT} → true
  ├─ message === "SSE read timed out" → true
  └─ 其它（401/403/400/404/422/abort）→ false（不再重试）
  ↓
delay(attempt, error)?
  ├─ error.data.responseHeaders["retry-after-ms"] → 用之
  ├─ error.data.responseHeaders["retry-after"] (seconds or HTTP date) → 解析后用之
  └─ 否则 → RETRY_INITIAL_DELAY (2000ms) × 2^attempt，cap 30s
  ↓
重试或上抛
```

## 关键设计

### 1. **Provider 注册表：动态 import 22 个 SDK**（`provider.ts:106-131`）
```typescript
const npm = {
  "@ai-sdk/amazon-bedrock": () => import("@ai-sdk/amazon-bedrock").then((m) => m.createAmazonBedrock),
  "@ai-sdk/anthropic":       () => import("@ai-sdk/anthropic").then((m) => m.createAnthropic),
  "@ai-sdk/azure":           () => import("@ai-sdk/azure").then((m) => m.createAzure),
  "@ai-sdk/google":          () => import("@ai-sdk/google").then((m) => m.createGoogleGenerativeAI),
  "@ai-sdk/google-vertex":   () => import("@ai-sdk/google-vertex").then((m) => m.createVertex),
  "@ai-sdk/google-vertex/anthropic": () => import("@ai-sdk/google-vertex/anthropic").then((m) => m.createVertexAnthropic),
  "@ai-sdk/openai":          () => import("@ai-sdk/openai").then((m) => m.createOpenAI),
  "@ai-sdk/openai-compatible": () => import("@ai-sdk/openai-compatible").then((m) => m.createOpenAICompatible),
  "@openrouter/ai-sdk-provider": () => import("@openrouter/ai-sdk-provider").then((m) => m.createOpenRouter),
  "@ai-sdk/xai":             () => import("@ai-sdk/xai").then((m) => m.createXai),
  "@ai-sdk/mistral":         () => import("@ai-sdk/mistral").then((m) => m.createMistral),
  "@ai-sdk/groq":            () => import("@ai-sdk/groq").then((m) => m.createGroq),
  "@ai-sdk/deepinfra":       () => import("@ai-sdk/deepinfra").then((m) => m.createDeepInfra),
  "@ai-sdk/cerebras":        () => import("@ai-sdk/cerebras").then((m) => m.createCerebras),
  "@ai-sdk/cohere":          () => import("@ai-sdk/cohere").then((m) => m.createCohere),
  "@ai-sdk/gateway":         () => import("@ai-sdk/gateway").then((m) => m.createGateway),
  "@ai-sdk/togetherai":      () => import("@ai-sdk/togetherai").then((m) => m.createTogetherAI),
  "@ai-sdk/perplexity":      () => import("@ai-sdk/perplexity").then((m) => m.createPerplexity),
  "@ai-sdk/vercel":          () => import("@ai-sdk/vercel").then((m) => m.createVercel),
  "@ai-sdk/alibaba":         () => import("@ai-sdk/alibaba").then((m) => m.createAlibaba),
  "gitlab-ai-provider":      () => import("gitlab-ai-provider").then((m) => m.createGitLab),
  "venice-ai-sdk-provider":  () => import("venice-ai-sdk-provider").then((m) => m.createVenice),
  "@ai-sdk/github-copilot":  () => import("./sdk/copilot").then((m) => m.createOpenaiCompatible),  // 本地 fork
}
```
**亮点**：**所有 SDK 都走动态 import**——未被使用的 provider 不会进 bundle；GitHub Copilot 走本地 fork 的 `./sdk/copilot`（详见 §4）。

### 2. **Model Catalog 远端拉取 + 5min TTL 缓存**（`models.ts`）
- 数据源：`https://models.dev` 或 `MODELS_DEV_URL` 环境变量覆盖
- 缓存：`{Global.Path.cache}/models.json`（或按 URL hash 命名）
- TTL：5 分钟
- 内容：cost / limit / modalities / tool_call / reasoning / temperature / experimental.modes

### 3. **GPT-5 走 Responses API**（`provider.ts:28-31`）
```typescript
function shouldUseCopilotResponsesApi(modelID: string): boolean {
  const match = /^gpt-(\d+)/.exec(modelID)
  if (!match) return false
  return Number(match[1]) >= 5 && !modelID.startsWith("gpt-5-mini")
}
```
**含义**：GPT-5 系列（除 mini）走 `responses()` 端点而非 `chat()`——为新模型用新协议。

### 4. **GitHub Copilot 本地 fork**（`provider/sdk/copilot/`）
- `chat/` 和 `responses/` 两个子目录各是一份 OpenAI 兼容实现的本地副本
- 用途：避免依赖上游 `@ai-sdk/github-copilot` 的卡版本，**自行维护可改**
- 文件：`openai-compatible-chat-language-model.ts`、`openai-compatible-responses-language-model.ts` 等

### 5. **跨 Provider 消息归一化**（`transform.ts`）
关键 patches：
- `supportsImageInput()`：`mimo-auto` / `mimo-v2.5` 允许图；`mimo-v2.5-pro` 不允许；Claude/GPT 强制多模态
- `normalizeMessages()`：Anthropic / Bedrock 拒绝空消息——过滤空字符串 / 空 text / 空 reasoning
- `providerOptions()`：把 `model.options` merge 到各 provider 的 option key（`anthropic`、`openai`、`vertex` 等）

### 6. **SSE 8 分钟 chunk 超时**（`provider.ts:33-91`）
```typescript
export const DEFAULT_CHUNK_TIMEOUT = 480_000  // 8 minutes
function wrapSSE(res: Response, ms: number, ctl: AbortController) {
  // ... 用 ReadableStream 包装 res.body
  // 每个 chunk 最多等 ms，否则 abort + reject("SSE read timed out")
}
```
**注释明确说**：调优 mimo-v2.5-pro 的"冷启动 TTFT 5 分钟"——但 reasoning model 多分钟思考期间仍会发 partial chunk / heartbeat，故 8 分钟是上限。可在 `mimocode.json` 的 `chunkTimeout` 按 provider 覆盖。

### 7. **Persistent-Retry Schedule**（`llm.ts:67-77`）
```typescript
export const persistentRetrySchedule = Schedule.exponential("500 millis", 2).pipe(
  Schedule.modifyDelay((_, delay) =>
    Effect.succeed(Duration.isLessThanOrEqualTo(delay, Duration.minutes(5)) ? delay : Duration.minutes(5)),
  ),
  Schedule.both(Schedule.recurs(10)),
)
```
- 指数退避：500ms × 2 → 0.5/1/2/4/8/16/32/64/128/256s
- 单次 delay cap：5 分钟
- 总尝试次数：10
- **最坏 11 attempts × 8min chunk + 9min 累计 backoff ≈ 97 min**

### 8. **isRetryableTransientError 单一来源**（`retry.ts:30-55`）
```typescript
export function isRetryableTransientError(error: unknown): boolean {
  if (!(error instanceof Error)) return false
  const status = (error as { status?: number }).status
    ?? (error as { statusCode?: number }).statusCode
    ?? (error as { response?: { status?: number } }).response?.status
  if (typeof status === "number" && RETRYABLE_HTTP_STATUS.has(status)) return true
  const code = (error as { code?: string }).code
  if (typeof code === "string" && NETWORK_ERROR_CODES.has(code)) return true
  if (error.message === SSE_TIMEOUT_MESSAGE) return true
  return false
}
```
- `RETRYABLE_HTTP_STATUS = {429, 500, 502, 503, 504, 529}`
- `NETWORK_ERROR_CODES = {ECONNRESET, EPIPE, ETIMEDOUT}`
- 401/403/400/404/422 和用户主动 abort **不**重试
- 注释明确说：之前 `retryable()` 和 `isTransientCapacityError()` 两套逻辑分叉过，曾导致"一个路径重试 SSE timeout、另一个挂死"——已统一

### 9. **MiMo 匿名免费通道**（`plugin/mimo-free.ts`）
- Bootstrap URL：`https://api.xiaomimimo.com/api/free-ai/bootstrap`（拿匿名 JWT）
- Chat URL：`https://api.xiaomimimo.com/api/free-ai/openai`（OpenAI 兼容）
- 客户端 fingerprint：CPU model + hostname + platform + arch + username → SHA256 → 存 `~/.local/share/mimocode/mimo-free-client`
- JWT 缓存 + inflight 去重 + 自动续期
- 不需要任何 API key 即可使用，是 MiMo-Code 的"零配置开箱即用"杀手锏

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| Provider 数 | ~20 | **22 + MiMo Free** | + alibaba + venice + **MiMo Free 插件** |
| Retry 策略 | Schedule.recurs(5) | **Schedule.recurs(10) + 5min 单次 cap** | 更激进地容忍上游瞬时不可用 |
| Retry 决策 | 分叉 | **`isRetryableTransientError` 单一来源** | 修了"两边各 retry 各的"bug |
| GitHub Copilot | 上游 `@ai-sdk/github-copilot` | **本地 fork** `provider/sdk/copilot/` | 摆脱版本卡死 |
| SSE 超时 | 无 / 短 | **8 分钟 default + per-provider 覆盖** | 适配 reasoning model 长沉默 |
| models.dev | TTL 1h | **TTL 5min** | provider 变动更快可见 |
| Anthropic 跨 region inference prefix | 部分 | **完整 US/EU/AP(JP/AU/APAC) 分支**（provider.ts:400+）| Bedrock 跨区模型自动加 `us.`/`eu.`/`jp.`/`au.`/`apac.` |
| MiMo vision | 走 models.dev data | **写死 patch**（`mimo-v2.5-pro` 文本、其它多模态） | 修 models.dev 不准的硬编码 |
| Mimo Free 通道 | 无 | **匿名 JWT 插件** | 零配置开箱即用 |
| `providerOptions` 合并 | 简单 | **根据 `npm` 包名映射成 `anthropic`/`openai`/`vertex` 等 key** | 跨 provider 统一 options |

## 关键引用

- `packages/opencode/src/provider/provider.ts:106-131` — 22 provider 注册表
- `packages/opencode/src/provider/provider.ts:28-91` — `shouldUseCopilotResponsesApi` + `wrapSSE`
- `packages/opencode/src/provider/transform.ts:30-46` — `supportsImageInput` 的硬编码 patch
- `packages/opencode/src/provider/transform.ts:50-70` — `sdkKey` npm→ai-sdk-key 映射
- `packages/opencode/src/session/llm.ts:30-50` — `OUTPUT_TOKEN_MAX` + mimo 128k override
- `packages/opencode/src/session/llm.ts:67-77` — `persistentRetrySchedule`
- `packages/opencode/src/session/retry.ts:30-55` — `isRetryableTransientError`
- `packages/opencode/src/session/retry.ts:57-90` — `delay()` 解析 `retry-after` / `retry-after-ms`
- `packages/opencode/src/plugin/mimo-free.ts:1-50` — fingerprint 缓存 + bootstrap 流程

## 学习检查清单

读完后应能回答：
- [ ] MiMo-Code 用了哪个 LLM SDK？支持多少个 provider？
- [ ] Persistent-retry 的最坏总时长是多少？单次 delay 和 attempt 数分别是多少？
- [ ] `isRetryableTransientError` 判定的可重试错误有哪三类？哪些**不**重试？
- [ ] MiMo Free 通道的认证机制是什么？为什么能"零配置开箱即用"？
- [ ] 为什么 GitHub Copilot 要本地 fork 一份 `@ai-sdk/openai-compatible`？
- [ ] SSE 8 分钟 chunk timeout 是为了解决什么场景的问题？
