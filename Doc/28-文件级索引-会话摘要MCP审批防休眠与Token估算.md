# 28｜文件级索引：会话摘要、MCP审批、防休眠与Token估算

> 目标：继续补齐 services 高价值剩余子域，覆盖“离开总结/审批交互/系统保活/token 估算”四类支撑能力。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/services/awaySummary.ts`
2. `restored-src/src/services/mcpServerApproval.tsx`
3. `restored-src/src/services/preventSleep.ts`
4. `restored-src/src/services/tokenEstimation.ts`

---

## 2. 会话摘要与审批交互

## 2.1 `services/awaySummary.ts`

已核对要点：

1. 负责生成“用户离开后回来”的短摘要，窗口截取最近消息并可拼接 session memory。
2. 通过 `queryModelWithoutStreaming` 走禁思考、无工具路径，仅做轻量 recap。
3. 对中断与异常场景返回 `null`，避免阻塞主流程。

---

## 2.2 `services/mcpServerApproval.tsx`

已核对要点：

1. 处理 project 作用域 MCP server 的 pending 审批对话流程。
2. 单个 pending 走单选审批弹窗，多个 pending 走多选弹窗。
3. 复用现有 Ink root 与 AppState/Keybinding 上下文，不另起渲染实例。

---

## 3. 运行时支撑能力

## 3.1 `services/preventSleep.ts`

已核对要点：

1. 提供 macOS 防休眠能力：以 `caffeinate` 子进程维持 idle sleep 抑制。
2. 通过 ref-count 控制启停，多任务并发下避免重复拉起与过早停止。
3. 周期重启 + cleanup 注册形成自愈与退出清理闭环。

---

## 3.2 `services/tokenEstimation.ts`

已核对要点：

1. 统一 token 计数入口，支持 API/provider 差异（Anthropic/Vertex/Bedrock）分流。
2. 对 tool-search 字段与 thinking block 做预处理，降低计数调用报错风险。
3. 提供 rough estimation 与按文件类型字节比修正（如 json 更密集）。
4. 集成 VCR token-count 录制回放路径，保证测试可重放。

---

## 4. 子系统不变量

1. away summary 必须轻量、可中断，失败时不影响主对话流程。
2. MCP 审批弹窗仅消费 pending 项，且渲染上下文必须与主应用一致。
3. preventSleep 只能在 darwin 生效，且 ref-count 与进程生命周期需一致。
4. token 估算在 provider/beta 差异下应优先“可用性”而非单一路径假设。

---

## 5. 改动检查清单（28卷子域）

1. 改 awaySummary：是否仍保证短输出、无工具与异常兜底
2. 改审批流程：是否仍区分单/多 pending 并正确结束 Promise
3. 改防休眠：是否破坏 ref-count 或导致孤儿 caffeinate
4. 改 token 计数：是否影响多 provider 分流、thinking/tool-search 兼容与 VCR 稳定性

---

## 6. 边界说明

- 本卷聚焦 services 支撑能力，不展开 UI 组件实现细节。
- 结论均可回溯到当前源码；版本变化后需重新核验。

