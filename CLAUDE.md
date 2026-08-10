# junie-api

## 构建与测试
- 构建：`npm run build`（使用 tsdown）
- 测试：`npm test`（通过 tsx 使用 Node 内置测试运行器——**未安装 bun**）
- 开发：`node --watch --import tsx ./src/main.ts`
- 类型检查：`npm run typecheck`（仅执行 `tsc`，不输出文件）
- 注意：`npm run lint` **无法运行**——`eslint` 未安装在 devDependencies 中，也没有 eslint 配置文件

## 框架栈
- **HTTP 框架**：Hono
- **服务器适配器**：srvx（跨运行时 HTTP 适配器）
- **CLI 解析**：citty
- **路径别名**：`~/*` → `./src/*`（在所有相对导入中使用）

## 架构

三条路由路径：

### 1. Anthropic 原生直通（模型名以 `claude-` 开头）
- 将 Anthropic `/v1/messages` 请求转发到 `ingrazzio-cloud-prod.labs.jb.gg/v1/messages`
- 转发前清理 payload（已知字段白名单）
- 无格式转换——请求/响应均为原生 Anthropic 协议
- 流式传输：直接将上游 Anthropic SSE 管道传输给客户端
- 路由：`handler.ts` → `create-anthropic-passthrough.ts` → 上游

### 2. OpenAI 原生直通（模型名以 `openai-` 开头）
- 将 OpenAI `/v1/chat/completions` 请求转发到 `ingrazzio-cloud-prod.labs.jb.gg/v1/chat/completions`
- 将 Grazie 配置文件 ID 映射为上游 OpenAI 模型名（例如 `openai-gpt4.1` → `gpt-4.1-2025-04-14`）
- 无格式转换——请求/响应为原生 OpenAI 协议
- 对于 `/v1/messages`：先进行 Anthropic→OpenAI 转换，通过直通发送，再将响应转换回来
- 路由：`handler.ts` → `create-openai-passthrough.ts` → 上游

### 3. 翻译路径（Google 模型）
- Anthropic 格式 → OpenAI 格式 → Grazie 原生协议
- 封装为 `{ profile, chat: { messages } }` 格式，发送至 `api.jetbrains.ai/llm/chat/stream/v9`
- 将 Grazie/OpenAI 响应翻译回 Anthropic 格式
- 路由：`handler.ts` → `create-chat-completions.ts` → Grazie 网关

## Ingrazzio API

**Anthropic 直通：**
- 端点：`https://ingrazzio-cloud-prod.labs.jb.gg/v1/messages`
- 认证：`Authorization: Bearer <token>`（JBA JWT 或 `perm-*` 密钥）
- 模型 ID：`claude-sonnet-5`、`claude-opus-5`
- 参见 `api-config.ts` 中的 `getAnthropicPassthroughHeaders()`

**OpenAI 直通：**
- 端点：`https://ingrazzio-cloud-prod.labs.jb.gg/v1/chat/completions`
- 认证：`Authorization: Bearer <token>`（JBA JWT 或 `perm-*` 密钥）
- 模型 ID：从 Grazie 配置文件映射到上游名称（`openai-gpt4.1` → `gpt-4.1-2025-04-14`）
- 参见 `api-config.ts` 中的 `getOpenAIPassthroughHeaders()` 和 `resolveOpenAIModelId()`

**Grazie 网关（翻译路径）：**
- 端点：`https://api.jetbrains.ai/llm/chat/stream/v9`
- 模型 ID：Grazie LLMProfileID，例如 `google-chat-gemini-pro-2.5`

**认证验证：** `GET https://ingrazzio-cloud-prod.labs.jb.gg/auth/test`

## 模型 ID

Anthropic 模型：`claude-sonnet-5`、`claude-opus-5`

OpenAI 模型（直通，映射为上游名称）：
- `openai-gpt4.1` → `gpt-4.1-2025-04-14`
- `openai-gpt4.1-mini` → `gpt-4.1-mini-2025-04-14`
- `openai-gpt-4o` → `gpt-4o`

Google 模型（Grazie LLMProfileID，翻译路径）：
- `google-chat-gemini-pro-2.5`、`google-chat-gemini-flash-2.5`

完整列表参见 `src/services/grazie/get-models.ts`。

## TypeScript 注意事项

- `noUnusedLocals` 和 `noUnusedParameters` 在构建时强制执行——添加未使用的变量会导致构建失败
- `@types/bun` 在 devDependencies 中，但不是因为使用了 Bun——这是 srvx 类型兼容所需的

## 认证 Token 行为
- `perm-*` 密钥和 JBA OAuth JWT：使用 `Authorization: Bearer <token>`
- GitHub token（`ghp_`、`github_pat_`）：使用 `Authorization: GitHub <token>`
