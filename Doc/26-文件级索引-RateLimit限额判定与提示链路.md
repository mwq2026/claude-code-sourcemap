# 26｜文件级索引：RateLimit限额判定与提示链路

> 目标：继续补齐高价值服务子域，聚焦 Claude AI 限额判定、mock 注入与用户提示闭环。  
> 口径：仅记录当前源码可核验行为。

---

## 1. 覆盖文件

1. `restored-src/src/services/claudeAiLimits.ts`
2. `restored-src/src/services/claudeAiLimitsHook.ts`
3. `restored-src/src/services/rateLimitMessages.ts`
4. `restored-src/src/services/rateLimitMocking.ts`
5. `restored-src/src/services/mockRateLimits.ts`

---

## 2. 限额状态主链

## 2.1 `services/claudeAiLimits.ts`

已核对要点：

1. 负责配额状态模型与全局 `currentLimits` 维护，统一状态变更广播。
2. 支持从响应头/错误对象提取限额与 overage 信息，并处理 early warning。
3. 维护原始利用率窗口（5h/7d）供外部状态线读取。
4. 在非交互与非必要流量场景存在前置门控，避免无效 quota 请求。

---

## 2.2 `services/claudeAiLimitsHook.ts`

已核对要点：

1. 以 React hook 订阅 `statusListeners`，将全局限额状态映射到 UI 层。
2. 通过浅拷贝更新避免外部直接持有可变引用。

---

## 3. 提示与 mock 适配层

## 3.1 `services/rateLimitMessages.ts`

已核对要点：

1. 集中生成 rate-limit 错误/警告文案，避免 UI 侧散落字符串判断。
2. 区分 `error` 与 `warning` 展示路径，并含订阅类型/计费权限分支。
3. 覆盖 overage、weekly/session、opus/sonnet 等差异化提示语义。

---

## 3.2 `services/rateLimitMocking.ts`

已核对要点：

1. 提供生产逻辑前的 mock facade：是否处理 mock、是否抛 429、mock 判定封装。
2. 将 mock 条件集中隔离，避免污染主业务判定路径。

---

## 3.3 `services/mockRateLimits.ts`

已核对要点：

1. 提供 [ANT-only] mock headers/scenario 机制，支持细粒度 header 开关。
2. 支持 exceeded limits/early warning/fast-mode 等测试场景拼装。
3. 会动态更新 representative claim 与 retry-after，维持 mock 行为一致性。

---

## 4. 子系统不变量

1. 限额状态来源需保持“单一真源”（header/error → normalized limits）。
2. warning 文案与 error 文案不可混用展示通道。
3. mock 逻辑必须通过 facade 进入，避免生产路径分叉失控。

---

## 5. 改动检查清单（RateLimit）

1. 改 header 解析：是否影响 early warning 与 overage 分流
2. 改文案策略：是否破坏 warning/error 展示边界
3. 改 mock 场景：是否仍能稳定复现 representative claim + retry-after
4. 改 hook 订阅：是否保持 UI 状态同步及时且无共享引用问题

---

## 6. 边界说明

- 本卷聚焦限额状态与提示子域，不覆盖 API 客户端实现细节。
- 所有结论锚定当前源码实现，版本变更需重新核验。

