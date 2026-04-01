# Architecture Overview — Claude Code Source

> Source: `src/` · ~1,884 files · ~512K lines · 34MB · Snapshot ngày 2026-03-31

---

## Mục lục

- [1. Tổng quan](#1-tổng-quan)
- [2. Entry Points](#2-entry-points)
- [3. Core Engine](#3-core-engine)
- [4. Tool System](#4-tool-system)
- [5. Command System](#5-command-system)
- [6. Multi-Agent Coordinator](#6-multi-agent-coordinator)
- [7. Memory System](#7-memory-system)
- [8. Service Layer](#8-service-layer)
- [9. Bridge System](#9-bridge-system)
- [10. Feature Flags](#10-feature-flags)
- [11. Dependency Graph](#11-dependency-graph)

---

## 1. Tổng quan

Claude Code là **CLI + REPL agentic tool** cho terminal, được viết hoàn toàn bằng TypeScript, chạy trên Bun runtime.

```
┌─────────────────────────────────────────────────────────────┐
│                        main.tsx                             │
│              (Commander.js + Ink/React)                    │
└─────────────────────────┬───────────────────────────────────┘
                          │ boot sequence
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
   entrypoints/      QueryEngine       services/
   init.ts           (~12K lines)      analytics/
                                       api/
                                       mcp/
                                       lsp/
                                       extractMemories/
                                       autoDream/
                                       teamMemorySync/
                                       plugins/
                                       compact/
```

### Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Bun |
| Language | TypeScript (strict mode) |
| Terminal UI | React 18 + Ink |
| CLI Parsing | Commander.js |
| Schema Validation | Zod v4 |
| AI API | Anthropic SDK |
| Observability | OpenTelemetry + gRPC |
| Feature Flags | GrowthBook |
| Auth | OAuth 2.0, JWT, macOS Keychain |

---

## 2. Entry Points

| File | Mô tả |
|---|---|
| `main.tsx` | React entry point chính — CLI parsing qua Commander.js, khởi tạo Ink renderer |
| `replLauncher.tsx` | REPL launcher — chế độ tương tác dòng lệnh |
| `setup.ts` | Setup logic — xử lý khởi động, auth, config initialization |
| `entrypoints/init.ts` | Application initialization pipeline |
| `entrypoints/mcp.ts` | MCP integration entry |
| `entrypoints/sdk/` | Agent SDK types & schemas |
| `entrypoints/sandboxTypes.ts` | Sandbox type definitions |

### Boot Sequence (từ `main.tsx`)

```typescript
// 1. Parallel prefetch — không blocking
startMdmRawRead()         // MDM settings (enterprise config)
startKeychainPrefetch()   // Secrets từ keychain

// 2. CLI argument parsing
const program = new Command()
program
  .name('claude')
  .argument('[prompt]', 'Initial prompt')
  .option('--resume', 'Resume previous session')
  .option('--dangerously-disable-permanent', ...)
  .option('--no-input', 'Non-interactive mode')
  .action(async (prompt, opts) => {
    // 3. Boot Ink renderer
    // 4. Init state via bootstrap/state.ts
    // 5. Load GrowthBook feature flags
    // 6. Start QueryEngine
  })
```

---

## 3. Core Engine

### 3.1 QueryEngine (`query.ts` ~12K lines — đây là phiên bản gốc trong repo)

**Core loop**: Nhận user input → build context → gọi Anthropic API → stream response → execute tools → repeat.

```
User Input
    │
    ▼
┌─────────────────────────────┐
│  buildSystemPrompt()        │  ← gắn memory, rules, system prompt
│  buildUserContext()         │  ← workspace files, recent changes
└─────────────┬───────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  anthropic.messages│
    │  .create()          │  ← streaming
    └─────────┬───────────┘
              │ text/event stream
              ▼
    ┌─────────────────────┐
    │  parseSSEStream()   │  ← xử lý Server-Sent Events
    │  handleTextDelta()  │  ← accumulate text
    │  handleContentBlock │  ← tool_use, thinking, etc.
    └─────────┬───────────┘
              │
       ┌──────┴──────┐
       │             │
   tool_use     thinking
       │             │
       ▼             ▼
  ┌────────────────────────┐
  │  Tool Executor Loop   │  ← đệ quy: gọi tool → gắn result → call LLM again
  └────────────┬───────────┘
               │ all tools done
               ▼
         Final Response
```

#### Thinking Mode (Extended Reasoning)

Claude Code hỗ trợ **extended thinking** — gọi nội bộ đến LLM với `thinking` block:

```typescript
// Signature protection: không để end-user thấy thinking nội bộ
const signature = crypto.randomUUID()
const protectedThinking = {
  type: 'thinking',
  thinking: encryptedContent,  // nội bộ, không exposed ra
  signature
}
```

#### Error Recovery (4 Layers)

| Layer | Trigger | Action |
|---|---|---|
| 1 | API error (rate limit, timeout) | Retry với exponential backoff |
| 2 | Context collapse (`prompt_too_long`) | Auto-compact — gọi `/compact` forked agent |
| 3 | Max output tokens exceeded | Gọi lại với higher limit |
| 4 | Persistent failure | Model fallback (Opus → Sonnet → Haiku) |

### 3.2 Task Management (`Task.ts`, `tasks.ts`)

Task được quản lý qua structured types:

```typescript
interface Task {
  id: string
  type: 'user' | 'system' | 'agent'
  status: 'pending' | 'in_progress' | 'completed' | 'failed'
  input: string
  output?: string
  createdAt: Date
  updatedAt: Date
  metadata: Record<string, unknown>
}
```

---

## 4. Tool System

### 4.1 Tool Registry (`tools.ts`)

```typescript
// Mỗi tool được register với metadata
export const tools = {
  BashTool: { schema: BashToolSchema, permission: 'dangerous' },
  FileReadTool: { schema: FileReadToolSchema, permission: 'default' },
  FileEditTool: { schema: FileEditToolSchema, permission: 'default' },
  GlobTool: { schema: GlobToolSchema, permission: 'default' },
  GrepTool: { schema: GrepToolSchema, permission: 'default' },
  WebFetchTool: { schema: WebFetchToolSchema, permission: 'default' },
  WebSearchTool: { schema: WebSearchToolSchema, permission: 'default' },
  AgentTool: { schema: AgentToolSchema, permission: 'plan' },
  MCPTool: { schema: MCPToolSchema, permission: 'default' },
  TaskCreateTool: { schema: TaskCreateToolSchema, permission: 'default' },
  TaskUpdateTool: { schema: TaskUpdateToolSchema, permission: 'default' },
  TeamCreateTool: { schema: TeamCreateToolSchema, permission: 'plan' },
  EnterPlanModeTool: { schema: EnterPlanModeToolSchema, permission: 'default' },
  ExitPlanModeTool: { schema: ExitPlanModeToolSchema, permission: 'default' },
  EnterWorktreeTool: { schema: EnterWorktreeToolSchema, permission: 'dangerous' },
  ExitWorktreeTool: { schema: ExitWorktreeToolSchema, permission: 'dangerous' },
  CronCreateTool: { schema: CronCreateToolSchema, permission: 'plan' },
  SkillTool: { schema: SkillToolSchema, permission: 'default' },
  NotebookEditTool: { schema: NotebookEditToolSchema, permission: 'default' },
  LSPTool: { schema: LSPToolSchema, permission: 'default' },
  SyntheticOutputTool: { schema: SyntheticOutputToolSchema, permission: 'default' },
  // ... ~20 tools nữa
}
```

### 4.2 Tool Base Class (`Tool.ts`)

Mỗi tool kế thừa base class với các capability:

```typescript
abstract class Tool<TInput, TOutput> {
  abstract name: string
  abstract description: string
  abstract schema: z.ZodType<TInput>

  // Permission level
  permission: ToolPermission = 'default'

  // Input validation
  validate(input: unknown): TInput { ... }

  // Execute
  abstract execute(input: TInput, context: ToolContext): Promise<TOutput>

  // Permission check
  async checkPermission(context: ToolContext): Promise<boolean> { ... }
}
```

### 4.3 Tool Permissions

| Level | Mô tả |
|---|---|
| `default` | User được hỏi trước khi execute lần đầu |
| `plan` | Chỉ chạy khi đang ở Plan Mode |
| `dangerous` | Write/Edit/Bash — luôn hỏi user hoặc cần `--dangerously-enable-all-commands` |
| `bypassPermissions` | Agent sub-tasks có thể bypass hoàn toàn |

### 4.4 Tool Implementation Locations

```
src/tools/
├── bash/                    # Shell command execution
├── fileRead/               # Read files (images, PDF, notebooks supported)
├── fileEdit/               # String replacement edits
├── fileWrite/              # Create/overwrite files
├── glob/                   # Pattern-based file search
├── grep/                   # Content search (ripgrep-powered)
├── webFetch/               # HTTP GET content
├── webSearch/              # Web search
├── agent/                  # Spawn sub-agents
├── skill/                  # Execute skills
├── mcp/                    # MCP protocol tools
├── lsp/                    # Language Server Protocol
├── task/                   # Task create/update
├── team/                   # Team management
├── planMode/               # Plan mode tools
├── worktree/               # Git worktree isolation
├── cron/                   # Scheduled triggers
├── syntheticOutput/        # Structured output
├── notebook/               # Jupyter notebook editing
└── ...
```

---

## 5. Command System

### 5.1 Command Registry (`commands.ts`)

```typescript
// Auto-discovery: scan commands/ directory
const commands = await Promise.all(
  commandsMeta.map(({ file, name }) =>
    import(`./commands/${file}`).then(m => ({ name, cmd: m.default }))
  )
)

// Slash command parsing
if (input.startsWith('/')) {
  const [cmdName, ...args] = input.slice(1).split(' ')
  const cmd = commands.find(c => c.name === cmdName)
  return cmd.execute(args)
}
```

### 5.2 Key Commands

| Command | Mô tả |
|---|---|
| `/commit` | Git commit với smart message generation |
| `/review` | Code review via sub-agent |
| `/compact` | Context compression — gọi compact forked agent |
| `/memory` | Memory management UI |
| `/tasks` | Task list management |
| `/mcp` | MCP server management |
| `/config` | Settings management |
| `/doctor` | Environment diagnostics |
| `/diff` | Git diff viewer |
| `/cost` | Token usage analysis |
| `/context` | Visualize current context window |
| `/resume` | Resume previous session |
| `/share` | Share session via link |
| `/skills` | Skill management |
| `/vim` | Vim mode |
| `/theme` | Theme customization |
| `/pr_comments` | PR comment viewer |
| `/desktop` | Switch to desktop app |
| `/mobile` | Switch to mobile app |

### 5.3 Command Implementation

```
src/commands/
├── commit/                  # Git commit command
├── review/                 # Code review
├── compact/                # Context compression
├── memory/                  # Memory management UI
├── tasks/                  # Task management
├── mcp/                    # MCP server management
├── config/                 # Settings
├── doctor/                 # Diagnostics
├── diff/                   # Git diff
├── cost/                   # Token analysis
├── context/                # Context visualization
├── resume/                 # Session resume
├── share/                  # Session sharing
├── skills/                # Skill management
├── vim/                    # Vim mode
├── theme/                  # Theming
├── pr_comments/            # PR comments
├── desktop/                # Desktop app
├── mobile/                 # Mobile app
└── ... (~50+ commands)
```

---

## 6. Multi-Agent Coordinator

Xem: [docs/2-Deep-Dives/MULTI-AGENT-COORDINATOR.md](../2-Deep-Dives/MULTI-AGENT-COORDINATOR.md)

**5 Task Types:**

| Type | Mô tả |
|---|---|
| `LocalAgentTask` | Chạy trong process hiện tại |
| `InProcessTeammateTask` | Chạy trong process, giao tiếp qua mailbox |
| `RemoteAgentTask` | Chạy trên Claude Code Remote (CCR) server |
| `DreamTask` | Background consolidation task |
| `LocalShellTask` | Shell command task |

**3 Execution Models:**

| Model | Hành vi |
|---|---|
| Sync Agent | Block cho đến khi hoàn thành |
| Async Agent | Fire-and-forget, kết quả qua task-notification XML |
| Teammate | Mailbox-based messaging |

**5 Isolation Modes:**

| Mode | Cơ chế |
|---|---|
| `worktree` | Git worktree isolation |
| `remote` | Claude Code Remote server |
| `in-process` | Same process, sandboxed |

**Key Pattern — Forked Subagent:**
```typescript
// Chia sẻ prompt cache với parent cho background tasks
const forkedAgent = await runForkedAgent({
  task: 'extractMemories',  // hoặc 'autoDream', 'compact'
  sharedPromptCache: true,   // key optimization
})
```

**Services dùng Forked Pattern:**
- `extractMemories` — auto-write memory sau mỗi turn
- `autoDream` — nightly consolidation
- `compact` — context compression
- `AgentSummary` — summarize agent output
- `MagicDocs` — auto-generate docs
- `SessionMemory` — context window summary

---

## 7. Memory System

Xem: [docs/2-Deep-Dives/MEMORY-SYSTEM.md](../2-Deep-Dives/MEMORY-SYSTEM.md)

**5 Subsystems:**

```
System Prompt Injection     ← MEMORY.md luôn được gắn vào system prompt
Query-Time Recall           ← chỉ đọc file liên quan (≤5 files)
Extract Memories            ← AI viết memory sau mỗi turn (forked agent)
Auto-Dream                   ← nightly consolidation (forked agent)
Team Memory Sync             ← sync qua server
```

**Disk Layout:**
```
~/.claude/projects/{git-root}/memory/
├── MEMORY.md              # Index (max 200 dòng, 25KB)
├── user_role.md           # Topic: user info
├── feedback_*.md          # Topic: rules/preferences
├── project_*.md           # Topic: project context
├── reference_*.md         # Topic: external system pointers
├── team/                  # Team-shared memories
│   ├── MEMORY.md
│   └── team_policy.md
└── logs/                  # KAIROS/Assistant mode only
```

**Memory Types:**

| Type | Privacy | Content |
|---|---|---|
| `user` | Private | Skills, preferences |
| `feedback` | Private/Team | Rules (do/don't) |
| `project` | Bias team | Deadlines, decisions |
| `reference` | Team | Linear, Grafana pointers |

---

## 8. Service Layer

Xem: [docs/2-Deep-Dives/SERVICE-LAYER.md](../2-Deep-Dives/SERVICE-LAYER.md)

**7 Architectural Patterns:**

| Pattern | Ví dụ | Mô tả |
|---|---|---|
| Queue Buffer | Analytics | Events queue → drain async via `queueMicrotask()` |
| Closure Factory | LSP, OAuth, Compact | State encapsulated in closure |
| Forked Subagent | extractMemories, autoDream | AI xử lý thay hardcode logic |
| Feature Gate Caching | Settings sync | Cached values có thể stale |
| Distributed Lock | autoDream | File `mtime` làm lock |
| Lazy Loading | Voice | Load NAPI chỉ khi cần |
| Marker Types | Analytics metadata | Compile-time PII enforcement |

**Dependency Graph:**
```
analytics/ (ZERO DEPS — BASE)
     ↑
api/client.ts ◄─── oauth/
     │
     ▼ forkedAgent
autoDream / extractMemories / SessionMemory / compact
     │
     ▼
remoteManagedSettings / settingsSync / teamMemorySync
     │
     ▼
INDEPENDENT: mcp/, lsp/, plugins/, voice/, claudeAiLimits/
```

**Key Services:**

| Service | Mô tả |
|---|---|
| `api/` | Anthropic API client |
| `mcp/` | MCP multi-transport (SSE, Stdio, HTTP, WS, In-process, SDK) |
| `oauth/` | OAuth 2.0 flow |
| `lsp/` | Language Server Protocol manager |
| `analytics/` | OpenTelemetry + GrowthBook |
| `plugins/` | Plugin loader (marketplace) |
| `compact/` | Context compression |
| `extractMemories/` | Auto memory extraction |
| `autoDream/` | Nightly memory consolidation |
| `teamMemorySync/` | Team memory server sync |
| `voice/` | Voice input (3 fallback: NAPI → SoX → ALSA) |
| `claudeAiLimits/` | Rate limiting |

---

## 9. Bridge System

Xem: [docs/2-Deep-Dives/BRIDGE-SYSTEM.md](../2-Deep-Dives/BRIDGE-SYSTEM.md)

Hệ thống cầu nối giao tiếp hai chiều giữa **IDE extensions** và **Claude Code CLI**.

```
┌──────────────┐     Bridge Protocol      ┌──────────────────┐
│  VS Code      │ ◄──────────────────────► │  Claude Code CLI │
│  Extension    │   JSON-RPC over stdin   │  (Bridge Mode)   │
├──────────────┤                          ├──────────────────┤
│  JetBrains    │                          │  Remote Session  │
│  Plugin       │                          │  (CCR Server)    │
└──────────────┘                          └──────────────────┘
```

**Key Files:**

| File | Mô tả |
|---|---|
| `bridgeMain.ts` | Bridge main loop |
| `bridgeMessaging.ts` | JSON-RPC message protocol |
| `bridgePermissionCallbacks.ts` | Permission request callbacks |
| `replBridge.ts` | REPL session bridge |
| `jwtUtils.ts` | JWT authentication |
| `sessionRunner.ts` | Session management |

---

## 10. Feature Flags

Claude Code dùng **GrowthBook** cho feature gates — cho phép toggle features mà không cần deploy:

```typescript
// src/services/analytics/featureFlags.ts
export const feature = (flag: FeatureFlag): boolean => {
  return growthBook.isOn(flag)
}
```

**Notable Flags:**

| Flag | Mục đích | Default |
|---|---|---|
| `tengu_passport_quail` | Enable extract memories | `false` |
| `tengu_slate_thimble` | Extract in non-interactive | `false` |
| `tengu_bramble_lintel` | Extract throttle (every N turns) | `1` |
| `tengu_onyx_plover` | Auto-dream config | `24h, 5 sessions` |
| `tengu_herring_clock` | Team memory | `false` |
| `KAIROS` | Assistant mode (daily logs) | gated |
| `PROACTIVE` | Proactive agent mode | gated |
| `BRIDGE_MODE` | IDE bridge mode | gated |
| `DAEMON` | Background daemon | gated |
| `VOICE_MODE` | Voice input | gated |

**Dead Code Elimination qua `bun:bundle`:**

```typescript
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
// → bị stripped hoàn toàn khi flag = false
```

---

## 11. Dependency Graph

```
                    ┌──────────────┐
                    │   analytics/ │  ZERO deps — BASE
                    └───────┬───────┘
                            │ tất cả services log vào đây
            ┌───────────────┼───────────────┐
            ▼               ▼                ▼
     ┌────────────┐  ┌──────────┐   ┌──────────────┐
     │  api/client │  │  oauth/  │   │ growthbook   │
     └─────┬───────┘  └──────────┘   └──────────────┘
           │                              ▲
           ▼            FORKED AGENT      │
    ┌──────────────────────────────┐       │
    │ extractMemories / autoDream  │──────┘
    │ SessionMemory / compact      │
    └──────────────┬───────────────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼              ▼
┌────────────┐ ┌──────────┐ ┌─────────────┐
│remoteMgd   │ │settings  │ │teamMemory   │
│Settings    │ │sync      │ │Sync         │
└────────────┘ └──────────┘ └─────────────┘
     │
     ▼
┌────────────────────────────────────────────┐
│         INDEPENDENT SERVICES               │
│  mcp/ (6 transports)  lsp/ (closure)       │
│  plugins/  voice/ (lazy)  claudeAiLimits/  │
└────────────────────────────────────────────┘
```

---

## Key Design Principles

### 1. Async-First, Functional

Services dùng **closure factories** thay vì classes — state hoàn toàn encapsulated:

```typescript
// Closure factory pattern
const createLspService = () => {
  let state = { connections: new Map(), pendingRequests: [] }
  return {
    connect: (uri: string) => { /* ... */ },
    disconnect: (uri: string) => { /* ... */ },
  }
}
```

### 2. AI Uses AI

Thay vì hardcode logic phức tạp, Claude Code **spawn sub-agents** để xử lý:
- extractMemories: AI quyết định gì cần nhớ
- autoDream: AI merge + cleanup memories
- compact: AI compress context
- SessionMemory: AI summarize conversation

### 3. Parallelism is a Superpower

```typescript
// Boot: parallel prefetch
startMdmRawRead()  // không blocking boot
startKeychainPrefetch()

// Runtime: independent tasks
const results = await Promise.all([
  agent1.run({ task: 'analyze', run_in_background: true }),
  agent2.run({ task: 'review',  run_in_background: true }),
  agent3.run({ task: 'test',    run_in_background: true }),
])
```

### 4. Immutable Patterns

Tuân thủ nghiêm ngặt immutable data — tất cả transforms tạo **new objects**, không mutate existing state.

### 5. Zero-Cost Abstraction

Feature flags + `bun:bundle` dead code elimination → features không dùng → không có runtime overhead.

---

## File Size Reference

| File | Dòng | Mô tả |
|---|---|---|
| `query.ts` | ~12,700 | Core query engine |
| `main.tsx` | ~4,683 | Entry point + CLI |
| `Tool.ts` | ~792 | Tool base types |
| `commands.ts` | ~754 | Command registry |
| `setup.ts` | ~477 | Boot sequence |
| `history.ts` | ~464 | History management |
| `context.ts` | ~189 | Context building |

---

> **Nguồn**: Source maps từ npm package ngày 2026-03-31. Bản quyền thuộc về Anthropic. Repository này phục vụ nghiên cứu bảo mật phòng thủ.
