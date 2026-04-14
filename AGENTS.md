# AGENTS.md

## 适用范围

- 本文件用于 `go-quality-plugin` 的本地工作说明。
- 本文件仅覆盖该 plugin 的 `skills/` 目录中当前实际存在的 skill。
- 本文件用于指导 AI coding agent 在该 plugin 内选择和组合 skill。

## 当前包含的 skills

- `golang-code-style`
- `golang-error-handling`
- `golang-linter`
- `golang-naming`
- `golang-security`
- `golang-testing`
- `bytedance-golang-expert`

## 每个 skill 的用途摘要

- `golang-code-style`：处理 Golang 代码风格、格式与约定相关任务。
- `golang-error-handling`：处理 Golang 错误创建、包装、传播、判定与日志表达相关任务。
- `golang-linter`：处理 Golang lint 流程、`golangci-lint` 配置与告警处置相关任务。
- `golang-naming`：处理 Golang 命名约定与命名决策相关任务。
- `golang-security`：处理 Golang 安全审计与安全编码相关任务。
- `golang-testing`：处理 Golang 测试设计、实现与质量保障相关任务。
- `bytedance-golang-expert`：处理 ByteDance 内部组件/中间件使用、Hertz/Kitex/可观测性实践与补充查询指引。

## 使用这些 skills 的主要规则和边界

- 仅在 Golang 任务中使用本 plugin skill。
- 触发依据以各 skill 的 `description` 与 `SKILL.md` 指令为准。
- 同一问题优先选择一个主 skill，其他 skill 仅作补充，避免重复输出同类规则。
- 若 skill 定义了 `Persona`、`Thinking mode`、`Modes`，按该 skill 内规则执行。
- 深度分析任务仅在对应 skill 要求时使用 `ultrathink`。
- 本文件不扩展、不覆盖各 skill 的原始指令内容。

## 多个 skill 一起使用时的注意事项

- 建议同时加载 2-4 个 skill，避免上下文拥挤。
- 组合建议：
- 代码质量：`golang-code-style` + `golang-naming` + `golang-linter`
- 正确性与可维护性：`golang-error-handling` + `golang-testing`
- 安全改造：`golang-security` + `golang-testing`（按需叠加 `golang-linter`）
- 多 skill 冲突时，以主 skill 的决策路径为准，其他 skill 做交叉检查。

## 明确排除项

- 不包含 `skills/` 中不存在的 skill 描述。
- 不包含 plugin 安装、marketplace、注册、发布、配置迁移等内容。
- 不包含仓库级运营流程（如版本发布流程、评测运营流程）。
- 不包含一次性临时任务说明。
