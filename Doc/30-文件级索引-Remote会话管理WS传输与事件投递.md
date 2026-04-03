# 30｜文件级索引：Remote 会话管理、WS 传输与事件投递

> 目标：继续推进后续卷，沉淀 remote 链路中“会话管理—传输重连—HTTP 事件投递”的文件级语义。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/remote/RemoteSessionManager.ts`
2. `restored-src/src/remote/SessionsWebSocket.ts`
3. `restored-src/src/utils/teleport/api.ts`

---

## 2. 会话管理层

## 2.1 `remote/RemoteSessionManager.ts`

已核对要点：

1. 统一管理 remote session 生命周期：`connect / sendMessage / cancelSession / disconnect / reconnect`。
2. 维护 `pendingPermissionRequests`，处理 `control_request/control_cancel_request/control_response` 分流。
3. 对未支持 control subtype 回发 error control_response，避免服务端请求悬挂。
4. 用户消息发送走 `sendEventToRemoteSession`（HTTP），消息接收走 `SessionsWebSocket`（WS）。

---

## 3. 传输层与重连策略

## 3.1 `remote/SessionsWebSocket.ts`

已核对要点：

1. 连接 `.../v1/sessions/ws/{sessionId}/subscribe?organization_uuid=...`，header 携带 OAuth 鉴权字段。
2. 统一维护 state（connecting/connected/closed）、ping 保活与 reconnect timer。
3. close code 策略：
   - `4003` 视为永久拒绝，不重连；
   - `4001` 走有限重试（session not found 可能是 compact 瞬态）。
4. `isSessionsMessage` 仅要求 `type` 为 string，新类型优先“透传给上层决定”。

---

## 4. Teleport API 投递层

## 4.1 `utils/teleport/api.ts`

已核对要点：

1. `prepareApiRequest` 统一校验 access token 与 org UUID，作为 sessions API 公共前置。
2. 提供 `axiosGetWithRetry`：瞬态网络错误/5xx 按指数退避重试。
3. `sendEventToRemoteSession` 负责 user event 投递，构造 `events:[{type:'user', ...}]` 请求体并返回成功布尔值。
4. Sessions API 请求统一注入 beta/org header，且为 remote 事件投递设置较宽超时窗口。

---

## 5. 子系统不变量

1. remote 消息收发是“双通道”：WS 订阅收消息，HTTP 投递发消息，二者不可互相替代。
2. control 请求必须有响应路径（success 或 error），否则会话控制面会卡死。
3. WS 重连必须区分瞬态与永久失败，避免无意义重连风暴或过早断连。
4. teleport API 必须维持统一鉴权与组织上下文，不可遗漏 org/beta 头。

---

## 6. 改动检查清单（30卷子域）

1. 改 RemoteSessionManager：是否破坏 control 分流与 pendingPermissionRequests 一致性
2. 改 SessionsWebSocket：是否破坏 close code 策略、重连预算与 ping 生命周期
3. 改 teleport/api：是否改变重试语义或事件投递请求体/鉴权头
4. 改 remote 交互：是否仍保持“WS 收 + HTTP 发”双通道契约

---

## 7. 边界说明

- 本卷聚焦 remote 传输与投递支撑层，不展开具体 UI 呈现逻辑。
- 结论均基于当前源码；若 API 版本/协议变更需重新核验。

