# Deep Dive: Service Layer — 7 Architectural Patterns

> Source: `src/services/` (~20 modules + nhiều single-file services)

---

## Mục lục

- [1. Dependency Graph](#1-dependency-graph)
- [2. 7 Architectural Patterns](#2-7-architectural-patterns)
- [3. Key Services](#3-key-services)
- [4. Strengths](#4-strengths)
- [5. Potential Weaknesses](#5-potential-weaknesses)

---

## 1. Dependency Graph

```
                    analytics/                        ← ZERO DEPS — BASE LAYER
                         ↑
          (tất cả services log vào đây)
                         │
            ┌─────────────┼──────────────────┐
            ▼             ▼                   ▼
      ┌──────────┐  ┌───────────┐    ┌──────────────────┐
      │api/client │◄─┤  oauth/   │    │  growthbook/     │
      └─────┬────┘  └───────────┘    └──────────────────┘
            │                              ↑
            │         FORKED AGENT         │
            ▼                             │
     ┌─────────────────────────────────────┤
     │  autoDream / extractMemories        │
     │  SessionMemory / compact            │
     └──────────────────┬──────────────────┘
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼               ▼
   ┌─────────────┐ ┌──────────┐ ┌─────────────┐
   │remoteMgd   │ │settings  │ │teamMemory   │
   │Settings     │ │sync      │ │Sync         │
   └─────────────┘ └──────────┘ └─────────────┘
          │
          ▼
   ┌─────────────────────────────────────────────┐
   │          INDEPENDENT SERVICES                │
   │ mcp/ (6 transports)  lsp/ (closure)        │
   │ plugins/  voice/ (lazy)  claudeAiLimits/   │
   └─────────────────────────────────────────────┘
```

**Key insight:** Analytics là **base layer** với 0 dependencies → không circular dependency.

---

## 2. 7 Architectural Patterns

### Pattern 1: Queue Buffer (Analytics)

```typescript
// Events được queue trước khi attachAnalyticsSink() gọi
// → Drain async qua queueMicrotask()
class AnalyticsService {
  private queue: TelemetryEvent[] = []

  emit(event: TelemetryEvent) {
    this.queue.push(event)  // O(1) push
  }

  attachAnalyticsSink(sink: AnalyticsSink) {
    // Chạy sau khi attach — không blocking startup
    queueMicrotask(() => {
      while (this.queue.length > 0) {
        const event = this.queue.shift()
        sink.flush(event)
      }
    })
  }
}
```

**Benefit:** Zero blocking tại startup — emit trước hay sau `attachAnalyticsSink()` đều OK.

### Pattern 2: Closure Factory (LSP, OAuth, Compact)

```typescript
// State hoàn toàn encapsulated trong closure
// → Không class instantiation, no `this`, no prototype pollution
const createLspService = () => {
  // Private state — không expose ra ngoài
  let connections = new Map<string, LspConnection>()
  let pendingRequests: PendingRequest[] = []
  let activeDiagnostics = new Map<string, Diagnostic[]>()

  return {
    connect(uri: string): Promise<void> {
      connections.set(uri, new LspConnection(uri))
    },
    disconnect(uri: string): void {
      connections.get(uri)?.dispose()
      connections.delete(uri)
    },
    getDiagnostics(uri: string): Diagnostic[] {
      return activeDiagnostics.get(uri) ?? []
    },
  }
}

const lsp = createLspService()  // factory call
```

**Benefit:** Immutable service instances, easy testing, no shared mutable state.

### Pattern 3: Forked Subagent (AI Calls AI)

```typescript
// AI xử lý thay hardcode logic
const runForkedAgent = async (task: ForkedTaskType) => {
  const systemPrompt = FORKED_PROMPTS[task]  // Carefully crafted
  const tools = FORKED_TOOLS[task]           // Minimal sandboxed set

  return spawnAgent({
    type: 'LocalAgentTask',
    model: 'sonnet',  // Dùng Sonnet — đủ thông minh, rẻ hơn Opus
    systemPrompt,
    tools,
    sharedPromptCache: true,  // Chia sẻ cache với parent
  })
}

// Usage
await runForkedAgent('extractMemories')   // AI decide what to remember
await runForkedAgent('autoDream')          // AI consolidate memories
await runForkedAgent('compact')            // AI compress context
await runForkedAgent('sessionMemory')     // AI summarize conversation
```

**Benefit:** Linh hoạt hơn hardcode logic — AI tự thích nghi với context.

### Pattern 4: Feature Gate Caching

```typescript
// Giá trị feature flag cached — có thể stale nhưng nhanh
const featureCache = new Map<FeatureFlag, boolean>()

function isSessionMemoryGateEnabled(): boolean {
  const cached = featureCache.get('tengu_passport_quail')
  if (cached !== undefined) return cached

  // GrowthBook check — có thể nặng
  const value = growthBook.isOn('tengu_passport_quail')
  featureCache.set('tengu_passport_quail', value)
  return value
}
```

**Trade-off:** Fast reads nhưng có thể serve stale value nếu flag thay đổi trong session.

### Pattern 5: Distributed Lock via File mtime

```typescript
// autoDream: tránh nhiều process chạy consolidation cùng lúc
const LOCK_FILE = '.dream.lock'

async function acquireDreamLock(): Promise<boolean> {
  try {
    const stat = await fs.stat(LOCK_FILE)
    const age = Date.now() - stat.mtimeMs

    // Lock cũ > 30 phút → coi như orphan, chiếm lock
    if (age > 30 * 60 * 1000) {
      await fs.writeFile(LOCK_FILE, String(Date.now()))
      return true
    }
    return false  // Lock đang active
  } catch {
    // Lock file không tồn tại → tạo mới
    await fs.writeFile(LOCK_FILE, String(Date.now()))
    return true
  }
}

async function releaseDreamLock(): Promise<void> {
  await fs.unlink(LOCK_FILE)
}
```

**Benefit:** Zero infrastructure — chỉ cần filesystem. Works across processes.

### Pattern 6: Lazy Loading (Voice)

```typescript
// Load audio NAPI chỉ khi user nhấn voice key lần đầu
const voiceService = (() => {
  let napiModule: VoiceNapiModule | null = null
  let backend: VoiceBackend | null = null

  return {
    async initialize(): Promise<void> {
      if (napiModule) return  // Already loaded

      // Dynamic import — heavy NAPI chỉ load khi cần
      napiModule = await import('./voice-napi.node')

      // 3-level fallback
      if (napiModule.supportsNapi()) {
        backend = new NapiBackend(napiModule)
      } else if (await commandExists('sox')) {
        backend = new SoxBackend()
      } else if (await commandExists('alsa')) {
        backend = new AlsaBackend()
      } else {
        throw new Error('No audio backend available')
      }
    },

    async transcribe(audio: Buffer): Promise<string> {
      if (!backend) await this.initialize()
      return backend.transcribe(audio)
    }
  }
})()
```

**Benefit:** Startup nhanh — NAPI (~MB) không load nếu user không dùng voice.

### Pattern 7: Marker Types cho PII Safety

```typescript
// Compile-time enforcement — không thể accidentally log PII
type AnalyticsMetadata_I_VERIFIED_SAFE = {
  userId?: string      // Verified safe — no PII
  sessionId?: string
  eventType: string
  timestamp: number
}

type AnalyticsMetadata_I_VERIFIED_HAS_PII = {
  // Never allowed in analytics
}

// Analytics flush chỉ accept SAFE type
function flushToTelemetry(
  event: AnalyticsMetadata_I_VERIFIED_SAFE  // ✅ Compile error nếu pass PII
): void {
  telemetry.export(event)  // Safe
}
```

**Benefit:** Type system ngăn accidental PII logging at compile time.

---

## 3. Key Services

### MCP Service (6 transports)

```typescript
// Model Context Protocol — kết nối Claude Code với external tools
const mcpTransports = {
  SSE:     'Server-Sent Events',    // stateless, firewall-friendly
  Stdio:   'Standard I/O',          // local processes
  HTTP:    'HTTP/REST',             // cloud services
  WebSocket: 'WebSocket',            // bidirectional
  InProcess: 'Direct import',       // zero network
  SDK:     'MCP SDK Client',         // official SDK
}

// Transport selection tự động dựa trên server config
function createMcpClient(config: McpServerConfig): McpClient {
  const transport = selectTransport(config.endpoint)
  return new McpClient({ transport })
}
```

### Voice Service (3 fallback backends)

```
NAPI (native)  ──fallback──►  SoX (shell)  ──fallback──►  ALSA (Linux)
     │                        │                        │
  ~5ms latency            ~50ms latency             ~100ms
  Best quality           Medium quality             Basic
  Requires .node file    Cross-platform            Linux only
```

### OAuth Service

```typescript
// Dual-mode: automatic browser redirect + manual copy-paste
const oauthFlow = {
  // Mode 1: Auto-redirect (desktop environments)
  automatic: async (config: OAuthConfig) => {
    const server = createLocalServer(8765)
    const authUrl = buildAuthUrl(config)
    await openBrowser(authUrl)

    // Intercept callback
    const callback = await server.waitForCallback()
    const tokens = await exchangeCode(callback.code)
    await storeTokens(tokens)
  },

  // Mode 2: Manual (CI, headless)
  manual: async (config: OAuthConfig) => {
    const authUrl = buildAuthUrl(config)
    console.log('Visit:', authUrl)
    const code = await prompt('Enter code:')
    const tokens = await exchangeCode(code)
    await storeTokens(tokens)
  }
}
```

---

## 4. Strengths

| Điểm mạnh | Chi tiết |
|---|---|
| **Zero-circular-deps** | Analytics base layer → tất cả services log vào đây, không ai depend vào nó |
| **Async-first** | Không sync I/O blocking startup |
| **Forked agent pattern** | Linh hoạt, AI thích nghi tốt hơn hardcode |
| **6 MCP transports** | Hỗ trợ mọi loại server infrastructure |
| **3 voice backends** | Graceful degradation — luôn có fallback |
| **Feature gate caching** | Fast reads, acceptable staleness |

---

## 5. Potential Weaknesses

| Điểm yếu | Rủi ro |
|---|---|
| `mcp/client.ts` ~122KB | Cần tách nhỏ — violate "many small files" principle |
| Feature gate caching | Stale values → user có thể thấy inconsistent behavior |
| Forked agent token cost | AI gọi AI = token consumption cao hơn |
| File mtime lock | Không robust trên network filesystems (NFS, etc.) |

---

## File Reference

| File | Trách nhiệm |
|---|---|
| `services/analytics/` | OpenTelemetry + GrowthBook, queue buffer pattern |
| `services/api/` | Anthropic API client |
| `services/oauth/` | OAuth 2.0 flow, dual-mode |
| `services/lsp/` | Language Server Protocol, closure factory |
| `services/mcp/` | MCP multi-transport client, ~122KB |
| `services/extractMemories/` | Forked agent memory extraction |
| `services/autoDream/` | Nightly consolidation, file lock |
| `services/compact/` | Context compression forked agent |
| `services/SessionMemory/` | Context window summary |
| `services/teamMemorySync/` | Team memory sync, secret scan |
| `services/plugins/` | Plugin loader, marketplace |
| `services/voice/` | Voice input, lazy NAPI loading, 3 backends |
| `services/claudeAiLimits/` | Rate limiting |
| `services/remoteManagedSettings/` | MDM/enterprise config |
| `services/settingsSync/` | Cross-instance settings sync |

---

> **Nguồn**: Source maps từ npm package ngày 2026-03-31. Bản quyền thuộc về Anthropic.
