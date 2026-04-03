# 23｜文件级索引：AutoDream与SessionMemory后台记忆链路

> 目标：继续完成后续卷，补齐自动记忆相关后台链路。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/services/autoDream/autoDream.ts`
2. `restored-src/src/services/SessionMemory/sessionMemory.ts`

---

## 2. AutoDream（跨会话 consolidation）

## 2.1 `services/autoDream/autoDream.ts`

已核对要点：

1. 以后台 forked subagent 方式触发 consolidation，不阻塞主对话流。
2. 触发门控顺序清晰：time gate → session gate → lock。
3. 使用 consolidation lock 防并发冲突，并在失败时支持 rollback。
4. 有 scan throttle，避免门控满足时高频重复扫描。
5. 与 DreamTask 状态机联动（注册、进度、完成、失败、kill）。
6. 对工具能力有显式约束文案（只读 shell 等），降低运行期误判。

---

## 3. SessionMemory（当前会话持续摘要）

## 3.1 `services/SessionMemory/sessionMemory.ts`

已核对要点：

1. 通过 post-sampling hook 周期性抽取会话记忆并写入 session memory 文件。
2. 触发需同时满足阈值策略（初始化阈值、更新间隔阈值、工具调用阈值/轮次条件）。
3. 使用 `runForkedAgent` 执行提取，隔离上下文，避免污染主状态。
4. 对 memory 文件读写前有目录/文件初始化与缓存状态处理。
5. gate/config 读取采用 cached may be stale 路径，避免阻塞。

---

## 4. 双链路协同边界

1. AutoDream 侧重“跨会话汇总”，SessionMemory 侧重“单会话持续摘要”。
2. 两者都走后台执行与门控触发，不应影响主线程请求稳定性。
3. 锁与阈值机制是防止重复执行/风暴触发的关键保障。

---

## 5. 改动检查清单（记忆链路）

1. 改触发门控：是否仍保持 time/session/lock 顺序与节流策略
2. 改后台任务状态：是否影响 DreamTask 生命周期一致性
3. 改 session memory 阈值：是否导致过密抽取或长期不抽取
4. 改 forked agent 调用：是否破坏主状态隔离与文件写入安全边界

---

## 6. 边界说明

- 本卷仅覆盖 autoDream/sessionMemory 主逻辑，不展开 prompt 模板细节。
- 结论均锚定当前源码实现，后续版本需重新核验。

