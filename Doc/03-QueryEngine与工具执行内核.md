# 03｜QueryEngine 与工具执行内核

## 1. 双核心文件关系

- `query.ts`：偏“函数式主循环”，负责消息迭代、工具调度、压缩与恢复逻辑
- `QueryEngine.ts`：偏“会话对象化封装”，管理跨 turn 状态，向 SDK/无头调用暴露统一接口

可理解为：`QueryEngine` 封装会话生命周期，`query.ts` 落地每轮推理与工具交互。

## 2. `QueryEngineConfig` 的设计意义

`QueryEngine.ts` 的配置项覆盖：

- 环境：`cwd`、`tools`、`commands`、`mcpClients`
- 状态访问：`getAppState` / `setAppState`
- 上下文控制：`customSystemPrompt` / `appendSystemPrompt`
- 预算控制：`maxTurns`、`maxBudgetUsd`、`taskBudget`
- 交互控制：`handleElicitation`、`setSDKStatus`、`abortController`

这使 QueryEngine 可以在不同上层（REPL、SDK、远程）复用，而不把 UI 细节绑死在引擎层。

## 3. `query.ts` 主循环核心职责

### 3.1 消息规范化与组装

通过 `createUserMessage`、`normalizeMessagesForAPI` 等方法：

- 统一用户/助手/tool_use/tool_result 格式
- 处理中断消息、错误消息、compact 边界消息
- 保证发送给 API 的消息序列符合约束

### 3.2 工具调用与回填

检测到 assistant `tool_use` 后，进入工具执行路径，结果通过 `tool_result` 回写到会话消息，再发起下一轮 API 调用。

### 3.3 上下文与 token 管控

与 `autoCompact/reactiveCompact/tokenBudget` 配合：

- 在 token 紧张时做自动压缩
- 处理 `max_output_tokens` 恢复循环
- 记录并校验当前轮预算与续跑次数

## 4. 工具协议中心：`Tool.ts`

`ToolUseContext` 是该项目最重要的“运行时上下文对象”之一，集中承载：

- 工具集/命令集/mcp 资源
- AppState 读写能力
- 中断控制与 JSX 渲染回调
- 通知、系统消息追加、文件历史、归因状态更新
- 执行中的工具 ID 集合与响应长度统计

意义：工具实现可以共享统一上下文，而不直接耦合到上层 UI/CLI。

## 5. 工具调度：`toolOrchestration.ts`

`runTools()` 的关键策略：

1. 先按 `isConcurrencySafe` 分批（并发安全批/串行批）
2. 并发批中先执行并收集 contextModifier
3. 再按原顺序回放 contextModifier，保证上下文演化顺序可控

这是一个“**并发执行 + 顺序生效**”的折中方案：性能与一致性兼顾。

## 6. 流式工具执行器：`StreamingToolExecutor.ts`

核心目标：工具流式到达时，边收边执行，且保持顺序输出语义。

关键机制：

- `canExecuteTool()`：判断当前并发窗口是否允许执行
- 维护 `queued/executing/completed/yielded` 状态机
- 支持“兄弟工具错误联动取消”（尤其并行 Bash 场景）
- 支持中断语义（`interruptBehavior`：cancel/block）
- streaming fallback 时可 `discard()` 全量丢弃未完成结果

这使模型工具调用在“并发、安全、中断、顺序一致性”四者间达成工程平衡。

## 7. 典型一轮执行序列

```text
用户输入 -> query.ts 组装消息 -> Claude API streaming
  -> 产生 tool_use
  -> toolOrchestration / StreamingToolExecutor 执行工具
  -> 回填 tool_result
  -> 再次调用 API
  -> 直到 stop_reason 非 tool_use
```

## 8. 稳定性设计观察

1. **强约束消息形态**（多 helper + type）
2. **并发保守策略**（异常即降级串行或取消）
3. **预算与压缩联动**
4. **中断语义明确**
5. **feature gate 隔离试验逻辑**（如 HISTORY_SNIP）

