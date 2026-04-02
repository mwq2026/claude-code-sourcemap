# 21｜文件级索引：Analytics与插件后台安装链路

> 目标：继续完成后续卷，补齐 analytics 事件总线与 plugins 后台安装链路。  
> 约束：只写当前实现已核对行为。

---

## 1. 覆盖文件

1. `restored-src/src/services/analytics/index.ts`
2. `restored-src/src/services/analytics/sink.ts`
3. `restored-src/src/services/analytics/growthbook.ts`
4. `restored-src/src/services/plugins/PluginInstallationManager.ts`
5. `restored-src/src/services/plugins/pluginOperations.ts`

---

## 2. Analytics 事件入口与路由

## 2.1 `services/analytics/index.ts`

已核对要点：

1. 作为 analytics 公共入口，设计上强调“低依赖避免循环引用”。
2. sink 未挂载时先入队，挂载后异步 drain 队列。
3. `logEvent/logEventAsync` 统一收口，支持测试态重置。
4. 明确 `_PROTO_*` 字段的 strip 逻辑入口（非 1P 后端应过滤）。

---

## 2.2 `services/analytics/sink.ts`

已核对要点：

1. sink 负责把事件路由到 Datadog 与 1P logging。
2. 先做事件采样，再做目标后端分发。
3. Datadog 路径会 strip `_PROTO_*`，1P 路径保留后交由 exporter 处理。
4. 有 gate 初始化与 kill switch 判定，支持动态关闭后端 sink。

---

## 2.3 `services/analytics/growthbook.ts`

已核对要点：

1. 维护 GrowthBook client、feature 值缓存、实验曝光记录与刷新信号。
2. 支持 env override 与 config override，且有优先级逻辑。
3. 提供 refresh listener 注册机制，供长生命周期模块在 gate 变化时重建配置。
4. 包含 reset/重初始化相关状态，处理认证变化与会话切换场景。

---

## 3. 插件后台安装与操作

## 3.1 `services/plugins/PluginInstallationManager.ts`

已核对要点：

1. 后台检查 marketplace 差异并驱动安装/更新流程，不阻塞启动。
2. 用 AppState 持续更新 marketplace 安装状态（pending/installing/installed/failed）。
3. 新安装成功后可触发插件刷新；更新场景会置 `needsRefresh` 走手动 reload 提示。
4. 全流程有 diagnostics 与 analytics 事件记录。

---

## 3.2 `services/plugins/pluginOperations.ts`

已核对要点：

1. 提供纯库函数式插件操作（install/uninstall/enable/disable/update），不直接 exit/console。
2. 明确 installable scopes 与 update scopes，含 runtime assert 与 type guard。
3. 插件定位、安装源解析、settings scope 映射、依赖反查等逻辑集中在此。
4. 与 marketplace manager、plugin loader、settings 持久化模块多点联动。

---

## 4. 子系统协同边界

1. analytics 是“事件管道”，业务模块只应调用统一入口，不直接耦合具体后端。
2. plugins 后台安装是“增量协调器”，不应在该层写交互式输出。
3. growthbook 刷新可影响插件/模型等长期配置消费者，应确保订阅方可重建状态。

---

## 5. 改动检查清单（analytics/plugins）

1. 改事件结构：是否保持 `_PROTO_*` 过滤边界
2. 改 gate 刷新：是否影响已订阅模块的运行时重建
3. 改插件后台安装：是否保持不阻塞启动与失败可回退
4. 改 scope 判定：是否影响 install/update 的权限与目标位置

---

## 6. 边界说明

- 本卷聚焦服务层与核心操作层，不覆盖插件 UI 组件细节。
- 所有结论锚定当前源码；后续版本需重新核验。

