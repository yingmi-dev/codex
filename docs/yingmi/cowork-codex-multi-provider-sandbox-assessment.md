# Codex 面向 Cowork 的多 Provider、Linux CI、沙箱与凭证隔离改造调查评估

> 调研日期：2026-08-27
>
> 范围：本仓库当前实现、GitHub Actions、Codex Rust workspace，以及 DeepSeek、Qwen、Z.AI/智谱 GLM、Moonshot/Kimi、MiniMax 和 Microsandbox 的公开官方资料。

## 结论摘要

这不是给 `base_url` 增加几个配置项的工作，而是一次“运行时边界重构”。当前 Codex 的模型主链路已经收敛到 OpenAI Responses API：`model-provider-info` 中 `WireApi::Chat` 已被明确拒绝，`core/src/client.rs` 直接构造 Responses request、reasoning、prompt cache key、Responses WebSocket 和 OpenAI 专用 header。因此，五家 provider 中只有明确提供 Responses-compatible 接口的模型可以低成本接入；仅提供 Chat Completions-compatible 接口的模型需要新增 wire adapter，或者在外部部署协议转换层。

建议采用以下目标架构：

1. 保留一个 provider-neutral 的 agent contract，把模型能力拆成 wire protocol、reasoning、tool calling、usage、cache 和 transport capability，而不是把所有能力压进 OpenAI 类型。
2. 将当前 `codex-network-proxy` 演进为本机 egress gateway：既做网络策略，也做按目标和调用来源限定的 credential broker；所有响应统一剥离 `Set-Cookie`、代理认证和内部 header。
3. 将执行器抽象为 `LocalSandbox`、现有 OS sandbox、remote exec 和 `Microsandbox` backend。Microsandbox 作为高隔离 backend，不应直接替换所有本地快速执行路径。
4. 将 ChatGPT/ OpenAI cloud/remote-control 作为一个可选 first-party integration 层，核心 agent、app-server、exec-server 和 provider runtime 不应再依赖 ChatGPT 登录或 `chatgpt.com/backend-api`。
5. CI 可以把“编译、静态检查、Linux 可运行测试、跨平台交叉构建”迁到 Linux；不能把“原生平台验证、macOS 签名/公证、Windows MSVC native 行为”简单等同为 Linux 构建。

推荐分四个阶段实施：先做 provider contract 与 capability matrix；再做 gateway/credential isolation；然后接入 Microsandbox；最后裁剪 Cowork 产品形态并迁移 remote/cloud 能力。这样每阶段都能独立验收，避免同时改变请求协议、进程隔离和产品入口。

## 1. 当前实现基线

### 1.1 Provider 与 Responses API

相关入口：

- `codex-rs/model-provider-info/src/lib.rs`
- `codex-rs/model-provider/src/lib.rs`
- `codex-rs/core/src/client.rs`
- `codex-rs/protocol/src/openai_models.rs`
- `codex-rs/protocol/src/config_types.rs`

当前 `ModelProviderInfo` 已支持：`base_url`、环境变量 API key、命令认证、Bearer token、额外静态/环境 header、query 参数、重试、stream idle timeout、WebSocket 和 standalone web search capability。对接普通 OpenAI-compatible 服务的配置面已经存在。

但关键限制也很明确：

- `WireApi` 目前只有 `Responses`；读取 `wire_api = "chat"` 会返回迁移错误。
- `core/src/client.rs` 会自动生成 `reasoning`、`prompt_cache_key`、`include = reasoning.encrypted_content`、`parallel_tool_calls`、Responses metadata 和 OpenAI 内部 header。
- `use_responses_lite`、Responses WebSocket、`/responses/compact`、OpenAI telemetry header 和内部 metadata 不是通用 OpenAI-compatible API 的稳定能力。
- 上层协议中的 `ReasoningEffort`、`ReasoningSummary` 和 `TokenUsage` 命名仍带有 OpenAI Responses 语义；不同 provider 的 reasoning token、cached token 和 total token 不能未经归一化直接填入。

结论：现有自定义 provider 配置适合“同样支持 Responses 的服务”，不够支撑五家 provider 的完整 agent loop。

### 1.2 网络代理与凭证保护的已有能力

`codex-rs/network-proxy` 已经是重要的可复用基础，而不是从零开始：

- HTTP proxy、SOCKS5、HTTPS CONNECT 和可选 HTTPS MITM；
- domain allow/deny、limited mode、local/private address 防护；
- MITM hook，可按 host、method、path 匹配并执行 `strip_auth`；
- `credential_broker`，目前已有 OpenAI、GitHub provider 方向的抽象；
- 通过环境变量把 proxy 路由注入子进程，并有 proxy policy audit event；
- `responses-api-proxy`、`http-client`、`linux-sandbox`、MCP HTTP 测试已经有 proxy 集成。

这说明用户提出的“网关”与当前设计方向一致。缺口主要是：凭证注入策略还没有形成通用的 destination-scoped contract；响应侧需要明确、强制和统一地清除 `Set-Cookie` 等 header；并且必须覆盖非 Codex 自带 HTTP client、工具进程直连、DNS/代理绕过、WebSocket 和日志路径。

### 1.3 CI 基线

当前 workflow 同时使用：

- `ubuntu-24.04`、`ubuntu-24.04-arm`；
- macOS xlarge runner，用于 Apple targets；
- Windows x64/arm64 runner，用于 MSVC 编译、Windows sandbox helper 和 native tests；
- Linux 上的 Windows cross-compile/Bazel path 已经存在。

因此 Linux runner 改造有可利用的现成路径，但不是把所有 `runs-on` 替换成 `ubuntu-24.04`：

- macOS/Windows 目标可以在 Linux 做 cross build，但 native linker、系统 API、代码签名、公证、Windows sandbox 运行行为不能由 cross build 证明；
- Microsandbox 本地运行需要 KVM；GitHub-hosted runner 是否暴露嵌套虚拟化必须单独验证；
- Linux CI 能覆盖 Linux sandbox、Bazel、Rust 单测和多数 integration test，但会改变 `runner.os`、路径、shell、证书库、网络和终端行为。

## 2. 五家 LLM Provider 兼容性评估

以下判断针对“让 Codex 运行可靠的 coding-agent loop”，不是仅验证一次普通问答请求。

| Provider | 官方 API 形态 | Reasoning / tool 关键差异 | Cache / usage 关键差异 | 初步结论 |
|---|---|---|---|---|
| DeepSeek | Chat Completions；同时已有 Responses API，当前 Responses 文档列出的模型范围有限 | thinking 返回 `reasoning_content`；带 tool call 的后续请求必须原样回传 reasoning content；thinking 模式不支持 temperature 等参数 | Chat usage 有 `prompt_cache_hit_tokens` / `prompt_cache_miss_tokens`；Responses 使用 `input_tokens_details.cached_tokens` 和 `output_tokens_details.reasoning_tokens` | `deepseek-v4-flash` 可优先走 Responses adapter；Chat 路径仍需 reasoning/tool adapter |
| Qwen / Model Studio | OpenAI-compatible Chat Completions 和 Responses | 可能使用非标准 `enable_thinking`，推荐逐步迁移到 `reasoning.effort`；reasoning 以 Responses reasoning output item 返回；tool_choice、并行工具和模型地域能力有差异 | Chat 使用 `usage.prompt_tokens_details.cached_tokens`；不同模型的 `max_output_tokens` 是否包含思维 token 不一致 | Responses 兼容模型可试点；必须按模型 capability 配置，不能全局假设 |
| Z.AI / 智谱 GLM | 主要为 Chat Completions-compatible | 默认/可选 thinking；interleaved/preserved thinking 要求完整、未修改地回传 `reasoning_content`；流式 tool call 有 `tool_stream` 和专有行为 | usage 与 reasoning 字段需以 Chat response 归一化；不能依赖 OpenAI encrypted reasoning | 需要 Chat wire adapter 和 preserved-reasoning transcript 支持 |
| Moonshot / Kimi | Chat Completions；新平台文档提供 thinking、tool use 和 `prompt_cache_key` | thinking 控制是否保留多轮 reasoning；coding agent 推荐稳定的 `prompt_cache_key`，tool loop 需验证 tool-call ID、部分输出和 thinking 回传 | cache key 是显式请求参数；usage 字段表面兼容但 cache hit 语义与 OpenAI 不同 | 需要 Chat adapter；应把 Codex session/task id 映射为 provider cache key |
| MiniMax | 自有 Text Chat Completion；OpenAI-compatible，同时官方推荐 Anthropic-compatible | M1/M2/M2.x 是 reasoning 模型；响应含 `reasoning_content`、tool calls，推荐 streaming；模型和 endpoint 命名不同 | usage 可能有 `completion_tokens_details.reasoning_tokens`、`base_resp` 等 provider 字段；总上下文和输出限制以 MiniMax 模型表为准 | 需要独立 MiniMax profile/adapter；不能只改 base URL |

官方参考： [DeepSeek Responses API](https://api-docs.deepseek.com/api/create-response/)、[DeepSeek thinking mode](https://api-docs.deepseek.com/guides/thinking_mode/)、[DeepSeek context cache](https://api-docs.deepseek.com/guides/kv_cache/)、[Qwen Responses API](https://help.aliyun.com/en/model-studio/qwen-api-via-openai-responses)、[Qwen function calling](https://help.aliyun.com/zh/model-studio/qwen-function-calling)、[Z.AI thinking mode](https://docs.z.ai/guides/capabilities/thinking-mode)、[Kimi chat API](https://platform.kimi.ai/docs/api/chat)、[MiniMax API overview](https://platform.minimaxi.com/docs/api-reference/api-overview) 和 [MiniMax text API](https://platform.minimax.io/docs/api-reference/text-post)。

### 2.1 必须抽象的协议层

建议新增内部 provider capability 与 adapter，而不是继续给 `ModelProviderInfo` 增加布尔字段：

```text
ProviderRuntime
  ├── WireAdapter: Responses | ChatCompletions | Anthropic | Native
  ├── ReasoningMode: Unsupported | Toggle | Effort | PreservedContent | PreservedBlocks
  ├── ToolMode: None | FunctionCalls | StreamingFunctionCalls | InterleavedCalls
  ├── PromptCache: None | ImplicitPrefix | ExplicitKey | ExplicitPrefixBlocks
  ├── UsageNormalizer
  └── Transport: HttpStreaming | ResponsesWebSocket | WebSocketUnsupported
```

每次请求前由 capability 生成 provider-specific request；每个流式事件先转换为内部事件，再交给 agent loop。这样 Codex 的 agent loop 只看到：assistant text、reasoning delta、tool call delta、tool result、usage、finish/error，而不直接依赖 `reasoning_content` 或 OpenAI response item。

### 2.2 Prompt cache

Codex 当前按 session/parent thread 计算 prompt cache key，并在 Responses request 中发送。迁移时应区分四种语义：

1. **显式 key**：Moonshot/Kimi 需要将稳定的 session/task id 作为 `prompt_cache_key`。
2. **隐式前缀缓存**：DeepSeek/Qwen 主要依靠重复前缀；Codex 必须保持 system/developer/tool schema 的稳定顺序，避免每轮无意义地改变前缀。
3. **Responses cached tokens**：只把 provider 返回的 cached input token 记为 observability，不假设它等于 OpenAI 的价格或 cache lifecycle。
4. **无缓存能力**：不能为了“统一”而伪造 cache hit；`cached_input_tokens = 0` 应与“provider 未提供该字段”区分。

建议内部 `Usage` 至少包含 `input_tokens`、`output_tokens`、`reasoning_tokens`、`cached_input_tokens`、`total_tokens` 和 `provider_raw`/`availability`，并允许每个字段为 unavailable。`TokenUsage` 对外保持兼容，但 analytics 和 UI 必须能显示“未知”而不是 0。

### 2.3 Reasoning 与 transcript

这是最高风险兼容点。DeepSeek、GLM、Kimi 等要求在 tool loop 中回传 reasoning content；而 Codex 当前主要处理 Responses reasoning item/encrypted content。不能把 reasoning 当成只展示给用户的可选文本后丢弃。

建议：

- 内部 transcript 保存 provider-neutral `ReasoningPart`，包含 opaque/provider-specific payload；
- 对要求 preserved reasoning 的 provider，下一轮按原始字节/结构回传，不重排、不摘要、不拼接修改；
- 对不允许或不需要回传的 provider，默认不把原始 reasoning 注入历史；
- UI/API 只暴露经过策略控制的 summary，默认不泄露原始 CoT；
- 对 stream reconnect/retry，要保存足够的已接收 reasoning/tool-call delta，避免重试请求缺少必要状态；
- provider capability 明确是否支持 temperature、top_p、parallel tool calls、structured output 和 max token 语义。

### 2.4 Tool call 兼容性

Codex 工具调用不是普通函数调用 demo，而是长循环：模型输出工具调用，执行器返回结果，再继续推理。接入验收必须覆盖：多个并行调用、同一轮多个 delta、半截 JSON、tool result 很大、reasoning + tool call 交错、断线重连、拒绝无效参数、tool call ID 稳定性。

建议为每家 provider 建立 contract test fixture，而不是只测 HTTP status 200。至少覆盖“无工具回答、单工具、连续两次工具、并行工具、工具错误后恢复、流式 usage、达到 token 上限”。

## 3. CI 全 Linux runner 改造

### 3.1 可迁移部分

可以优先迁移到 Linux：

- Rust fmt、clippy、argument-comment-lint、schema 生成检查；
- Linux native build/test、Bazel、nextest 和大多数 protocol/core 单测；
- macOS/Windows target 的 cross compilation、归档前的可执行文件存在性和资源打包检查；
- GitHub script、文档、TypeScript/Python SDK 检查；
- provider contract tests，使用 mock HTTP/SSE server，不依赖真实 provider key。

### 3.2 必须保留或替代的 native 阶段

- macOS binary signing/notarization 仍需 macOS 或服务端签名环境；
- Windows MSVC 编译、Windows sandbox helper、Windows path/ACL/ConPTY 和 native smoke test 仍需 Windows；
- Apple Silicon/Intel 的真实动态链接和 codesign entitlements 不能由 Linux cross linker 完整证明；
- 需要 KVM 的 Microsandbox integration test 应使用明确标记的 Linux runner，先验证 `/dev/kvm`，失败时不能静默降级为“通过”；
- 如果产品发布政策要求 native artifact provenance，Linux 只能负责构建，native runner 负责签名/验证。

### 3.3 建议的 workflow 形态

拆成四类 job：

1. `linux-check`: fmt、lint、unit/integration、Bazel、schema、provider mock tests。
2. `linux-cross-build`: 用固定 Linux image 和 target matrix 构建 macOS/Windows artifacts，生成未签名包。
3. `native-validation`: 少量 macOS/Windows smoke/signing jobs；只保留无法在 Linux 证明的内容。
4. `release-assembly`: Linux 汇总 artifact、校验 hash、生成 metadata；签名后的 artifact 只能由 native job 上传。

不要把 `runner.os` 当作 target；所有脚本应显式使用 `target` 和 cross toolchain。对 `path`、shell、压缩工具、证书和权限的差异增加 Linux fixtures。第一阶段只改 CI 组织和矩阵，不同时修改业务代码。

## 4. Microsandbox 作为工具执行沙箱

Microsandbox 官方项目定位为本地优先的 microVM runtime，使用 OCI image、独立 guest kernel、volume/command workflow，并提供 Rust/Python/TypeScript/Go SDK；官方资料同时强调当前仍处于 beta，Linux 依赖 KVM，macOS 主要要求 Apple Silicon，Windows 支持处于预览状态。参考：[官方 GitHub](https://github.com/superradcompany/microsandbox)、[SDK overview](https://github.com/superradcompany/microsandbox/blob/main/docs/sdk/overview.mdx)、[官方隔离说明](https://microsandbox.dev/features/isolation)。

### 4.1 集成边界

建议新增独立 crate，例如 `codex-microsandbox`，实现现有 exec backend 所需的窄接口：

```text
create(task_scope, workspace, policy) -> SandboxHandle
run(handle, argv, env, stdin) -> streamed stdout/stderr + exit status
interrupt(handle)
terminate(handle)
collect_artifacts(handle)
```

Core 只负责工具调用和生命周期，不直接依赖 Microsandbox 的具体 CLI 输出。backend 负责：

- 将 workspace 以受控 volume 挂载，默认不挂载 home、凭证目录和宿主 socket；
- 将 cwd、argv、环境变量、stdin/stdout/stderr、退出码映射到 Codex exec protocol；
- 为每个 task 或 session 选择 ephemeral/persistent sandbox；
- 将 network policy 转换为 Microsandbox 的 egress allowlist；
- 通过 gateway 使用 credential placeholder，真实 credential 不进入 guest；
- 对 image digest、runtime version、CPU/memory/PIDs/time limit 做审计记录。

### 4.2 不应直接替代现有 sandbox

Microsandbox 的 microVM 隔离强于仅 syscall/filesystem policy，但启动、镜像拉取、KVM、workspace 同步和调试成本更高。建议保留：

- 普通只读/工作区写入任务：现有本地 OS sandbox；
- 高风险不可信代码、第三方依赖、浏览器/脚本执行：Microsandbox；
- 远程 Cowork 执行：remote exec 或托管 Microsandbox；
- 无虚拟化能力环境：明确降级策略，默认拒绝高风险工具，而不是悄悄退回 host execution。

### 4.3 验收重点

必须测试 guest 无法读取 host 文件、环境变量和 Unix socket；workspace 写入边界；sandbox 删除后无残留；网络 allow/deny；凭证 placeholder 不可还原；超时/取消/断线；长输出截断；非零退出；TTY 和非 TTY；并发 sandbox 数量及资源上限。

## 5. 出站网关、header 注入与 credential 隔离

### 5.1 推荐方案：扩展现有 network-proxy

不建议另起一个与 `codex-network-proxy` 平行的代理。推荐将其明确分层：

```text
Agent / Tool process
        │ HTTP(S)_PROXY / ALL_PROXY
        ▼
Egress Gateway
  1. identify source scope (session/tool/sandbox)
  2. evaluate destination + method + path
  3. resolve credential policy
  4. inject short-lived Authorization/Cookie/header upstream
  5. proxy/MITM request
  6. strip Set-Cookie and sensitive response headers
  7. redact audit logs
        ▼
Provider / external service
```

credential policy 应是结构化配置，不允许任意 header 注入：

```toml
[[permissions.workspace.network.credentials]]
name = "github_read"
destinations = ["api.github.com"]
methods = ["GET"]
path_prefixes = ["/repos/owned-org/"]
source = "credential_broker.github"
inject = ["authorization"]
```

规则原则：deny 优先；destination、method、path、source scope 全部匹配才注入；credential 只存在 proxy memory 或 OS secret store；不进入 agent env、sandbox filesystem、命令行、请求日志、错误 body 和 trace attribute；token 轮换/撤销不要求重启 agent。

### 5.2 入站响应清理

代理返回给 agent/tool 前至少清除：

- `Set-Cookie`、`Set-Cookie2`；
- `Authorization`、`Proxy-Authenticate`、内部 session/auth header；
- 可能包含真实 credential 的 provider-specific header；
- 对受控场景可选清理 `Location` 中的 userinfo/query secret。

注意：不能无条件删除所有 cookie，因为部分公开网站工具需要会话 cookie。正确方式是“默认不保存/不转发上游 cookie；只有显式声明的 destination-scoped、非敏感 cookie jar 才允许保留”，并将 cookie jar 与 tool/session 隔离。

### 5.3 TLS、绕过和威胁边界

- HTTPS CONNECT 隧道下无法修改 HTTP header，必须对需要 header 注入/响应剥离的 host 启用 MITM；guest/tool 必须信任代理 CA，CA 私钥只能在 proxy 内存。
- 不支持 MITM 的 TLS client、QUIC/HTTP3、原生 TCP、DNS-over-HTTPS、直连 IP 和自行实现 socket 的工具可能绕过 HTTP proxy；sandbox 层必须同时限制网络能力。
- DNS rebinding 不能只靠域名 allowlist 解决；连接建立时还要检查解析地址，并在更低层（防火墙/VPC/VM egress）再次执行 private/link-local/metadata block。
- gateway 不应成为任意内网转发器；监听默认 loopback，Unix socket 代理必须显式 allowlist。
- 不记录完整 URL、query、cookie、Authorization 和响应 body；错误信息也要做 secret redaction。

### 5.4 当前代码已考虑到什么

当前 `network-proxy/README.md` 和实现已经考虑 allowlist、deny precedence、MITM hook、`strip_auth`、credential broker、local binding 和 OTEL 脱敏边界，这是最好的落点。尚需补齐：通用 provider credential schema、response header scrubber、source scope 绑定、credential injection 的集成测试，以及非 HTTP 绕过的明确产品策略。

## 6. OpenAI 强依赖改造清单

### 6.1 第一类：模型请求主链路

涉及 `codex-rs/core/src/client.rs`、`model-provider-info`、`protocol/openai_models.rs`、responses metadata 和 compact remote：

- 把 Responses request/event 转成 provider-neutral request/event；
- 将 OpenAI-only `reasoning.encrypted_content`、Responses lite header、WebSocket beta header、OpenAI timing header 变为 capability-gated；
- `/responses/compact` 只作为 OpenAI optimization，provider 不支持时走本地 compact；
- 重新定义 usage、reasoning summary 和 retry 的 provider contract；
- 不向非 OpenAI endpoint 发送 ChatGPT account、installation、subagent 或内部 header。

### 6.2 第二类：认证与账户

`codex-rs/chatgpt`、`login`、`backend-client`、`cloud-tasks`、`account` 和 auth manager 中存在 ChatGPT token、account id、`chatgpt.com/backend-api`、WHAM path 及 usage/profile/settings 语义。

建议抽象：

- `ModelAuth`: API key、OAuth/token、command-backed、cloud identity；
- `AccountFeatures`: usage/profile/billing/remote task 是否可用；
- `ProviderControlPlane`: provider-specific model catalog、remote jobs、plugin catalog；
- `CloudTaskBackend`: ChatGPT backend 与未来 Cowork backend 的独立实现。

非 OpenAI provider 不应被迫登录 ChatGPT；缺少 usage endpoint 时返回 provider usage 或 unavailable，不显示 ChatGPT subscription 文案和链接。

### 6.3 第三类：remote control / remote exec

`app-server-transport/src/transport/remote_control`、`app-server-daemon/src/remote_control_client.rs`、`exec-server`、`core-plugins/src/remote*` 和相关测试包含明显的 first-party remote assumptions，包括 enrollment、websocket refresh、registry authorization、ChatGPT cloud 任务和插件目录。

应把它们分成两个层次：

1. **通用 session transport**：app-server protocol、thread/event、exec-server、noise/transport、pairing 和 device lifecycle，不依赖 OpenAI。
2. **服务端控制面 adapter**：ChatGPT remote control、Cowork control plane、企业自建 gateway，各自提供 enrollment、authorization、catalog、task APIs。

Remote control 的认证不能从“模型 provider API key”推导。模型 key、gateway credential、device pairing credential 和 cloud session token 要有不同 scope、存储和撤销路径。

## 7. 仅 Cowork（Web/Electron UI）时的裁剪建议

### 7.1 保留为核心运行时

- `codex-rs/core`：agent loop、turn、context、tool orchestration；后续需继续拆 provider/context，避免 core 继续膨胀；
- `app-server-protocol`、`app-server-transport`：Web/Electron 的稳定 API 和事件流；
- `app-server`/`app-server-daemon`：按部署形态保留一个明确入口；
- `exec-server`、`exec-server-protocol`、`linux-sandbox`/`sandboxing`、`network-proxy`；
- `model-provider`、`model-provider-info` 和新的 provider adapters；
- `config`、`protocol`、`thread-store`、`history`、`secrets`、`file-system`、MCP/connector runtime；
- telemetry、audit 和 crash reporting，但需要移除 ChatGPT 专属字段或改成可选 integration。

### 7.2 可从 Cowork binary 移除或独立部署

- `codex-rs/tui` 及其 keymap、rendering、ratatui snapshot、terminal onboarding；
- CLI-only command parsing、interactive login UI、terminal raw mode 和 shell completion；
- TUI 专用的 `chatwidget`、bottom pane、footer、terminal detection；
- `codex-rs/chatgpt` 中只服务于 ChatGPT UI/账户/云任务的部分；
- 仅为本地 CLI 提供的 `codex` binary 入口，改为 app-server daemon 或 library embedding；
- macOS/Windows 本地 CLI sandbox helper，前提是 Cowork 执行统一在 Linux remote/Microsandbox；若 Electron 支持本地执行，则不能删除对应平台 backend。

### 7.3 不建议第一阶段删除

- `protocol` 中的 legacy event mapping：Web/Electron 迁移期间可能仍有客户端依赖；
- `remote control` 全部代码：如果 Cowork 的产品本身需要远程接管，应该先抽象 backend，再删除 ChatGPT implementation；
- MCP、connector、plugin runtime：Cowork 通常比 TUI 更依赖这些能力；
- sandboxing/network-proxy：去掉 TUI 不等于降低 agent 执行风险；
- `responses-api-proxy`：如果 Cowork 仍需给兼容客户端提供统一 Responses endpoint，应保留为独立服务而非塞回 UI binary。

裁剪方式应优先使用 Cargo feature 或独立 binary target，而不是大面积删除 crate。建议产出 `codex-cowork-server`，依赖 app-server/core/exec/provider/network；`codex-cli` 和 `codex-tui` 作为另一组可选产品入口。

## 8. 推荐实施顺序与验收标准

### Phase 0：建立兼容性基线

- 固化当前 OpenAI Responses contract tests；
- 为每个 provider 建立 mock SSE/JSON fixtures；
- 定义 provider-neutral request/event/usage/reasoning types；
- 给所有 OpenAI-only header 和 endpoint 加 capability gate；
- 记录每个 workflow 的 target、runner、native-only reason。

验收：现有 OpenAI tests 不回归；五家 provider 的差异矩阵能映射到明确 capability，而不是散落条件判断。

### Phase 1：Provider adapters

- 先实现 Chat Completions adapter，再实现 Responses adapter；
- 先接 DeepSeek Responses 与 Qwen Responses-compatible 模型；
- 再接 GLM、Kimi、MiniMax Chat path；
- 完成 preserved reasoning、stream tool call、usage normalization、cache key 和 retry tests；
- provider 不支持的能力必须可观测地降级或拒绝。

验收：每家 provider 至少通过单工具、连续工具、流式断线、reasoning 回传、usage 和 context limit contract test。

### Phase 2：Egress gateway 与 Microsandbox

- 在 `codex-network-proxy` 增加 response scrubber 和 destination-scoped credential broker；
- 将 source scope 从 exec/sandbox 传到 proxy policy；
- 接入 `codex-microsandbox` backend；
- 先做 Linux KVM integration test，再决定是否支持其他平台；
- 验证 host credential、cookie、Set-Cookie、proxy CA 和直连绕过。

验收：恶意工具无法从环境、文件、错误或响应 header 获得真实 credential；允许的上游请求仍能成功；所有拒绝都有审计原因但不含 secret。

### Phase 3：Cowork runtime 与 Linux CI

- 生成不依赖 TUI 的 Cowork app-server binary；
- 把 ChatGPT control-plane implementation 与通用 remote transport 解耦；
- 将 CI check/cross-build 迁至 Linux；
- 保留必要 native signing/validation jobs；
- 删除或 feature-gate TUI/CLI-only dependencies。

验收：Web/Electron 可通过 app-server 完成创建 thread、turn、tool approval、exec streaming、文件变更、取消和恢复；Linux CI 对所有目标产物完成构建；native job 仅覆盖不可交叉验证的证明。

## 9. 主要风险与决策建议

| 风险 | 影响 | 建议 |
|---|---|---|
| 把 OpenAI-compatible 等同于 Responses-compatible | tool/reasoning loop 在第一轮后失败 | 以 capability + adapter 为准，禁止只改 URL |
| 丢弃 preserved reasoning | DeepSeek/GLM/Kimi 在 tool loop 返回 400 或质量下降 | transcript 保存 opaque reasoning，并按 provider 原样回传 |
| 将未知 usage 当作 0 | 预算、UI、analytics 误导 | 字段支持 unavailable，provider raw usage 可审计 |
| 只做 HTTP proxy | 工具通过 TCP、QUIC、DNS 或本地 socket 泄露 | proxy + sandbox egress 双层控制 |
| MITM 过度扩大信任面 | proxy CA 或上游 credential 泄露 | 仅对显式 host 开启，CA 私钥内存化，response scrubber 默认开启 |
| 全部 CI Linux 后误以为跨平台验证完成 | 发布后 native 崩溃或签名失败 | native validation 保留为窄 job |
| Microsandbox 失败后静默退回 host | 安全边界被绕过 | 高风险 profile 失败即拒绝，降级需显式策略 |
| 一次性删除 remote/TUI/ChatGPT 代码 | Cowork 客户端或远程执行功能断裂 | 先拆 feature/backend，再按依赖图删除 |

最终建议是“先建立边界，再迁移实现”：provider adapter、egress gateway、sandbox backend、remote control backend 和 Cowork app-server 都应有独立接口及 contract tests。这样既能接入五家模型，也能让后续新增 provider 或自建部署不再继续扩大 `codex-core` 与 OpenAI-specific code 的耦合。
