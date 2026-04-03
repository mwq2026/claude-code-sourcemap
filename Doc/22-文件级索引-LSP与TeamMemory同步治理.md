# 22｜文件级索引：LSP与TeamMemory同步治理

> 目标：继续补齐后续卷，覆盖编辑反馈链（LSP）与团队记忆同步链（TeamMemory）。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/services/lsp/LSPServerManager.ts`
2. `restored-src/src/services/lsp/LSPDiagnosticRegistry.ts`
3. `restored-src/src/services/lsp/manager.ts`
4. `restored-src/src/services/teamMemorySync/index.ts`
5. `restored-src/src/services/teamMemorySync/watcher.ts`
6. `restored-src/src/services/teamMemorySync/teamMemSecretGuard.ts`

---

## 2. LSP 子系统

## 2.1 `services/lsp/LSPServerManager.ts`

已核对要点：

1. 采用 manager 工厂 + 闭包状态（servers、extensionMap、openedFiles）。
2. 支持按文件扩展名路由 server，并按需启动（lazy start）。
3. 封装 `open/change/save/close` 文件生命周期同步到 LSP server。
4. 初始化与请求失败都有明确日志与错误路径，不阻断其他 server 注册。

---

## 2.2 `services/lsp/LSPDiagnosticRegistry.ts`

已核对要点：

1. 维护 pending diagnostics 注册表，供后续 attachment 生成消费。
2. 做批次内与跨轮次去重（含 LRU 上限）避免重复刷屏。
3. 有每文件和总量上限，按 severity 排序优先保留高严重度项。
4. `attachmentSent` 后会从 pending map 移除，避免重复投递。

---

## 2.3 `services/lsp/manager.ts`

已核对要点：

1. 提供全局单例生命周期管理：initialize / reinitialize / shutdown。
2. 用 `initializationState + generation` 防止过期初始化回写状态。
3. `getInitializationStatus/isLspConnected` 为上层提供可观测状态。
4. `reinitializeLspServerManager` 解决插件刷新后 LSP 配置重载问题。

---

## 3. Team Memory 同步子系统

## 3.1 `services/teamMemorySync/index.ts`

已核对要点：

1. 围绕 repo 作用域 team memory 做 pull/push，同步语义明确：
   - pull 以服务端为准覆盖本地
   - push 仅上传 checksum 差异项（delta upload）
2. 维护 `SyncState`（etag/checksum/max_entries）作为显式可传状态，避免模块全局可变共享。
3. 有请求超时、重试、413/409 等分支与错误类型分类。
4. 含 secret scan、条目大小/体积上限等安全与稳定性边界。

---

## 3.2 `services/teamMemorySync/watcher.ts`

已核对要点：

1. 启动流程：先 pull，再启动目录级 watch，并做 debounce push。
2. 区分“可恢复失败”和“永久失败”并可进入 suppression，避免无限重试风暴。
3. suppression 可由 unlink 事件清除，支持“删除后恢复”路径。
4. watcher 与 cleanup 注册联动，退出时可回收资源。

---

## 3.3 `services/teamMemorySync/teamMemSecretGuard.ts`

已核对要点：

1. 写入 team memory 前可做 secrets 检测。
2. 命中潜在敏感内容时返回阻断消息，防止共享敏感信息到协作者。
3. 内部有 TEAMMEM feature guard，关闭时逻辑惰性失效。

---

## 4. 子系统不变量

1. LSP 诊断投递必须去重并限量，防止上下文噪声扩散。
2. TeamMemory push 应始终按 delta + checksum 比对，不做盲量全传。
3. 同步失败应 fail-open 且可观测，不影响主会话可用性。

---

## 5. 改动检查清单（LSP/TeamMemory）

1. 改 LSP 生命周期：是否破坏 init/reinit generation 保护
2. 改诊断注册：是否仍保持跨轮次去重与上限截断
3. 改 TeamMemory 同步：是否保持 etag/checksum 与 delta 上传一致性
4. 改 watcher 抑制：是否可能重新引入失败重试风暴

---

## 6. 边界说明

- 本卷聚焦服务层核心逻辑，不覆盖 UI 呈现与文案。
- 所有结论锚定当前源码实现，后续版本需重新核验。

