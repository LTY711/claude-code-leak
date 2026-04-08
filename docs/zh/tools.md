# 工具参考

> Claude Code 中全部 40 个代理工具的完整目录。

---

## 概述

每个工具以独立模块的形式存放于 `src/tools/<ToolName>/` 目录下。每个工具定义了以下内容：

- **输入模式（Input schema）** — 经 Zod 验证的参数
- **权限模型（Permission model）** — 哪些操作需要用户审批
- **执行逻辑（Execution logic）** — 工具的具体实现
- **UI 组件（UI components）** — 调用与结果的终端渲染
- **并发安全性（Concurrency safety）** — 是否可以并行运行

工具在 `src/tools.ts` 中注册，并由查询引擎（Query Engine）在 LLM 工具调用循环中调用。

### 工具定义模式

```typescript
export const MyTool = buildTool({
  name: 'MyTool',
  aliases: ['my_tool'],
  description: 'What this tool does',
  inputSchema: z.object({
    param: z.string(),
  }),
  async call(args, context, canUseTool, parentMessage, onProgress) {
    // Execute and return { data: result, newMessages?: [...] }
  },
  async checkPermissions(input, context) { /* Permission checks */ },
  isConcurrencySafe(input) { /* Can run in parallel? */ },
  isReadOnly(input) { /* Non-destructive? */ },
  prompt(options) { /* System prompt injection */ },
  renderToolUseMessage(input, options) { /* UI for invocation */ },
  renderToolResultMessage(content, progressMessages, options) { /* UI for result */ },
})
```

**每个工具的目录结构：**

```
src/tools/MyTool/
├── MyTool.ts        # Main implementation
├── UI.tsx           # Terminal rendering
├── prompt.ts        # System prompt contribution
└── utils.ts         # Tool-specific helpers
```

---

## 文件系统工具

| 工具 | 描述 | 只读 |
|------|------|------|
| **FileReadTool** | 读取文件内容（文本、图片、PDF、Notebook），支持行范围 | 是 |
| **FileWriteTool** | 创建或覆盖文件 | 否 |
| **FileEditTool** | 通过字符串替换对文件进行局部修改 | 否 |
| **GlobTool** | 使用 glob 模式查找文件（如 `**/*.ts`） | 是 |
| **GrepTool** | 使用 ripgrep 进行内容搜索（支持正则表达式） | 是 |
| **NotebookEditTool** | 编辑 Jupyter Notebook 单元格 | 否 |
| **TodoWriteTool** | 写入结构化的待办/任务文件 | 否 |

## Shell 与执行工具

| 工具 | 描述 | 只读 |
|------|------|------|
| **BashTool** | 在 bash 中执行 shell 命令 | 否 |
| **PowerShellTool** | 执行 PowerShell 命令（Windows） | 否 |
| **REPLTool** | 在 REPL 会话中运行代码（Python、Node 等）— *仅限 Anthropic 内部用户（`ant`）* | 否 |

## 代理与编排工具

| 工具 | 描述 | 只读 |
|------|------|------|
| **AgentTool** | 为复杂任务创建子代理 | 否 |
| **SendMessageTool** | 在代理之间发送消息 | 否 |
| **TeamCreateTool** | 创建并行代理团队 | 否 |
| **TeamDeleteTool** | 删除团队代理 | 否 |
| **EnterPlanModeTool** | 切换到规划模式（不执行操作） | 否 |
| **ExitPlanModeTool** | 退出规划模式，恢复执行 | 否 |
| **EnterWorktreeTool** | 在 git worktree 中隔离工作 | 否 |
| **ExitWorktreeTool** | 退出 worktree 隔离 | 否 |
| **SleepTool** | 暂停执行（主动模式）— *需要 `PROACTIVE` / `KAIROS` feature flag* | 是 |
| **SyntheticOutputTool** | 生成结构化输出 | 是 |

## 任务管理工具

| 工具 | 描述 | 只读 |
|------|------|------|
| **TaskCreateTool** | 创建新的后台任务 | 否 |
| **TaskUpdateTool** | 更新任务状态或详情 | 否 |
| **TaskGetTool** | 获取特定任务的详情 | 是 |
| **TaskListTool** | 列出所有任务 | 是 |
| **TaskOutputTool** | 获取已完成任务的输出 | 是 |
| **TaskStopTool** | 停止正在运行的任务 | 否 |

## 网络工具

| 工具 | 描述 | 只读 |
|------|------|------|
| **WebFetchTool** | 从 URL 获取内容 | 是 |
| **WebSearchTool** | 搜索网络 | 是 |

## MCP（模型上下文协议）工具

| 工具 | 描述 | 只读 |
|------|------|------|
| **MCPTool** | 调用已连接 MCP 服务器上的工具 | 不定 |
| **ListMcpResourcesTool** | 列出 MCP 服务器暴露的资源 | 是 |
| **ReadMcpResourceTool** | 读取特定 MCP 资源 | 是 |
| **McpAuthTool** | 处理 MCP 服务器认证 | 否 |
| **ToolSearchTool** | 从 MCP 服务器发现延迟/动态工具 | 是 |

## 集成工具

| 工具 | 描述 | 只读 |
|------|------|------|
| **LSPTool** | 语言服务器协议操作（跳转定义、查找引用等） | 是 |
| **SkillTool** | 执行已注册的技能 | 不定 |

## 调度与触发器

| 工具 | 描述 | 只读 |
|------|------|------|
| **ScheduleCronTool** | 创建定时 cron 触发器 | 否 |
| **RemoteTriggerTool** | 触发远程触发器 | 否 |

## 实用工具

| 工具 | 描述 | 只读 |
|------|------|------|
| **AskUserQuestionTool** | 在执行过程中向用户提示输入 | 是 |
| **BriefTool** | 生成摘要/简报 — *需要 `KAIROS_BRIEF` feature flag* | 是 |
| **ConfigTool** | 读取或修改 Claude Code 配置 | 否 |

---

## 权限模型

每次工具调用都会经过权限系统（`src/hooks/toolPermission/`）。权限模式说明：

| 模式 | 行为 |
|------|------|
| `default` | 对每个可能有破坏性的操作提示用户确认 |
| `plan` | 展示完整计划，只询问一次 |
| `bypassPermissions` | 自动批准所有操作（危险） |
| `auto` | 由基于机器学习的分类器决定 |

权限规则使用通配符模式：

```
Bash(git *)           # Allow all git commands
FileEdit(/src/*)      # Allow edits to anything in src/
FileRead(*)           # Allow reading any file
```

每个工具实现 `checkPermissions()`，返回 `{ granted: boolean, reason?, prompt? }`。

---

## 工具预设

工具在 `src/tools.ts` 中按不同使用场景分组为预设集合（例如：用于代码审查的只读工具集、用于开发的完整工具集）。

---

## 参见

- [架构说明](architecture.md) — 工具如何融入整体流水线
- [子系统指南](subsystems.md) — MCP、权限及其他工具相关子系统
- [探索指南](exploration-guide.md) — 如何阅读工具源码
