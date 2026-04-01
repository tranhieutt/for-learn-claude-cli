# Deep Dive: Multi-Agent Coordinator

> Source: `src/coordinator/` + `src/AgentTool` + `src/services/`

---

## Mục lục

- [1. Tổng quan](#1-tổng-quan)
- [2. 5 Task Types](#2-5-task-types)
- [3. 3 Execution Models](#3-3-execution-models)
- [4. AgentTool — Spawning Sub-Agents](#4-agenttool--spawning-sub-agents)
- [5. Forked Pattern — AI Calls AI](#5-forked-pattern--ai-calls-ai)
- [6. Isolation Modes](#6-isolation-modes)
- [7. Message Passing](#7-message-passing)
- [8. Teams & Swarms](#8-teams--swarms)
- [9. Concurrency Philosophy](#9-concurrency-philosophy)

---

## 1. Tổng quan

Multi-agent coordinator quản lý việc spawn và điều phối nhiều AI agents — cả synchronous (block) và asynchronous (fire-and-forget).

```
┌─────────────────────────────────────────────────────────────┐
│              Claude Code (Parent Agent)                      │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐  │
│  │ LocalAgent   │ │ RemoteAgent  │ │ InProcessTeammate   │  │
│  │ (worktree)   │ │ (CCR Server) │ │ (mailbox messaging) │  │
│  └──────┬───────┘ └──────┬───────┘ └──────────┬───────────┘  │
│         │                │                     │              │
│    ┌────┴────┐      ┌────┴────┐         ┌────┴────┐         │
│    │ Worker  │      │ Worker  │         │ Mailbox │         │
│    │ (async) │      │ (async) │         │ Queue   │         │
│    └─────────┘      └─────────┘         └─────────┘         │
└─────────────────────────────────────────────────────────────┘
```

**Coordinator Mode:** Spawns parallel workers, merges results via XML notifications.

---

## 2. 5 Task Types

| Type | Mô tả | Khi nào dùng |
|---|---|---|
| `LocalAgentTask` | Chạy trong process hiện tại | Default, tốc độ cao |
| `InProcessTeammateTask` | Chạy trong process, giao tiếp qua mailbox | Teammate agents |
| `RemoteAgentTask` | Chạy trên Claude Code Remote (CCR) server | Heavy/parallel work |
| `DreamTask` | Background consolidation task | autoDream |
| `LocalShellTask` | Shell command task | System commands |

---

## 3. 3 Execution Models

### Model 1: Sync Agent — Block cho đến khi hoàn thành

```typescript
// User đợi kết quả — blocking
const result = await runAgent({
  type: 'LocalAgentTask',
  model: 'sonnet',
  task: 'Analyze this codebase for security vulnerabilities',
})

console.log(result.summary) // Kết quả có ngay
```

### Model 2: Async Agent — Fire-and-forget

```typescript
// Không block — kết quả gửi qua task-notification XML
const agent = await runAgent({
  type: 'LocalAgentTask',
  model: 'sonnet',
  task: 'Run comprehensive tests',
  run_in_background: true,  // Key flag
})

// User tiếp tục làm việc khác
// Khi agent done → notification được gửi
// Claude tự đọc và xử lý kết quả
```

### Model 3: Teammate — Mailbox-based messaging

```typescript
// Agent sống trong process, giao tiếp qua messages
const teammate = await createTeammate({
  type: 'InProcessTeammateTask',
  name: 'reviewer',
  role: 'Senior code reviewer',
})

// Gửi message
await teammate.send({
  type: 'task',
  content: 'Review PR #123',
})

// Lắng nghe response
teammate.on('message', (msg) => {
  handleTeammateResponse(msg)
})
```

---

## 4. AgentTool — Spawning Sub-Agents

```typescript
interface AgentToolInput {
  prompt: string              // Task description
  model?: 'opus' | 'sonnet' | 'haiku'  // Override model
  subagent_type?: string       // Agent type (code-reviewer, planner, etc.)
  run_in_background?: boolean // Async execution
  isolation?: 'worktree' | 'remote' | 'in-process'
  tools?: string[]            // Restrict available tools
  system_prompt?: string      // Override system prompt
}

// Ví dụ: Spawn reviewer agent
const result = await runAgent({
  type: 'LocalAgentTask',
  prompt: `
    Review the code in src/auth/
    Focus on: security, performance, error handling
    Return a structured report with severity ratings
  `,
  model: 'sonnet',
  tools: ['Read', 'Grep', 'Glob'],  // Restrict — read-only
  subagent_type: 'code-reviewer',
})
```

### Agent Tool Permissions

| Mode | Description |
|---|---|
| Default parent tools | Agent được cấp tất cả tools của parent |
| Restricted tools | Chỉ được dùng subset đã chỉ định |
| Sandboxed | Write/Edit chỉ trong worktree directory |
| Bypass | Sub-agents có thể bypass permission prompts |

---

## 5. Forked Pattern — AI Calls AI

Pattern đặc trưng nhất của Claude Code: **spawn AI agent để xử lý thay vì hardcode logic**.

```
Main Agent (User-facing)
    │
    │ Turn end → handleStopHooks()
    │
    ▼
runForkedAgent({ task: 'extractMemories' })
    │
    ├── Shares prompt cache với parent
    ├── Tool permissions: read-only + memory-dir write
    └── Không block — kết quả ghi vào disk

Services dùng Forked Pattern:
┌─────────────────────┬──────────────────────────────────────────┐
│ extractMemories     │ AI quyết định gì cần nhớ                │
│ autoDream           │ AI merge + cleanup memories              │
│ compact             │ AI compress context                       │
│ AgentSummary        │ AI summarize agent output                 │
│ MagicDocs           │ AI generate documentation                 │
│ SessionMemory       │ AI summarize conversation                 │
└─────────────────────┴──────────────────────────────────────────┘
```

```typescript
// Forked agent chia sẻ prompt cache với parent
async function runForkedAgent(params: {
  task: ForkedTaskType
  sharedPromptCache?: boolean  // Key optimization
  context?: ForkContext
}): Promise<void> {
  const agent = await spawnAgent({
    type: 'LocalAgentTask',
    model: params.model ?? 'sonnet',
    systemPrompt: FORKED_AGENT_PROMPTS[params.task],
    sharedPromptCache: true,  // Chia sẻ cache — giảm cost
    tools: getForkedAgentTools(params.task),  // Sandboxed tools
  })
}
```

---

## 6. Isolation Modes

| Mode | Cơ chế | Use case |
|---|---|---|
| `worktree` | Git worktree isolation | Risky refactoring |
| `remote` | CCR server | Heavy parallel work |
| `in-process` | Same process, sandboxed | Default |

### Worktree Isolation

```typescript
// Khi refactoring nguy hiểm — chạy trong worktree riêng
const result = await runAgent({
  type: 'LocalAgentTask',
  isolation: 'worktree',
  task: 'Migrate from REST to GraphQL',
  tools: ['Read', 'Write', 'Edit', 'Bash', 'Glob'],
})

// Main codebase KHÔNG bị ảnh hưởng
// Agent làm việc trong .git/worktrees/claude-refactor-xxx/
// Khi xong → review → merge hoặc discard
```

---

## 7. Message Passing

Claude Code hỗ trợ **rich message passing** giữa các agents:

```typescript
// Resume agents
await coordinator.resumeAgent(agentId, {
  content: 'Continue from where you left off',
})

// Direct messages (teammate-to-teammate)
await teammate.send({
  type: 'message',
  to: 'reviewer',
  content: 'Found an issue in auth.ts, check it out',
})

// Broadcast
await coordinator.broadcast({
  type: 'broadcast',
  content: 'All agents: stop current tasks, new priority task incoming',
})

// Structured messages
await agent.send({
  type: 'tool_result',
  toolUseId: 'tool_abc123',
  content: { status: 'success', output: '...', logs: '...' },
})
```

---

## 8. Teams & Swarms

```typescript
// Tạo team gồm nhiều agents
const team = await createTeam({
  name: 'release-team',
  agents: [
    { name: 'writer', role: 'Write code' },
    { name: 'reviewer', role: 'Review code' },
    { name: 'tester', role: 'Run tests' },
  ],
  planApprovalGate: true,  // Require plan approval before execution
})

// Plan approval gate
team.on('planProposed', async (plan) => {
  const approved = await promptUser(plan)
  if (approved) {
    team.executePlan(plan)
  }
})
```

### Orchestration Pattern

```
User: "Deploy v1.2.0"
     │
     ▼
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Builder  │────►│ Reviewer │────►│ Tester  │
│ (async)  │     │ (async)  │     │ (async) │
└──────────┘     └──────────┘     └──────────┘
     │               │                 │
     └───────────────┼─────────────────┘
                     ▼
              Coordinator merges
              results, decides next step
```

---

## 9. Concurrency Philosophy

> **"Parallelism is your superpower. Workers are async. Launch independent workers concurrently whenever possible."**

```typescript
// ✅ ĐÚNG: Independent tasks → parallel
const [analysis, review, tests] = await Promise.all([
  runAgent({ type: 'LocalAgentTask', task: 'analyze', run_in_background: true }),
  runAgent({ type: 'LocalAgentTask', task: 'review',  run_in_background: true }),
  runAgent({ type: 'LocalAgentTask', task: 'test',    run_in_background: true }),
])

// ❌ SAI: Sequential khi không cần
const result1 = await runAgent({ task: 'analyze' })
const result2 = await runAgent({ task: 'review' })  // Đợi analyze xong mới chạy
```

### When to Use Each Model

| Scenario | Model |
|---|---|
| Parallel code analysis | Async Agent (background) |
| Multi-agent code review | Async Agent (background) |
| Heavy refactoring | Worktree Isolation |
| Tool result summary | Forked Subagent |
| Team task coordination | Teammate + Message Passing |
| User đợi kết quả | Sync Agent |

---

> **Nguồn**: Source maps từ npm package ngày 2026-03-31. Bản quyền thuộc về Anthropic.
