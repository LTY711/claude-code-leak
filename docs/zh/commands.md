# 命令参考

> Claude Code 中全部斜杠命令的完整目录。

---

## 概述

命令是用户通过在 REPL 中输入 `/` 前缀来触发的操作（例如 `/commit`、`/review`）。命令文件存放于 `src/commands/` 目录，并在 `src/commands.ts` 中统一注册。

### 命令类型

| 类型 | 描述 | 示例 |
|------|------|------|
| **PromptCommand** | 将格式化的提示词连同注入的工具一起发送给 LLM | `/review`、`/commit` |
| **LocalCommand** | 在进程内运行，返回纯文本 | `/cost`、`/version` |
| **LocalJSXCommand** | 在进程内运行，返回 React JSX | `/install`、`/doctor` |

### 命令定义模式

```typescript
const command = {
  type: 'prompt',
  name: 'my-command',
  description: 'What this command does',
  progressMessage: 'working...',
  allowedTools: ['Bash(git *)', 'FileRead(*)'],
  source: 'builtin',
  async getPromptForCommand(args, context) {
    return [{ type: 'text', text: '...' }]
  },
} satisfies Command
```

---

## Git 与版本控制

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/commit` | `commit.ts` | 使用 AI 生成的提交信息创建 git commit |
| `/commit-push-pr` | `commit-push-pr.ts` | 一步完成 commit、push 并创建 PR |
| `/branch` | `branch/` | 创建或切换 git 分支 |
| `/diff` | `diff/` | 查看文件变更（已暂存、未暂存，或与某个引用对比） |
| `/pr_comments` | `pr_comments/` | 查看并处理 PR 审查意见 |
| `/rewind` | `rewind/` | 回退到之前的某个状态 |

## 代码质量

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/review` | `review.ts` | 对已暂存/未暂存的变更进行 AI 驱动的代码审查 |
| `/security-review` | `security-review.ts` | 以安全为重点的代码审查 |
| `/advisor` | `advisor.ts` | 获取架构或设计方面的建议 |
| `/bughunter` | `bughunter/` | 在代码库中查找潜在 bug |
| `/ultrareview` | `review/ultrareviewCommand.tsx` | 带有扩展分析的 AI 超级代码审查 |

## 会话与上下文

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/compact` | `compact/` | 压缩对话上下文以容纳更多历史记录 |
| `/context` | `context/` | 可视化当前上下文（文件、内存等） |
| `/resume` | `resume/` | 恢复之前的对话会话 |
| `/session` | `session/` | 管理会话（列出、切换、删除） |
| `/share` | `share/` | 通过链接分享会话 |
| `/export` | `export/` | 将对话导出到文件 |
| `/summary` | `summary/` | 生成当前会话的摘要 |
| `/clear` | `clear/` | 清除对话历史记录 |

## 配置与设置

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/config` | `config/` | 查看或修改 Claude Code 设置 |
| `/permissions` | `permissions/` | 管理工具权限规则 |
| `/theme` | `theme/` | 更改终端颜色主题 |
| `/output-style` | `output-style/` | 更改输出格式样式 |
| `/color` | `color/` | 切换彩色输出 |
| `/keybindings` | `keybindings/` | 查看或自定义快捷键绑定 |
| `/vim` | `vim/` | 切换输入的 vim 模式 |
| `/effort` | `effort/` | 调整响应的精力等级 |
| `/model` | `model/` | 切换当前使用的模型 |
| `/privacy-settings` | `privacy-settings/` | 管理隐私/数据设置 |
| `/fast` | `fast/` | 切换快速模式（更简短的响应） |
| `/brief` | `brief.ts` | 切换简洁输出模式 |

## 内存与知识

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/memory` | `memory/` | 管理持久化内存（CLAUDE.md 文件） |
| `/add-dir` | `add-dir/` | 将目录添加到项目上下文 |
| `/files` | `files/` | 列出当前上下文中的文件 |

## MCP 与插件

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/mcp` | `mcp/` | 管理 MCP 服务器连接 |
| `/plugin` | `plugin/` | 安装、移除或管理插件 |
| `/reload-plugins` | `reload-plugins/` | 重新加载所有已安装的插件 |
| `/skills` | `skills/` | 查看和管理技能 |

## 认证

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/login` | `login/` | 通过 Anthropic 进行身份验证 |
| `/logout` | `logout/` | 退出登录 |
| `/oauth-refresh` | `oauth-refresh/` | 刷新 OAuth 令牌 |

## 任务与代理

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/tasks` | `tasks/` | 管理后台任务 |
| `/agents` | `agents/` | 管理子代理 |
| `/ultraplan` | `ultraplan.tsx` | 生成详细的执行计划 |
| `/plan` | `plan/` | 进入规划模式 |

## 诊断与状态

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/doctor` | `doctor/` | 运行环境诊断 |
| `/status` | `status/` | 显示系统与会话状态 |
| `/stats` | `stats/` | 显示会话统计信息 |
| `/cost` | `cost/` | 显示 token 用量和预估费用 |
| `/version` | `version.ts` | 显示 Claude Code 版本号 |
| `/usage` | `usage/` | 显示详细 API 用量 |
| `/extra-usage` | `extra-usage/` | 显示扩展用量详情 |
| `/rate-limit-options` | `rate-limit-options/` | 查看速率限制配置 |

## 安装与初始化

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/install` | `install.tsx` | 安装或更新 Claude Code |
| `/upgrade` | `upgrade/` | 升级到最新版本 |
| `/init` | `init.ts` | 初始化项目（创建 CLAUDE.md） |
| `/init-verifiers` | `init-verifiers.ts` | 设置验证器钩子 |
| `/onboarding` | `onboarding/` | 运行首次使用设置向导 |
| `/terminalSetup` | `terminalSetup/` | 配置终端集成 |
| `/install-github-app` | `install-github-app/` | 安装 Claude Code GitHub App |
| `/install-slack-app` | `install-slack-app/` | 安装 Claude Code Slack 集成 |

## IDE 与桌面集成

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/bridge` | `bridge/` | 管理 IDE 桥接连接 |
| `/bridge-kick` | `bridge-kick.ts` | 强制重启 IDE 桥接 |
| `/ide` | `ide/` | 在 IDE 中打开 |
| `/desktop` | `desktop/` | 切换到桌面应用 |
| `/mobile` | `mobile/` | 切换到移动应用 |
| `/teleport` | `teleport/` | 将会话迁移到另一台设备 |

## 远程与环境

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/remote-env` | `remote-env/` | 配置远程环境 |
| `/remote-setup` | `remote-setup/` | 设置远程会话 |
| `/env` | `env/` | 查看环境变量 |
| `/sandbox-toggle` | `sandbox-toggle/` | 切换沙箱模式 |

## 其他

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/help` | `help/` | 显示帮助信息及可用命令列表 |
| `/exit` | `exit/` | 退出 Claude Code |
| `/copy` | `copy/` | 复制内容到剪贴板 |
| `/feedback` | `feedback/` | 向 Anthropic 发送反馈 |
| `/release-notes` | `release-notes/` | 查看发行说明 |
| `/rename` | `rename/` | 重命名当前会话 |
| `/tag` | `tag/` | 为当前会话打标签 |
| `/insights` | `insights.ts` | 显示代码库洞察 |
| `/stickers` | `stickers/` | 彩蛋功能 — 贴纸 |
| `/good-claude` | `good-claude/` | 彩蛋功能 — 夸夸 Claude |
| `/voice` | `voice/` | 切换语音输入模式 |
| `/chrome` | `chrome/` | Chrome 扩展集成 |
| `/issue` | `issue/` | 提交 GitHub Issue |
| `/statusline` | `statusline.tsx` | 自定义状态栏 |
| `/think-back` | `thinkback/` | 回放 Claude 的思考过程 |
| `/thinkback-play` | `thinkback-play/` | 以动画方式回放 Claude 的思考过程 |
| `/passes` | `passes/` | 多轮执行 |
| `/x402` | `x402/` | x402 支付协议集成 |

## 内部 / 调试命令

| 命令 | 源文件 | 描述 |
|------|--------|------|
| `/ant-trace` | `ant-trace/` | Anthropic 内部链路追踪 |
| `/autofix-pr` | `autofix-pr/` | 自动修复 PR 问题 |
| `/backfill-sessions` | `backfill-sessions/` | 回填会话数据 |
| `/break-cache` | `break-cache/` | 使缓存失效 |
| `/btw` | `btw/` | "顺带一提"插入语 |
| `/ctx_viz` | `ctx_viz/` | 上下文可视化（调试） |
| `/debug-tool-call` | `debug-tool-call/` | 调试特定工具调用 |
| `/heapdump` | `heapdump/` | 导出堆内存用于内存分析 |
| `/hooks` | `hooks/` | 管理钩子脚本 |
| `/mock-limits` | `mock-limits/` | 模拟速率限制（用于测试） |
| `/perf-issue` | `perf-issue/` | 报告性能问题 |
| `/reset-limits` | `reset-limits/` | 重置速率限制计数器 |

---

## 参见

- [架构说明](architecture.md) — 命令系统如何融入整体流水线
- [工具参考](tools.md) — 代理工具（与斜杠命令不同）
- [探索指南](exploration-guide.md) — 查找命令源码的方法
