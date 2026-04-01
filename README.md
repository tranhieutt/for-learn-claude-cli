# Claude Code Source — Defensive Research Archive

> **Bản sao mã nguồn Claude Code bị lộ công khai** (31-03-2026) + **tài liệu phân tích chuyên sâu**.
> Phục vụ nghiên cứu bảo mật phòng thủ, kiến trúc hệ thống Agentic CLI, và phân tích rủi ro chuỗi cung ứng phần mềm.

---

## ⚠️ Legal Notice

- **Bản quyền thuộc về Anthropic PBC**
- Repository này **không trực thuộc, không được sự chấp thuận, và không đại diện cho Anthropic**
- Mục đích: **Educational & Defensive Security Research only**
- Xem [LICENSE](LICENSE) và [CONTRIBUTING](CONTRIBUTING.md)

---

## Repository Structure

```
claude-code-source/
├── src/                      ← ~1,884 files · ~512K lines · 34MB
│   ├── query.ts              ← Core query engine (~12.7K lines)
│   ├── QueryEngine.ts        ← State machine + recovery (~1.3K lines)
│   ├── Tool.ts               ← Tool base types (~792 lines)
│   ├── commands.ts           ← Command registry (~754 lines)
│   ├── tools/                ← 40+ tool implementations
│   ├── commands/             ← 50+ slash commands
│   ├── coordinator/          ← Multi-agent orchestration
│   ├── memdir/               ← Persistent memory system
│   ├── services/             ← Service layer (~20 modules)
│   └── bridge/               ← IDE bridge (VS Code, JetBrains)
│
├── docs/                     ← 📚 Tài liệu phân tích
│   ├── 1-Architecture/
│   │   └── OVERVIEW.md       ← Tổng quan kiến trúc
│   ├── 2-Deep-Dives/
│   │   ├── QUERY-ENGINE.md   ← Central orchestrator (phần hay nhất)
│   │   ├── MEMORY-SYSTEM.md  ← 5-layer memory architecture
│   │   ├── MULTI-AGENT.md    ← Multi-agent coordinator
│   │   ├── SERVICE-LAYER.md  ← 7 architectural patterns
│   │   └── TOOL-SYSTEM.md    ← 40+ tools architecture
│   └── 3-Guides/
│       ├── LESSONS-LEARNED.md ← Best practices từ source
│       └── REFERENCE.md       ← Quick reference
│
├── reports/                  ← Original analysis reports (tiếng Việt)
└── CODEMAP-*.md             ← Source code codemaps
```

---

## 📖 Tài liệu đọc (Recommended Reading Order)

### Level 1 — Bắt đầu ở đây

| Doc | Thời gian | Nội dung |
|---|---|---|
| [docs/1-Architecture/OVERVIEW.md](docs/1-Architecture/OVERVIEW.md) | 15 phút | Tổng quan toàn bộ hệ thống |
| [docs/3-Guides/LESSONS-LEARNED.md](docs/3-Guides/LESSONS-LEARNED.md) | 10 phút | Bài học sử dụng Claude Code hiệu quả |

### Level 2 — Deep Dives

| Doc | Thời gian | Nội dung |
|---|---|---|
| [docs/2-Deep-Dives/QUERY-ENGINE.md](docs/2-Deep-Dives/QUERY-ENGINE.md) | 30 phút | **Phần hay nhất** — query loop, error recovery, streaming |
| [docs/2-Deep-Dives/MEMORY-SYSTEM.md](docs/2-Deep-Dives/MEMORY-SYSTEM.md) | 20 phút | 5 subsystems memory, AI viết memory |
| [docs/2-Deep-Dives/MULTI-AGENT.md](docs/2-Deep-Dives/MULTI-AGENT.md) | 15 phút | 5 task types, 3 execution models |
| [docs/2-Deep-Dives/SERVICE-LAYER.md](docs/2-Deep-Dives/SERVICE-LAYER.md) | 20 phút | 7 architectural patterns |
| [docs/2-Deep-Dives/TOOL-SYSTEM.md](docs/2-Deep-Dives/TOOL-SYSTEM.md) | 20 phút | 40+ tools, permission system |

### Level 3 — Source Code

```
src/query.ts              ← Start here — understand the main loop
src/QueryEngine.ts        ← State machine + recovery
src/Tool.ts               ← Tool base class
src/tools.ts              ← Tool registry
src/memdir/               ← Memory system implementation
src/coordinator/           ← Multi-agent orchestration
src/services/              ← All services
```

---

## 🎯 Key Insights từ Source

### 1. AI Uses AI
```typescript
// Thay vì hardcode logic, Claude spawn sub-agent để:
runForkedAgent('extractMemories')  // AI quyết định gì cần nhớ
runForkedAgent('autoDream')         // AI merge + cleanup memories
runForkedAgent('compact')           // AI compress context
```
→ **Pattern này có thể áp dụng cho bất kỳ AI tool nào.**

### 2. 4-Layer Error Recovery
```
API error → Retry
Context overflow → Auto-compact (forked agent)
Token limit → Escalate limit
Model fail → Fallback chain (Opus → Sonnet → Haiku)
```
→ **Zero single-point-of-failure design.**

### 3. Parallelism is a Superpower
```typescript
// Boot: parallel prefetch
startMdmRawRead()  // không blocking
startKeychainPrefetch()

// Runtime: independent agents → parallel
await Promise.all([
  agent.run({ task: 'analyze', run_in_background: true }),
  agent.run({ task: 'review',  run_in_background: true }),
  agent.run({ task: 'test',    run_in_background: true }),
])
```

### 4. Memory là Priority #1
```
🥇 MEMORY.md investment      → Persistent knowledge
🥈 CLAUDE.md writing         → Per-session guidance
🥉 Parallel task decomposition
```

### 5. 7 Service Patterns

| Pattern | Ví dụ |
|---|---|
| Queue Buffer | Analytics — zero blocking startup |
| Closure Factory | LSP, OAuth — encapsulated state |
| Forked Subagent | extractMemories — AI xử lý thay hardcode |
| Feature Gate Caching | Settings sync — fast reads, stale OK |
| Distributed Lock | autoDream — file mtime làm lock |
| Lazy Loading | Voice — NAPI load khi cần |
| Marker Types | Analytics — compile-time PII safety |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Bun |
| Language | TypeScript (strict mode) |
| Terminal UI | React 18 + Ink |
| CLI Parsing | Commander.js |
| Schema | Zod v4 |
| AI API | Anthropic SDK |
| Observability | OpenTelemetry + gRPC |
| Feature Flags | GrowthBook |
| Auth | OAuth 2.0, JWT, macOS Keychain |

---

## Nguồn gốc

Mã nguồn bị lộ qua **npm source maps** ngày **2026-03-2026** bởi [@Fried_rice](https://x.com/Fried_rice), được xác nhận bởi Chaofan Shou.

File source map trỏ đến TypeScript source gốc (unobfuscated) trên R2 bucket của Anthropic.

---

## Contributing

Xem [CONTRIBUTING.md](CONTRIBUTING.md).

---

## License

Xem [LICENSE](LICENSE). **Bản quyền gốc thuộc về Anthropic PBC.**
