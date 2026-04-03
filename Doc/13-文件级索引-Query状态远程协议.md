# 13｜文件级索引：Query / 状态 / 远程协议（细颗粒度）

> 目标：把 Query 主循环、状态容器、SDK 控制协议、remote 会话链路下沉到文件级，便于研发排障与二次开发。  
> 口径：仅记录当前源码可核验实现。

---

## 1. Query 执行域（`query.ts`）

## 1.1 模块职责（已核对）

`query.ts` 是单轮执行主循环，负责：

1. 组装/规范化消息（含 attachment/system context）
2. 调用 API 流式消费并落地 message/event
3. tool_use 检测、工具执行与 tool_result 回填
4. compact（snip/micro/auto/reactive）与重试协同
5. 各类错误恢复（prompt-too-long、max_output_tokens、media 等）

---

## 1.2 关键实现锚点（已核对）

1. `yieldMissingToolResultBlocks(...)`：在异常路径补齐缺失 `tool_result`
2. thinking 规则说明注释：明确“thinking block 保留期与消息轨迹约束”
3. `isWithheldMaxOutputTokens(...)`：控制 SDK 不过早暴露可恢复错误
4. `QueryParams`：
   - `taskBudget`（注释已区分于 tokenBudget）
   - `maxTurns/maxOutputTokensOverride/querySource`
5. `queryLoop` 维护状态：
   - `autoCompactTracking`
   - `maxOutputTokensRecoveryCount`
   - `hasAttemptedReactiveCompact`
   - `transition`（用于恢复路径断言）

---

## 1.3 开发者易错点（基于源码行为）

1. 把 `content_block_stop` 当作最终 stop 点（实际 stop_reason/usage 在 message_delta）
2. 重试时混入旧 executor 结果导致 orphan/错配 tool_result
3. 压缩后 token/预算判断仍用旧上下文估计，造成误阻断或误重试

---

## 2. API 协议与流式解析域（`services/api/claude.ts`）

## 2.1 模块职责（已核对）

1. 组装 API 请求参数（model/messages/system/tools/beta/effort/output 等）
2. 处理 provider 差异与 extra body 参数
3. 流式事件解析与消息归一化
4. 失败重试与 fallback（与 withRetry 协同）
5. usage/cost/request-id 追踪写回状态

---

## 2.2 关键实现锚点（已核对）

1. `getExtraBodyParams(betaHeaders?)`：
   - 读取 `CLAUDE_CODE_EXTRA_BODY`
   - 对对象做浅拷贝，避免缓存对象被污染
2. 请求构造依赖：
   - `normalizeMessagesForAPI`
   - `ensureToolResultPairing`
   - `toolToAPISchema`
3. 大量 beta/header latch 状态取自 `bootstrap/state.ts`（缓存与一致性相关）
4. 与 `withRetry.ts` 的 `FallbackTriggeredError/CannotRetryError` 协作

---

## 3. 工具执行编排域（`services/tools/StreamingToolExecutor.ts`）

## 3.1 模块职责（已核对）

1. 维护工具状态机：`queued/executing/completed/yielded`
2. 处理串并发执行窗口
3. 处理 sibling error / user interrupt / streaming fallback
4. 保证结果有序产出并补齐必要的合成错误结果

---

## 3.2 关键实现锚点（已核对）

1. per-tool child abort controller：
   - sibling error 可级联中止同批工具
   - 需要时向上传播到 query abort
2. Bash tool error 会触发 sibling cancel（其余工具不默认联动）
3. progress 消息单独通道（`pendingProgress`）可即时 yield
4. 非并发安全工具执行中会阻断后续结果抢跑（保持顺序）

---

## 4. 应用状态域（`state/AppStateStore.ts`）

## 4.1 模块职责（已核对）

`AppState` 负责 interactive 层和会话协作态：

1. UI 与视图态（expanded/footer selection/brief mode）
2. 权限与提示态（toolPermissionContext/elicitation/notifications）
3. 远程态（remoteConnectionStatus/remoteBackgroundTaskCount）
4. bridge/tungsten/bagel 等功能态
5. mcp/plugins/tasks/fileHistory/attribution/todos 等业务态

---

## 4.2 关键实现锚点（已核对）

1. `toolPermissionContext` 是权限决策中心态
2. remote viewer 与本地任务态明确分离（注释解释本地 tasks 为空场景）
3. plugins 与 mcp 各自有 `commands/tools/resources/errors/refresh` 机制

---

## 5. 全局会话状态域（`bootstrap/state.ts`）

## 5.1 模块职责（已核对）

全局状态持有会话级长期信息：

1. project/cwd/session lineage
2. 费用/时延/token/模型 usage 聚合
3. telemetry/logger/meter/tracer 对象
4. 缓存策略 latch（prompt cache / fast mode / thinking clear 等）
5. strictToolResultPairing、pendingPostCompaction 等关键运行开关

---

## 5.2 关键实现锚点（已核对）

1. 注释明确：慎加全局状态（“DO NOT ADD MORE STATE HERE”）
2. `strictToolResultPairing`：可切换为 mismatch 直接抛错而非修复
3. `lastAPIRequest` 与 `lastAPIRequestMessages`：用于回溯与分享一致性

---

## 6. Remote 会话域（`remote/RemoteSessionManager.ts`）

## 6.1 模块职责（已核对）

1. 管理 WS 订阅与回调分发
2. 通过 HTTP 发送用户事件到 remote session
3. 处理 permission request/response/cancel
4. 暴露 interrupt/close/reconnect 能力

---

## 6.2 关键实现锚点（已核对）

1. `pendingPermissionRequests: Map<request_id, SDKControlPermissionRequest>`
2. `handleMessage` 区分：
   - `control_request`
   - `control_cancel_request`
   - `control_response`
   - SDKMessage
3. 未支持的 control subtype 会回发 error response，避免服务端悬挂等待

---

## 7. WS 传输域（`remote/SessionsWebSocket.ts`）

## 7.1 模块职责（已核对）

1. 连接 `/v1/sessions/ws/{id}/subscribe`
2. header 鉴权与 ping 保活
3. close code 策略化重连
4. 消息 parse + 透传

---

## 7.2 关键实现锚点（已核对）

1. 常量：
   - `RECONNECT_DELAY_MS=2000`
   - `MAX_RECONNECT_ATTEMPTS=5`
   - `MAX_SESSION_NOT_FOUND_RETRIES=3`
2. `PERMANENT_CLOSE_CODES` 默认含 `4003`（unauthorized）
3. `4001`（session not found）走有限重试分支（注释说明 compact 期间可能瞬态）
4. `isSessionsMessage` 只校验 `type` 为 string，避免后端新增类型被静默丢弃

---

## 8. SDK 控制协议域（`entrypoints/sdk/controlSchemas.ts`）

## 8.1 模块职责（已核对）

1. 定义 control request/response/cancel/keep_alive schema
2. 定义高频 subtype schema（permission、model、mcp、settings、rewind 等）
3. 聚合 stdin/stdout 消息联合 schema

---

## 8.2 关键实现锚点（已核对）

### 控制壳

- `SDKControlRequestSchema`：
  - `type='control_request'`
  - `request_id`
  - `request`（union）
- `SDKControlResponseSchema`：
  - `type='control_response'`
  - `response`（success/error）
- `SDKControlCancelRequestSchema`：
  - `type='control_cancel_request'`
  - `request_id`

### 典型 subtype（部分）

1. `can_use_tool`（含 `tool_use_id/tool_name/input/...`）
2. `set_permission_mode` / `set_model` / `set_max_thinking_tokens`
3. `mcp_status` / `mcp_set_servers` / `mcp_reconnect` / `mcp_toggle`
4. `reload_plugins` / `stop_task` / `apply_flag_settings` / `get_settings`
5. `rewind_files` / `cancel_async_message` / `seed_read_state` / `elicitation`

---

## 9. 跨域排障对账表（文件级）

1. 权限卡住：`RemoteSessionManager.pendingPermissionRequests` + `controlSchemas.can_use_tool`
2. 工具错配：`query.ts yieldMissingToolResultBlocks` + `claude.ts ensureToolResultPairing`
3. WS 频繁断开：`SessionsWebSocket.handleClose` close code 分支
4. 模型参数异常：`claude.ts` 请求组装 + `bootstrap/state.ts` latch 状态

---

## 10. 研发改动最小核对清单

1. 改 query：是否破坏 tool_result 闭环与 recovery transition
2. 改 tool executor：是否破坏有序 yield 与中断传播
3. 改 control schema：是否同步 remote manager 与消费者处理分支
4. 改 global state：是否新增了不必要长期状态，是否影响缓存稳定性
5. 改 ws 策略：是否影响 4001/4003 的重连语义

---

## 11. 快速阅读顺序（文件级）

1. `restored-src/src/query.ts`
2. `restored-src/src/services/api/claude.ts`
3. `restored-src/src/services/tools/StreamingToolExecutor.ts`
4. `restored-src/src/state/AppStateStore.ts`
5. `restored-src/src/bootstrap/state.ts`
6. `restored-src/src/remote/RemoteSessionManager.ts`
7. `restored-src/src/remote/SessionsWebSocket.ts`
8. `restored-src/src/entrypoints/sdk/controlSchemas.ts`

---

## 12. 边界说明

- 本文为细颗粒“文件级索引”，强调定位与对账，不等价于逐函数 API 手册。
- 全部内容均锚定当前仓库可见源码；若后续版本更新，请按本索引重新核验。

