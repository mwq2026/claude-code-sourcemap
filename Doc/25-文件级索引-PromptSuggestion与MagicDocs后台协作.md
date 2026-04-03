# 25｜文件级索引：PromptSuggestion与MagicDocs后台协作

> 目标：继续按同节奏补齐后续卷，聚焦提示建议与文档自动更新两条后台协作链。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/services/PromptSuggestion/promptSuggestion.ts`
2. `restored-src/src/services/PromptSuggestion/speculation.ts`
3. `restored-src/src/services/MagicDocs/magicDocs.ts`
4. `restored-src/src/services/MagicDocs/prompts.ts`

---

## 2. PromptSuggestion 主链

## 2.1 `services/PromptSuggestion/promptSuggestion.ts`

已核对要点：

1. 负责 suggestion 开关判定、抑制条件判定、生成流程与状态写回。
2. 抑制条件覆盖：非交互会话、计划模式、权限等待、速率限制、对话早期等。
3. suggestion 通过 forked agent 生成，并使用 deny-tools 策略保持“仅建议文本”。
4. 与 speculation 可选联动：建议生成后可触发预执行分支。

---

## 2.2 `services/PromptSuggestion/speculation.ts`

已核对要点：

1. 负责投机执行状态机（active/accepted/aborted 等）与边界判定。
2. 包含 overlay 路径管理、写入复制、消息注入清洗、统计与反馈消息构造。
3. 对中断消息、未完成 tool_use、失败结果有过滤与修正逻辑，避免脏上下文回灌。
4. 与 promptSuggestion 主链共享 suppress/filter 语义，保持一致体验。

---

## 3. MagicDocs 主链

## 3.1 `services/MagicDocs/magicDocs.ts`

已核对要点：

1. 通过文件头 `# MAGIC DOC: ...` 识别并跟踪文档文件。
2. 在 post-sampling 空闲窗口（上轮无工具调用）触发后台更新。
3. 更新时会重新读取文档并再次检测 header，失效文档自动移出跟踪。
4. 子代理仅允许 Edit 指向目标 magic doc 路径，限制越界修改风险。

---

## 3.2 `services/MagicDocs/prompts.ts`

已核对要点：

1. 提供默认更新提示模板，并支持 `~/.claude/magic-docs/prompt.md` 自定义覆盖。
2. 变量替换采用单次替换策略，避免二次替换与 `$` 反向引用污染。
3. 支持可选文档级 instructions 注入，并与通用规则合并成最终 prompt。

---

## 4. 子系统不变量

1. PromptSuggestion 不应在不适合时机强行出建议（需严格遵守 suppress 条件）。
2. speculation 注入前必须清洗消息，避免把中断/未完成工具结果当真实上下文。
3. MagicDocs 更新必须保留 magic header，不得扩散到非目标文件。

---

## 5. 改动检查清单（Prompt/MagicDocs）

1. 改建议触发：是否影响 suppress/filter 判定一致性
2. 改 speculation：是否破坏 overlay 合并与消息清洗安全边界
3. 改 MagicDocs 工具权限：是否仍保证“仅目标文件可编辑”
4. 改 prompt 模板：是否保持变量替换稳定与 header 保留约束

---

## 6. 边界说明

- 本卷聚焦后台协作服务层，不展开前端展示行为。
- 全部结论以当前源码实现为准，版本变化需重新核验。

