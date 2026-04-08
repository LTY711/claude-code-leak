# 子系统指南

> Claude Code 各主要子系统的详细文档。此为英文原版，中文翻译版正在整合中。

---

## Table of Contents

- [Bridge (IDE Integration)](#bridge-ide-integration)
- [MCP (Model Context Protocol)](#mcp-model-context-protocol)
- [Permission System](#permission-system)
- [Plugin System](#plugin-system)
- [Skill System](#skill-system)
- [Task System](#task-system)
- [Memory System](#memory-system)
- [Coordinator (Multi-Agent)](#coordinator-multi-agent)
- [Voice System](#voice-system)
- [Service Layer](#service-layer)
- [Buddy System (Virtual Companion)](#buddy-system-virtual-companion)
- [Vim Mode](#vim-mode)
- [Keybindings](#keybindings)
- [OAuth Authentication](#oauth-authentication)

---

## Bridge (IDE Integration)

**Location:** `src/bridge/`

The bridge is a bidirectional communication layer connecting Claude Code's CLI with IDE extensions (VS Code, JetBrains). It allows the CLI to run as a backend for IDE-based interfaces.

### Architecture

```
┌──────────────────┐         ┌──────────────────────┐
│   IDE Extension  │◄───────►│   Bridge Layer       │
│  (VS Code, JB)   │  JWT    │  (src/bridge/)       │
│                  │  Auth   │                      │
│  - UI rendering  │         │  - Session mgmt      │
│  - File watching │         │  - Message routing    │
│  - Diff display  │         │  - Permission proxy   │
└──────────────────┘         └──────────┬───────────┘
                                        │
                                        ▼
                              ┌──────────────────────┐
                              │   Claude Code Core   │
                              │  (QueryEngine, Tools) │
                              └──────────────────────┘
```

### Key Files

| File | Purpose |
|------|---------|
| `bridgeMain.ts` | Main bridge loop — starts the bidirectional channel |
| `bridgeMessaging.ts` | Message protocol (serialize/deserialize) |
| `bridgePermissionCallbacks.ts` | Routes permission prompts to the IDE |
| `bridgeApi.ts` | API surface exposed to the IDE |
| `bridgeConfig.ts` | Bridge configuration |
| `replBridge.ts` | Connects the REPL session to the bridge |
| `jwtUtils.ts` | JWT-based authentication between CLI and IDE |
| `sessionRunner.ts` | Manages bridge session execution |
| `createSession.ts` | Creates new bridge sessions |
| `trustedDevice.ts` | Device trust verification |
| `workSecret.ts` | Workspace-scoped secrets |
| `inboundMessages.ts` | Handles messages coming from the IDE |
| `inboundAttachments.ts` | Handles file attachments from the IDE |
| `types.ts` | TypeScript types for the bridge protocol |

### Feature Flag

The bridge is gated behind the `BRIDGE_MODE` feature flag and is stripped from non-IDE builds.

---

## MCP (Model Context Protocol)

**Location:** `src/services/mcp/`

Claude Code acts as both an **MCP client** (consuming tools/resources from MCP servers) and can run as an **MCP server** (exposing its own tools via `src/entrypoints/mcp.ts`).

### Client Features

- **Tool discovery** — Enumerates tools from connected MCP servers
- **Resource browsing** — Lists and reads MCP-exposed resources
- **Dynamic tool loading** — `ToolSearchTool` discovers tools at runtime
- **Authentication** — `McpAuthTool` handles MCP server auth flows
- **Connectivity monitoring** — `useMcpConnectivityStatus` hook tracks connection health

### Server Mode

When launched via `src/entrypoints/mcp.ts`, Claude Code exposes its own tools and resources via the MCP protocol, allowing other AI agents to use Claude Code as a tool server.

### Related Tools

| Tool | Purpose |
|------|---------|
| `MCPTool` | Invoke tools on connected MCP servers |
| `ListMcpResourcesTool` | List available MCP resources |
| `ReadMcpResourceTool` | Read a specific MCP resource |
| `McpAuthTool` | Authenticate with an MCP server |
| `ToolSearchTool` | Discover deferred tools from MCP servers |

### Configuration

MCP servers are configured via `/mcp` command or settings files. The server approval flow lives in `src/services/mcpServerApproval.tsx`.

---

## Permission System

**Location:** `src/hooks/toolPermission/`

Every tool invocation passes through a centralized permission check before execution.

### Permission Modes

| Mode | Behavior |
|------|----------|
| `default` | Prompts the user for each potentially destructive operation |
| `plan` | Shows the full execution plan, asks once for batch approval |
| `bypassPermissions` | Auto-approves all operations (dangerous — for trusted environments) |
| `auto` | ML-based classifier automatically decides (experimental) |

### How It Works

1. Tool is invoked by the Query Engine
2. `checkPermissions(input, context)` is called on the tool
3. Permission handler checks against configured rules
4. If not auto-approved, user is prompted via terminal or IDE

### Permission Rules

Rules use wildcard patterns to match tool invocations:

```
Bash(git *)           # Allow all git commands without prompt
Bash(npm test)        # Allow 'npm test' specifically
FileEdit(/src/*)      # Allow edits to anything under src/
FileRead(*)           # Allow reading any file
```

### Key Files

| File | Path |
|------|------|
| Permission context | `src/hooks/toolPermission/PermissionContext.ts` |
| Permission handlers | `src/hooks/toolPermission/handlers/` |
| Permission logging | `src/hooks/toolPermission/permissionLogging.ts` |
| Permission types | `src/types/permissions.ts` |

---

## Plugin System

**Location:** `src/plugins/`, `src/services/plugins/`

Claude Code supports installable plugins that can extend its capabilities.

### Structure

| Component | Location | Purpose |
|-----------|----------|---------|
| Plugin loader | `src/services/plugins/` | Discovers and loads plugins |
| Built-in plugins | `src/plugins/builtinPlugins.ts` | Plugins that ship with Claude Code |
| Bundled plugins | `src/plugins/bundled/` | Plugin code bundled into the binary |
| Plugin types | `src/types/plugin.ts` | TypeScript types for plugin API |

### Plugin Lifecycle

1. **Discovery** — Scans plugin directories and marketplace
2. **Installation** — Downloaded and registered (`/plugin` command)
3. **Loading** — Initialized at startup or on-demand
4. **Execution** — Plugins can contribute tools, commands, and prompts
5. **Auto-update** — `usePluginAutoupdateNotification` handles updates

### Related Commands

| Command | Purpose |
|---------|---------|
| `/plugin` | Install, remove, or manage plugins |
| `/reload-plugins` | Reload all installed plugins |

---

## Skill System

**Location:** `src/skills/`

Skills are reusable, named workflows that bundle prompts and tool configurations for specific tasks.

### Structure

| Component | Location | Purpose |
|-----------|----------|---------|
| Bundled skills | `src/skills/bundled/` | Skills that ship with Claude Code |
| Skill loader | `src/skills/loadSkillsDir.ts` | Loads skills from disk |
| MCP skill builders | `src/skills/mcpSkillBuilders.ts` | Creates skills from MCP resources |
| Skill registry | `src/skills/bundledSkills.ts` | Registration of all bundled skills |

### Bundled Skills (15)

> `claudeApiContent.ts` 是 `claudeApi` skill 的内容加载器（多语言代码示例），不是独立 skill，不计入此列表。

| Skill | Purpose |
|-------|---------|
| `batch` | 批量操作多个文件 |
| `claudeApi` | 调用 Anthropic API，附带多语言代码示例 |
| `claudeInChrome` | Chrome 扩展集成 |
| `debug` | 调试工作流 |
| `keybindings` | 键位绑定配置 |
| `loop` | 迭代优化循环 |
| `loremIpsum` | 生成占位文本 |
| `remember` | 将信息持久化到 Memory |
| `scheduleRemoteAgents` | 调度 Agent 远程执行 |
| `simplify` | 简化复杂代码 |
| `skillify` | 从现有工作流创建新 Skill |
| `stuck` | 遇到阻塞时获取帮助 |
| `updateConfig` | 以编程方式修改配置 |
| `verify` | 验证代码正确性（主流程） |
| `verifyContent` | 验证代码正确性（内容校验） |

### Execution

Skills are invoked via the `SkillTool` or the `/skills` command. Users can also create custom skills.

---

## Task System

**Location:** `src/tasks/`

Manages background and parallel work items — shell tasks, agent tasks, and teammate agents.

### Task Types

| Type | Location | Purpose |
|------|----------|---------|
| `LocalShellTask` | `LocalShellTask/` | Background shell command execution |
| `LocalAgentTask` | `LocalAgentTask/` | Sub-agent running locally |
| `RemoteAgentTask` | `RemoteAgentTask/` | Agent running on a remote machine |
| `InProcessTeammateTask` | `InProcessTeammateTask/` | Parallel teammate agent |
| `DreamTask` | `DreamTask/` | Background "dreaming" process |
| `LocalMainSessionTask` | `LocalMainSessionTask.ts` | Main session as a task |

### Task Tools

| Tool | Purpose |
|------|---------|
| `TaskCreateTool` | Create a new background task |
| `TaskUpdateTool` | Update task status |
| `TaskGetTool` | Retrieve task details |
| `TaskListTool` | List all tasks |
| `TaskOutputTool` | Get task output |
| `TaskStopTool` | Stop a running task |

---

## Memory System

**Location:** `src/memdir/`

Claude Code's persistent memory system, based on `CLAUDE.md` files.

### Memory Hierarchy

| Scope | Location | Purpose |
|-------|----------|---------|
| Project memory | `CLAUDE.md` in project root | Project-specific facts, conventions |
| User memory | `~/.claude/CLAUDE.md` | User preferences, cross-project |
| Extracted memories | `src/services/extractMemories/` | Auto-extracted from conversations |
| Team memory sync | `src/services/teamMemorySync/` | Shared team knowledge |

### Related

- `/memory` command for managing memories
- `remember` skill for persisting information
- `useMemoryUsage` hook for tracking memory size

---

## Coordinator (Multi-Agent)

**Location:** `src/coordinator/`

Orchestrates multiple agents working in parallel on different aspects of a task.

### How It Works

- `coordinatorMode.ts` manages the coordinator lifecycle
- `TeamCreateTool` and `TeamDeleteTool` manage agent teams
- `SendMessageTool` enables inter-agent communication
- `AgentTool` spawns sub-agents

Gated behind the `COORDINATOR_MODE` feature flag.

---

## Voice System

**Location:** `src/voice/` + `src/services/voice*.ts`

Push-to-talk voice input with real-time speech-to-text transcription. Gated behind `VOICE_MODE` feature flag and requires OAuth authentication.

### Architecture

```
Hold keybinding
      ↓
startRecording()  ──► native audio module (cpal)
                  ──► arecord (Linux fallback)
                  ──► SoX rec (final fallback)
      ↓
Audio chunks (Buffer)
      ↓
connectVoiceStream() → WebSocket → api.anthropic.com/voice_stream
      ↓
TranscriptText (interim) / TranscriptEndpoint (final)
      ↓
Text inserted into REPL input
```

### Recording Backend Fallback Chain

| Priority | Backend | Platform |
|----------|---------|---------|
| 1 | Native audio module (`cpal`) | macOS / Linux / Windows |
| 2 | `arecord` | Linux (ALSA) |
| 3 | SoX `rec` with silence detection | All (final fallback) |

Native module is lazy-loaded on first keypress (~1–8s cold, ~1s warm) to avoid blocking startup.

### WebSocket STT

- Connects to `api.anthropic.com/voice_stream` (not `claude.ai` — avoids CloudFlare TLS fingerprinting)
- Sends audio chunks as binary WebSocket frames
- KeepAlive sent every 8s to prevent idle timeout
- STT provider: `conversation_engine` (Deepgram Nova 3 via `tengu_cobalt_frost` gate)

### Message Types

| Message | Direction | Meaning |
|---------|-----------|---------|
| `TranscriptText` | Server → Client | Interim transcription (may revise) |
| `TranscriptEndpoint` | Server → Client | Final utterance boundary |
| `TranscriptError` | Server → Client | STT error |
| `CloseStream` | Server → Client | Stream ending; wait for final endpoint |

### Silence Detection

SoX fallback uses `silence` filter: **3% threshold, 2.0s duration** — auto-stops recording after 2 seconds of silence.

### Keyterms

`getVoiceKeyterms()` pulls context-aware hints to improve STT accuracy:
- Project root directory name
- Current git branch
- Recently opened file names

Sent as URL params on WebSocket connection.

### Key Files

| File | Location | Purpose |
|------|----------|---------|
| Feature gate | `src/voice/voiceModeEnabled.ts` | Enable/disable check + kill switch |
| Core service | `src/services/voice.ts` | Recording, WebSocket, stream lifecycle |
| STT streaming | `src/services/voiceStreamSTT.ts` | WebSocket protocol, message parsing, auto-finalize |
| Key terms | `src/services/voiceKeyterms.ts` | Context-aware STT hints |
| React hooks | `src/hooks/useVoice.ts`, `useVoiceEnabled.ts`, `useVoiceIntegration.tsx` | UI integration |
| Voice command | `src/commands/voice/` | `/voice` slash command |

### Feature Gates

| Gate | Type | Effect |
|------|------|--------|
| `VOICE_MODE` | Build-time | Master gate; strips voice code if false |
| `tengu_amber_quartz_disabled` | GrowthBook kill switch | Emergency off (defaults to false — voice works by default) |
| `tengu_cobalt_frost` | GrowthBook | Routes STT to Deepgram Nova 3 provider |

### Constraints

- OAuth-only — no API key, Bedrock, or Vertex support
- Uses mTLS for WebSocket connections
- WSL2+WSLg works (PulseAudio via RDP); WSL1 and Win10-WSL2 have no audio devices
- Bun WebSocket on Windows may spuriously fire `unexpected-response` with 101 — whitelisted

---

## Service Layer

**Location:** `src/services/`

External integrations and shared services.

| Service | Path | Purpose |
|---------|------|---------|
| **API** | `api/` | Anthropic SDK client, file uploads, bootstrap |
| **MCP** | `mcp/` | MCP client connections and tool discovery |
| **OAuth** | `oauth/` | OAuth 2.0 authentication flow |
| **LSP** | `lsp/` | Language Server Protocol manager |
| **Analytics** | `analytics/` | GrowthBook feature flags, telemetry |
| **Plugins** | `plugins/` | Plugin loader and marketplace |
| **Compact** | `compact/` | Conversation context compression |
| **Policy Limits** | `policyLimits/` | Organization rate limits/quota |
| **Remote Settings** | `remoteManagedSettings/` | Enterprise managed settings sync |
| **Token Estimation** | `tokenEstimation.ts` | Token count estimation |
| **Team Memory** | `teamMemorySync/` | Team knowledge synchronization |
| **Tips** | `tips/` | Contextual usage tips |
| **Agent Summary** | `AgentSummary/` | Agent work summaries |
| **Prompt Suggestion** | `PromptSuggestion/` | Suggested follow-up prompts |
| **Session Memory** | `SessionMemory/` | Session-level memory |
| **Magic Docs** | `MagicDocs/` | Documentation generation |
| **Auto Dream** | `autoDream/` | 后台自动整理和发散思维（空闲时运行） |
| **Settings Sync** | `settingsSync/` | 跨 Claude Code 环境同步用户设置和 Memory 文件 |
| **Tool Use Summary** | `toolUseSummary/` | 用 Haiku 为已完成的工具批次生成可读摘要，供 SDK 向客户端推送进度更新 |
| **x402** | `x402/` | x402 支付协议集成 |

---

## Buddy System (Virtual Companion)

**Location:** `src/buddy/`

A procedurally-generated virtual companion that sits alongside the input box and occasionally comments. The entire system is gated behind the `BUDDY` feature flag.

### Generation Logic

Each companion is **deterministically generated from the user's account UUID** — the same user always gets the same companion. Bones (appearance) are regenerated on every read to prevent config-editing exploits; only the soul (name + personality) is persisted.

```
User UUID → salt 'friend-2026-401' → Mulberry32 PRNG → CompanionBones
                                                          ↓
                                                    StoredCompanion (soul only)
                                                          ↓
                                                    Companion = bones + soul
```

### Data Structures

| Type | Location | Purpose |
|------|----------|---------|
| `CompanionBones` | `types.ts` | Deterministic appearance: rarity, species, eye, hat, shiny, stats |
| `CompanionSoul` | `types.ts` | Persisted: name + personality description (generated by LLM on first hatch) |
| `StoredCompanion` | `types.ts` | Written to config: soul + hatchedAt timestamp |
| `Companion` | `types.ts` | Final combined object shown to user |

### Species & Rarity

- **18 species**: duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk
- **5 rarity tiers**: Common (60%), Uncommon (25%), Rare (10%), Epic (4%), Legendary (1%)
- **7 hats**: crown, tophat, propeller, halo, wizard, beanie, tinyduck
- **6 eye types**: ·, ✦, ×, ◉, @, °
- **1% shiny chance**: sparkle variant of any species

### Stats

5 attributes scored 1–100, with rarity setting a minimum floor:

| Stat | Description |
|------|-------------|
| DEBUGGING | Problem-solving strength |
| PATIENCE | Tolerance for long tasks |
| CHAOS | Unpredictability factor |
| WISDOM | Knowledge depth |
| SNARK | Commentary style |

Legendary companions have a stat floor of 50; one peak stat (50–100) and one intentional dump stat.

### Sprite Rendering

ASCII art with 3 idle animation frames per species. The `{E}` placeholder is replaced with the companion's eye character at render time. Hat sprite is overlaid on frame 0.

### Canary Bypass

Species names are encoded via `String.fromCharCode()` to avoid triggering `scripts/excluded-strings.txt` — the build system's canary scanner greps compiled output, and "capybara" (a species name) collides with a model codename.

### Feature Gate

- `feature('BUDDY')` — master gate
- `isBuddyLive()` — returns true after 2026-04-01 (rolling by timezone)
- `config.companionMuted` — user can silence companion commentary

### Integration Points

- **Config**: Reads/writes `config.companion` (StoredCompanion)
- **System prompt**: `companionIntroText()` injects companion context as an attachment
- **UI**: `useBuddyNotification()` manages the teaser popup on startup

---

## Vim Mode

**Location:** `src/vim/`

A complete vim modal editing state machine for the REPL input box. Implements INSERT and NORMAL modes with operators, motions, text objects, registers, dot-repeat, and find-repeat.

### State Machine

```
INSERT ←──────────────────────────── Esc
  │                                    │
  │ i/a/I/A/o/O                        │
  ↓                                    │
NORMAL ── idle ── count ── operator ── find ── g ── replace
```

`VimState` is either `INSERT { insertedText }` or `NORMAL { command: CommandState }`.

`CommandState` has 13 variants for multi-key command sequences (e.g., `d` → operator state, `dw` → execute delete-word).

### Supported Commands

**Motions:**

| Key | Motion |
|-----|--------|
| h/l | character left/right |
| j/k | line down/up |
| w/b/e | word forward/backward/end |
| W/B/E | WORD variants (whitespace-delimited) |
| 0/^/$ | line start/first-non-blank/end |
| gg/G | file start/end |
| f/F/t/T | find character forward/backward |

**Operators:** `d` (delete), `c` (change), `y` (yank)

**Text Objects:** `iw/aw` (word), `i"/a"` (quotes), `i(/a(` (brackets), and variants for `[]`, `{}`, `<>`

**Special Commands:** `.` (dot-repeat), `;`/`,` (repeat find), `p/P` (paste), `x` (delete char), `~` (toggle case), `J` (join lines), `r` (replace char), `>>`/`<<` (indent)

### Implementation Highlights

- **Grapheme-safe**: All operations use `Intl.Segmenter` for correct emoji/diacritic handling
- **Immutable cursor**: Navigation via immutable `Cursor` objects with method chaining
- **cw/cW special case**: Matches vim's actual behavior (change to end of word, not start of next)
- **Count cap**: Maximum count of 10,000 to prevent DoS via `999999999j`
- **Dot-repeat**: Replays `RecordedChange` (9 variants capturing all change types and parameters)

### Feature Gate

No feature gate — vim mode is core functionality, toggled via `/vim` command or keybinding.

---

## Keybindings

**Location:** `src/keybindings/`

Manages keyboard shortcuts across 18 UI contexts with platform-specific defaults, user customization (gated), hot-reload, and validation.

### Contexts

18 named contexts: `Chat`, `Global`, `Settings`, `Diff`, `Tabs`, `Vim Normal`, `Vim Insert`, and more — each with independent binding tables.

### Architecture

```
~/.claude/keybindings.json  ──┐
                               ├──► merge ──► validate ──► ParsedBinding[]
Default bindings  ─────────────┘
                                                │
                                      chokidar file watcher
                                      (hot-reload on change)
```

### Key Types

| Type | Description |
|------|-------------|
| `ParsedKeystroke` | Normalized key + modifier flags (ctrl, alt, shift, meta, super) |
| `Chord` | Sequence of keystrokes (e.g., `ctrl+k ctrl+s`) |
| `ParsedBinding` | chord + action name + context |
| `KeybindingWarning` | Validation result with severity and suggestion |

### Platform-Specific Defaults

| Binding | macOS/Linux | Windows |
|---------|-------------|---------|
| Image paste | `ctrl+v` | `alt+v` (ctrl+v is system paste) |
| Mode cycle | `shift+tab` | `meta+m` (older Windows Terminal) |

### Reserved Shortcuts

| Category | Keys | Reason |
|----------|------|--------|
| Non-rebindable | ctrl+c, ctrl+d, ctrl+m | Interrupt / exit |
| Terminal reserved | ctrl+z, ctrl+\ | SIGTSTP / SIGQUIT signals |
| macOS reserved | cmd+c/v/x/q/w | OS-level intercepts |

### Actions

172 named actions (e.g., `app:interrupt`, `chat:submit`, `select:accept`) or `null` to unbind. Slash command bindings use `command:*` prefix.

### Validation

- Detects duplicate keys in JSON (JSON.parse silently drops earlier values — raw text parsing needed)
- Warns on reserved shortcut conflicts
- Normalizes modifier order for consistent comparison (sorted: ctrl, alt, shift, meta, super)

### Feature Gate

- `tengu_keybinding_customization_release` (GrowthBook): User customization enabled for Anthropic employees only; all others use defaults
- Several actions are conditionally included based on `VOICE_MODE`, `KAIROS`, `TERMINAL_PANEL` feature flags

### User Config Format

```json
[
  {
    "context": "Chat",
    "bindings": {
      "ctrl+enter": "chat:submit",
      "ctrl+l": null
    }
  }
]
```

Config location: `~/.claude/keybindings.json`. Hot-reloaded on save (500ms stability threshold).

---

## OAuth Authentication

**Location:** `src/services/oauth/`

OAuth 2.0 + PKCE flow for Claude.ai and Anthropic Console authentication. Handles browser-based and headless (manual code paste) authorization, token refresh, and profile fetching.

### Authorization Flow

```
1. Generate code_verifier (32 random bytes, base64url)
2. Generate code_challenge = SHA256(verifier), base64url
3. Generate state = 32 random bytes (CSRF protection)
4. Open browser → api.anthropic.com/oauth/authorize?...
5. AuthCodeListener captures redirect on localhost:PORT/callback
6. Validate state, extract auth code
7. POST /token → access_token + refresh_token
8. Fetch profile (subscription, rate limits, org info)
9. Save tokens to secure storage + config
```

> Routes through `api.anthropic.com` (not `claude.ai`) to avoid CloudFlare TLS fingerprinting that blocks non-browser clients.

### Token Types

| Scope | Use |
|-------|-----|
| Full OAuth scopes | Interactive sessions — full feature access |
| Inference-only | Long-lived tokens for SDK use; no UI features |

### Token Refresh

- Proactive refresh when `expiresAt` is within 5 minutes
- POST `/token` with `grant_type=refresh_token`
- Scopes can be **expanded** on refresh (server allows `ALLOWED_SCOPE_EXPANSIONS`)
- Profile round-trip is **skipped** if cached data exists — saves ~7M requests/day fleet-wide

### Profile Data

Fetched from `/api/oauth/profile` after successful token exchange:

| Field | Source |
|-------|--------|
| `subscriptionType` | `organization.organization_type` → max/pro/enterprise/team |
| `rateLimitTier` | `organization.rate_limit_tier` |
| `billingType` | `organization.billing_type` |
| `accountUUID` | `account.uuid` |
| `organizationUUID` | `organization.uuid` |

### Headless / Manual Flow

For SDK or CI environments without a browser:

1. Print authorization URL to stdout
2. User opens URL manually, authorizes, copies code from redirect
3. User pastes code into terminal
4. Same token exchange flow as automatic

### Key Files

| File | Purpose |
|------|---------|
| `client.ts` | Token exchange, refresh, profile fetch logic |
| `index.ts` | High-level auth orchestration, config integration |
| `auth-code-listener.ts` | Local HTTP server capturing the OAuth redirect |
| `crypto.ts` | PKCE helpers: base64url encode + SHA256 |
| `getOauthProfile.ts` | Profile endpoint fetch and parsing |

### Analytics Events

| Event | Trigger |
|-------|---------|
| `tengu_oauth_token_exchange_success` | Successful code exchange |
| `tengu_oauth_token_refresh_success/failure` | Token refresh result |
| `tengu_oauth_profile_fetch_success` | Profile fetch success |
| `tengu_oauth_auth_code_received` | Code captured (automatic flag indicates flow type) |

---

## See Also

- [Architecture](architecture.md) — How subsystems connect in the core pipeline
- [Tools Reference](tools.md) — Tools related to each subsystem
- [Commands Reference](commands.md) — Commands for managing subsystems
- [Exploration Guide](exploration-guide.md) — Finding subsystem source code
