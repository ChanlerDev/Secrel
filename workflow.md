# Secrel AI Coding Workflow

本文定义 Secrel 的 AI coding 工作流。Secrel 采用 human-in-the-loop 的 spec-driven 开发方式：用户提出方向，AI 辅助写 proposal、plan、代码和验证，但关键产品边界由用户人工确认。

## 1. 权威文档

- `docs/SPEC.md`：当前完整产品规格，是实现时的最高优先级依据。
- `docs/ROADMAP.md`：总任务板，记录 Now / Next / Later / Done。
- `docs/proposals/*.md`：新功能、新产品能力或较大规格变化的草案。
- `docs/plans/*.md`：已接受 proposal 或 SPEC 内能力的实现计划。
- `docs/CHANGELOG.md`：按版本记录用户可见变化。

如果文档冲突，优先级为：

```text
用户当前明确指令
  > docs/SPEC.md
  > Accepted proposal
  > implementation plan
  > ROADMAP / CHANGELOG
  > Draft proposal
```

Draft proposal 不能作为直接实现依据。

## 2. 需求入口

### 2.1 新功能或产品行为变化

新功能应先进入 proposal 流程：

1. 阅读 `docs/SPEC.md`，确认当前规格是否已覆盖。
2. 如果未覆盖，创建或更新 `docs/proposals/<feature>.md`。
3. 与用户一起 review proposal。
4. 用户明确认可后，将 proposal 状态改为 `Accepted`。
5. 更新 `docs/ROADMAP.md`，把任务放入合适状态。
6. 创建 `docs/plans/<feature>.md` 实现计划。
7. 用户确认计划或明确要求实现后，开始编码。

### 2.2 Bugfix

Bugfix 可以跳过 proposal，但必须说明：

- 复现方式。
- 根因。
- 修复方案。
- 回归测试或验证方式。

如果 bug 暴露出产品语义变化，应同步更新 `docs/SPEC.md` 或补一个 proposal。

### 2.3 小型文档或配置修正

不改变产品行为的小修正可以直接处理，但交付前仍需检查是否影响 `SPEC`、`ROADMAP` 或 `CHANGELOG`。

## 3. Proposal 流程

Proposal 使用 `docs/proposals/TEMPLATE.md`。

状态流：

```text
Draft -> Accepted -> Implemented -> Merged
          └────────> Rejected / Superseded
```

状态含义：

- `Draft`：讨论中，不能实现。
- `Accepted`：产品边界已确认，可以写实现计划。
- `Implemented`：代码已实现，但尚未合并进主 SPEC 或发布。
- `Merged`：稳定规格已进入主产品规格，proposal 仅作历史记录。
- `Rejected`：明确不做。
- `Superseded`：被新的 proposal 或 SPEC 内容替代。

Proposal 应关注产品行为，不应在草案阶段写成完整代码实现计划。

## 4. Implementation Plan 流程

实现前应创建 `docs/plans/<feature>.md`，使用 `docs/plans/TEMPLATE.md`。

Plan 应明确：

- 要实现的用户价值。
- 关联 SPEC / proposal。
- 改动范围。
- 数据模型和 API 影响。
- 测试策略。
- 验证命令。
- 回滚或降级策略。

Plan 被用户确认后才能进入实现，除非用户明确要求直接实现。

## 5. 实现流程

实现时遵循：

1. 保持改动聚焦，不做无关重构。
2. 优先遵循现有架构和命名。
3. 用户可见行为变化必须更新文档。
4. 涉及安全、凭证、日志、协议转换和路由策略时必须补测试或明确验证方式。
5. 完成后运行统一检查脚本。

## 6. 文档更新规则

- 产品规格变化：更新 `docs/SPEC.md`。
- 新功能草案：更新 `docs/proposals/*.md`。
- 已接受功能的开发计划：更新 `docs/plans/*.md`。
- 任务状态：更新 `docs/ROADMAP.md`。
- 用户可见发布变化：更新 `docs/CHANGELOG.md`。

Proposal 合并进主 SPEC 后，应把 proposal 状态改为 `Merged`。

## 7. Changelog 规则

`docs/CHANGELOG.md` 用来按版本记录用户可见变化。它只记录已经发布或准备发布给用户的变化，不记录内部讨论过程、未接受的 proposal 或未进入发布的临时实验。

开发期间先写入 `## Unreleased`。发布时创建新版本段落，把 Unreleased 中的条目移动进去。

条目建议格式：

```text
- **feat**: 描述新增能力
- **fix**: 描述修复
- **changed**: 描述行为变化
- **docs**: 描述用户可见文档变化
```

内部重构如果不影响用户可见行为，可以不写 changelog。

## 8. Roadmap 规则

`docs/ROADMAP.md` 是任务 Board，不是产品规格。

建议分区：

- `Now`：正在做或马上要做。
- `Next`：已经认可但尚未开始。
- `Later`：方向存在，但近期不做。
- `Done`：完成的主要任务，保留历史。

Roadmap 不应覆盖 `SPEC` 的权威性。具体产品行为仍以 `SPEC` 或 Accepted proposal 为准。

## 9. 目标脚本

当前项目尚未 scaffold，脚本先作为目标契约保留。代码结构确定后再实现。

### 9.1 `agent:check`

统一验证入口。应根据项目结构运行必要检查。

目标覆盖：

- Rust: `cargo fmt --check`
- Rust: `cargo clippy --workspace --all-targets --all-features --tests -- -D warnings`
- Rust: `cargo test --workspace`
- Web: `pnpm lint`
- Web: `pnpm test`
- Web: `pnpm build`
- Docker: `docker build` 或等价镜像构建检查

脚本应尽量根据变更文件选择检查范围，但不能牺牲安全性。

### 9.2 `agent:plan-check`

检查 implementation plan 是否存在并引用了 SPEC 或 Accepted proposal。

目标行为：

- 对非 bugfix 的功能实现，要求存在 `docs/plans/<feature>.md`。
- 检查相关 proposal 状态不是 `Draft`。
- 检查 plan 包含测试和验证章节。

### 9.3 `agent:docs-check`

检查文档一致性。

目标行为：

- `docs/SPEC.md` 必须存在。
- `docs/ROADMAP.md` 必须存在。
- `docs/CHANGELOG.md` 必须存在。
- `docs/proposals/TEMPLATE.md` 必须存在。
- `docs/plans/TEMPLATE.md` 必须存在。
- proposal 状态必须是允许值。

### 9.4 `agent:release`

发布脚本。当前仅定义目标行为，不实现。

目标行为：

- 要求工作区干净。
- 要求 `agent:check` 通过。
- 要求 `docs/CHANGELOG.md` 有对应版本记录。
- 自动 bump 版本。
- 创建 release commit。
- 创建 git tag。
- push 分支和 tag。
- 触发 GitHub Actions 构建发布。

## 10. CI/CD 目标

后续应通过 GitHub Actions 提供：

- `check.yml`：PR 和 push 时运行格式、lint、测试、构建。
- `docker.yml`：构建服务端 Docker 镜像。
- `release.yml`：tag 触发 release 构建，产出服务端镜像、本地 App artifact 或其他发布产物。

CI/CD 不应绕过本地脚本。GitHub Actions 应优先调用与本地一致的 `agent:*` 脚本，避免本地和 CI 行为漂移。

## 11. 交付前检查

每次开发交付前，AI 应确认：

- 是否回答了用户最新需求。
- 是否更新了受影响文档。
- 是否运行了可用验证命令。
- 如果未能验证，明确说明原因。
- 没有把 Draft proposal 当作实现依据。
