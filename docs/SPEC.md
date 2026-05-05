# LLM Gateway 产品规格

## 1. 产品定位

Secrel 是一个 LLM API Gateway，支持本地桌面 App 和服务端部署两种形态。它为多个上游 LLM Provider、自定义 Base URL、模型路由、协议转换、凭证管理、用量统计和 MCP Proxy 提供统一入口。

本产品不应被限制为“本地 MCP Proxy”。MCP 是建立在同一套 Gateway 基础上的资源代理能力，LLM API Proxy 是核心主路径。

## 2. 交付形态

系统采用一套共享 Gateway Core，外加两个产品壳。

### 2.1 本地 App

本地产品是 Tauri 桌面 App。

- 运行本地 Gateway daemon。
- 默认只监听 `127.0.0.1`。
- 使用 OS Keychain 存储本地敏感凭证。
- 提供 Provider、模型、Client Key、日志和用量的可视化管理。
- 面向 Codex、Claude Code、Cursor、Cherry Studio、Open WebUI、本地脚本和 SDK 开发等场景。
- App 不只管理本机，也可以通过切换 Admin API Endpoint 管理远程 Server。

### 2.2 服务端部署

服务端产品是后端服务加 Web Admin。

- 通过 Docker 部署。
- 暴露可被网络访问的 LLM API Endpoint。
- 提供 Web Admin 进行配置管理。
- 使用服务端加密存储 Provider 凭证。
- 采用单 Owner + 多 Client API Key 模型。
- Admin 鉴权与 LLM 调用 API Key 必须分离。
- 面向远程服务、多设备、自动化任务和私有共享调用场景。

## 3. 核心架构

本地 App 和服务端部署共享同一套 Gateway Core。

```text
Client / SDK / Agent / Service
        |
Gateway API Endpoint
        |
识别入口协议
        |
按请求中的 model ID 路由
        |
协议是否一致？
  ├── 是：Raw Passthrough
  └── 否：Protocol Transform
        |
Upstream Provider / AppStream
```

Gateway Core 负责：

- Provider 配置。
- 上游凭证引用。
- 模型发现和手动模型注册。
- 自动生成对外暴露的 Model ID 索引。
- 模型别名和可选前缀规则。
- 路由策略。
- 协议适配。
- Streaming 处理。
- Client API Key 鉴权。
- 用量日志和成本估算。
- 敏感信息脱敏。

## 4. Provider 与上游配置

Provider 表示一个上游 API 服务。它可以是官方 Provider、第三方中转站、自托管模型服务，或任何实现受支持协议的自定义 Base URL。

Provider 字段：

- `name`
- `protocol`: `openai_chat`、`openai_responses`、`anthropic`、`gemini`
- `base_url`
- `credential_ref`
- `models_endpoint`
- `enabled`
- `health_status`

系统必须支持：

- 配置任意 Base URL。
- 配置 API Key 或等价 Bearer 凭证。
- 从 Provider 拉取模型列表。
- 当 Provider 不提供可用 models endpoint 时，允许手动添加模型。
- 为模型配置别名。
- 为 Provider 或模型配置可选前缀规则。

## 5. 模型注册与对外暴露

模型路由以客户端请求中的 `model` 字段为主键。

用户可以配置多个 upstream。多个 upstream 可能暴露重复的 model ID，例如多个 Anthropic 协议中转站都暴露 `claude-sonnet-4.6`。这种重复是允许的，不应被视为冲突。

正确理解是：

- Provider / upstream 是独立配置。
- Upstream model ID 可以重复。
- 每个 upstream model ID 默认都会被注册为可对外请求的 model ID。
- 用户不需要手动创建 `exposed_model_id`。
- 路由层根据请求中的 model ID 找到候选 upstream，再决定实际选择哪个 upstream。

### 5.1 Upstream Model

Upstream Model 是从 Provider 拉取或手动添加的真实上游模型。

字段：

- `provider_id`
- `upstream_model_id`
- `upstream_protocol`
- `display_name`
- `enabled`
- `metadata`

示例：

```text
provider: relay-a
upstream_model_id: claude-sonnet-4.6
upstream_protocol: anthropic

provider: relay-b
upstream_model_id: claude-sonnet-4.6
upstream_protocol: anthropic
```

这两个是不同的 Upstream Model，只是上游 model ID 相同。

### 5.2 自动生成的 Exposed Model Index

Exposed Model Index 是 Gateway 自动维护的对外模型索引，不是用户手动配置的资源。

来源：

- Provider 拉取到的 upstream model ID。
- 用户手动添加的 upstream model ID。
- 用户为某个模型配置的 alias。
- 可选的 Provider / model prefix 规则。

当多个 upstream 暴露同一个 model ID 时，Gateway 自动把它们归入同一个对外 model entry 的候选列表。

字段：

- `exposed_model_id`
- `aliases`
- `source_model_id`
- `candidate_upstreams`
- `effective_route_policy`

示例：

```text
exposed_model_id: claude-sonnet-4.6
candidate_upstreams:
  - relay-a / claude-sonnet-4.6
  - relay-b / claude-sonnet-4.6
```

如果用户给 `relay-a / claude-sonnet-4.6` 或该模型组配置别名：

```text
alias: sonnet
alias: code-sonnet
```

这些别名也会注册为可请求的 model ID，并路由到对应候选 upstream。

客户端请求：

```json
{
  "model": "claude-sonnet-4.6",
  "messages": []
}
```

Gateway 根据请求中的 `model` 找到自动生成的 model entry，再按路由策略选择其中一个 upstream。

## 6. 路由策略

当一个对外 model entry 有多个候选 upstream 时，Gateway 按路由策略选择目标。

用户不需要为每个对外 model ID 手动建路由。常见配置入口应该是：

- Provider 级默认 priority。
- Upstream model 级 priority 覆盖。
- 同 model ID 候选之间的 route policy。
- 模型 alias。

系统应支持：

- `fill_first`：优先使用第一个健康 upstream，直到失败、限流、余额不足或不可用后再切换。
- `priority`：选择优先级最高的健康 upstream。
- `round_robin`：在健康 upstream 之间轮询。
- `fallback`：先尝试主 upstream，符合条件的错误发生后再尝试备选。

可扩展策略：

- `weighted_round_robin`
- quota-aware routing
- cost-aware routing
- latency-aware routing
- client-specific routing

Gateway 应支持显式模型前缀以避免歧义，但当前端请求不带前缀时，也可以通过自动生成的 model entry 和路由策略完成选择。

## 7. 协议支持

系统重点支持四类协议：

- OpenAI-compatible Chat Completions。
- OpenAI Responses API，用于 Codex 类客户端。
- Anthropic Messages API，用于 Claude Code 类客户端。
- Gemini API。

Gateway 对外暴露这些协议兼容入口，并按请求中的 `model` 字段路由到具体 upstream。

## 8. 协议转换策略

Gateway 不应对所有请求强制转换。

### 8.1 同协议 Fast Path

当入口协议与选中的 upstream 协议一致时，Gateway 走 Raw Passthrough。

该路径只做：

- Client 鉴权。
- 解析并路由 model ID。
- 将 exposed model ID 替换为 upstream model ID。
- 注入 upstream 凭证。
- 日志脱敏。
- 尽量保持请求和响应结构不变。

这样可以保留 Provider 专有能力，避免无意义字段损失。

### 8.2 跨协议 Transform Path

当入口协议与 upstream 协议不一致时，Gateway 执行协议转换。

流程：

```text
source protocol request
  -> typed source request
  -> gateway chat core representation
  -> typed target request
  -> upstream
  -> typed target response
  -> gateway response representation
  -> source protocol response
```

系统支持四种协议之间的 Chat Core best-effort 转换。

Chat Core 包括：

- system prompt
- user / assistant messages
- text content
- temperature
- max tokens
- stop sequences
- 基础 tool definitions
- 基础 tool calls
- streaming text deltas
- finish reason
- 可获得时的 usage

跨协议转换遇到未知或小众字段时，默认跳过，不应直接让整个请求失败。Gateway 应在日志里记录 compatibility warning，便于用户排查。

同协议 passthrough 应保留未知字段。

## 9. Streaming

系统必须支持 Streaming。

### 9.1 同协议 Streaming

当入口协议与 upstream 协议一致时，stream 数据应直接透传。

### 9.2 跨协议 Streaming

当协议不一致时，Gateway 做事件级转换。

```text
upstream stream parser
  -> gateway stream event
  -> downstream stream encoder
```

Stream Event：

- start
- text delta
- tool call start
- tool call delta
- tool call end
- usage
- finish
- error

系统不承诺在跨协议转换时完整保留 Provider 专有 streaming 事件，例如 reasoning stream、computer use stream、多模态二进制流或私有事件图。同协议 passthrough 应尽量保留这些事件。

## 10. 鉴权与安全

服务端模式必须区分 Admin 鉴权和 LLM API 调用鉴权。

- Admin token/session 只能访问管理 API。
- Client API Key 只能访问 LLM inference API。
- Client API Key 不能访问 Admin API。
- Admin 凭证不能被当作 LLM Client API Key 使用。

服务端用户模型：

- 单 Owner。
- 多 Client API Key。
- Client Key 可启用/禁用。
- 可选按 Client Key 配置可访问模型。
- 按 Client Key 统计用量。

本地模式可以生成默认本地 Client Key，但鉴权模型应与服务端保持兼容。

敏感数据要求：

- 不记录 upstream API key。
- 对 Authorization Header 脱敏。
- 错误信息中不得暴露 Provider 凭证。
- Provider 凭证必须加密存储。
- 保存后 UI 不回显完整 Secret。

## 11. 用量与成本

系统应记录：

- 请求次数。
- 可获得时的 input tokens。
- 可获得时的 output tokens。
- 可获得时的 cache tokens。
- 可获得时的 reasoning tokens。
- 请求中的 model ID。
- 实际选中的 upstream provider 和 upstream model。
- Client Key。
- latency。
- status 和 error category。

成本估算通过手动配置 price table 实现。Price table 按真实 model ID 配置。为了降低配置成本，UI 可以允许用户从已发现或手动添加的 upstream models 中选择 model ID 并填入价格。

Price Table 可包含：

- model ID。
- input token 单价。
- output token 单价。
- cache read/write 单价。
- reasoning token 单价。
- currency。
- effective date。

系统计算 usage 成本时不能使用客户端请求中的 model ID 或 alias 直接匹配价格，而应使用路由后实际调用的 upstream model ID 匹配 price table。

## 12. 管理 UI

本地 Tauri App 和服务端 Web Admin 可以共享大部分页面，但它们是不同产品壳。

共享页面：

- Provider 管理。
- 模型发现。
- 手动模型注册。
- 模型别名和前缀管理。
- 自动生成的对外模型索引查看。
- 路由策略配置。
- Client API Key 管理。
- 用量和日志。
- Price Table 配置。

本地专属页面：

- 本地 daemon 状态。
- 本地监听地址。
- OS Keychain 状态。
- App 启动设置。
- 本机客户端一键配置。

服务端专属页面：

- Admin 登录和 session 管理。
- 部署状态。
- 服务端存储设置。
- Public API Endpoint 设置。

本地 App 应支持管理本地 daemon 或远程 server，只需要切换 Admin API Endpoint。

## 13. 自定义 Provider

自定义 Provider 指：

- 自定义 Base URL。
- 选择一个已支持协议。
- 配置 API Key。
- 配置 models endpoint。
- 支持手动添加模型作为 fallback。

系统不要求用户编写任意协议脚本。自定义 Provider 必须映射到已支持协议之一。

## 14. 产品边界

产品边界：

- 不把公共 SaaS 计费系统作为核心能力。
- 不把 CDN 类加速作为核心能力。
- 不要求 Provider 账号池作为基础模型。
- 不要求 CLI / OAuth account proxy 作为基础 Provider 类型。
- 不承诺完全无损协议转换。
- 不承诺在跨协议转换时完整保留 computer use、私有 reasoning event 等高级 Provider 专有能力。

## 15. 扩展方向

可扩展方向：

- MCP Proxy 和 MCP 配置管理。
- CLI / OAuth Provider Adapter。
- Weighted routing。
- Budget 和 quota policy。
- Provider health check 和自动禁用。
- Cost-aware routing。
- Latency-aware routing。
- 团队 / 多用户管理。
- SaaS 部署形态。
- 按协议和字段维护更完整的 compatibility matrix。

## 16. 产品成功标准

产品成功标准：

- 用户可以配置多个 upstream provider、自定义 Base URL 和 API Key。
- 用户可以从 Provider 拉取模型，或手动添加模型。
- 多个 Provider 暴露重复 upstream model ID 时不会冲突。
- Upstream model ID 和 alias 会自动注册为可对外请求的 model ID。
- 用户可以配置模型别名、前缀、priority 和路由策略。
- 客户端可以用 OpenAI-compatible、Responses、Anthropic 或 Gemini 风格 API 调用 Gateway。
- 同协议请求走 raw passthrough。
- 跨协议请求支持 Chat Core 转换。
- Streaming 支持同协议透传和跨协议文本增量转换。
- 服务端可通过 Docker 部署。
- 本地端通过 Tauri App 运行。
- Admin 鉴权和 LLM Client API Key 明确分离。
- 管理 UI 可查看用量和成本估算。
