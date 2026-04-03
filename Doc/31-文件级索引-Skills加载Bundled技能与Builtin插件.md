# 31｜文件级索引：Skills 加载、Bundled 技能与 Builtin 插件

> 目标：继续推进后续卷，补齐扩展平面中 skills/plugins 的核心注册与加载语义。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/skills/loadSkillsDir.ts`
2. `restored-src/src/skills/mcpSkillBuilders.ts`
3. `restored-src/src/skills/bundledSkills.ts`
4. `restored-src/src/plugins/builtinPlugins.ts`
5. `restored-src/src/plugins/bundled/index.ts`

---

## 2. Skills 加载主链

## 2.1 `skills/loadSkillsDir.ts`

已核对要点：

1. 统一 skills 路径来源（user/project/policy/plugin），并处理 frontmatter 字段解析与校验。
2. 支持路径约束、参数替换、hooks/schema 解析、模型与 effort 配置等技能元数据。
3. 通过 `createSkillCommand` 产出 Command，保持技能文件与运行时命令语义一致。
4. 在模块初始化阶段注册 MCP skill builders，避免 mcpSkills 与 loadSkillsDir 循环依赖。

---

## 2.2 `skills/mcpSkillBuilders.ts`

已核对要点：

1. 作为“只存函数引用”的注册器，承接 `createSkillCommand/parseSkillFrontmatterFields`。
2. 以 write-once registry 方式让 MCP 技能发现链路复用 loadSkillsDir 能力。
3. builders 未注册时直接抛错，避免静默降级导致技能发现失真。

---

## 3. Bundled 技能注册与文件落地

## 3.1 `skills/bundledSkills.ts`

已核对要点：

1. 提供 bundled skill 注册表（`registerBundledSkill/getBundledSkills`），用于内置技能命令装配。
2. 支持首调懒提取 `files` 到本地目录，并在 prompt 前注入“Base directory for this skill”。
3. 文件写入采用路径逃逸检查 + 安全 flag（含 nofollow/excl）策略，降低目录穿越与覆盖风险。
4. 支持技能可见性、allowedTools、context/agent、hooks 等行为参数。

---

## 4. Builtin 插件注册与启停

## 4.1 `plugins/builtinPlugins.ts`

已核对要点：

1. 维护 built-in 插件注册表，插件 ID 采用 `{name}@builtin`。
2. 启用态判定遵循“用户设置优先，其次 defaultEnabled，再次 true”。
3. 可导出 enabled/disabled 插件列表，并将启用插件的 skills 转换为 Command。
4. built-in 插件与 bundled skill 的概念分离：前者强调可在 `/plugin` UI 中启停。

---

## 4.2 `plugins/bundled/index.ts`

已核对要点：

1. 作为 built-in plugin 初始化入口（`initBuiltinPlugins`）。
2. 当前代码保留框架与注释说明，尚未注册具体 built-in 插件定义。

---

## 5. 子系统不变量

1. MCP 技能构建能力必须通过注册器注入，避免循环依赖与运行时缺失。
2. bundled 技能文件提取必须保持“不可逃逸 skill dir + 安全写入”约束。
3. builtin 插件启停状态必须稳定可重现（设置优先级明确）。
4. 插件“可启停”与技能“可调用”是两层语义，不能混用概念导致行为漂移。

---

## 6. 改动检查清单（31卷子域）

1. 改 loadSkillsDir：是否破坏 frontmatter 解析、Command 生成或路径来源优先级
2. 改 mcpSkillBuilders：是否破坏 builders 注册时序与 MCP 技能发现
3. 改 bundledSkills：是否破坏安全写入、目录注入提示或首调懒提取语义
4. 改 builtinPlugins：是否破坏 enabled 判定或 pluginId 约定（`@builtin`）

---

## 7. 边界说明

- 本卷聚焦 skills/plugins 加载注册层，不展开具体每个 bundled skill 内容细节。
- 结论均基于当前源码；若插件体系演进需重新核验。

