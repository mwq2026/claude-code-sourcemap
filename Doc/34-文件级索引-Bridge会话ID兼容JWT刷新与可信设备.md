# 34｜文件级索引：Bridge 会话ID兼容、JWT刷新与可信设备

> 目标：继续推进后续卷，补齐 bridge 安全与会话协商支撑层的文件级语义。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/bridge/workSecret.ts`
2. `restored-src/src/bridge/sessionIdCompat.ts`
3. `restored-src/src/bridge/jwtUtils.ts`
4. `restored-src/src/bridge/trustedDevice.ts`

---

## 2. WorkSecret 与会话URL构造

## 2.1 `bridge/workSecret.ts`

已核对要点：

1. 负责 work secret base64url 解码与 version/token/baseUrl 基础校验。
2. 提供 SDK URL 构造（本地与生产路径差异）及 CCR v2 session URL 构造。
3. 提供 `sameSessionId` 比较逻辑，按后缀 body 比较兼容不同 tag 前缀。
4. 提供 registerWorker 调用并校验 `worker_epoch` 可用性。

---

## 3. Session ID 兼容层

## 3.1 `bridge/sessionIdCompat.ts`

已核对要点：

1. 提供 `toCompatSessionId`（`cse_* -> session_*`）与 `toInfraSessionId` 反向映射。
2. 通过可注入 gate（`setCseShimGate`）控制兼容 shim 生效，避免 sdk bundle 引入重依赖。
3. 目标是让 compat API 与基础设施层在同一 UUID 的不同 tag 语义下可互通。

---

## 4. JWT 解析与刷新调度

## 4.1 `bridge/jwtUtils.ts`

已核对要点：

1. 提供 JWT payload/exp 解析（不做签名校验）作为刷新调度输入。
2. 提供 token refresh scheduler，支持按 `exp` 或 `expires_in` 两种方式排程。
3. 维护 generation/failure 计数，避免并发刷新导致过期回调覆盖与链路断裂。
4. 对无 token、重试、取消与全量取消均有显式处理路径。

---

## 5. 可信设备令牌链路

## 5.1 `bridge/trustedDevice.ts`

已核对要点：

1. 由 gate 控制是否启用 trusted device token 头注入。
2. token 读取走 secure storage + memoize，支持显式清缓存与清除持久化 token。
3. 支持登录后 enroll 逻辑：调用 trusted_devices 接口并将 device token 落盘。
4. 失败路径为 best-effort，避免阻塞主登录流程。

---

## 6. 子系统不变量

1. 会话 ID 标签映射必须保持双向可逆，否则 compat 与 infra 调用会出现“同UUID不同ID”错配。
2. JWT 刷新必须具备并发隔离与代际约束，避免旧回调覆盖新调度。
3. trusted device token 读取优先级与 gate 判定必须稳定，否则会出现不一致鉴权头。
4. work secret 与 worker_epoch 校验必须严格，避免会话接入使用无效协商参数。

---

## 7. 改动检查清单（34卷子域）

1. 改 workSecret：是否破坏 secret 解码校验、URL 构造与 worker_epoch 解析
2. 改 sessionIdCompat：是否破坏 cse/session tag 转换与 gate 生效路径
3. 改 jwtUtils：是否破坏刷新排程、失败重试与 cancel 语义
4. 改 trustedDevice：是否破坏 gate 控制、token 持久化或登录后 enroll 流程

---

## 8. 边界说明

- 本卷聚焦 bridge 鉴权与兼容支撑层，不展开 transport/UI 主流程。
- 结论均基于当前源码；后续协议与安全策略变化需重新核验。

