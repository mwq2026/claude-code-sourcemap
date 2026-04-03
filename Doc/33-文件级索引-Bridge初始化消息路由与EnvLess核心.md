# 33｜文件级索引：Bridge 初始化、消息路由与 Env-less 核心

> 目标：继续推进后续卷，补齐 bridge 子域中 REPL 初始化、收发路由与 env-less 核心链路。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/bridge/bridgeMain.ts`
2. `restored-src/src/bridge/initReplBridge.ts`
3. `restored-src/src/bridge/bridgeMessaging.ts`
4. `restored-src/src/bridge/remoteBridgeCore.ts`

---

## 2. Bridge 主循环与会话生命周期

## 2.1 `bridge/bridgeMain.ts`

已核对要点：

1. 维护 bridge loop 的会话生命周期（active sessions、heartbeat、timeout、cleanup、reconnect）。
2. 区分 v1/v2 会话路径并处理 token refresh 策略（含 v2 场景的 reconnectSession 重派发）。
3. 轮询与退避参数集中管理，并对长期失败、auth 失效等场景有显式分支。
4. 与 worktree/session runner/logger 协同，形成“拉取工作→执行→回收”闭环。

---

## 3. REPL 侧 Bridge 初始化入口

## 3.1 `bridge/initReplBridge.ts`

已核对要点：

1. 作为 REPL 包装层：读取 bootstrap state/gate/oauth/policy/title，再委托 core 初始化。
2. 启动前有多重门控：bridge enable、版本检查、OAuth 可用、组织 policy allow_remote_control。
3. 处理会话标题来源优先级（显式名、存储名、消息推导、fallback slug），并在用户消息中增量更新。
4. 兼容 v1/v2 与 env-less 选择分支，且保留状态回调与控制回调注入点。

---

## 4. 收发消息与控制路由

## 4.1 `bridge/bridgeMessaging.ts`

已核对要点：

1. 提供 ingress 消息解析与分流：SDKMessage、control_request、control_response。
2. 维护 echo/re-delivery 去重缓冲（posted/inbound UUID）防止重复处理。
3. `handleServerControlRequest` 统一处理 initialize/set_model/interrupt/permission 等控制请求响应。
4. outbound-only 模式下对可变请求回 error，避免“假成功”反馈。

---

## 5. Env-less 核心链路

## 5.1 `bridge/remoteBridgeCore.ts`

已核对要点：

1. env-less 路径直接走 session + bridge credential + v2 transport，不依赖 Environments API dispatch 层。
2. 建立后维护 FlushGate、去重集合、连接状态与 transport rebuild（401 恢复/主动 refresh）。
3. JWT 刷新与 epoch 更新联动，避免 token 更新但 epoch 失配导致的 409/连接抖动。
4. 通过纯参数注入方式接入消息映射、回调、title 更新策略，降低核心层耦合。

---

## 6. 子系统不变量

1. bridge 启动必须先过 gate+auth+policy 三层门控，否则应快速失败并给出可操作反馈。
2. 控制请求必须有明确 response 路径，防止远端等待超时导致连接被杀。
3. ingress 去重必须稳定，否则 echo/replay 会污染本地会话状态。
4. env-less v2 刷新必须同步处理 JWT 与 epoch，不能只换 token 不换连接上下文。

---

## 7. 改动检查清单（33卷子域）

1. 改 bridgeMain：是否破坏轮询退避、heartbeat、timeout 与会话回收一致性
2. 改 initReplBridge：是否破坏启动门控顺序与 title 推导/更新策略
3. 改 bridgeMessaging：是否破坏 control request 响应与去重行为
4. 改 remoteBridgeCore：是否破坏 env-less 初始化、刷新恢复或 transport rebuild 语义

---

## 8. 边界说明

- 本卷聚焦 bridge 支撑主链，不展开每个辅助文件（UI/debug/config）细节。
- 结论均基于当前源码；协议或网关策略变化需重新核验。

