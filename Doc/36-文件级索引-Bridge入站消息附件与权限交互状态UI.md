# 36｜文件级索引：Bridge 入站消息、附件与权限交互状态 UI

> 目标：继续补齐 bridge 子域文档，覆盖“入站消息清洗 + 附件解析 + 权限交互 + 状态 UI”的用户输入面。  
> 口径：仅记录当前源码可直接核验行为。

---

## 1. 覆盖文件

1. `restored-src/src/bridge/inboundMessages.ts`
2. `restored-src/src/bridge/inboundAttachments.ts`
3. `restored-src/src/bridge/bridgePermissionCallbacks.ts`
4. `restored-src/src/bridge/bridgeUI.ts`
5. `restored-src/src/bridge/types.ts`

---

## 2. 入站消息提取与图片块规范化

## 2.1 `bridge/inboundMessages.ts`

已核对要点：

1. 仅提取 `user` 类型消息，忽略无内容或空数组消息。
2. 兼容提取 `uuid`，作为后续入队上下文标识。
3. 对图片 block 执行标准化，补齐 `media_type`（兼容 `mediaType` 或缺省场景）。
4. 在无需修复时走 fast-path，保持零额外分配。

---

## 3. 入站附件解析与 @path 前缀注入

## 3.1 `bridge/inboundAttachments.ts`

已核对要点：

1. 解析 `file_attachments`（zod 校验），提取 `file_uuid/file_name`。
2. 通过 OAuth 鉴权下载附件内容并写入本地 uploads 目录。
3. 文件名做安全净化，避免路径穿越与危险字符传播。
4. 生成 `@"absolute/path"` 前缀并注入消息内容，兼容字符串与 block 数组。
5. 整体按 best-effort 处理：单个附件失败不会中断整条消息。

---

## 4. 权限回调协议与响应判定

## 4.1 `bridge/bridgePermissionCallbacks.ts`

已核对要点：

1. 定义 `sendRequest/sendResponse/cancelRequest/onResponse` 回调契约。
2. 定义权限响应结构（allow/deny + 可选更新字段）。
3. 提供 `isBridgePermissionResponse` 类型守卫，避免不安全强转。

---

## 5. 状态 UI 与日志展示面

## 5.1 `bridge/bridgeUI.ts`

已核对要点：

1. 提供 bridge logger，管理连接态/空闲态/失败态与状态行渲染。
2. 集成 QR 码显示、session 计数、spawn mode 提示与活动摘要展示。
3. 对终端可视行数有显式计算与清屏恢复逻辑，降低状态刷新错位。

## 5.2 `bridge/types.ts`（本卷关联部分）

已核对要点：

1. 提供 bridge 协议与配置核心类型（WorkResponse/BridgeConfig/SessionHandle 等）。
2. 定义权限响应事件、session 活动、spawn mode 等跨模块共享结构。
3. 为 UI/logger 与 bridge 运行态提供稳定类型边界。

---

## 6. 子系统不变量

1. 入站消息必须先归一化再入队，避免坏 block 污染后续会话调用。
2. 附件解析必须坚持 best-effort，不因单个文件失败阻断主消息。
3. 权限交互必须维持 request/response/cancel 可闭环，避免前端悬挂提示。
4. 状态 UI 必须可重复渲染且可清理，避免终端残留和状态漂移。

---

## 7. 改动检查清单（36卷子域）

1. 改 inboundMessages：是否破坏 user 消息筛选与图片 block 规范化逻辑
2. 改 inboundAttachments：是否破坏附件下载、路径净化与 @path 注入语义
3. 改 bridgePermissionCallbacks：是否破坏权限回调契约与响应判定边界
4. 改 bridgeUI/types：是否破坏状态渲染、session 展示与共享类型兼容性

---

## 8. 边界说明

- 本卷聚焦 bridge 输入与展示面，不展开 bridgeMain/transport/poll 主循环细节。
- 结论均基于当前源码；后续协议字段调整需同步复核。

