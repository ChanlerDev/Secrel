# Proposal: MCP Proxy

## 1. 元信息

- Status: Draft
- Owner:
- Created: 2026-05-06
- Updated: 2026-05-06
- Related Product Spec Sections: 产品定位、核心架构、交付形态、鉴权与安全、扩展方向
- Target Product Surface: Gateway Core / Local App / Server Web Admin / MCP

## 2. 背景

当前 Agent、IDE 和 Coding Assistant 通常通过 `mcp.json`、环境变量或客户端本地配置直接接入 MCP Server。这种方式有几个问题：

- MCP 配置分散在多个客户端中，难以统一维护。
- MCP Token、OAuth Token 和命令环境变量容易以明文形式暴露。
- 用户需要手写或复制复杂配置，普通用户不易理解。
- 本地工具、远程服务和不同 Agent 客户端无法共享同一套 MCP 配置。
- 缺少统一的 MCP 调用日志、错误排查、启停管理和权限控制。

Secrel 已经定位为 LLM Gateway。MCP Proxy 应作为同一套 Gateway Core 上的资源代理能力，复用 Credential Store、Admin 鉴权、Client Key、日志脱敏和本地 / 服务端两种交付形态。

## 3. 用户场景

- 作为本地开发者，我希望在 Tauri App 中可视化管理 MCP Server，而不是手写多个客户端的 `mcp.json`。
- 作为使用多个 Agent 客户端的用户，我希望只在 Secrel 中配置一次 MCP，然后让 Codex、Claude Code、Cursor 等客户端统一连接。
- 作为服务端部署用户，我希望把部分 MCP 能力通过 Server Gateway 暴露给远程服务或自动化任务。
- 作为安全敏感用户，我希望 MCP Token 和 OAuth Token 不出现在客户端配置、环境变量和日志中。
- 作为排查问题的用户，我希望能看到 MCP Server 连接状态、工具列表、调用记录和脱敏错误信息。

## 4. 产品目标

- 统一管理 MCP Server 配置，减少手写 `mcp.json`。
- 代理 MCP Client 与真实 MCP Server 之间的连接和工具调用。
- 托管 MCP 所需 Token、OAuth Token、环境变量密钥和其他敏感配置。
- 在 Local App 和 Server Web Admin 中提供一致的 MCP 管理体验。
- 与 LLM API Gateway 共享鉴权、安全、日志和可观测性模型。
- 支持本地 stdio MCP 和远程 HTTP / SSE 类 MCP 接入。

## 5. 非目标

- 不把 MCP Proxy 设计成独立于 LLM Gateway 的另一套产品。
- 不在 proposal 阶段定义完整 MCP runtime 调度平台。
- 不承诺任意 MCP Server 的高级故障恢复、自动扩缩容或复杂热迁移。
- 不要求把所有 MCP 协议差异抽象成统一工具语义。
- 不把 Client API Key 和 Admin Token 混用。

## 6. 产品规格

### 6.1 用户流程

用户在 Local App 或 Server Web Admin 中新增 MCP Server：

1. 选择 MCP 类型：`stdio`、`http`、`sse` 或其他受支持传输类型。
2. 填写名称、描述、命令 / URL、参数、环境变量和所需凭证引用。
3. 保存后，Secrel 对敏感字段加密存储，UI 不回显完整 secret。
4. 用户可执行连接测试，查看工具列表和基础 server metadata。
5. 用户为 MCP Server 设置启用状态、可访问 Client Key 和可选标签。
6. 客户端只连接 Secrel 暴露的 MCP Proxy Endpoint，不直接持有真实 MCP 配置和 secret。

用户可以从现有 `mcp.json` 导入配置，也可以导出由 Secrel 代理后的客户端配置。

### 6.2 配置模型

MCP Server 配置字段：

- `name`
- `description`
- `transport`: `stdio`、`http`、`sse`
- `command`、`args`、`cwd`：仅 stdio 类型需要。
- `url`：HTTP / SSE 类型需要。
- `headers`
- `env`
- `credential_refs`
- `enabled`
- `tags`
- `client_key_policy`
- `timeout`
- `restart_policy`

MCP Tool Index 由连接发现生成，不应要求用户手动维护。

Tool Index 字段：

- `mcp_server_id`
- `tool_name`
- `description`
- `input_schema`
- `enabled`
- `last_seen_at`

### 6.3 API / 协议行为

Secrel 对外提供 MCP Proxy Endpoint。客户端看到的是 Secrel 管理后的 MCP Server，而不是真实 upstream MCP 配置。

代理行为：

- 对 stdio MCP，Secrel 负责启动和管理真实 MCP Server 进程。
- 对 HTTP / SSE MCP，Secrel 负责连接真实远程 MCP Server。
- Secrel 在代理时注入真实 Token、Headers、Env 或其他凭证。
- 客户端侧只使用 Secrel 提供的本地或远程 Endpoint。
- Secrel 应保留 MCP 请求语义，不主动改写 tool 输入输出，除非为了凭证注入、权限控制或错误脱敏。

### 6.4 UI 行为

共享 MCP 管理页面：

- MCP Server 列表。
- 新增 / 编辑 / 删除 MCP Server。
- 启用 / 禁用 MCP Server。
- 连接测试。
- 工具列表查看。
- 调用日志查看。
- 从 `mcp.json` 导入。
- 导出代理后的客户端配置。

Local App 额外页面 / 状态：

- 本地 stdio 进程状态。
- 进程启动失败、退出码和 stderr 脱敏摘要。
- 本机客户端一键配置。
- OS Keychain 状态。

Server Web Admin 额外页面 / 状态：

- 远程访问 Endpoint。
- Client Key 访问策略。
- Server 侧进程运行限制提示。
- 服务端加密存储状态。

### 6.5 权限与安全

MCP Proxy 必须复用 Secrel 的鉴权边界：

- Admin Token / Session 用于管理 MCP Server。
- Client API Key 用于访问被允许的 MCP Proxy Endpoint。
- Client API Key 不能访问 MCP 管理接口。
- MCP Token、OAuth Token、Header secret、env secret 必须加密存储。
- 日志中不得记录完整 secret、Authorization Header 或敏感环境变量。
- UI 保存 secret 后不回显完整值，只允许替换或删除。

本地模式优先使用 OS Keychain 保存主密钥或敏感凭证。服务端模式使用服务端加密存储。

### 6.6 日志、用量与可观测性

系统应记录：

- MCP Server 启停事件。
- 连接测试结果。
- Tool discovery 结果。
- Tool call 开始、结束、耗时和状态。
- 调用方 Client Key。
- 目标 MCP Server。
- 错误类型和脱敏错误信息。

系统不应记录：

- 完整 tool 输入中的显式 secret。
- Authorization Header。
- 完整环境变量。
- OAuth Token、API Key 或 Cookie。

## 7. 与主产品规格的关系

影响主产品规格：

- 产品定位：MCP Proxy 是 Gateway Core 上的资源代理能力。
- 核心架构：Gateway Core 需要支持 MCP resource proxy。
- 交付形态：Local App 和 Server Web Admin 都需要 MCP 管理入口。
- 鉴权与安全：MCP 管理接口与 MCP 调用接口必须区分权限。
- 日志与可观测性：需要加入 MCP 连接和 tool call 日志。

本文在 Draft 阶段不直接替代主产品规格。当 MCP Proxy 产品边界确认后，应把稳定内容合并进主产品规格，并将本文状态改为 `Merged`。

## 8. 技术影响

- Core 影响：新增 MCP server registry、credential refs、tool index、MCP proxy runtime 抽象。
- Local App 影响：需要管理 stdio 子进程、OS Keychain、客户端配置导入导出。
- Server 影响：需要限制服务端 stdio 进程能力，避免任意命令执行风险；远程 HTTP / SSE MCP 更适合服务端默认路径。
- 数据存储影响：新增 MCP server、tool index、call log、credential refs。
- API 影响：新增 Admin MCP 管理 API 和 MCP Proxy 访问 Endpoint。
- 测试影响：需要覆盖 stdio 启动失败、远程连接失败、secret 脱敏、权限隔离和 tool discovery。

## 9. 边界情况

- MCP Server 启动后立即退出：UI 应显示失败状态、退出码和脱敏 stderr。
- MCP Server tool schema 变化：Tool Index 应支持刷新，并记录 `last_seen_at`。
- 远程 MCP 返回认证错误：日志应提示认证失败，但不能显示真实 token。
- Client Key 无权限访问某个 MCP Server：返回明确权限错误。
- stdio MCP 需要 secret env：secret env 只在进程启动时注入，不写入客户端配置。
- 服务端部署允许 stdio MCP 时存在任意命令执行风险，应提供明确安全提示或默认禁用。

## 10. 待确认问题

- 服务端模式是否默认禁用 stdio MCP，只允许 HTTP / SSE MCP？
- MCP Proxy Endpoint 对客户端应呈现为单个聚合 MCP Server，还是每个 MCP Server 一个独立 Endpoint？
- Tool 级权限是否需要进入基础规格，还是只做到 MCP Server 级权限？
- 是否需要支持 MCP 配置市场 / 模板库？
- MCP 调用日志是否要记录 tool 输入输出摘要，还是只记录 metadata？

## 11. 接受标准

- Proposal 明确 MCP Proxy 是 Secrel Gateway 的资源代理能力，而不是独立产品。
- Proposal 明确 Local App 和 Server Web Admin 中的 MCP 管理体验。
- Proposal 明确 stdio、HTTP、SSE 类 MCP 的配置和代理边界。
- Proposal 明确 Admin 鉴权、Client Key 和 MCP secret 的隔离要求。
- Proposal 明确从 `mcp.json` 导入和导出代理配置的产品行为。
- Proposal 明确服务端 stdio MCP 的安全风险和待确认决策。
