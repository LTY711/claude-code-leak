# Claude Code v2.1.88 — 源码分析

> **免责声明**: 本仓库中所有源码版权归 **Anthropic 和 Claude** 所有。本仓库仅用于技术研究和科研爱好者交流学习参考，**严禁任何个人、机构及组织将其用于商业用途、盈利性活动、非法用途及其他未经授权的场景。** 若内容涉及侵犯您的合法权益、知识产权或存在其他侵权问题，请及时联系我们，我们将第一时间核实并予以删除处理。

> 从 npm 包 `@anthropic-ai/claude-code` **2.1.88** 版本中提取。
> 发布的包只有一个打包后的 `cli.js`（~12MB）。本仓库的 `src/` 目录包含从 npm 包中解包的 **TypeScript 源码**。

**语言**: [English](README.md) | **中文** | [한국어](README_KR.md) | [日本語](README_JA.md)

---

## 目录

- [文档 (`docs/`)](#文档-docs) — 架构、子系统、工具、命令、桥接层
- [缺失模块说明](#缺失模块说明108-个模块) — 108 个被 feature gate 移除的模块
- [架构概览](#架构概览) — 入口 → 查询引擎 → 工具/服务/状态
- [构建说明](#构建说明) — 如何尝试本地构建

---

## 文档 (`docs/`)

代码库的结构性文档，便于研究和探索。

```
docs/
├── architecture.md       # 整体架构深度解析
├── subsystems.md         # 主要子系统详细说明
├── tools.md              # 全部 ~40 个 Agent 工具参考
├── commands.md           # 全部斜杠命令参考
├── bridge.md             # VS Code / JetBrains IDE 桥接层
└── exploration-guide.md  # 如何导航和研究本源码库
```

---

## 缺失模块说明（108 个模块）

> **源码不完整。** 108 个被 `feature()` 门控的模块**未包含**在 npm 包中。
> 它们仅存在于 Anthropic 的内部 monorepo 中，在编译时被死代码消除。
> **无法**从 `cli.js`、`sdk-tools.d.ts` 或任何已发布的制品中恢复。

### Anthropic 内部代码（~70 个模块，从未发布）

这些模块在 npm 包中没有任何源文件。它们是 Anthropic 内部基础设施。

<details>
<summary>点击展开完整列表</summary>

| Module | Purpose | Feature Gate |
|--------|---------|-------------|
| `daemon/main.js` | 后台守护进程管理器 | `DAEMON` |
| `daemon/workerRegistry.js` | 守护进程 worker 注册 | `DAEMON` |
| `proactive/index.js` | 主动通知系统 | `PROACTIVE` |
| `contextCollapse/index.js` | 上下文折叠服务（实验性） | `CONTEXT_COLLAPSE` |
| `contextCollapse/operations.js` | 折叠操作 | `CONTEXT_COLLAPSE` |
| `contextCollapse/persist.js` | 折叠持久化 | `CONTEXT_COLLAPSE` |
| `skillSearch/featureCheck.js` | 远程技能特性检查 | `EXPERIMENTAL_SKILL_SEARCH` |
| `skillSearch/remoteSkillLoader.js` | 远程技能加载器 | `EXPERIMENTAL_SKILL_SEARCH` |
| `skillSearch/remoteSkillState.js` | 远程技能状态 | `EXPERIMENTAL_SKILL_SEARCH` |
| `skillSearch/telemetry.js` | 技能搜索遥测 | `EXPERIMENTAL_SKILL_SEARCH` |
| `skillSearch/localSearch.js` | 本地技能搜索 | `EXPERIMENTAL_SKILL_SEARCH` |
| `skillSearch/prefetch.js` | 技能预取 | `EXPERIMENTAL_SKILL_SEARCH` |
| `coordinator/workerAgent.js` | 多代理协调器 worker | `COORDINATOR_MODE` |
| `bridge/peerSessions.js` | 桥接对等会话管理 | `BRIDGE_MODE` |
| `assistant/index.js` | KAIROS 助手模式 | `KAIROS` |
| `assistant/AssistantSessionChooser.js` | 助手会话选择器 | `KAIROS` |
| `compact/reactiveCompact.js` | 响应式上下文压缩 | `CACHED_MICROCOMPACT` |
| `compact/snipCompact.js` | 基于裁剪的压缩 | `HISTORY_SNIP` |
| `compact/snipProjection.js` | 裁剪投影 | `HISTORY_SNIP` |
| `compact/cachedMCConfig.js` | 缓存微压缩配置 | `CACHED_MICROCOMPACT` |
| `sessionTranscript/sessionTranscript.js` | 会话转录服务 | `TRANSCRIPT_CLASSIFIER` |
| `commands/agents-platform/index.js` | 内部代理平台 | `ant` (内部) |
| `commands/assistant/index.js` | 助手命令 | `KAIROS` |
| `commands/buddy/index.js` | Buddy 系统通知 | `BUDDY` |
| `commands/fork/index.js` | Fork 子代理命令 | `FORK_SUBAGENT` |
| `commands/peers/index.js` | 多对等命令 | `BRIDGE_MODE` |
| `commands/proactive.js` | 主动命令 | `PROACTIVE` |
| `commands/remoteControlServer/index.js` | 远程控制服务器 | `DAEMON` + `BRIDGE_MODE` |
| `commands/subscribe-pr.js` | GitHub PR 订阅 | `KAIROS_GITHUB_WEBHOOKS` |
| `commands/torch.js` | 内部调试工具 | `TORCH` |
| `commands/workflows/index.js` | 工作流命令 | `WORKFLOW_SCRIPTS` |
| `jobs/classifier.js` | 内部任务分类器 | `TEMPLATES` |
| `memdir/memoryShapeTelemetry.js` | 记忆形态遥测 | `MEMORY_SHAPE_TELEMETRY` |
| `services/sessionTranscript/sessionTranscript.js` | 会话转录 | `TRANSCRIPT_CLASSIFIER` |
| `tasks/LocalWorkflowTask/LocalWorkflowTask.js` | 本地工作流任务 | `WORKFLOW_SCRIPTS` |
| `protectedNamespace.js` | 内部命名空间守卫 | `ant` (内部) |
| `protectedNamespace.js` (envUtils) | 受保护命名空间运行时 | `ant` (内部) |
| `coreTypes.generated.js` | 生成的核心类型 | `ant` (内部) |
| `devtools.js` | 内部开发工具 | `ant` (内部) |
| `attributionHooks.js` | 内部归属钩子 | `COMMIT_ATTRIBUTION` |
| `systemThemeWatcher.js` | 系统主题监视器 | `AUTO_THEME` |
| `udsClient.js` / `udsMessaging.js` | UDS 消息客户端 | `UDS_INBOX` |

</details>

### Feature-Gated 工具（~20 个模块）

这些工具有类型签名，但实现被编译时移除。

<details>
<summary>点击展开完整列表</summary>

| Tool | Purpose | Feature Gate |
|------|---------|-------------|
| `REPLTool` | 交互式 REPL（VM 沙箱） | `ant` (内部) |
| `SnipTool` | 上下文裁剪 | `HISTORY_SNIP` |
| `SleepTool` | 代理循环中的休眠/延迟 | `PROACTIVE` / `KAIROS` |
| `MonitorTool` | MCP 监控 | `MONITOR_TOOL` |
| `OverflowTestTool` | 溢出测试 | `OVERFLOW_TEST_TOOL` |
| `WorkflowTool` | 工作流执行 | `WORKFLOW_SCRIPTS` |
| `WebBrowserTool` | 浏览器自动化 | `WEB_BROWSER_TOOL` |
| `TerminalCaptureTool` | 终端捕获 | `TERMINAL_PANEL` |
| `TungstenTool` | 内部性能监控 | `ant` (内部) |
| `VerifyPlanExecutionTool` | 计划验证 | `CLAUDE_CODE_VERIFY_PLAN` |
| `SendUserFileTool` | 向用户发送文件 | `KAIROS` |
| `SubscribePRTool` | GitHub PR 订阅 | `KAIROS_GITHUB_WEBHOOKS` |
| `SuggestBackgroundPRTool` | 建议后台 PR | `KAIROS` |
| `PushNotificationTool` | 推送通知 | `KAIROS` |
| `CtxInspectTool` | 上下文检查 | `CONTEXT_COLLAPSE` |
| `ListPeersTool` | 列出活跃对等方 | `UDS_INBOX` |
| `DiscoverSkillsTool` | 技能发现 | `EXPERIMENTAL_SKILL_SEARCH` |

</details>

### 文本/Prompt 资源（~6 个文件）

| File | Purpose |
|------|---------|
| `yolo-classifier-prompts/auto_mode_system_prompt.txt` | 自动模式分类器系统提示 |
| `yolo-classifier-prompts/permissions_anthropic.txt` | Anthropic 内部权限提示 |
| `yolo-classifier-prompts/permissions_external.txt` | 外部用户权限提示 |
| `verify/SKILL.md` | 验证技能文档 |
| `verify/examples/cli.md` | CLI 验证示例 |
| `verify/examples/server.md` | 服务端验证示例 |

### 为什么缺失

```
  Anthropic 内部 Monorepo              发布的 npm 包
  ──────────────────────               ─────────────────────
  feature('DAEMON') → true    ──构建──→   feature('DAEMON') → false
  ↓                                         ↓
  daemon/main.js  ← 包含        ──打包──→  daemon/main.js  ← 删除 (DCE)
  tools/REPLTool  ← 包含        ──打包──→  tools/REPLTool  ← 删除 (DCE)
  proactive/      ← 包含        ──打包──→  （被引用但 src/ 中不存在）
```

  Bun 的 `feature()` 是**编译时内建函数**：
  - 在 Anthropic 内部构建中返回 `true` → 代码保留在 bundle 中
  - 在发布构建中返回 `false` → 代码被死代码消除
  - 108 个模块在已发布的制品中根本不存在

---

## 版权与免责声明

```
Copyright (c) Anthropic. All rights reserved.

本仓库中所有源码均为 Anthropic 和 Claude 的知识产权。
本仓库仅用于技术研究和教育目的。严禁商业使用。

如果您是版权所有者并认为本仓库侵犯了您的权利，
请联系仓库所有者立即删除。
```

---

## 统计数据

| 项目 | 数量 |
|------|------|
| 源文件 (.ts/.tsx) | ~1,916 |
| 代码行数 | ~519,426 |
| 最大单文件 | `query.ts` (~785KB) |
| 内置工具 | ~43 |
| 斜杠命令 | ~100+ |
| 依赖 (node_modules) | ~192 个包 |
| 运行时 | Bun（编译为 Node.js >= 18 bundle）|

---

## 代理模式

```
                    核心循环
                    ========

    用户 --> messages[] --> Claude API --> 响应
                                          |
                                stop_reason == "tool_use"?
                               /                          \
                             是                           否
                              |                             |
                        执行工具                        返回文本
                        追加 tool_result
                        循环回退 -----------------> messages[]


    这就是最小的代理循环。Claude Code 在此循环上
    包裹了生产级线束：权限、流式传输、并发、
    压缩、子代理、持久化和 MCP。
```

---

## 目录参考

```
src/
├── main.tsx                 # REPL 引导程序，4,683 行
├── QueryEngine.ts           # SDK/headless 查询生命周期引擎
├── query.ts                 # 主代理循环 (785KB，最大文件)
├── Tool.ts                  # 工具接口 + buildTool 工厂
├── Task.ts                  # 任务类型、ID、状态基类
├── tools.ts                 # 工具注册、预设、过滤
├── commands.ts              # 斜杠命令定义
├── context.ts               # 用户输入上下文
├── cost-tracker.ts          # API 成本累积
├── setup.ts                 # 首次运行设置流程
│
├── bridge/                  # Claude Desktop / 远程桥接
│   ├── bridgeMain.ts        #   会话生命周期管理器
│   ├── bridgeApi.ts         #   HTTP 客户端
│   ├── bridgeConfig.ts      #   连接配置
│   ├── bridgeMessaging.ts   #   消息中继
│   ├── sessionRunner.ts     #   进程生成
│   ├── jwtUtils.ts          #   JWT 刷新
│   ├── workSecret.ts        #   认证令牌
│   └── capacityWake.ts      #   基于容量的唤醒
│
├── cli/                     # CLI 基础设施
│   ├── handlers/            #   命令处理器
│   └── transports/          #   I/O 传输 (stdio, structured)
│
├── commands/                # ~100 个斜杠命令
├── components/              # React/Ink 终端 UI
├── entrypoints/             # 应用入口点（cli.tsx、mcp.ts、sdk/ 等）
├── hooks/                   # React hooks
├── screens/                 # 顶层界面组件（REPL.tsx 等）
├── services/                # 业务逻辑层
├── state/                   # 应用状态
├── tasks/                   # 任务实现
├── tools/                   # 43 个工具实现
├── types/                   # 类型定义
├── utils/                   # 工具函数（最大目录）
├── vim/                     # Vim 模式支持
├── voice/                   # 语音模式（push-to-talk）
└── shims/                   # 运行时垫片
```

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         入口层                                      │
│  entrypoints/cli.tsx ──> main.tsx ──> screens/REPL.tsx (交互式)      │
│                                └──> QueryEngine.ts (headless/SDK)    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       查询引擎                                      │
│  submitMessage(prompt) ──> AsyncGenerator<SDKMessage>               │
│    ├── fetchSystemPromptParts()    ──> 组装系统提示词               │
│    ├── processUserInput()          ──> 处理 /命令                   │
│    ├── query()                     ──> 主代理循环                   │
│    │     ├── StreamingToolExecutor ──> 并行工具执行                  │
│    │     ├── autoCompact()         ──> 上下文压缩                   │
│    │     └── runTools()            ──> 工具编排                     │
│    └── yield SDKMessage            ──> 流式传输给消费者             │
└──────────────────────────────┬──────────────────────────────────────┘
```

---

## 构建说明

本仓库包含完整的构建配置（`tsconfig.json`、`bunfig.toml`、`biome.json`、`scripts/`），但**实际构建仍受限**：

- `feature()` 调用是 Bun 编译时内建函数 — 在打包时解析，缺少 Anthropic 内部的 feature flag 值
- `MACRO.VERSION` 在构建时注入
- `process.env.USER_TYPE === 'ant'` 部分是 Anthropic 内部的
- 108 个被 feature gate 移除的模块在 `src/` 中不存在，编译会报错
- 编译后的 `cli.js` 是一个自包含的 12MB bundle，只需 Node.js >= 18

可以运行 `bun run build` 尝试构建，但预期会因缺失模块而失败。

---

## MCP 探索服务器 (`mcp-server/`)

独立的 [Model Context Protocol](https://modelcontextprotocol.io/) 服务器，让任何 MCP 客户端都能探索本源码库。支持 **STDIO**、**Streamable HTTP**、**SSE** 三种传输方式。

暴露了 **8 个工具、3 个资源、5 个 prompt**，用于导航 ~1,900 个文件、512K+ 行的代码库。

| 传输方式 | 适用场景 |
|----------|----------|
| STDIO | Claude Desktop、本地 Claude Code、VS Code |
| Streamable HTTP (`POST/GET /mcp`) | 现代 MCP 客户端、远程部署 |
| Legacy SSE | 旧版 MCP 客户端 |

---

## Web 界面 (`web/`)

基于 Next.js 的聊天界面（`claude-code-web`），提供对源码的可视化交互探索入口。

---

## 许可证

本仓库中所有源码版权归 **Anthropic 和 Claude** 所有。本仓库仅用于技术研究和教育目的。完整许可条款请参阅原始 npm 包。
