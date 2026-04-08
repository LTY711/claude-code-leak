# 架构

> 深入介绍 Claude Code 的内部结构。

---

## 总体概览

Claude Code 是一个以终端为原生环境的 AI 编程助手，以单一二进制文件的形式作为 CLI 发布。其架构遵循流水线模型：

```
User Input → CLI Parser → Query Engine → LLM API → Tool Execution Loop → Terminal UI
```

整个 UI 层基于 **React + Ink**（面向终端的 React）构建，使其成为一个完全响应式的 CLI 应用，具备组件、钩子、状态管理等所有你在 React Web 应用中所期待的模式——只是渲染到终端而已。

---

## 核心流水线

### 1. 入口点（`src/entrypoints/cli.tsx`）

CLI 解析器基于 [Commander.js](https://github.com/tj/commander.js)（`@commander-js/extra-typings`）构建。启动时，它会：

- 在加载重量级模块之前，并行触发预加载副作用（MDM 设置、Keychain、API 预连接）
- 解析 CLI 参数和标志
- 初始化 React/Ink 渲染器
- 移交给 REPL 启动器（`src/replLauncher.tsx`）

### 2. 初始化（`src/entrypoints/`）

| 文件 | 职责 |
|------|------|
| `cli.tsx` | CLI 会话编排——从启动到 REPL 的主路径 |
| `init.ts` | 配置、遥测、OAuth、MDM 策略初始化 |
| `mcp.ts` | MCP 服务器模式入口点（将 Claude Code 作为 MCP 服务器运行） |
| `sdk/` | Agent SDK——用于嵌入 Claude Code 的编程接口 |

启动时执行并行初始化：MDM 策略读取、Keychain 预加载、功能标志检查，然后进行核心初始化。

### 3. 查询引擎（`src/QueryEngine.ts` + `src/query.ts`）

Claude Code 的核心。负责处理：

- **流式响应** — 来自 Anthropic API 的流式输出
- **工具调用循环** — 当 LLM 请求工具时，执行该工具并将结果回传
- **思考模式** — 带预算管理的扩展思考
- **重试逻辑** — 针对瞬时故障的自动退避重试
- **令牌计数** — 追踪每轮对话的输入/输出令牌数及费用
- **上下文管理** — 管理对话历史和上下文窗口

### 4. 工具系统（`src/Tool.ts` + `src/tools/`）

Claude 可调用的每项能力都是一个**工具**。每个工具自成一体，包含：

- **输入模式**（Zod 校验）
- **权限模型**（哪些操作需要用户审批）
- **执行逻辑**（实际实现）
- **UI 组件**（调用/结果在终端中的渲染方式）

工具在 `src/tools.ts` 中注册，并由查询引擎在工具调用循环中发现。

完整目录请参见[工具参考](tools.md)。

### 5. 命令系统（`src/commands.ts` + `src/commands/`）

面向用户的斜杠命令（`/commit`、`/review`、`/mcp` 等），可在 REPL 中输入。共三种类型：

| 类型 | 描述 | 示例 |
|------|------|------|
| **PromptCommand** | 向 LLM 发送带注入工具的格式化提示 | `/review`、`/commit` |
| **LocalCommand** | 在进程内运行，返回纯文本 | `/cost`、`/version` |
| **LocalJSXCommand** | 在进程内运行，返回 React JSX | `/doctor`、`/install` |

命令在 `src/commands.ts` 中注册，通过 REPL 中的 `/command-name` 调用。

完整目录请参见[命令参考](commands.md)。

---

## 状态管理

Claude Code 采用 **React context + 自定义 store** 模式：

| 组件 | 位置 | 用途 |
|------|------|------|
| `AppState` | `src/state/AppStateStore.ts` | 全局可变状态对象 |
| Context Providers | `src/context/` | 用于通知、统计、FPS 的 React context |
| Selectors | `src/state/` | 派生状态函数 |
| Change Observers | `src/state/onChangeAppState.ts` | 状态变更时的副作用处理 |

`AppState` 对象会被传入工具上下文，使工具能够访问对话历史、设置和运行时状态。

---

## UI 层

### 组件（`src/components/`，约 140 个组件）

- 使用 Ink 原语（`Box`、`Text`、`useInput()`）的函数式 React 组件
- 使用 [Chalk](https://github.com/chalk/chalk) 为终端添加颜色样式
- 启用 React Compiler 以优化重渲染
- 设计系统原语位于 `src/components/design-system/`

### 屏幕（`src/screens/`）

全屏 UI 模式：

| 屏幕 | 用途 |
|------|------|
| `REPL.tsx` | 主交互式 REPL（默认屏幕） |
| `Doctor.tsx` | 环境诊断（`/doctor`） |
| `ResumeConversation.tsx` | 会话恢复（`/resume`） |

### 钩子（`src/hooks/`，约 80 个钩子）

标准 React 钩子模式。主要类别：

- **权限钩子** — `useCanUseTool`、`src/hooks/toolPermission/`
- **IDE 集成** — `useIDEIntegration`、`useIdeConnectionStatus`、`useDiffInIDE`
- **输入处理** — `useTextInput`、`useVimInput`、`usePasteHandler`、`useInputBuffer`
- **会话管理** — `useSessionBackgrounding`、`useRemoteSession`、`useAssistantHistory`
- **插件/技能钩子** — `useManagePlugins`、`useSkillsChange`
- **通知钩子** — `src/hooks/notifs/`（频率限制、弃用警告等）

---

## 配置与模式

### 配置模式（`src/schemas/`）

基于 Zod v3 的所有配置模式：

- 用户设置
- 项目级设置
- 组织/企业策略
- 权限规则

### 迁移（`src/migrations/`）

处理版本间的配置格式变更——读取旧配置并将其转换为当前模式。

---

## 构建系统

### Bun 运行时

Claude Code 运行在 [Bun](https://bun.sh) 上（而非 Node.js）。主要影响：

- 原生 JSX/TSX 支持，无需转译步骤
- `bun:bundle` 功能标志用于死代码消除
- 使用带 `.js` 扩展名的 ES 模块（Bun 惯例）

### 功能标志（死代码消除）

```typescript
import { feature } from 'bun:bundle'

// Code inside inactive feature flags is completely stripped at build time
if (feature('VOICE_MODE')) {
  const voiceCommand = require('./commands/voice/index.js').default
}
```

主要标志：

| 标志 | 功能 |
|------|------|
| `PROACTIVE` | 主动代理模式（自主操作） |
| `KAIROS` | Kairos 子系统 |
| `BRIDGE_MODE` | IDE 桥接集成 |
| `DAEMON` | 后台守护进程模式 |
| `VOICE_MODE` | 语音输入/输出 |
| `AGENT_TRIGGERS` | 触发式代理操作 |
| `MONITOR_TOOL` | 监控工具 |
| `COORDINATOR_MODE` | 多代理协调器 |
| `WORKFLOW_SCRIPTS` | 工作流自动化脚本 |

### 懒加载

重量级模块通过动态 `import()` 延迟到首次使用时加载：

- OpenTelemetry（约 400KB）
- gRPC（约 700KB）
- 其他可选依赖

---

## 错误处理与遥测

### 遥测（`src/services/analytics/`）

- [GrowthBook](https://www.growthbook.io/) 用于功能标志和 A/B 测试
- [OpenTelemetry](https://opentelemetry.io/) 用于分布式追踪和指标
- 用于使用情况统计的自定义事件追踪

### 费用追踪（`src/cost-tracker.ts`）

追踪每轮对话的令牌用量和预估费用，可通过 `/cost` 命令查看。

### 诊断（`/doctor` 命令）

`Doctor.tsx` 屏幕运行环境检查：API 连通性、身份认证、工具可用性、MCP 服务器状态等。

---

## 并发模型

Claude Code 采用**单线程事件循环**（Bun/Node.js 模型），具备：

- 用于 I/O 操作的 async/await
- React 的并发渲染用于 UI 更新
- Web Workers 或子进程用于 CPU 密集型任务（gRPC 等）
- 工具并发安全性——每个工具通过声明 `isConcurrencySafe()` 来表明其是否可与其他工具并行运行

---

## 参见

- [工具参考](tools.md) — 全部 40 个代理工具的完整目录
- [命令参考](commands.md) — 所有斜杠命令的完整目录
- [子系统指南](subsystems.md) — Bridge、MCP、权限、技能、插件等
- [探索指南](exploration-guide.md) — 如何浏览本代码库
