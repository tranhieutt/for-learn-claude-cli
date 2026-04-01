# Deep Dive: Tool System — 40+ Tools Architecture

> Source: `src/tools.ts` + `src/Tool.ts` + `src/tools/`

---

## Mục lục

- [1. Tool Overview](#1-tool-overview)
- [2. Base Tool Architecture](#2-base-tool-architecture)
- [3. Tool Permission System](#3-tool-permission-system)
- [4. Tool Registry](#4-tool-registry)
- [5. Tool Implementations](#5-tool-implementations)
- [6. Tool Discovery & Lazy Loading](#6-tool-discovery--lazy-loading)
- [7. Input Validation](#7-input-validation)
- [8. Tool Concurrency Model](#8-tool-concurrency-model)

---

## 1. Tool Overview

Claude Code có **40+ tools** mà LLM có thể gọi — mỗi tool là một independent module với schema, permissions, và execution logic riêng.

```
┌──────────────────────────────────────────────────────────┐
│                    Tool System                            │
│                                                          │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐ │
│  │ Permission   │   │   Registry   │   │   Executor   │ │
│  │   Checker    │◄──│  (tools.ts)  │◄──│   (query.ts) │ │
│  └──────────────┘   └──────────────┘   └──────────────┘ │
│                                                          │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Tool Implementations (src/tools/)                   │ │
│  │  bash/  fileRead/  fileEdit/  fileWrite/  glob/      │ │
│  │  grep/  webFetch/  webSearch/  agent/  mcp/  lsp/   │ │
│  │  task/  team/  planMode/  worktree/  cron/  skill/  │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Base Tool Architecture

### Tool Base Class

```typescript
// src/Tool.ts (~792 lines) — Base type definitions
abstract class Tool<
  TInput extends z.ZodType,
  TOutput = unknown
> {
  abstract readonly name: string
  abstract readonly description: string
  abstract readonly schema: TInput

  // Permission level
  permission: ToolPermission = 'default'

  // Input validation
  validate(input: unknown): z.infer<TInput> {
    const result = this.schema.safeParse(input)
    if (!result.success) {
      throw new ToolValidationError(this.name, result.error)
    }
    return result.data
  }

  // Execute — override này
  abstract execute(
    input: z.infer<TInput>,
    context: ToolContext
  ): Promise<TOutput>

  // Dry run — preview without execution
  async dryRun?(input: z.infer<TInput>): Promise<string>

  // Permission check
  async checkPermission(context: ToolContext): Promise<boolean> {
    return permissionManager.check(this.permission, context)
  }
}
```

### Tool Context

```typescript
interface ToolContext {
  sessionId: string
  workingDirectory: string
  gitRoot: string | null
  permissionMode: PermissionMode
  isPlanMode: boolean
  isSubagent: boolean
  abortSignal: AbortSignal
  analytics: AnalyticsContext
}
```

---

## 3. Tool Permission System

### Permission Levels

| Level | Mô tả | Auto-approve in `auto` mode |
|---|---|---|
| `default` | User được hỏi trước lần đầu | ❌ |
| `plan` | Chỉ chạy khi Plan Mode | ❌ |
| `dangerous` | Write/Edit/Bash — luôn hỏi hoặc cần `--dangerously-enable-all-commands` | ❌ |
| `bypass` | Sub-agents có thể bypass hoàn toàn | ✅ |

### Safety Rules

```typescript
// Auto-approve chỉ cho operations không destructive
const SAFETY_RULES: Record<string, {
  autoApproved: boolean
  paths?: string[]           // Allowed paths (glob)
  commands?: string[]        // Allowed commands (bash)
}> = {
  Read: { autoApproved: true },
  Glob: { autoApproved: true },
  Grep: { autoApproved: true },
  Bash: {
    autoApproved: false,
    commands: ['ls', 'git', 'cat', 'head', 'tail', 'wc', 'stat', 'find', 'grep']
    // NOT: rm, mv, dd, chmod, chown, shutdown...
  },
  Write: {
    autoApproved: false,
    paths: ['!**/.git/**', '!**/node_modules/**']  // Avoid overwriting git/module files
  },
  Edit: {
    autoApproved: false,
    paths: ['!**/.git/**', '!**/node_modules/**']
  },
  Agent: { autoApproved: false },
  Team: { autoApproved: false },
}
```

### Permission Flow

```
Tool Call Request
       │
       ▼
┌─────────────────────┐
│ Check PermissionMode │
└─────────┬───────────┘
          │
    ┌─────┴──────┐
    │            │
 bypass/      default/
 auto        plan mode
    │            │
    ▼            ▼
  Approved    ┌─────────────────────┐
              │ Is Plan Mode?       │
              └──────────┬──────────┘
                   ┌────┴────┐
                   │         │
                  Yes        No
                   │         │
                   ▼         ▼
              Approved   ┌─────────────────┐
                         │  Prompt User    │
                         │  (Interactive)  │
                         └────────┬────────┘
                                  │
                         ┌────────┴────────┐
                         │                 │
                     Approved           Denied
                         │                 │
                         ▼                 ▼
                    Execute          Return Error
```

---

## 4. Tool Registry

```typescript
// src/tools.ts (~389 lines) — Central registry
export const toolRegistry = new Map<string, ToolRegistration>()

interface ToolRegistration {
  tool: Tool<any, any>
  schema: z.ZodType
  permission: ToolPermission
  experimental?: boolean
  tools?: string[]  // Sub-tools (MCP)
}

// Auto-discovery — scan src/tools/ directory
async function registerTools() {
  const toolDirs = await fs.readdir('src/tools/')

  for (const dir of toolDirs) {
    const module = await import(`./tools/${dir}/index.ts`)

    if (module.default) {
      // Single tool export
      registerTool(module.default)
    } else if (module.tools) {
      // Multiple tools export (MCP servers)
      for (const tool of module.tools) {
        registerTool(tool)
      }
    }
  }
}

function registerTool(tool: Tool<any, any>) {
  toolRegistry.set(tool.name, {
    tool,
    schema: tool.schema,
    permission: tool.permission,
  })
}
```

---

## 5. Tool Implementations

### Read Tools

| Tool | Schema | Mô tả |
|---|---|---|
| `ReadTool` | `{ file_path, offset?, limit? }` | Đọc file, hỗ trợ images, PDF, notebooks |
| `GlobTool` | `{ pattern, path? }` | Pattern-based file search |
| `GrepTool` | `{ pattern, path?, glob?, context? }` | Content search (ripgrep-powered) |
| `WebFetchTool` | `{ url, prompt? }` | Fetch + process URL content |

### Write Tools

| Tool | Schema | Mô tả |
|---|---|---|
| `WriteTool` | `{ file_path, content }` | Create/overwrite file |
| `EditTool` | `{ file_path, old_string, new_string }` | String replacement |
| `BashTool` | `{ command, timeout?, cwd? }` | Shell command execution |

### Agent Tools

| Tool | Schema | Mô tả |
|---|---|---|
| `AgentTool` | `{ prompt, model?, subagent_type?, run_in_background? }` | Spawn sub-agent |
| `SkillTool` | `{ skill, args? }` | Execute skill |
| `MCPTool` | `{ server, tool, args }` | Call MCP server tool |
| `LSPTool` | `{ action, uri, symbol? }` | LSP symbol/definition lookup |

### Management Tools

| Tool | Schema | Mô tả |
|---|---|---|
| `TaskCreateTool` | `{ title, description?, priority? }` | Tạo task |
| `TaskUpdateTool` | `{ id, status?, priority? }` | Cập nhật task |
| `TeamCreateTool` | `{ name, agents }` | Tạo agent team |
| `TeamDeleteTool` | `{ name }` | Xóa team |
| `SendMessageTool` | `{ to, content, type? }` | Inter-agent messaging |

### Mode Tools

| Tool | Schema | Mô tả |
|---|---|---|
| `EnterPlanModeTool` | `{ prompt? }` | Bật plan mode |
| `ExitPlanModeTool` | `{}` | Tắt plan mode |
| `EnterWorktreeTool` | `{ name?, base_branch? }` | Tạo worktree |
| `ExitWorktreeTool` | `{ action }` | Rời worktree (keep/remove) |
| `CronCreateTool` | `{ cron, prompt }` | Lên lịch task |
| `SyntheticOutputTool` | `{ schema, prompt }` | Structured output |

### Editor Tools

| Tool | Schema | Mô tả |
|---|---|---|
| `NotebookEditTool` | `{ notebook_path, cell_id?, edit_mode, new_source? }` | Jupyter notebook |

---

## 6. Tool Discovery & Lazy Loading

```typescript
// Deferred tool discovery — không load tất cả tool ngay startup
const toolDiscovery = {
  // Eager: load ngay (tools thường dùng)
  eager: ['Read', 'Write', 'Edit', 'Glob', 'Grep', 'Bash'],

  // Lazy: load khi cần (MCP, LSP, rarely-used tools)
  lazy: ['MCP', 'LSP', 'Skill', 'Notebook', 'Voice'],

  // Dynamic: discover tại runtime (MCP servers)
  dynamic: async () => {
    const mcpServers = await loadMcpServerConfigs()
    return mcpServers.flatMap(server => server.tools)
  }
}

// Lazy load khi LLM request
async function loadTool(name: string): Promise<Tool> {
  if (toolCache.has(name)) return toolCache.get(name)!

  const tool = await import(`./tools/${name.toLowerCase()}/index.ts`)
  toolCache.set(name, tool.default)
  return tool.default
}
```

---

## 7. Input Validation

```typescript
// Tất cả tools dùng Zod v4 cho input validation
const BashToolSchema = z.object({
  command: z.string().min(1).describe('Shell command to execute'),
  timeout: z.number().int().positive().max(300000).optional()
    .describe('Timeout in milliseconds (max 5 minutes)'),
  cwd: z.string().optional().describe('Working directory'),
  ENVIRONMENT: z.record(z.string()).optional()
    .describe('Environment variables'),
})

class BashTool extends Tool<typeof BashToolSchema> {
  name = 'Bash'
  description = 'Execute shell command'
  schema = BashToolSchema
  permission = 'dangerous'

  async execute(input, context) {
    const { command, timeout, cwd } = this.validate(input)

    // Command injection prevention
    if (containsDangerousPattern(command)) {
      throw new ToolSafetyError('Dangerous command pattern detected')
    }

    return this.runCommand(command, { timeout, cwd, signal: context.abortSignal })
  }
}
```

---

## 8. Tool Concurrency Model

### Parallel Execution

```typescript
// Claude Code gửi multiple tool_use trong 1 LLM response
// → Execute in parallel nếu không phụ thuộc nhau
interface ToolExecutionPlan {
  groups: ToolGroup[]
}

interface ToolGroup {
  tools: ToolCall[]
  execution: 'parallel' | 'sequential'
  dependsOn?: string[]  // tool_use IDs
}

// Example: parallel reads, sequential writes
const plan: ToolExecutionPlan = {
  groups: [
    { tools: [read1, read2, grep1], execution: 'parallel' },  // Không phụ thuộc
    { tools: [write1], execution: 'sequential' },             // Depends on reads
    { tools: [bash1], execution: 'sequential', dependsOn: [write1.id] } // Depends on write
  ]
}

async function executePlan(plan: ToolExecutionPlan) {
  for (const group of plan.groups) {
    if (group.execution === 'parallel') {
      await Promise.all(group.tools.map(t => executeTool(t)))
    } else {
      for (const tool of group.tools) {
        if (group.dependsOn) {
          await waitForTools(group.dependsOn)
        }
        await executeTool(tool)
      }
    }
  }
}
```

### Max Recursion Limit

```typescript
const MAX_TOOL_RECURSION = 50

class ToolExecutor {
  private depth = 0

  async execute(toolCall: ToolCall): Promise<ToolResult> {
    if (this.depth > MAX_TOOL_RECURSION) {
      throw new Error(
        `Tool recursion limit (${MAX_TOOL_RECURSION}) exceeded. ` +
        `Possible infinite loop in tool ${toolCall.name}`
      )
    }

    this.depth++
    try {
      return await this.runTool(toolCall)
    } finally {
      this.depth--
    }
  }
}
```

---

## Key Files Reference

| File | Lines | Trách nhiệm |
|---|---|---|
| `src/Tool.ts` | ~792 | Base class, types, validation |
| `src/tools.ts` | ~389 | Registry, discovery, lazy loading |
| `src/tools/bash/` | — | Shell execution |
| `src/tools/fileRead/` | — | File reading |
| `src/tools/fileEdit/` | — | String replacement |
| `src/tools/fileWrite/` | — | File creation/overwrite |
| `src/tools/glob/` | — | Pattern search |
| `src/tools/grep/` | — | Content search (ripgrep) |
| `src/tools/webFetch/` | — | URL fetching |
| `src/tools/webSearch/` | — | Web search |
| `src/tools/agent/` | — | Sub-agent spawning |
| `src/tools/mcp/` | — | MCP protocol tools |
| `src/tools/lsp/` | — | Language Server Protocol |
| `src/tools/task/` | — | Task management |
| `src/tools/team/` | — | Team management |
| `src/tools/planMode/` | — | Plan mode tools |
| `src/tools/worktree/` | — | Git worktree isolation |
| `src/tools/cron/` | — | Scheduled triggers |
| `src/tools/syntheticOutput/` | — | Structured output |
| `src/tools/notebook/` | — | Jupyter notebook |

---

> **Nguồn**: Source maps từ npm package ngày 2026-03-31. Bản quyền thuộc về Anthropic.
