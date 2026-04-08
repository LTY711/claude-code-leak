# 探索指南

> 如何浏览和学习 Claude Code 源代码。

---

## 快速入门

这是一个**只读参考代码库** —— 没有构建系统或测试套件。其目标是理解一个生产级 AI 编程助手是如何构建的。

### 概览

| 内容 | 位置 |
|------|-------|
| CLI 入口点 | `src/entrypoints/cli.tsx` |
| 核心 LLM 引擎 | `src/QueryEngine.ts`（1,297 行）+ `src/query.ts`（1,730 行）|
| 工具定义 | `src/Tool.ts`（794 行）|
| 命令注册表 | `src/commands.ts`（758 行）|
| 工具注册表 | `src/tools.ts` |
| 上下文收集 | `src/context.ts` |
| 所有工具实现 | `src/tools/`（43 个子目录，共约 50K 行）|
| 所有命令实现 | `src/commands/`（约 102 个条目，共约 27K 行）|

---

## 查找内容

### "工具 X 是如何工作的？"

1. 前往 `src/tools/{ToolName}/`
2. 主要实现位于 `{ToolName}.ts` 或 `.tsx`
3. UI 渲染位于 `UI.tsx`
4. 系统提示词贡献位于 `prompt.ts`

示例 —— 理解 BashTool：
```
src/tools/BashTool/
├── BashTool.ts      ← 核心执行逻辑
├── UI.tsx           ← bash 输出在终端中的渲染方式
├── prompt.ts        ← 系统提示词中关于 bash 的描述
└── ...
```

### "命令 X 是如何工作的？"

1. 查看 `src/commands/{command-name}/`（目录）或 `src/commands/{command-name}.ts`（文件）
2. 寻找 `getPromptForCommand()` 函数（PromptCommands）或直接实现（LocalCommands）

### "功能 X 是如何工作的？"

| 功能 | 从这里开始 |
|---------|-----------|
| 权限 | `src/hooks/toolPermission/` |
| IDE 桥接 | `src/bridge/bridgeMain.ts` |
| MCP 客户端 | `src/services/mcp/` |
| 插件系统 | `src/plugins/` + `src/services/plugins/` |
| 技能 | `src/skills/` |
| 语音输入 | `src/voice/` + `src/services/voice.ts` |
| 多智能体 | `src/coordinator/` |
| 记忆 | `src/memdir/` |
| 身份验证 | `src/services/oauth/` |
| 配置模式 | `src/schemas/` |
| 状态管理 | `src/state/` |

### "API 调用流程是怎样的？"

从用户输入到 API 响应的调用链追踪：

```
src/entrypoints/cli.tsx         ← CLI 解析
  → src/replLauncher.tsx        ← REPL 会话启动
    → src/QueryEngine.ts        ← 核心引擎
      → src/services/api/       ← Anthropic SDK 客户端
        → (Anthropic API)       ← HTTP/流式传输
      ← 工具使用响应
      → src/tools/{ToolName}/   ← 工具执行
      ← 工具结果
      → (反馈回 API)            ← 继续循环
```

---

## 需要识别的代码模式

### `buildTool()` —— 工具工厂

每个工具都使用以下模式：

```typescript
export const MyTool = buildTool({
  name: 'MyTool',
  inputSchema: z.object({ ... }),
  async call(args, context) { ... },
  async checkPermissions(input, context) { ... },
})
```

### 功能标志门控

```typescript
import { feature } from 'bun:bundle'

if (feature('VOICE_MODE')) {
  // 如果 VOICE_MODE 关闭，此代码在构建时会被剔除
}
```

### Anthropic 内部门控

```typescript
if (process.env.USER_TYPE === 'ant') {
  // 仅限 Anthropic 员工使用的功能
}
```

### 索引重导出

大多数目录都有一个 `index.ts` 文件，用于重导出公共 API：

```typescript
// src/tools/BashTool/index.ts
export { BashTool } from './BashTool.js'
```

### 懒加载动态导入

重量级模块仅在需要时才加载：

```typescript
const { OpenTelemetry } = await import('./heavy-module.js')
```

### 带 `.js` 扩展名的 ESM

Bun 约定 —— 所有导入均使用 `.js` 扩展名，即使对应的是 `.ts` 文件：

```typescript
import { something } from './utils.js'  // 实际上导入的是 utils.ts
```

---

## 按大小排列的关键文件

最大的文件包含最多的逻辑，值得重点学习：

| 文件 | 行数 | 内容 |
|------|-------|---------------|
| `main.tsx` | 4,684 | REPL 引导程序、CLI 解析器、启动优化 |
| `query.ts` | 1,730 | 核心智能体循环实现 |
| `QueryEngine.ts` | 1,297 | SDK/无头查询生命周期引擎 |
| `Tool.ts` | 794 | 工具接口 + `buildTool` 工厂 |
| `commands.ts` | 758 | 斜杠命令注册表与加载 |
| `setup.ts` | 478 | 首次运行设置流程 |
| `tools/`（整个目录） | ~50K | 全部 43 个工具实现 |
| `commands/`（整个目录） | ~27K | 全部约 100 个斜杠命令实现 |

> 注意：其他地方引用的打包体积（约 46K、约 29K）是指编译后的 `cli.js` 各部分，而非 TypeScript 源文件。

---

## 学习路径

### 路径 1："工具端到端是如何工作的？"

1. 阅读 `src/Tool.ts` —— 理解 `buildTool` 接口
2. 选择一个简单的工具，例如 `src/tools/FileReadTool/` 中的 `FileReadTool`
3. 追踪 `QueryEngine.ts` 在工具循环中调用工具的方式
4. 查看 `src/hooks/toolPermission/` 中权限的检查方式

### 路径 2："UI 是如何工作的？"

1. 阅读 `src/screens/REPL.tsx` —— 主屏幕
2. 探索 `src/components/` —— 选择几个组件研究
3. 查看 `src/hooks/useTextInput.ts` —— 用户输入的捕获方式
4. 检查 `src/ink/` —— Ink 渲染器封装

### 路径 3："IDE 集成是如何工作的？"

1. 从 `src/bridge/bridgeMain.ts` 开始
2. 跟踪 `bridgeMessaging.ts` 了解消息协议
3. 查看 `bridgePermissionCallbacks.ts` 了解权限如何路由到 IDE
4. 检查 `replBridge.ts` 了解 REPL 会话桥接

### 路径 4："插件如何扩展 Claude Code？"

1. 阅读 `src/types/plugin.ts` —— 插件 API 接口
2. 查看 `src/services/plugins/` —— 插件的加载方式
3. 检查 `src/plugins/builtinPlugins.ts` —— 内置示例
4. 查看 `src/plugins/bundled/` —— 打包的插件代码

### 路径 5："MCP 是如何工作的？"

1. 阅读 `src/services/mcp/` —— MCP 客户端
2. 查看 `src/tools/MCPTool/` —— MCP 工具的调用方式
3. 检查 `src/entrypoints/mcp.ts` —— 作为 MCP 服务器的 Claude Code
4. 查看 `src/skills/mcpSkillBuilders.ts` —— 来自 MCP 的技能

---

## 使用 MCP 服务器进行探索

本代码库包含一个独立的 MCP 服务器（`mcp-server/`），可让任何兼容 MCP 的客户端探索源代码。请参阅 [MCP 服务器 README](../mcp-server/README.md) 了解设置方法。

连接后，你可以让 AI 助手探索源代码：

- "BashTool 是如何工作的？"
- "搜索权限检查的位置"
- "列出 bridge 目录中的所有文件"
- "读取 QueryEngine.ts 第 1-100 行"

---

## Grep 模式

用于查找内容的常用 grep/ripgrep 模式：

```bash
# 查找所有工具定义
rg "buildTool\(" src/tools/

# 查找所有命令定义
rg "satisfies Command" src/commands/

# 查找功能标志用法
rg "feature\(" src/

# 查找 Anthropic 内部门控
rg "USER_TYPE.*ant" src/

# 查找所有 React 钩子
rg "^export function use" src/hooks/

# 查找所有 Zod 模式
rg "z\.object\(" src/schemas/

# 查找所有系统提示词贡献
rg "prompt\(" src/tools/*/prompt.ts

# 查找权限规则模式
rg "checkPermissions" src/tools/
```

---

## 参见

- [架构](architecture.md) — 整体系统设计
- [工具参考](tools.md) — 完整工具目录
- [命令参考](commands.md) — 所有斜杠命令
- [子系统指南](subsystems.md) — Bridge、MCP、权限等深度解析
