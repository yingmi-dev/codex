# Codex 架构文档

> 本文档描述 Codex CLI 的整体架构、模块划分及关键设计决策。
> 适用于开发者快速了解项目结构和模块间关系。

## 目录

- [1. 总体架构](#1-总体架构)
- [2. 分层架构](#2-分层架构)
- [3. 核心模块详解](#3-核心模块详解)
  - [3.1 CLI 前端](#31-cli-前端-codex-rscli)
  - [3.2 TUI 终端界面](#32-tui-终端界面-codex-rstui)
  - [3.3 App-Server JSON-RPC 服务](#33-app-server-json-rpc-服务-codex-rsapp-server)
  - [3.4 Core 核心引擎](#34-core-核心引擎-codex-rscore)
  - [3.5 Protocol 通信协议](#35-protocol-通信协议-codex-rsprotocol)
  - [3.6 Exec-Server 远程执行服务](#36-exec-server-远程执行服务-codex-rsexec-server)
  - [3.7 Sandboxing 沙箱](#37-sandboxing-沙箱-codex-rssandboxing)
  - [3.8 Tools 工具系统](#38-tools-工具系统-codex-rstools)
  - [3.9 Codex-MCP MCP 管理](#39-codex-mcp-mcp-管理-codex-rscodex-mcp)
  - [3.10 Extension 扩展系统](#310-extension-扩展系统-codex-rsext)
  - [3.11 Config 配置系统](#311-config-配置系统-codex-rsconfig)
  - [3.12 State 状态存储](#312-state-状态存储-codex-rsstate)
  - [3.13 Rollout 会话记录](#313-rollout-会话记录-codex-rsrollout)
  - [3.14 Login 认证](#314-login-认证-codex-rslogin)
  - [3.15 Plugin 插件系统](#315-plugin-插件系统-codex-rsplugin)
- [4. 关键数据流](#4-关键数据流)
- [5. 跨平台支持](#5-跨平台支持)
- [6. 构建与测试](#6-构建与测试)
- [7. 模块依赖关系图](#7-模块依赖关系图)

---

## 1. 总体架构

Codex 是一个用 Rust 编写的 AI 编码助手平台，采用 Cargo Workspace 组织，包含 **130+ 个 crate**，所有 crate 名称以 `codex-` 为前缀。整个系统围绕 **ThreadManager** 和 **CodexThread** 为核心，管理 AI 与用户之间的交互轮次（turn），通过沙箱化的执行环境安全地运行命令和操作文件系统。

### 核心设计原则

- **安全优先**：所有命令执行均在沙箱中运行，通过网络和文件系统隔离防止未授权操作
- **多前端架构**：同一核心引擎支持 CLI、TUI、IDE 插件（通过 app-server）等多种前端
- **可扩展**：通过 Extension 系统和 Plugin 系统支持功能扩展
- **跨平台**：支持 macOS、Linux 和 Windows
- **协议驱动**：各组件间通过明确定义的协议（Protocol）通信

---

## 2. 分层架构

```
┌─────────────────────────────────────────────────────────┐
│                   Frontend Layer                         │
│   CLI (codex-cli)  │  TUI (codex-tui)  │  IDE/Remote    │
└────────────────────┬────────────────────┴────────────────┘
                     │ JSON-RPC / IPC
┌────────────────────▼────────────────────────────────────┐
│               App-Server Layer                           │
│   (codex-app-server) — JSON-RPC 服务, 多连接管理       │
└────────────────────┬────────────────────────────────────┘
                     │ Op / Event (SQ/EQ)
┌────────────────────▼────────────────────────────────────┐
│                 Core Engine Layer                        │
│   (codex-core) — ThreadManager, Session, Client         │
│   Compact, Agent, Guardian, Context, ExecPolicy        │
└──────┬──────────┬──────────┬───────────┬────────────────┘
       │          │          │           │
┌──────▼──┐ ┌────▼────┐ ┌───▼────┐ ┌────▼─────────┐
│ Exec    │ │ MCP     │ │ Tools  │ │ Extension     │
│ Server  │ │ Manager │ │ System │ │ System        │
└──────┬──┘ └─────────┘ └────────┘ └──────────────┘
       │
┌──────▼──────────────────────────────────────────────────┐
│              Sandbox / Platform Layer                    │
│   Seatbelt (macOS) │ Landlock/bwrap (Linux) │ Win (Win) │
└──────────────────────────────────────────────────────────┘
```

---

## 3. 核心模块详解

### 3.1 CLI 前端 (`codex-rs/cli`)

CLI 是用户进入 Codex 的主要入口，基于 `clap` 解析命令行参数。

**入口**：`codex-rs/cli/src/main.rs`（约 4800 行）

**主要子命令**：
- 默认（无子命令）：启动 TUI 交互界面
- `exec`：非交互式执行模式，用于脚本和自动化
- `mcp`：MCP 服务器管理
- `login` / `logout`：认证管理
- `doctor`：系统诊断（磁盘、网络、沙箱、安全等）
- `plugin`：插件管理
- `sandbox`：沙箱配置和测试
- `state-db`：状态数据库恢复
- `marketplace`：插件市场
- `remote-control`：远程控制管理

**关键设计**：
- 使用 `arg0` 机制根据可执行文件名分发不同的运行模式（如 `codex-linux-sandbox`）
- 支持 `-c key=value` 格式的配置覆盖
- 支持 `--profile` 加载 v2 配置文件
- 平台特定命令（`app_cmd`、`desktop_app`、`sandbox_setup`、`wsl_paths`）通过条件编译启用

### 3.2 TUI 终端界面 (`codex-rs/tui`)

基于 `ratatui` 构建的终端用户界面，是 Codex 最主要的前端。

**关键文件**：
- `lib.rs`（约 3400 行）：TUI 库入口，负责配置加载、认证、会话恢复/创建
- `app.rs`：主应用状态机，协调所有 UI 组件
- `app_server_session.rs`：与 app-server 的会话管理
- `bottom_pane/`：底部交互区域（聊天输入框、审批覆盖层、命令弹窗等）
- `chatwidget.rs`：聊天界面核心组件

**通信方式**：
- 通过 `AppServerClient` 与 app-server 通信（支持 in-process 和 remote 两种模式）
- 接收 `ServerNotification` 事件流并渲染
- 发送 `thread/start`、`thread/interrupt` 等 JSON-RPC 请求

**特性**：
- 快照测试（insta）：所有 UI 变更必须有对应的快照覆盖
- 文本换行：使用 `textwrap` 和自定义 `wrapping.rs` 辅助函数
- 历史搜索：通过 `chat_composer_history` 模块提供

### 3.3 App-Server JSON-RPC 服务 (`codex-rs/app-server`)

App-Server 是 Codex 的核心服务层，为 CLI、TUI、IDE 插件等提供统一的 JSON-RPC API。

**入口**：`codex-rs/app-server/src/lib.rs`

**传输层**（`Transport` 枚举）：
- **Stdio**：单客户端模式，用于 CLI/TUI 进程内通信
- **UnixSocket**：Unix 域套接字，支持多客户端
- **WebSocket**：WebSocket 监听，支持远程客户端
- **Off**：不启动本地传输，仅依赖 remote control

**核心组件**：
- `MessageProcessor`：处理入站 JSON-RPC 消息，分发到各请求处理器
- `ConfigManager`：配置管理，支持热重载
- `EnvironmentManager`：执行环境管理（本地/远程）
- `OutgoingMessageSender`：出站消息路由
- `ConnectionCleanupTasks`：连接关闭后的异步清理

**关键设计**：
- 双循环架构：处理器循环 + 出站路由循环，通过 `OutboundControlEvent` 协调
- 优雅关闭：支持信号驱动的 graceful restart drain，等待运行中的 assistant turn 完成
- Remote Control：支持远程控制连接，受 managed requirements 策略约束
- Code Mode：支持本地和远程（WebSocket/gRPC）的 code mode 会话提供者

**协议版本**：
- v1：遗留协议，不再新增 API
- v2：所有新 API 开发的目标，使用 `#[ts(export_to = "v2/")]` 导出 TypeScript 类型

### 3.4 Core 核心引擎 (`codex-rs/core`)

Codex 的核心业务逻辑库，是最庞大也最核心的 crate。**AGENTS.md 明确指出应避免向 codex-core 添加新代码**，鼓励提取新 crate。

**入口**：`codex-rs/core/src/lib.rs`

#### 关键组件

**ThreadManager** (`thread_manager.rs`, ~2200 行)：
- 顶层管理器，管理多个 `CodexThread` 的生命周期
- 支持 fork 子 agent、创建/恢复/截断 thread
- 集成 `AgentGraphStore`（多 agent 关系图）
- 集成 `ThreadStore`（线程持久化）

**CodexThread** (`codex_thread.rs`, ~800 行)：
- 单个对话线程，封装 `Session`
- 管理 thread 级配置（模型、审批策略、权限配置等）
- 支持后台终端、elicitation 注册

**Session** (`session/` 模块)：
- 对话会话引擎，驱动模型 API 调用和工具执行
- 使用 SQ（Submission Queue）/ EQ（Event Queue）模式
- 管理 context、compact（上下文压缩）、retry

**Client / ModelClient** (`client.rs`, ~2500 行)：
- 与 OpenAI Responses API 通信
- 处理 SSE 流式响应
- 请求元数据（installation ID、routing hint、turn metadata）

**Agent 系统** (`agent/` 模块)：
- `AgentControl`：agent 生命周期控制
- `AgentRegistry`：agent 注册表
- `AgentRole`：agent 角色定义
- 支持 spawn、residency、legacy 等控制模式

**Compact** (`compact.rs`, ~800 行)：
- 上下文压缩系统，当对话超过 token 限制时触发
- 支持本地和远程（compact_remote_v2）压缩
- Token 预算管理

**ExecPolicy** (`exec_policy.rs`, ~1100 行)：
- 命令执行策略，使用 Starlark DSL 定义规则
- 在命令执行前进行安全检查和策略评估

**Guardian** (`guardian.rs` + `ext/guardian-v2/`)：
- 安全评估系统，评估命令风险
- 支持异步评分和子 agent 评审

**Context** (`context/` 模块)：
- 模型上下文管理，实现 `ContextualUserFragment` trait
- 所有注入模型上下文的片段必须有界大小和硬上限
- 禁止历史重写，必须增量构建

**MCP** (`mcp.rs`, `mcp_tool_call.rs` 等)：
- MCP 工具调用处理
- MCP 工具审批模板
- MCP 工具暴露控制

### 3.5 Protocol 通信协议 (`codex-rs/protocol`)

定义了 Codex 内部各组件间通信的核心协议类型。

**入口**：`codex-rs/protocol/src/lib.rs` + `protocol.rs`（~6000 行）

**核心概念**：
- **SQ/EQ 模式**：Submission Queue（用户提交）和 Event Queue（事件流）异步通信
- `Op`：用户/系统向 agent 提交的操作
- `Event` / `EventMsg`：agent 产生的事件通知
- `ResponseItem`：模型响应的结构化数据

**主要模块**：
- `protocol`：Op、Event、EventMsg、SessionMeta 等核心类型
- `approvals`：审批请求（exec、apply_patch、guardian assessment 等）
- `config_types`：配置类型（AskForApproval、Personality、ReasoningEffort 等）
- `models`：数据模型（ResponseItem、ContentItem、PermissionProfile 等）
- `permissions`：文件系统权限（SandboxPolicy、FileSystemAccessMode 等）
- `mcp`：MCP 相关类型（CallToolResult、McpServer 等）
- `turn_input`：轮次输入（TurnInputRequest、TurnInputSubmission 等）
- `dynamic_tools`：动态工具类型
- `network_policy`：网络策略

### 3.6 Exec-Server 远程执行服务 (`codex-rs/exec-server`)

Exec-Server 是 Codex 的命令执行和文件系统操作服务，支持本地和远程执行。

**入口**：`codex-rs/exec-server/src/lib.rs`

**核心功能**：
- **进程执行**：沙箱化的进程 spawn、信号管理、输出流
- **文件系统操作**：文件读写、目录遍历、元数据查询、文件复制/删除
- **网络代理**：HTTP 请求代理，受网络策略约束
- **Shell 快照**：捕获和恢复 shell 环境状态（仅 Unix）

**架构**：
- `Environment` / `EnvironmentManager`：执行环境管理（本地/远程）
- `ExecServerClient`：exec-server 客户端
- `NoiseChannel`：基于 Noise Protocol 的加密通道（用于远程连接）
- `RemoteEnvironmentConfig`：远程环境配置

**环境类型**：
- `LOCAL_ENVIRONMENT_ID`：本地执行环境
- `REMOTE_ENVIRONMENT_ID`：远程执行环境

**能力发现**：
- `CapabilityDiscovery`：发现远程环境的可用能力
- `ExecutorCapabilityDiscoveryCache`：能力发现缓存

### 3.7 Sandboxing 沙箱 (`codex-rs/sandboxing`)

跨平台的沙箱实现，确保所有命令在受限环境中执行。

**平台实现**：
- **macOS**：Seatbelt（`/usr/bin/sandbox-exec`），通过 `seatbelt` 模块
- **Linux**：Landlock + Bubblewrap（bwrap），通过 `landlock` 和 `bwrap` 模块
- **Windows**：Windows Restricted Token Sandbox，通过 `windows` 模块

**核心组件**：
- `SandboxManager`：沙箱管理器，处理命令转换和执行
- `SandboxPolicy`：沙箱策略，控制可读/可写根目录和网络访问
- `SandboxViolation`：沙箱违规检测和记录（文件系统/网络）
- `SpawnRequest` / `spawn_process`：进程生成

**策略转换**：
- `SandboxTransformRequest`：将命令转换为沙箱化的执行请求
- `compatibility_sandbox_policy_for_permission_profile`：权限配置兼容性

### 3.8 Tools 工具系统 (`codex-rs/tools`)

定义了模型可调用的工具（function calling）的抽象。

**入口**：`codex-rs/tools/src/lib.rs`

**核心类型**：
- `ToolSpec`：工具规范（名称、描述、参数 schema）
- `ToolDefinition`：工具定义
- `ToolExecutor`：工具执行器 trait
- `ToolCall`：工具调用上下文
- `ToolOutput` / `JsonToolOutput`：工具输出
- `ToolExposure` / `ToolExposures`：工具暴露控制
- `ToolEnvironment`：工具执行环境
- `ConversationHistory`：对话历史

**Responses API 集成**：
- `create_tools_json_for_responses_api`：生成 Responses API 工具 JSON
- `ResponsesApiTool` / `LoadableToolSpec`：Responses API 工具类型
- `mcp_tool_to_responses_api_tool`：MCP 工具转换

**工具发现**：
- `DiscoverableTool` / `DiscoverablePluginInfo`：可发现工具和插件信息
- `ToolSearch`：工具搜索
- `REQUEST_PLUGIN_INSTALL_TOOL_NAME`：插件安装请求工具

### 3.9 Codex-MCP MCP 管理 (`codex-rs/codex-mcp`)

Model Context Protocol 的连接管理、工具目录和运行时。

**入口**：`codex-rs/codex-mcp/src/lib.rs`

**核心组件**：
- `McpManager`：MCP 服务器管理器
- `McpRuntime` / `McpRuntimeContext`：MCP 运行时
- `McpBinding` / `PreparedMcpCall`：MCP 绑定和预调用
- `McpCatalogBuilder` / `ResolvedMcpCatalog`：MCP 目录
- `McpResourceClient`：MCP 资源客户端
- `McpToolCatalogCache`：工具目录缓存
- `ConnectionManager`：MCP 连接管理器（`codex-rs/codex-mcp/src/connection_manager.rs`）

**Codex Apps**：
- `CodexAppsToolsCache`：Codex Apps 工具缓存（基于 ConnectorRuntimeManager）
- `codex_apps_mcp_server_config`：Codex Apps MCP 服务器配置
- OAuth 认证支持（`McpOAuthLoginSupport`、`resolve_oauth_scopes`）

**设计要点**：
- AGENTS.md 指出：MCP 工具调用应优先通过 `mcp_connection_manager.rs` 处理
- 避免不必要地调用 `reset_client_session`，让增量检查逻辑决定是否重用

### 3.10 Extension 扩展系统 (`codex-rs/ext`)

Extension API 定义了扩展点的 trait 接口，各内置扩展实现这些 trait 来注入行为。

**Extension API** (`codex-rs/ext/extension-api`)：
- `ExtensionRegistry` / `ExtensionRegistryBuilder`：扩展注册表
- `ExtensionData` / `ExtensionDataInit`：扩展数据

**Contributor Traits**（扩展点）：
- `ThreadLifecycleContributor`：线程生命周期（start、stop、resume、idle）
- `TurnLifecycleContributor`：轮次生命周期
- `ToolLifecycleContributor`：工具生命周期（start、finish）
- `ToolContributor`：工具贡献
- `ContextContributor`：上下文贡献
- `ConfigContributor`：配置贡献
- `ApprovalReviewContributor`：审批评审
- `McpServerContributor`：MCP 服务器贡献
- `TokenUsageContributor`：Token 使用
- `ThreadOriginator`：线程发起
- `SkillInvocationContributor`：技能调用
- `PromptFragment` / `PromptSlot`：提示词片段

**内置扩展**：

| 扩展 | 路径 | 职责 |
|------|------|------|
| `agent` | `ext/agent` | Agent 生成和运行（`AgentRunner`） |
| `guardian-v2` | `ext/guardian-v2` | 安全评估和异步评分 |
| `mcp` | `ext/mcp` | MCP 工具集成 |
| `memories` | `ext/memories` | 记忆管理 |
| `skills` | `ext/skills` | 技能加载和调用 |
| `web-search` | `ext/web-search` | 网络搜索 |
| `image-generation` | `ext/image-generation` | 图像生成 |
| `goal` | `ext/goal` | 目标管理 |
| `queue` | `ext/queue` | 任务队列 |
| `git-attribution` | `ext/git-attribution` | Git 归属 |
| `history-notes` | `ext/history-notes` | 历史笔记 |
| `connectors` | `ext/connectors` | 连接器管理 |
| `items` | `ext/items` | 扩展项（图像生成失败等） |

### 3.11 Config 配置系统 (`codex-rs/config`)

配置加载、分层叠加、合并和验证系统。

**入口**：`codex-rs/config/src/lib.rs`

**配置分层**：
- `ConfigLayerStack`：配置层栈，支持多来源叠加
- `ConfigLayerSource`：配置来源（User、Project、Managed、CLI override 等）
- 配置文件为 TOML 格式（`config.toml`）

**配置类型**：
- `ConfigToml`：用户配置 TOML 结构
- `PermissionsToml`：权限配置
- `ProfileToml`：v2 Profile 配置
- `HooksToml`：Hook 配置
- `SkillsConfig`：技能配置
- `ThreadConfig`：线程配置（支持远程加载）

**关键概念**：
- `LoaderOverrides`：加载器覆盖（忽略用户配置、自定义路径等）
- `CloudConfigBundle` / `CloudConfigFragment`：云端配置包和片段
- `ConfigRequirements`：管理要求（允许/禁止 remote control、sandbox mode 等）
- `Constrained` / `Constraint`：约束系统，确保配置符合管理策略

**Profile v2**：
- `ProfileV2Name`：v2 Profile 名称
- 支持 `$CODEX_HOME/<name>.config.toml` 层叠
- `resolve_profile_v2_config_path`：解析 Profile 路径

### 3.12 State 状态存储 (`codex-rs/state`)

基于 SQLite 的状态存储系统，从 JSONL rollout 中提取元数据并镜像到本地数据库。

**入口**：`codex-rs/state/src/lib.rs`

**核心组件**：
- `StateRuntime`：首选入口，拥有配置和指标
- `SqliteConfig`：SQLite 配置
- `log_db`：日志数据库（tracing layer）

**数据模型**：
- `ThreadMetadata`：线程元数据
- `ThreadSection`：线程分节
- `Project` / `ProjectRoot`：项目
- `LogEntry` / `LogRow`：日志条目
- `GoalStore` / `MemoryStore`：目标和记忆存储
- `SqliteQueueStore`：SQLite 队列存储

**关键设计**：
- 使用 bundled SQLite（含 WAL-reset 修复）
- 支持 corruption recovery（自动备份损坏的数据库并重建）
- 回填系统：从 JSONL rollout 回填元数据到 SQLite
- 指标系统：DB 初始化、回填、fallback 等指标

### 3.13 Rollout 会话记录 (`codex-rs/rollout`)

会话持久化和发现系统，记录 Codex 的对话历史。

**入口**：`codex-rs/rollout/src/lib.rs`

**核心功能**：
- `RolloutRecorder`：会话记录器，将 `RolloutItem` 追加到 JSONL 文件
- `RolloutLineReader`：rollout 行读取器（支持压缩格式）
- `ReverseJsonlScanner`：反向 JSONL 扫描器
- `RolloutReferenceIndex`：rollout 引用索引
- `ModelContextScan`：模型上下文扫描

**文件组织**：
- `sessions/`：活跃会话
- `archived_sessions/`：归档会话
- 支持压缩（zstd）和非压缩格式

**State DB 集成**：
- `state_db::StateDbHandle`：状态数据库句柄
- 从 rollout 回填元数据到 SQLite

### 3.14 Login 认证 (`codex-rs/login`)

认证管理，支持多种认证方式。

**认证方式**：
- ChatGPT 登录（OAuth device code flow）
- API Key
- Access Token（stdin）
- Workload Identity
- AWS Auth（`codex-rs/aws-auth`）

**核心组件**：
- `AuthManager`：认证管理器（共享实例）
- `CodexAuth`：Codex 认证 trait
- `AuthConfig`：认证配置
- `AuthMode`：认证模式

### 3.15 Plugin 插件系统 (`codex-rs/plugin`)

插件包模型、来源提供者和标识符。

**入口**：`codex-rs/plugin/src/lib.rs`

**核心类型**：
- `PluginId`：插件标识符
- `PluginProvider` / `ResolvedPlugin`：插件来源和解析
- `LoadedPlugin` / `PluginLoadOutcome`：已加载插件
- `PluginCapabilitySummary`：插件能力摘要
- `PluginHookSource`：插件 Hook 来源
- `AppConnectorId` / `AppDeclaration`：应用连接器

---

## 4. 关键数据流

### 4.1 用户交互流程

```
用户输入
  │
  ▼
TUI/CLI ──JSON-RPC──▶ App-Server ──Op──▶ ThreadManager
                                                    │
                                                    ▼
                                                CodexThread
                                                    │
                                                    ▼
                                                 Session
                                                    │
                                          ┌─────────┼──────────┐
                                          ▼          ▼          ▼
                                     ModelClient  Tools    ExecPolicy
                                          │          │          │
                                          ▼          ▼          ▼
                                   OpenAI API   Exec-Server   Guardian
                                    (SSE)         │          Assessment
                                          │       ▼
                                          │   Sandbox
                                          │       │
                                          ▼       ▼
                                     Event ◀──────┘
                                          │
                                          ▼
                                    App-Server ──▶ TUI/CLI ──▶ 用户
```

### 4.2 工具调用流程

```
模型返回 function_call
      │
      ▼
  Session 解析
      │
      ├── 内置工具 (shell/apply_patch) ──▶ Exec-Server ──▶ Sandbox
      │
      ├── MCP 工具 ──▶ McpManager ──▶ MCP Server (stdio/sse/websocket)
      │
      └── Extension 工具 ──▶ Extension Registry ──▶ 对应扩展
      │
      ▼
  ToolOutput → 返回模型上下文
```

### 4.3 上下文压缩流程

```
对话接近 token 限制
      │
      ▼
  Compact 触发
      │
      ├── 本地压缩 (compact.rs)
      │     └── 保留最近 N 条消息 + 摘要
      │
      └── 远程压缩 (compact_remote_v2.rs)
            └── 调用模型 API 生成摘要
      │
      ▼
  替换模型上下文（不重写历史）
```

---

## 5. 跨平台支持

Codex 支持 macOS、Linux 和 Windows 三大平台，通过条件编译实现平台特定功能。

### 5.1 沙箱

| 平台 | 技术 | 模块 |
|------|------|------|
| macOS | Seatbelt (`/usr/bin/sandbox-exec`) | `sandboxing::seatbelt` |
| Linux | Landlock + Bubblewrap (bwrap) | `sandboxing::landlock`, `sandboxing::bwrap` |
| Windows | Windows Restricted Token | `sandboxing::windows`, `windows-sandbox-rs` |

### 5.2 远程执行测试

Codex 支持在不同于主机的操作系统上运行 app-server 和 exec-server，集成测试使用以下跳过宏：

- `skip_if_target_windows!`：Windows 目标行为
- `skip_if_host_windows!`：Windows 主机限制
- `skip_if_remote!`：仅本地测试
- `skip_if_no_remote_env!`：仅远程测试
- `skip_if_wine_exec!`：Wine 特定运行器

### 5.3 平台特定 crate

- `codex-rs/windows-sandbox-rs`：Windows 沙箱
- `codex-rs/linux-sandbox`：Linux 沙箱
- `codex-rs/bwrap`：Bubblewrap 封装
- `codex-rs/process-hardening`：进程加固（跨平台）

---

## 6. 构建与测试

### 6.1 构建系统

- **Cargo**：主要构建系统，`codex-rs/Cargo.toml` 为 workspace 根
- **Bazel**：也支持 Bazel 构建，`BUILD.bazel` 文件遍布各 crate
- **Just**：任务运行器，`justfile` 定义常用命令

### 6.2 常用命令

| 命令 | 用途 |
|------|------|
| `just fmt` | 格式化 Rust 代码 |
| `just fix -p <project>` | 修复 lint 问题（建议指定项目） |
| `just test -p <project>` | 运行特定项目测试 |
| `just test` | 运行完整测试套件 |
| `just bench` | 运行 benchmark（使用 divan） |
| `just write-config-schema` | 更新配置 schema |
| `just write-app-server-schema` | 更新 app-server schema |
| `just bazel-lock-update` | 刷新 Bazel lockfile |
| `just argument-comment-lint` | 参数注释 lint |

### 6.3 测试约定

- **集成测试优先**：agent 逻辑变更必须添加集成测试（`core/suite`）
- **快照测试**：TUI 使用 `insta` 进行快照测试
- **测试工具**：`TestCodexBuilder::build_with_auto_env()`、`TestAppServer::builder().build()`
- **SSE Mock**：使用 `responses::mount_sse_once` 等 helper 模拟 API 响应
- **断言**：使用 `pretty_assertions::assert_eq` 进行深比较
- **二进制解析**：使用 `codex_utils_cargo_bin::cargo_bin("...")`

### 6.4 代码规范

- Crate 名称前缀 `codex-`（如 `core` → `codex-core`）
- 格式化参数内联：`format!("{x}")` 而非 `format!("{}", x)`
- 避免 `unwrap()` / `expect()`（workspace lint: deny）
- 合并 if 语句（clippy::collapsible_if）
- 优先方法引用而非闭包（clippy::redundant_closure_for_method_calls）
- 模块目标 < 500 LoC，文件不超过 ~800 LoC
- 新增 trait 需有文档注释，优先 RPINIT（`fn foo(&self, ...) -> impl Future<...> + Send`）

---

## 7. 模块依赖关系图

```
                    ┌──────────┐
                    │ protocol │ (核心类型, 无依赖)
                    └────┬─────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
     ┌────▼────┐  ┌──────▼──────┐  ┌───▼────┐
     │ config  │  │  tools      │  │ state  │
     └────┬────┘  └──────┬──────┘  └───┬────┘
          │              │              │
          └──────────────┼──────────────┘
                         │
                    ┌────▼─────┐
                    │  core    │ (核心引擎)
                    └────┬─────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
     ┌────▼────┐  ┌──────▼──────┐  ┌───▼────────┐
     │exec-srv │  │ codex-mcp   │  │  ext/*     │
     └────┬────┘  └──────┬──────┘  └────────────┘
          │              │
          └──────┬───────┘
                 │
            ┌────▼─────┐
            │app-server│
            └────┬─────┘
                 │
          ┌──────┼──────┐
          │      │      │
     ┌────▼──┐ ┌─▼──┐ ┌─▼──────┐
     │  TUI  │ │CLI │ │ app-srv│
     │       │ │    │ │ client │
     └───────┘ └────┘ └────────┘
```

### 依赖层次（从底到顶）

1. **基础层**：`protocol`（核心类型定义，无内部依赖）
2. **配置/工具层**：`config`、`tools`、`state`（依赖 protocol）
3. **扩展 API 层**：`ext/extension-api`（依赖 protocol、tools）
4. **核心引擎层**：`core`（依赖以上所有层）
5. **服务层**：`exec-server`、`codex-mcp`、`ext/*`（依赖 core 或其子集）
6. **API 层**：`app-server-protocol`、`app-server`（依赖 core、exec-server）
7. **前端层**：`cli`、`tui`（依赖 app-server）

### 关键设计约束

- `codex-core` 已过度膨胀，新增功能应优先考虑提取新 crate
- `app-server` 的新 API 只在 v2 开发，不新增 v1
- `protocol` 是最底层的共享类型库，应保持精简
- 模型上下文注入必须通过 `ContextualUserFragment` trait 且有界大小
- MCP 工具调用应通过 `mcp_connection_manager.rs` 集中处理

---

*本文档基于代码库结构分析生成，可能随代码演进而需要更新。*
