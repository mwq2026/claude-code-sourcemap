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

## 9. 二次审核补充（证据锚点）

为避免“只讲结论不讲依据”，本卷关键结论对应的源码锚点如下：

- 会话级封装与状态持有：`src/QueryEngine.ts`（`QueryEngineConfig`、`submitMessage()`）
- 查询主循环与 token/compact 处理：`src/query.ts`（`query()` 与 compact/recovery 相关逻辑）
- 工具上下文统一协议：`src/Tool.ts`（`ToolUseContext`）
- 串并行工具调度：`src/services/tools/toolOrchestration.ts`（`runTools()`）
- 流式执行与取消语义：`src/services/tools/StreamingToolExecutor.ts`

## 10. 工具通信详细链路（API Streaming 报文级）

> 本节聚焦“本地 CLI ↔ Anthropic Messages API”这条主链路；远程 CCR 会话链路见 `05`。

### 10.1 请求封装（发送前）

在 `services/api/claude.ts` 的 `paramsFromContext()` 中，最终组装 `anthropic.beta.messages.create({... stream: true })` 参数，核心字段如下：

- `model`：模型标识（会先归一化）。
- `messages`：历史消息数组（已做 `normalizeMessagesForAPI`、tool pairing 修复、媒体裁剪）。
- `system`：系统提示块数组（含 attribution/header/feature 指令）。
- `tools`：工具 schema 列表（由 `toolToAPISchema()` 生成；可带 `defer_loading`）。
- `tool_choice`：工具选择策略（auto/specific）。
- `max_tokens`：本轮最大输出 token。
- `thinking`：思考配置（`adaptive` 或 `enabled + budget_tokens`）。
- `metadata`：请求归因信息（`device_id/account_uuid/session_id` 等）。
- `betas`：beta header 列表（fast mode/context mgmt/tool search 等）。
- `output_config`：结构化输出、effort、task budget 等扩展配置。
- `context_management`：上下文管理策略（满足 beta 条件时注入）。

### 10.2 流式响应解析（事件到消息）

`queryModel()` 使用 `for await (const part of stream)` 按事件解析，关键事件与处理：

1. `message_start`  
   - 初始化 `partialMessage` 与 usage 基线。
2. `content_block_start`  
   - 创建 block 容器：`text/thinking/tool_use/server_tool_use`。
3. `content_block_delta`  
   - 追加增量：  
     - `text_delta` → 文本拼接  
     - `thinking_delta/signature_delta` → thinking 拼接与签名  
     - `input_json_delta` → 工具输入 JSON 片段拼接
4. `content_block_stop`  
   - 将完整 block 正规化为内部 `AssistantMessage` 并 `yield`。
5. `message_delta`  
   - 回填最终 `usage` 与 `stop_reason`（例如 `tool_use`、`end_turn`、`max_tokens`）。
6. `message_stop`  
   - 该条 assistant 消息流结束。

### 10.3 `tool_use` → 本地工具执行 → `tool_result` 回填

主循环位于 `query.ts`：

1. 从 assistant 内容块提取 `tool_use`（含 `id/name/input`）。
2. 交给 `StreamingToolExecutor` / `toolOrchestration.runTools()` 执行。
3. 执行结果包装为 user 消息中的 `tool_result` block，并放回会话：
   - 关键绑定字段：`tool_use_id`（必须与上一步 `tool_use.id` 一致）。
4. 触发下一轮 API 调用，模型继续消费 `tool_result` 并生成后续回答/新工具调用。

### 10.4 关键报文示例（简化）

**A. 请求报文（节选）**

```json
{
  "model": "claude-sonnet-4-5",
  "messages": [
    { "role": "user", "content": "请读取 package/package.json 并总结 scripts" }
  ],
  "system": [{ "type": "text", "text": "<system prompt...>" }],
  "tools": [
    { "name": "Read", "description": "...", "input_schema": { "type": "object" } }
  ],
  "max_tokens": 32000,
  "stream": true
}
```

**B. 流式事件（assistant 发起工具）**

```json
{ "type": "message_start", "message": { "id": "msg_x", "usage": { "input_tokens": 1234, "output_tokens": 0 } } }
{ "type": "content_block_start", "index": 0, "content_block": { "type": "tool_use", "id": "toolu_1", "name": "Read", "input": {} } }
{ "type": "content_block_delta", "index": 0, "delta": { "type": "input_json_delta", "partial_json": "{\"file_path\":\"package/package.json\"}" } }
{ "type": "content_block_stop", "index": 0 }
{ "type": "message_delta", "delta": { "stop_reason": "tool_use" }, "usage": { "output_tokens": 87 } }
{ "type": "message_stop" }
```

**C. 回填 `tool_result`（下一次请求中的 user 消息）**

**说明**：以下 `content` 为示例，实际格式取决于工具实现（可能为结构化 JSON、文本片段或混合内容块）。

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_1",
      "content": "Read succeeded: { \"scripts\": { \"prepare\": \"...\" } }",
      "is_error": false
    }
  ]
}
```

### 10.5 字段语义速查

- `tool_use.id`： 本次工具调用唯一 ID。  
- `tool_result.tool_use_id`： 关联回哪一次 `tool_use`。  
- `stop_reason`： 本轮停止原因，常见 `tool_use/end_turn/max_tokens`。  
- `usage`： token 计量（输入/输出/缓存读写），在 `message_delta` 阶段最终稳定。  
- `request_id`： HTTP 请求维度 ID（从 SDK response 读取，用于追踪与诊断）。

### 10.6 解析细节补充（避免常见误读）

1. **`content_block_stop` 先产出消息，`message_delta` 再回填最终统计**  
   - 代码中 assistant 消息在 `content_block_stop` 即被构造并 `yield`。  
   - 但 `usage/stop_reason` 的最终值在后续 `message_delta` 才写回。  
   - 因此若只看首个 yield，可能会误认为 `usage.output_tokens=0` 或 `stop_reason=null`。

2. **工具输入不是一次性 JSON，而是 `input_json_delta` 片段拼接**  
   - `tool_use.input` 初始为空字符串，随后连续拼接 `partial_json`。  
   - 这解释了为什么抓包中经常看到多段 `content_block_delta` 才形成完整参数。

3. **每个原始流事件都会再包装成内部 `stream_event` 转发**  
   - 即除了 assistant/user/system 消息外，还会看到 `type='stream_event'` 的透传事件。  
   - 这层转发用于 UI 实时渲染、诊断与 SDK 消费。

### 10.7 异常链路与报文影响（查漏补缺）

#### A. Streaming 卡住（watchdog）→ 非流式回退

```text
stream 长时间无 chunk
  -> watchdog 触发（streamIdleAborted=true）
  -> 记录 tengu_streaming_fallback_to_non_streaming
  -> executeNonStreamingRequest()
  -> 产出完整 assistant message（非 stream）
```

报文含义影响：

- 你可能看到同一轮先有部分 stream 事件，随后出现一次完整 non-streaming 结果。
- 这是容错路径，不是重复调用工具（代码里有对应防重与丢弃逻辑）。

#### B. `stop_reason=max_tokens` / `model_context_window_exceeded`

- 不直接结束为“正常成功”，而会额外生成 API error message（`apiError=max_output_tokens`），
  供上层进入恢复/续写逻辑。

#### C. 中断（用户 ESC 或 abort signal）

- 如果流式执行中断，执行器会补齐/生成必要的工具结果占位，避免 `tool_use` 与 `tool_result` 失配。
- 这是保证后续轮次报文一致性的关键保护。

### 10.8 请求报文字段“全量清单”（按当前代码路径）

> 口径说明：以下以 `paramsFromContext()` 实际构造的请求体为准，分为“恒定字段”和“条件字段”。

#### A. 恒定字段（每次请求都会出现）

- `model`：`normalizeModelStringForAPI(options.model)` 归一化后的模型名。
- `messages`：经 cache breakpoint/compact 处理后的消息数组。
- `system`：系统消息块数组。
- `tools`：工具 schema 数组。
- `tool_choice`：工具策略。
- `metadata`：`getAPIMetadata()` 返回值。
- `max_tokens`：本轮最大输出 token。

#### B. 条件字段（满足条件才注入）

1. `betas`  
   - 条件：`useBetas` 为真。  
   - 来源：基础 beta + 动态追加（如 1M/fast/context mgmt/structured output 等）。

2. `thinking`  
   - 条件：thinking 未禁用且模型支持。  
   - 形态：
     - `{ type: "adaptive" }`
     - `{ type: "enabled", budget_tokens: number }`

3. `temperature`  
   - 条件：仅在 thinking 关闭时发送（thinking 开启时依赖 API 默认 `1`）。

4. `context_management`  
   - 条件：`getAPIContextManagement()` 有返回，且 beta 开关满足并包含 `CONTEXT_MANAGEMENT_BETA_HEADER`。

5. `output_config`  
   - 条件：`outputConfig` 非空。  
   - 可能包含：
     - `effort`
     - `task_budget`
     - `format`（structured outputs）
     - 以及 provider extra body 里的兼容字段。

6. `speed`  
   - 条件：fast mode 可用且本次 retryContext 允许。

7. provider 额外字段（`...extraBodyParams`）  
   - 条件：provider（例如 bedrock）路径下由 `getExtraBodyParams()` 返回。

### 10.9 流式事件字段“全量清单”（按解析分支）

> 口径说明：这里列的是 `queryModel()` 显式消费/依赖的字段全集，而不是 SDK 原始事件 schema 的理论全集。

#### 1) `message_start`

- `type`
- `message`（至少消费到）：
  - `usage`
  - 其余字段透传进入 `partialMessage`
- （internal）`research` 可能出现并被记录

#### 2) `content_block_start`

- `type`
- `index`
- `content_block`（按 block type 分支）：
  - `tool_use`：`id/name/input`（其中 `input` 在本地重置为空字符串再拼接）
  - `server_tool_use`：`id/name/input`（同样走增量拼接）
  - `text`：`text`（起始值被置空，防重复）
  - `thinking`：`thinking/signature`（均初始化为空字符串）
  - `connector_text`（feature 打开时）：走专门 delta 分支
  - 其他 block：按原样浅拷贝保留

#### 3) `content_block_delta`

- `type`
- `index`
- `delta`：
  - `input_json_delta.partial_json`
  - `text_delta.text`
  - `thinking_delta.thinking`
  - `signature_delta.signature`
  - `connector_text_delta.connector_text`（feature 条件）
  - `citations_delta`（当前分支仅识别，未落地处理）
- （internal）`research` 可能出现并覆盖

#### 4) `content_block_stop`

- `type`
- `index`
- 行为：把 `contentBlocks[index]` 归一化后构造成 `AssistantMessage` 并立刻 `yield`

#### 5) `message_delta`

- `type`
- `delta.stop_reason`
- `usage`
- （internal）`research`
- 行为：回填最近一条已 yield 的 assistant 消息：
  - `message.usage = usage`
  - `message.stop_reason = stop_reason`

#### 6) `message_stop`

- `type`
- 行为：单条 assistant 消息流结束标记

#### 7) 内部二次转发事件（SDK 层可见）

每个原始 `part` 还会被包装为：

```json
{
  "type": "stream_event",
  "event": "<raw stream part>",
  "ttftMs": "<only for message_start>"
}
```
