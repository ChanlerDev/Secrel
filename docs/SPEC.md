# Secrel LLM Gateway 产品规格

本文定义 Secrel 的当前产品边界和核心行为，是设计、实现和验收时的主规格。未被本文覆盖的新能力应先通过 proposal 澄清产品价值、边界和风险，再进入实现计划。

## 1. 产品定位

Secrel 是一个面向个人开发者、小团队和自动化服务的 LLM API Gateway，支持本地桌面 App 和服务端部署两种形态。它为多个上游 LLM Provider、自定义 Base URL、模型路由、协议转换、凭证管理和用量统计提供统一入口。

核心主路径是 LLM API Proxy：客户端通过 Secrel 调用 OpenAI-compatible、OpenAI Responses、Anthropic Messages 或 Gemini 风格 API，由 Gateway 完成鉴权、路由、凭证注入、协议适配和日志记录。MCP Proxy 属于规划中的扩展资源代理能力；在对应 proposal 被接受或合并前，它不属于本文定义的基础交付范围。

## 2. 交付形态

系统采用一套共享 Gateway Core 和统一 Admin Surface，外加两个产品壳。服务端部署是优先交付形态；本地 App 复用同一套管理能力，既可以管理本机 daemon，也可以作为远程 Server 的桌面管理端。

统一 Admin Surface 包括：

- Provider、模型、alias、路由策略和 Price Table 配置。
- Client Key 管理。
- 日志、用量、成本估算和健康状态查看。
- Admin API 和管理 UI 的核心信息架构。

两个产品壳的差异主要来自部署环境、凭证存储、安全边界和系统集成，而不是管理能力本身。

### 2.1 本地 App

本地产品是 Tauri 桌面 App。

- 运行本地 Gateway daemon。
- 默认只监听 `127.0.0.1`。
- 使用 OS Keychain 存储本地敏感凭证。
- 复用统一 Admin Surface，提供 Provider、模型、Client Key、日志、用量、Price Table 和健康状态的可视化管理。
- 面向 Codex、Claude Code、Cursor、Cherry Studio、Open WebUI、本地脚本和 SDK 开发等场景。
- App 不只管理本机，也可以通过切换 Admin API Endpoint 管理远程 Server。
- 本机客户端一键配置、本地 daemon 状态和 OS Keychain 状态属于本地 App 专属能力。

### 2.2 服务端部署

服务端产品是后端服务加 Web Admin。

- 通过 Docker 部署。
- 暴露可被网络访问的 LLM API Endpoint。
- 复用统一 Admin Surface，提供 Web Admin 进行配置管理、日志查看、用量统计、成本估算和健康状态查看。
- 使用服务端加密存储 Provider 凭证。
- 采用单 Owner + 多 Client API Key 模型。
- Admin 鉴权与 LLM 调用 API Key 必须分离。
- 面向远程服务、多设备、自动化任务和私有共享调用场景。

## 3. 核心架构

本地 App 和服务端部署共享同一套 Gateway Core。早期实现优先服务端路径：先让 Gateway Core 在服务端稳定提供远程 LLM API、Admin API、配置管理、日志和用量，再复用这些能力封装本地 App。

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
Upstream Provider / Model Service
```

Gateway Core 负责：

- Provider 配置。
- 上游凭证引用。
- 模型发现和手动模型注册。
- 自动生成对外暴露的 Model ID 索引。
- 模型别名和可选前缀规则。
- 路由策略。
- 路由优先级、回退规则、重试预算和可用性状态管理。
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

路由层以 upstream 为主要可用性单位。健康探测和连续失败会优先标记 upstream 是否可路由；如果某个具体 model 在特定 upstream 上持续失败，系统可以记录为补充性的模型可用性状态，但不取代 upstream 级别的可用性判断。

## 5. 模型注册与对外暴露

模型路由以客户端请求中的 `model` 字段为主键。Gateway 先按请求中最具体的可匹配名称解析模型，再把同一个 canonical model ID 下的多个来源合并为一个对外 model entry 的候选集合。

用户可以配置多个 upstream。多个 upstream 可能暴露重复的 model ID，例如多个 Anthropic 协议中转站都暴露 `claude-sonnet-4.6`。Gateway 把重复 model ID 视为同一个对外 model entry 的多个候选 upstream，而不是配置冲突。

模型注册规则：

- Provider / upstream 是独立配置。
- Upstream model ID 可以重复。
- 每个 upstream model ID 默认都会被注册为可对外请求的 model ID。
- 用户不需要手动创建 `exposed_model_id`。
- 路由层先解析请求中的 model，再决定是否进入 prefix 作用域或 canonical fallback，最后从候选 upstream 中选择实际目标。

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

### 5.3 Prefix 与作用域

Prefix 不是独立的计费或模型真值，它只定义模型的可见范围和路由候选边界。标准 model ID 始终以不带 prefix 的 canonical 形式存在。

路由解析规则：

- 请求不带 prefix 时，可以回退到带 prefix 的 upstream 候选。
- 请求带 prefix 时，是否可以回退到无 prefix 的同名 model，必须由该 prefix 作用域的配置决定。
- 当多个 upstream 命中同一个 Model ID 时，不论是否带 prefix，都先按 `priority` 排序，再应用路由策略。

Prefix fallback 只影响候选集合，不改变 canonical model ID，也不改变成本计算所使用的价格键。

## 6. 路由策略

当一个对外 model entry 有多个候选 upstream 时，Gateway 按路由策略选择目标。

用户不需要为每个对外 model ID 手动建路由。常见配置入口是：

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

Gateway 应支持显式模型前缀以避免歧义；当客户端请求不带前缀时，也可以通过自动生成的 model entry 和路由策略完成选择。

### 6.1 重试与回退

当 Gateway 已经选定某个 upstream 后，应优先在该 upstream 上消耗重试预算，再把请求切换到下一个候选 upstream。

重试节奏如下：

- 第一次立即重试。
- 第二次重试前等待 1 秒。
- 第三次重试前等待 2 秒。
- 第四次重试前等待 4 秒。

这意味着单个 upstream 的一次尝试最多包含 1 次初始请求和 4 次重试。Retry 只针对可恢复的传输错误、超时、限流、上游不可用以及同类可重试失败；不可恢复的客户端错误不进入重试流程。具体错误码映射由协议适配层和 Provider 适配层统一解释。

### 6.2 路由优先级与策略

`priority` 是跨 prefix 作用域的排序依据。它决定当一个请求命中多个具有相同 Model ID 的 upstream 时，Gateway 优先考虑哪些候选。

在同一优先级组内：

- `fill_first`：优先使用最先可用的健康 upstream。
- `round_robin`：在可用 upstream 间轮询。
- `fallback`：先尝试主 upstream，主 upstream 不可用时再切换备选。

路由策略负责决定候选顺序，重试机制负责决定单个候选的重复尝试，两者是不同层次的控制。

## 7. 协议支持

系统重点支持四类协议：

- OpenAI-compatible Chat Completions。
- OpenAI Responses API，用于 Codex 类客户端。
- Anthropic Messages API，用于 Claude Code 类客户端。
- Gemini API。

Gateway 对外暴露这些协议兼容入口，并按请求中的 `model` 字段路由到具体 upstream。

## 8. 协议转换策略

协议转换采用 passthrough 优先策略。只有入口协议与选中的 upstream 协议不一致时，Gateway 才执行跨协议转换。

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

跨协议转换遇到未知或小众字段时，默认跳过该字段并继续处理请求。Gateway 应在日志里记录 compatibility warning，便于用户排查。

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

跨协议转换的基础范围是文本增量、基础 tool call 事件、usage、finish 和 error。Provider 专有 streaming 事件，例如 reasoning stream、computer use stream、多模态二进制流或私有事件图，只在同协议 passthrough 中尽量保留；跨协议转换不保证完整保留。

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

Price Table 的真值键是标准、无 prefix 的 canonical model ID。Price Table 可以来自内置默认值，也可以来自用户手动维护的配置；无论来源如何，usage 成本都必须按路由后实际调用的 canonical model ID 计价，而不能按请求中的原始 model 字符串或 alias 计价。

## 12. 管理 UI

本地 Tauri App 和服务端 Web Admin 共享同一套管理信息架构和主要页面。产品应优先把配置、日志、用量、成本和健康状态建成统一 Admin Surface，再由本地 App 和服务端 Web Admin 以各自 shell 承载。

共享页面：

- Provider 管理。
- 模型发现。
- 手动模型注册。
- 模型别名和前缀管理。
- 自动生成的对外模型索引查看。
- 路由策略配置。
- Client API Key 管理。
- 用量和日志。
- 健康状态。
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

基础产品边界：

- 公共 SaaS 计费系统不属于核心能力。
- CDN 类加速不属于核心能力。
- Provider 账号池不属于基础模型。
- CLI / OAuth account proxy 不属于基础 Provider 类型。
- 协议转换以 Chat Core 的 best-effort 兼容为目标，不保证完全无损。
- 跨协议转换不保证完整保留 computer use、私有 reasoning event 等高级 Provider 专有能力。

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
