# 18｜文件级索引：OAuth认证与API客户端装配

> 目标：继续补齐后续卷，聚焦 OAuth 登录流与 API client/provider 装配路径。  
> 口径：仅记录当前源码可核验实现。

---

## 1. 覆盖文件

1. `restored-src/src/services/oauth/index.ts`
2. `restored-src/src/services/oauth/client.ts`
3. `restored-src/src/services/oauth/auth-code-listener.ts`
4. `restored-src/src/services/api/client.ts`
5. `restored-src/src/services/api/claude.ts`

---

## 2. OAuth 主流程（服务层）

## 2.1 `services/oauth/index.ts`

已核对要点：

1. `OAuthService.startOAuthFlow` 同时构造 automatic/manual 两种授权 URL。
2. automatic 路径依赖 localhost 回调监听；manual 路径依赖用户输入授权码。
3. `skipBrowserOpen` 支持“由外部调用方控制打开方式”（SDK 控制协议场景）。
4. 收到授权码后执行 token 交换，再拉取 profile 信息并格式化为 `OAuthTokens`。
5. automatic 成功/失败都会尝试完成浏览器重定向收尾并清理 listener。

---

## 2.2 `services/oauth/auth-code-listener.ts`

已核对要点：

1. 本地 HTTP server 只用于捕获 OAuth 回调，不承担 OAuth 授权本身。
2. 启动时绑定 localhost 随机端口（或指定端口），避免固定端口冲突。
3. 通过 `state` 参数校验做 CSRF 防护。
4. 回调成功时暂存 `pendingResponse`，由后续 success/error redirect 统一收口。
5. `close()` 会清理 listener，并在有 pendingResponse 时先发错误重定向。

---

## 2.3 `services/oauth/client.ts`

已核对要点：

1. `buildAuthUrl` 支持 inference-only、orgUUID、login_hint、login_method 等参数。
2. token exchange/refresh 均走 axios，包含超时与状态码错误处理。
3. refresh 路径会尽量复用现有 profile/subscription 信息，减少额外 profile 请求。
4. OAuth 与本地配置/安全存储有联动更新逻辑（如 displayName、billing 等字段）。

---

## 3. API 客户端装配与 provider 分流

## 3.1 `services/api/client.ts`

已核对要点：

1. `getAnthropicClient` 统一装配 headers/session id/user-agent 等公共参数。
2. 按环境与 provider 分流：Bedrock / Foundry / Vertex / Direct Anthropic。
3. 各 provider 分支有独立认证路径（AWS 凭证刷新、Azure AD token provider、GoogleAuth）。
4. 先执行 `checkAndRefreshOAuthTokenIfNeeded`，再决定 OAuth token 或 API key 头。
5. 支持 `CLAUDE_CODE_EXTRA_BODY`、custom headers、debug logger 等诊断能力。

---

## 3.2 `services/api/claude.ts`

已核对要点：

1. 负责 query/stream 级请求拼装与事件处理，是 API 请求执行核心层。
2. 挂接 betas、effort、fast mode、cache 策略、tool schema 转换等多维参数。
3. 与 retry、logging、prompt cache break detection、compact 等子模块深度耦合。
4. 对 assistant/tool/result/message normalize 与 pairing 有专门处理路径。

---

## 4. 认证与客户端装配不变量

1. OAuth 回调必须 state 匹配才可接受授权码。
2. client 装配先做 token freshness，再决定认证头落位。
3. provider 分支认证失败应可观测（日志/事件），不应静默吞错。
4. automatic/manual 授权都要保证 listener 清理与流程收口。

---

## 5. 改动检查清单（OAuth/API）

1. 改 OAuth 参数：是否影响 manual/automatic 双路径一致性
2. 改 token 刷新：是否破坏 profile/subscription 缓存回填逻辑
3. 改 provider 分支：是否保持各云平台认证前置条件
4. 改 stream/query 处理：是否保持 message/tool 结构化归一

---

## 6. 边界说明

- 本卷关注认证与客户端装配，不覆盖 UI 登录交互细节。
- 所有描述基于当前可见源码，后续版本需重新核验。

