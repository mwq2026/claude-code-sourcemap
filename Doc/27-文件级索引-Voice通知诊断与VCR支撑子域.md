# 27｜文件级索引：Voice通知诊断与VCR支撑子域

> 目标：持续补高价值卷，覆盖语音链路、通知发送、诊断追踪与测试录制支撑能力。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/services/voice.ts`
2. `restored-src/src/services/voiceKeyterms.ts`
3. `restored-src/src/services/voiceStreamSTT.ts`
4. `restored-src/src/services/notifier.ts`
5. `restored-src/src/services/diagnosticTracking.ts`
6. `restored-src/src/services/internalLogging.ts`
7. `restored-src/src/services/vcr.ts`

---

## 2. Voice 链路

## 2.1 `services/voice.ts`

已核对要点：

1. 负责录音能力探测、依赖检测与录音启动/停止路径。
2. 优先 native audio（cpal/NAPI），Linux 下含 arecord/SoX 回退链。
3. 对 WSL/远程环境与无音频设备场景提供明确不可用原因。

---

## 2.2 `services/voiceKeyterms.ts`

已核对要点：

1. 生成 STT keyterms（全局词 + 项目名 + 分支词 + 最近文件词）。
2. 通过上限截断与分词规则控制噪声与长度。

---

## 2.3 `services/voiceStreamSTT.ts`

已核对要点：

1. 实现 voice_stream WebSocket 客户端，管理 keepalive/finalize/close 生命周期。
2. 负责 OAuth token 刷新后连接、query 参数装配、音频帧发送与 transcript 事件处理。
3. finalize 路径有 no-data/safety/ws-close 等收敛分支，降低挂起风险。

---

## 3. 通知与诊断子域

## 3.1 `services/notifier.ts`

已核对要点：

1. 对外统一通知入口，先执行 notification hooks 再按 channel 分发。
2. 支持 auto/iterm2/kitty/ghostty/bell 等路径并记录 method telemetry。

---

## 3.2 `services/diagnosticTracking.ts`

已核对要点：

1. 诊断追踪为单例服务，维护 baseline 与 right-file 诊断状态。
2. 通过 IDE RPC 获取诊断并做“新增诊断差异”提取。
3. 对路径做规范化比较，覆盖 Windows 大小写/分隔符差异。

---

## 3.3 `services/internalLogging.ts`

已核对要点：

1. 记录 ANT 内部权限上下文，包含 namespace/containerId。
2. 使用 memoized 文件读取探测运行环境并上报 telemetry。

---

## 4. VCR 测试支撑

## 4.1 `services/vcr.ts`

已核对要点：

1. 提供 fixture 读写与 API 消息脱敏/还原流程。
2. CI 下缺失 fixture 且未开启录制会直接失败，保证可重现性。
3. 通过路径与动态字段归一化提升跨平台 fixture 稳定性。

---

## 5. 子系统不变量

1. Voice finalize 需保证可收敛，避免连接悬挂。
2. 诊断新增判定需与 baseline 对账，避免重复噪声输出。
3. VCR 脱敏/归一化规则需稳定，避免 fixture 哈希漂移。

---

## 6. 改动检查清单（Voice/诊断/VCR）

1. 改语音连接：是否仍保持 keepalive 与 finalize 多分支收敛
2. 改诊断跟踪：是否仍正确识别“新增诊断”且跨平台路径一致
3. 改通知分发：是否保持 hook 先执行与 method telemetry 完整
4. 改 VCR：是否破坏 CI fixture 约束与跨平台归一化

---

## 7. 边界说明

- 本卷聚焦服务子域协作，不展开 UI 组件层实现。
- 结论以当前源码为准，后续版本需重新核验。

