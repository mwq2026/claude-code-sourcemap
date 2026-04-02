# 32｜文件级索引：插件 CLI 命令与 SessionMemory 工具层

> 目标：继续推进后续卷，补齐“插件命令封装层 + SessionMemory 工具/模板层”的文件级语义。  
> 口径：仅记录当前源码可直接核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/services/plugins/pluginCliCommands.ts`
2. `restored-src/src/services/SessionMemory/sessionMemoryUtils.ts`
3. `restored-src/src/services/SessionMemory/prompts.ts`

---

## 2. 插件 CLI 命令封装

## 2.1 `services/plugins/pluginCliCommands.ts`

已核对要点：

1. 提供 install/uninstall/enable/disable/disable-all/update 的 CLI 包装函数。
2. 核心操作委托给 `pluginOperations.ts`，本层负责 console 输出、退出码、错误处理与 telemetry 上报。
3. 统一失败路径 `handlePluginCommandError`：记录错误分类、插件标识字段并退出。
4. 成功路径按动作发出对应 analytics 事件，且区分 plugin name/marketplace 与 scope 等字段。

---

## 3. SessionMemory 状态与配置工具

## 3.1 `services/SessionMemory/sessionMemoryUtils.ts`

已核对要点：

1. 管理 session memory 的共享状态：初始化标记、提取中状态、上次摘要点、提取时 token 基线。
2. 提供阈值判定函数（初始化阈值、更新阈值、工具调用间隔）供主流程复用。
3. 提供提取等待机制（超时/过期保护），避免 compaction 等场景无限等待。
4. `getSessionMemoryContent` 读取失败时对 FS 不可用场景返回 `null`，并保留事件埋点。

---

## 4. SessionMemory 模板与提示词工具

## 4.1 `services/SessionMemory/prompts.ts`

已核对要点：

1. 内置默认 template/prompt，并支持从 `~/.claude/session-memory/config/` 覆盖加载。
2. 生成 update prompt 时会做变量替换（`{{currentNotes}}/{{notesPath}}`）并附加超长 section 提醒。
3. 对总 token 与分节 token 超限有专门压缩提示逻辑，约束记忆文件体积。
4. 提供 compact 场景下的 session memory 截断辅助函数，防止记忆文本挤占预算。

---

## 5. 子系统不变量

1. 插件 CLI 层应保持“薄封装”定位：业务语义在 operations，CLI 只承接交互与退出语义。
2. SessionMemory 共享状态更新必须有明确入口，避免并发提取导致状态错乱。
3. 提示词生成必须保留模板结构约束，防止 session memory 文件被错误改写。
4. 总量/分节预算提醒必须可触发，避免 memory 膨胀破坏 compact 可用性。

---

## 6. 改动检查清单（32卷子域）

1. 改 pluginCliCommands：是否破坏失败退出码、telemetry 字段或 scope 判定
2. 改 sessionMemoryUtils：是否破坏提取状态机、阈值判定或等待超时语义
3. 改 prompts：是否破坏变量替换、模板保真与超限提醒逻辑
4. 改 compact 相关截断：是否导致 session memory 注入后 token 预算失衡

---

## 7. 边界说明

- 本卷聚焦插件 CLI 命令层与 SessionMemory 工具层，不重复展开主提取流程（见前卷）。
- 结论均基于当前源码；版本演进后需重新核验。

