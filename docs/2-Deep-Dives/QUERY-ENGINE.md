# Deep Dive: QueryEngine — Central Orchestrator

> Source: `src/query.ts` (~12,700 lines) + `src/QueryEngine.ts` (~1,295 lines)
> Đây là phần hay nhất của Claude Code.

---

## Mục lục

- [1. Tổng quan](#1-tổng-quan)
- [2. Main Query Loop](#2-main-query-loop)
- [3. Loop State Machine](#3-loop-state-machine)
- [4. Agentic Tool-Call Loop](#4-agentic-tool-call-loop)
- [5. Error Recovery — 4 Layers](#5-error-recovery--4-layers)
- [6. Thinking Mode](#6-thinking-mode)
- [7. Streaming Architecture](#7-streaming-architecture)
- [8. Token Counting & Cost Tracking](#8-token-counting--cost-tracking)
- [9. Context Building (4 Layers)](#9-context-building-4-layers)
- [10. Abort/Cancellation](#10-abortcancellation)
- [11. Permission System](#11-permission-system)
- [12. Stop Hooks System](#12-stop-hooks-system)
- [13. Key Design Decisions](#13-key-design-decisions)

---

## 1. Tổng quan

QueryEngine là **central orchestrator** của Claude Code — quản lý toàn bộ conversation lifecycle từ user input → LLM response → tool execution → error recovery.

```
┌─────────────────────────────────────────────────────────┐
│                    QueryEngine                          │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │ State Machine│  │ Tool Executor │  │Recovery Loop│  │
│  │  7 states    │  │   (recursive) │  │  4 layers   │  │
│  └──────────────┘  └──────────────┘  └─────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │Token Counter │  │ Stop Hooks   │  │PermissionMgr│  │
│  │ per-turn     │  │  (end-turn)  │  │             │  │
│  └──────────────┘  └──────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Trách nhiệm:**
- Gọi Anthropic API (streaming)
- Xử lý tool_use blocks từ LLM
- Quản lý tool-call recursion loop
- Error recovery (API errors, context overflow, rate limits)
- Thinking mode orchestration
- Token budget management
- Abort/cancellation signal propagation
- Stop hooks execution (extractMemories, analytics)

---

## 2. Main Query Loop

```typescript
async function runQueryLoop(input: UserInput, state: LoopState): Promise<void> {
  // Layer 1: Build context
  const systemPrompt = buildSystemPrompt(state)
  const userContext = await buildUserContext(state)
  const conversationHistory = state.history.get()

  // Layer 2: Token budget check
  const estimatedTokens = countTokens(systemPrompt, userContext, conversationHistory)
  if (estimatedTokens > state.tokenBudget * 0.7) {
    await triggerProactiveCompact(state)
  }

  // Layer 3: API call
  const stream = await anthropic.messages.create({
    model: state.model,
    max_tokens: state.maxOutputTokens,
    system: systemPrompt,
    messages: [...conversationHistory, { role: 'user', content: input }],
    stream: true,
    thinking: state.thinkingEnabled ? { type: 'enabled', budget_tokens: 10000 } : undefined,
  })

  // Layer 4: Stream processing
  for await (const event of stream) {
    switch (event.type) {
      case 'content_block_start':
        if (event.content_block.type === 'thinking') {
          state.enterState('THINKING')
        } else if (event.content_block.type === 'tool_use') {
          state.enterState('TOOL_CALL')
        }
        break
      case 'content_block_delta':
        if (event.delta.type === 'thinking_delta') {
          state.appendThinking(event.delta.thinking)
        } else if (event.delta.type === 'text_delta') {
          state.appendText(event.delta.text)
        } else if (event.delta.type === 'input_reasoning_delta') {
          state.appendInputReasoning(event.delta.input_reasoning)
        }
        break
      case 'content_block_stop':
        if (state.currentState === 'TOOL_CALL') {
          await executeToolCalls(state.toolCalls)
        }
        break
      case 'message_delta':
        state.appendUsage(event.usage)
        break
      case 'message_stop':
        state.enterState('DONE')
        await handleStopHooks(state)
        break
    }
  }
}
```

---

## 3. Loop State Machine

```
┌─────────┐  user input   ┌───────────┐  tool_use   ┌───────────┐
│  IDLE   │──────────────►│  RUNNING  │────────────►│TOOL_CALL  │
└─────────┘               └─────┬─────┘             └─────┬─────┘
                                 │                       │
                                 │                       │ tool result
                                 │                       ▼
                                 │               ┌───────────────┐
                                 │               │TOOL_RESULT    │
                                 │               └───────┬───────┘
                                 │                       │ loop back
                                 │                       ▼
                                 │               ┌───────────────┐
                                 │               │ BACK_TO_LLM   │──► Anthropic API
                                 │               └───────────────┘
                                 │
                                 ▼
                         ┌───────────────┐  stream done  ┌──────────┐
                         │   THINKING    │──────────────►│   DONE   │
                         └───────────────┘               └──────────┘
```

### 7 State Transition Types

| Transition | Trigger | Action |
|---|---|---|
| `USER_INPUT` | User sends message | Reset state, start new query |
| `LLM_OUTPUT` | LLM produces text/tool_use | Accumulate output |
| `TOOL_EXECUTION` | LLM requests tool | Execute tool(s), attach results |
| `CONTEXT_OVERFLOW` | prompt_too_long error | Trigger auto-compact |
| `TOKEN_BUDGET_EXCEEDED` | Output token limit | Re-call with higher limit |
| `RECOVERY` | API/rate limit error | Retry với exponential backoff |
| `DONE` | Stream completes | Execute stop hooks |

---

## 4. Agentic Tool-Call Loop

### Concurrency Model

Claude Code hỗ trợ **parallel tool execution** — nhiều tool có thể chạy đồng thời:

```typescript
// Batch tool calls: gửi nhiều tool_use trong 1 response
const toolCalls = [
  { name: 'Bash', input: { command: 'ls -la' } },
  { name: 'Grep', input: { pattern: 'TODO', path: './src' } },
  { name: 'Glob', input: { pattern: '**/*.ts' } },
]

// Execute in parallel — key performance optimization
const results = await Promise.all(
  toolCalls.map(tool => executeTool(tool, context))
)

// Attach ALL results trước khi gọi LLM tiếp
const followUp = await anthropic.messages.create({
  messages: [...history, { role: 'assistant', content: [
    ...toolCalls.map(tc => ({ type: 'tool_use', id: tc.id, name: tc.name })),
    ...results.map(r => ({ type: 'tool_result', tool_use_id: r.id, content: r.output }))
  ]}]
})
```

### Batching Rules

1. **Independent tools** → parallel (Bash, Glob, Grep không phụ thuộc nhau)
2. **Sequential tools** → theo thứ tự (FileWrite sau Bash tạo directory)
3. **Max batch size**: 10 tools/call (tránh overwhelm LLM context)
4. **Tool permission** → nếu 1 tool bị deny → **hault entire batch**

---

## 5. Error Recovery — 4 Layers

```
┌─────────────────────────────────────────────────────────┐
│                    ERROR RECOVERY STACK                 │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Layer 4: MODEL FALLBACK                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Opus ──fail──► Sonnet ──fail──► Haiku ──fail──►│   │
│  │ Error: "Model unavailable"                      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Layer 3: OUTPUT TOKEN ESCALATION                       │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 4096 tokens ──exceeded──► 8192 ──exceeded──►   │   │
│  │ 16384 ──exceeded──► stop (user must truncate)   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Layer 2: CONTEXT COLLAPSE AUTO-COMPACT                │
│  ┌─────────────────────────────────────────────────┐   │
│  │ prompt_too_long ──► runForkedAgent(compact)     │   │
│  │ Resume với compressed context                  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Layer 1: API ERROR RETRY                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Rate limit ──wait──► Retry (exponential backoff)│   │
│  │ 429 / 500 / 503 ──► Retry 1, 2, 3, give up       │   │
│  │ Timeout ──► Retry                               │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Layer 1: API Error Retry

```typescript
async function callAnthropicWithRetry(params: ApiParams, attempt = 1): Promise<Stream> {
  try {
    return await anthropic.messages.create({ ...params, stream: true })
  } catch (error) {
    if (isRateLimit(error) && attempt < MAX_RETRIES) {
      const delay = Math.min(1000 * Math.pow(2, attempt), 30000)
      await sleep(delay)
      return callAnthropicWithRetry(params, attempt + 1)
    }
    if (isTimeout(error) && attempt < MAX_RETRIES) {
      return callAnthropicWithRetry(params, attempt + 1)
    }
    throw error  // Layer 2/3 sẽ catch
  }
}
```

### Layer 2: Auto-Compact (Context Collapse)

```typescript
// Khi API trả về "prompt_too_long"
catch (error) {
  if (error.type === 'prompt_too_long') {
    log('Context overflow, triggering auto-compact')
    await runForkedAgent({ task: 'compact' })  // non-blocking
    // Re-build context với truncated history
    const compressedHistory = await compactHistory(state.history)
    state.history = compressedHistory
    return runQueryLoop(input, state)  // retry
  }
}
```

### Layer 3: Output Token Escalation

```typescript
const OUTPUT_TOKEN_TIERS = [4096, 8192, 16384, 32768]

async function callWithEscalatingTokens(params: ApiParams): Promise<Stream> {
  for (const limit of OUTPUT_TOKEN_TIERS) {
    try {
      return await anthropic.messages.create({ ...params, max_tokens: limit, stream: true })
    } catch (error) {
      if (error.type === 'max_tokens_exceeded') {
        log(`Output limit ${limit} exceeded, escalating...`)
        continue  // thử tier cao hơn
      }
      throw error
    }
  }
  throw new Error('Max output tokens exceeded at all tiers')
}
```

### Layer 4: Model Fallback

```typescript
const MODEL_CHAIN = ['claude-opus-4-5', 'claude-sonnet-4-6', 'claude-haiku-4']

async function callWithModelFallback(params: ApiParams): Promise<Stream> {
  for (const model of MODEL_CHAIN) {
    try {
      return await anthropic.messages.create({ ...params, model, stream: true })
    } catch (error) {
      if (error.type === 'model_not_available') {
        log(`Model ${model} unavailable, trying next...`)
        continue
      }
      throw error
    }
  }
  throw new Error('All models unavailable')
}
```

---

## 6. Thinking Mode

Claude Code hỗ trợ **extended thinking** — LLM tự generate internal reasoning trước khi produce final answer.

```typescript
interface ThinkingConfig {
  type: 'enabled'
  budget_tokens: number  // 1000 - 45000
}

// Signature protection: đảm bảo thinking không bị leak ra UI
const thinkingBlock = {
  type: 'thinking',
  thinking: encryptedContent,     // Nội bộ, LLM-only
  signature: crypto.randomUUID() // Integrity check
}

// Extended reasoning vs quick response
type ThinkingMode = 'adaptive'  // default: tự quyết dựa trên query complexity
        | 'enabled'   // luôn bật thinking
        | 'disabled'  // không bao giờ bật (simple CRUD queries)

// Adaptive: complexity detection
const shouldThink = detectComplexity(input) > COMPLEXITY_THRESHOLD
```

**Adaptive Thinking Flow:**

```
Query Complexity Score
    │
    ├── < 30 (simple) ──► Thinking disabled ──► fast response
    ├── 30-70 (medium) ─► Thinking enabled, 5K tokens
    └── > 70 (complex) ──► Thinking enabled, 20K tokens
```

---

## 7. Streaming Architecture

Claude Code xử lý SSE (Server-Sent Events) từ Anthropic API với ultra-low latency:

```typescript
// Stream processing với concurrent tool execution
for await (const event of stream) {
  switch (event.type) {
    case 'content_block_start': {
      // Khởi tạo block
      const block = createBlock(event.content_block)
      state.blocks.push(block)
      break
    }
    case 'content_block_delta': {
      // Incremental accumulation — zero latency UI update
      if (event.delta.type === 'text_delta') {
        state.currentText += event.delta.text  // append ngay, không buffer
        emitToUI({ type: 'text', delta: event.delta.text })
      }
      if (event.delta.type === 'thinking_delta') {
        // Thinking được masked — user không thấy
        state.thinkingBuffer += event.delta.thinking
      }
      break
    }
    case 'content_block_stop': {
      // Block hoàn chỉnh
      if (state.currentBlock.type === 'tool_use') {
        state.collectedToolCalls.push(state.currentBlock)
      }
      break
    }
    case 'message_delta': {
      // Usage stats — cập nhật token counter
      state.usageStats = event.usage
      emitToUI({ type: 'usage', stats: event.usage })
      break
    }
  }
}
```

**Zero-buffering principle:** Text được emit ngay khi nhận được từng delta — không đợi full message.

---

## 8. Token Counting & Cost Tracking

### Per-Turn Usage

```typescript
interface TurnUsage {
  inputTokens: number
  outputTokens: number
  cacheCreationTokens: number   // Prompt caching
  cacheReadTokens: number       // Cache hits
  thinkingTokens: number        // Extended thinking
  total: number
  costUSD: number
}

class CostTracker {
  private turns: TurnUsage[] = []

  onMessageDelta(usage: Usage) {
    this.turns.push({
      inputTokens: usage.input_tokens,
      outputTokens: usage.output_tokens,
      cacheCreationTokens: usage.cache_creation,
      cacheReadTokens: usage.cache_read,
      thinkingTokens: usage.thinking_tokens ?? 0,
      total: usage.total_tokens,
      costUSD: this.calculateCost(usage)
    })
  }

  getSessionTotal(): TurnUsage { /* sum all turns */ }
  getAverageCostPerTurn(): number { /* */ }
  getProjectedSessionCost(): number { /* based on current rate */ }
}
```

### Token Budget System

```typescript
interface TokenBudget {
  maxContext: number           // e.g., 200K
  maxOutput: number            // e.g., 8K
  warningThreshold: number      // e.g., 0.7 (70%)
  criticalThreshold: number    // e.g., 0.9 (90%)
}

// Proactive compact trigger
function shouldTriggerCompact(state: LoopState): boolean {
  const { maxContext } = state.budget
  const current = countTokens(state.fullContext)
  const ratio = current / maxContext
  return ratio > state.budget.warningThreshold  // 70%
}
```

---

## 9. Context Building (4 Layers)

```
┌─────────────────────────────────────────────────────────────┐
│                    CONTEXT LAYERS                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Layer 1: SYSTEM PROMPT                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ • Base instruction (always)                          │   │
│  │ • MEMORY.md index (always, ~200 lines)               │   │
│  │ • CLAUDE.md content (if exists)                     │   │
│  │ • Permission mode instruction                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Layer 2: USER CONTEXT (Workspace)                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ • Relevant files (file reads gần đây)               │   │
│  │ • Git diff (unstaged changes)                        │   │
│  │ • Recent command outputs                            │   │
│  │ • MCP resource data                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Layer 3: CONVERSATION HISTORY                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ • Previous turns (compressed when long)            │   │
│  │ • Tool call/result pairs                             │   │
│  │ • Thinking blocks (internal, masked)                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Layer 4: ON-DEMAND INJECTION                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ • findRelevantMemories() → relevant topic files (≤5)│   │
│  │ • LSP symbol data (on-demand)                       │   │
│  │ • MCP tool results (fresh each turn)                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Layer 1: System Prompt

```typescript
function buildSystemPrompt(state: LoopState): string {
  const parts = [
    BASE_INSTRUCTION,                                          // Always
    state.memoryIndex ? injectMemory(state.memoryIndex) : '',  // MEMORY.md
    state.claudeMd ? injectClaudeMd(state.claudeMd) : '',      // CLAUDE.md
    buildPermissionInstruction(state.permissionMode),          // Permission rules
    buildFeatureFlagInstructions(state.activeFlags),           // Feature flags
  ]
  return parts.filter(Boolean).join('\n\n')
}
```

### Layer 4: Query-Time Recall (Selective Memory)

```typescript
// Chỉ inject memories liên quan — không phải tất cả
async function findRelevantMemories(query: string): Promise<MemoryFile[]> {
  // 1. Scan frontmatter của tất cả topic files (chỉ đọc description)
  const manifest = await scanMemoryDirectory(state.memoryDir)
  // → { path, name, description, type, modifiedAt }

  // 2. Side-query: "File nào match query này?"
  const relevant = await sonnetSideQuery(`
    Query: ${query}
    Available memories:
    ${manifest.map(m => `- ${m.name}: ${m.description}`).join('\n')}
    Select ≤ 5 most relevant.
  `)

  // 3. Chỉ đọc full content của file được chọn
  const selectedFiles = relevant.map(r => r.path)
  const memories = await Promise.all(
    selectedFiles.map(p => readFile(p))
  )

  // 4. Inject với freshness warning nếu cũ
  return memories.map(m => ({
    content: m.content,
    warning: m.age > 1 ? `⚠️ Memory is ${m.age} days old.` : undefined
  }))
}
```

---

## 10. Abort/Cancellation

### Signal Architecture

```typescript
// AbortController chain — propagate cancellation qua async stack
class AbortChain {
  private controllers: AbortController[] = []

  spawn(): AbortSignal {
    const controller = new AbortController()
    this.controllers.push(controller)
    return controller.signal
  }

  abort(reason: string) {
    for (const c of this.controllers) {
      c.abort(reason)
    }
  }

  cleanup(controller: AbortController) {
    this.controllers = this.controllers.filter(c => c !== controller)
  }
}

// Tool behavior on abort
const toolAbortPolicy = {
  Bash:     'cancel_immediately',      // Không blocking — kill process
  WebFetch: 'cancel_immediately',      // Abort fetch
  Write:    'block_until_complete',   // Đợi write xong để data integrity
  Edit:     'block_until_complete',   // Đợi edit xong
  Agent:    'signal_subagent',         // Propagate to sub-agent
}
```

### Recovery After Abort

```
User presses ESC / Ctrl+C
    │
    ├── Current tool: cancel or block
    ├── LLM stream: abort() → stop receiving
    ├── State: save partial output
    │
    ▼
On Abort Complete:
    │
    ├── Emit "Interrupted" to UI
    ├── Show partial output if any
    ├── Store interrupted state
    │
    ▼
User resumes:
    ├── Load partial state
    ├── Provide recovery options:
    │   ├── "Continue from here" → resume stream
    │   ├── "Retry tool" → re-execute
    │   └── "Cancel" → discard
    │
    ▼
Re-inject: "You were interrupted. Continue."
```

---

## 11. Permission System

```typescript
// Permission modes
type PermissionMode =
  | 'default'      // Ask per new command type
  | 'plan'         // Ask only in plan mode
  | 'auto'         // Auto-approve safe operations
  | 'bypass'       // No prompts (CI/subagent)
  | 'dangerously_bypass'  // Full bypass

// Tool permission check
async function checkToolPermission(
  tool: ToolCall,
  context: ToolContext
): Promise<PermissionResult> {
  const mode = context.permissionMode

  if (mode === 'bypass' || mode === 'dangerously_bypass') {
    return { approved: true }
  }

  if (mode === 'auto') {
    const isSafe = SAFETY_RULES[tool.name]?.autoApproved
    return isSafe ? { approved: true } : { approved: false, reason: 'not_auto_approved' }
  }

  if (mode === 'plan' && !context.isPlanMode) {
    return { approved: false, reason: 'plan_mode_required' }
  }

  // default: prompt user
  return promptUser(tool, context)
}
```

**Auto-approve rules:**

| Tool | Safe to auto-approve |
|---|---|
| Read, Glob, Grep | ✅ Yes |
| Bash (read-only: ls, git, cat) | ✅ Yes |
| Bash (write: rm, mv, sed) | ❌ No |
| Write, Edit | ⚠️ Depends on path |
| Agent, Team | ❌ No |

---

## 12. Stop Hooks System

Stop hooks chạy sau mỗi turn — thực hiện background maintenance:

```typescript
// handleStopHooks() — called on every turn end
async function handleStopHooks(context: StopHookContext) {
  // 1. Analytics flush
  await analytics.flush()

  // 2. Extract Memories (forked, non-blocking)
  if (isExtractMemoriesEnabled()) {
    runForkedAgent({
      task: 'extractMemories',
      context: { turnHistory: context.history }
    })
  }

  // 3. Auto-Dream check (nightly)
  if (shouldRunDream()) {
    runForkedAgent({ task: 'dream' })
  }

  // 4. Telemetry export
  exportTelemetryEvents(context.events)

  // 5. Session Memory check
  if (context.turnCount % SESSION_MEMORY_INTERVAL === 0) {
    runForkedAgent({ task: 'sessionMemory' })
  }
}
```

---

## 13. Key Design Decisions

### 1. Withheld Messages

Một số message types bị **suppress** khỏi UI nhưng vẫn được xử lý:

```typescript
const uiSuppressedTypes = [
  'thinking',              // Extended reasoning nội bộ
  'input_reasoning',       // Input reasoning nội bộ
  'redacted_reasoning',    // Safety redactions
]

function shouldRenderInUI(block: ContentBlock): boolean {
  return !uiSuppressedTypes.includes(block.type)
}
```

### 2. Recursive Tool Execution Limit

```typescript
const MAX_TOOL_RECURSION = 50  // Prevent infinite loops

if (state.toolCallDepth > MAX_TOOL_RECURSION) {
  throw new Error('Tool recursion limit exceeded. Possible infinite loop.')
}
```

### 3. Prompt Cache Utilization

Claude Code **reuse prompt cache** giữa turns để giảm cost:

```typescript
// Cache key = hash(system_prompt + recent_files)
const cacheKey = computeCacheKey(state.systemPrompt, state.recentFiles)
const cachedPrompt = await promptCache.get(cacheKey)

// Forked agents share parent's cache
const forkedResult = await runForkedAgent({
  sharedPromptCache: true,  // inherit cache từ parent
  task: 'extractMemories'
})
```

### 4. Diminishing Returns Detection

```typescript
// QueryEngine tự detect khi nào thêm context không còn useful
async function detectDiminishingReturns(
  query: string,
  addedContext: string[]
): Promise<boolean> {
  const score = await sonnetSideQuery(`
    Context: ${addedContext.join('\n---\n')}
    Query: ${query}
    Rate relevance 1-10. Return JSON: { "score": number, "reasoning": string }
  `)
  return score < 3  // context không liên quan → skip
}
```

---

## File Reference

| File | Lines | Trách nhiệm |
|---|---|---|
| `src/query.ts` | ~12,700 | Main query loop, streaming, tool execution |
| `src/QueryEngine.ts` | ~1,295 | State machine, recovery orchestration |
| `src/tools.ts` | ~389 | Tool registry |
| `src/context.ts` | ~189 | Context building |
| `src/cost-tracker.ts` | ~323 | Token counting, cost tracking |

---

> **Nguồn**: Source maps từ npm package ngày 2026-03-31. Bản quyền thuộc về Anthropic.
