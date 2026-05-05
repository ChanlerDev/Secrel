# Roadmap

本文是 Secrel 的任务 Board，用来记录工作项的当前状态和优先级。它不定义具体产品行为；具体产品行为由主产品规格和已接受的 proposal 定义。

状态含义：

- `Now`：正在进行或马上开始的工作。
- `Next`：已经认可，但尚未开始的工作。
- `Later`：方向存在，但暂不进入近期开发。
- `Done`：已经完成的主要工作，保留历史。

## Now

- [x] 建立主产品规格 `docs/SPEC.md`
- [x] 建立 proposal 机制和 MCP Proxy proposal
- [x] 建立 AI coding workflow 文档

## Next

- [ ] 初始化项目代码结构
- [ ] 建立 `agent:check`、`agent:docs-check`、`agent:plan-check` 脚本
- [ ] 建立基础 GitHub Actions check workflow
- [ ] 实现 Gateway Core 基础骨架
- [ ] 实现 Provider 配置和模型发现基础能力

## Later

- [ ] Tauri Local App
- [ ] Server Web Admin
- [ ] Docker 部署
- [ ] 多协议请求代理与转换
- [ ] Streaming 转换
- [ ] MCP Proxy
- [ ] CLI / OAuth Provider Adapter

## Done

- [x] 产品规格文档整理
- [x] Proposal 模板
- [x] MCP Proxy proposal
