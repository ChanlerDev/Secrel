# Roadmap

本文是 Secrel 的任务 Board，用来记录工作项的当前状态和优先级。它帮助团队判断接下来做什么，但不定义具体产品行为。

产品行为以 `docs/SPEC.md` 为准；尚未进入主规格的新能力，以状态为 `Accepted` 的 `docs/proposals/*.md` 为准。`Draft` proposal 只表示讨论方向，不能作为实现依据。

状态含义：

- `Now`：正在进行或马上开始的工作。
- `Next`：已经认可，但尚未开始的工作。
- `Later`：方向存在，但暂不进入近期开发。
- `Done`：已经完成的主要工作，保留历史。

## Now

- [ ] 初始化服务端优先的项目代码结构

## Next

- [ ] 建立 `agent:check`、`agent:docs-check`、`agent:plan-check` 脚本
- [ ] 建立基础 GitHub Actions check workflow
- [ ] 实现 Gateway Core 基础骨架
- [ ] 实现服务端 Gateway API 和 Admin API 基础骨架
- [ ] 实现 Provider 配置和模型发现基础能力
- [ ] 实现服务端 Web Admin 基础配置、日志和用量页面

## Later

- [ ] Tauri Local App
- [ ] Docker 部署
- [ ] 多协议请求代理与转换
- [ ] Streaming 转换
- [ ] MCP Proxy 产品边界确认与 proposal review
- [ ] CLI / OAuth Provider Adapter

## Done

- [x] 产品规格文档整理
- [x] Proposal 模板
- [x] MCP Proxy proposal 草案已建立
- [x] AI coding workflow 文档
