# 29｜文件级索引：Coordinator 模式与远程消息/权限桥接

> 目标：继续推进后续卷，补齐 coordinator 与 remote 侧关键“调度上下文—消息适配—权限桥接”子域。  
> 口径：仅记录可由当前源码直接核验的实现事实。

---

## 1. 覆盖文件

1. `restored-src/src/coordinator/coordinatorMode.ts`
2. `restored-src/src/assistant/sessionHistory.ts`
3. `restored-src/src/remote/sdkMessageAdapter.ts`
4. `restored-src/src/remote/remotePermissionBridge.ts`

---

## 2. Coordinator 调度语义

## 2.1 `coordinator/coordinatorMode.ts`

已核对要点：

1. coordinator 开关由 feature gate + 环境变量共同决定，支持会话恢复时模式对齐（`matchSessionMode`）。
2. 提供 coordinator user context 组装：worker 可用工具、MCP server 名称、scratchpad 目录提示等。
3. 内嵌 coordinator system prompt，明确协调者职责、worker 生命周期、并发与验证要求。

---

## 3. Assistant 历史分页拉取

## 3.1 `assistant/sessionHistory.ts`

已核对要点：

1. 统一历史分页大小常量（`HISTORY_PAGE_SIZE`），抽象 latest/older 两类翻页查询。
2. 认证上下文（baseUrl + OAuth headers + org uuid）先构造后复用，减少重复准备成本。
3. 请求失败走 debug 日志并返回 `null`，调用方可按“可空页”语义处理。

---

## 4. Remote 适配与桥接

## 4.1 `remote/sdkMessageAdapter.ts`

已核对要点：

1. 将 SDKMessage 映射为 REPL 内部消息类型（assistant/user/system/stream_event）。
2. 对 result/system/tool_progress 等类型做显示裁剪与降噪（如 success result 默认忽略）。
3. 对未知 message type 采用“记录并忽略”策略，保证前后端版本错位时会话不崩溃。

---

## 4.2 `remote/remotePermissionBridge.ts`

已核对要点：

1. 为远端权限请求构造 synthetic AssistantMessage，满足本地权限确认组件输入约束。
2. 当远端工具在本地未加载时，构造最小 Tool stub 承接权限流，不中断审批链路。
3. stub 保持 `needsPermissions=true`，确保远端工具依然进入权限闸门而非直接放行。

---

## 5. 子系统不变量

1. coordinator 模式切换必须与恢复会话模式一致，否则会导致工具集和行为预期偏移。
2. 历史分页接口应保持“失败可空、成功有序”的稳定 contract，避免 UI 死锁。
3. SDK→REPL 适配层应优先保证兼容与可恢复，未知类型不得导致崩溃。
4. 远端工具即便本地无定义，也必须经过权限桥接与审批，不可绕过安全门。

---

## 6. 改动检查清单（29卷子域）

1. 改 coordinatorMode：是否破坏 mode 对齐、workerToolsContext 或 prompt 约束
2. 改 sessionHistory：是否影响 latest/older 游标语义与 header 认证一致性
3. 改 sdkMessageAdapter：是否误显示噪声消息或错误丢弃关键消息
4. 改 remotePermissionBridge：是否削弱远端工具权限确认链路

---

## 7. 边界说明

- 本卷聚焦 coordinator/assistant/remote 的支撑层，不展开 UI 组件细节。
- 结论均基于当前仓库源码；版本变化需重新核对。

