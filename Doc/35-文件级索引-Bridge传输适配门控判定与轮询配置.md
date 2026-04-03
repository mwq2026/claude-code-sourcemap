# 35｜文件级索引：Bridge 传输适配、门控判定与轮询配置

> 目标：继续推进后续卷，补齐 bridge 子域“传输抽象 + 能力门控 + 轮询配置”控制面。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/bridge/replBridgeTransport.ts`
2. `restored-src/src/bridge/bridgeConfig.ts`
3. `restored-src/src/bridge/bridgeEnabled.ts`
4. `restored-src/src/bridge/pollConfig.ts`
5. `restored-src/src/bridge/pollConfigDefaults.ts`

---

## 2. 传输适配层

## 2.1 `bridge/replBridgeTransport.ts`

已核对要点：

1. 定义统一 `ReplBridgeTransport` 接口，屏蔽 v1 HybridTransport 与 v2 SSE+CCRClient 差异。
2. v1 走现有 HybridTransport 能力透传；v2 组合 SSE 读流与 CCR 写流。
3. 支持 sequence num 延续、flush、delivery/state/metadata 上报等桥接能力。
4. 对 epoch mismatch、close 回调、outbound-only 路径有显式处理。

---

## 3. Bridge 鉴权与 URL 解析配置

## 3.1 `bridge/bridgeConfig.ts`

已核对要点：

1. 收敛 ant-only `CLAUDE_BRIDGE_*` override 的读取逻辑，减少多处复制。
2. `getBridgeAccessToken` 实现 override 优先、否则回退 OAuth token。
3. `getBridgeBaseUrl` 实现 override 优先、否则回退 OAuth 配置基址。

---

## 4. 功能门控与可用性判定

## 4.1 `bridge/bridgeEnabled.ts`

已核对要点：

1. 提供 bridge enable 判定与 blocking 版本，覆盖缓存/阻塞查询场景。
2. 提供禁用原因诊断（订阅、scope、组织信息、gate）用于可操作错误反馈。
3. 包含 env-less/v2、cse shim、min version、auto connect、mirror 等子门控判定。
4. 通过命名空间导入与异常兜底规避初始化周期依赖问题。

---

## 5. 轮询配置与默认值治理

## 5.1 `bridge/pollConfig.ts`

已核对要点：

1. 定义 poll interval 配置 schema（含 at-capacity 与 multisession 字段）。
2. 对关键字段施加下界与联动约束，防止错误配置导致 tight-loop。
3. 从 GrowthBook 拉取并校验配置，失败时回退默认值。

## 5.2 `bridge/pollConfigDefaults.ts`

已核对要点：

1. 提供 bridge 轮询默认值常量与类型定义。
2. 覆盖 not-at-capacity / at-capacity / multisession / reclaim / keepalive 默认语义。
3. 作为无需实时 GrowthBook 的调用方轻依赖入口。

---

## 6. 子系统不变量

1. 传输抽象必须保证 v1/v2 行为边界清晰，避免写路由与读路由混淆。
2. 功能门控必须区分“是否可用”与“为何不可用”，否则无法给出可执行诊断。
3. poll 配置必须具备防误配约束与默认回退，防止异常值引发高频空转。
4. override 配置优先级必须稳定，避免同一会话出现 token/baseUrl 来源漂移。

---

## 7. 改动检查清单（35卷子域）

1. 改 replBridgeTransport：是否破坏 v1/v2 适配、close/flush/sequence 行为
2. 改 bridgeConfig：是否破坏 override 优先级与 OAuth 回退语义
3. 改 bridgeEnabled：是否破坏 bridge 可用性判定、禁用原因与子 gate 语义
4. 改 pollConfig/pollConfigDefaults：是否破坏校验约束、默认回退与轮询节流语义

---

## 8. 边界说明

- 本卷聚焦 bridge 控制面配置与适配层，不展开 remote/bridge 业务链路细节。
- 结论均基于当前源码；后续 gate 策略或传输实现变更需重新核验。

