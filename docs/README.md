# Documentation Index

## 📚 Tài liệu phân tích Claude Code Source

---

## Reading Guide

### 🟢 Level 1 — Getting Started (30 phút)

| File | Mô tả |
|---|---|
| [1-Architecture/OVERVIEW.md](1-Architecture/OVERVIEW.md) | Tổng quan kiến trúc toàn bộ hệ thống |
| [3-Guides/LESSONS-LEARNED.md](3-Guides/LESSONS-LEARNED.md) | Bài học sử dụng Claude Code hiệu quả |

### 🟡 Level 2 — Deep Dives (2 giờ)

| File | Mô tả |
|---|---|
| [2-Deep-Dives/QUERY-ENGINE.md](2-Deep-Dives/QUERY-ENGINE.md) | ⭐ **Phần hay nhất** — query loop, error recovery 4 layers, streaming, thinking mode |
| [2-Deep-Dives/MEMORY-SYSTEM.md](2-Deep-Dives/MEMORY-SYSTEM.md) | 5-layer memory: extractMemories, autoDream, team sync |
| [2-Deep-Dives/MULTI-AGENT.md](2-Deep-Dives/MULTI-AGENT.md) | 5 task types, 3 execution models, Forked Pattern |
| [2-Deep-Dives/SERVICE-LAYER.md](2-Deep-Dives/SERVICE-LAYER.md) | 7 patterns: Queue Buffer, Closure Factory, Forked Subagent |
| [2-Deep-Dives/TOOL-SYSTEM.md](2-Deep-Dives/TOOL-SYSTEM.md) | 40+ tools, permission system, Zod validation |

### 🔴 Level 3 — Source Code (nâng cao)

Đọc trực tiếp source code trong `src/`:

```
src/query.ts              ← Main query loop
src/QueryEngine.ts        ← State machine
src/Tool.ts               ← Tool base class
src/memdir/               ← Memory implementation
src/coordinator/           ← Multi-agent
src/services/              ← Services
```

---

## Quick Navigation

```
docs/
├── 1-Architecture/
│   └── OVERVIEW.md              ← Tổng quan (Start here)
│
├── 2-Deep-Dives/
│   ├── QUERY-ENGINE.md          ← ⭐ Must read
│   ├── MEMORY-SYSTEM.md
│   ├── MULTI-AGENT.md
│   ├── SERVICE-LAYER.md
│   └── TOOL-SYSTEM.md
│
└── 3-Guides/
    ├── LESSONS-LEARNED.md       ← Best practices
    └── REFERENCE.md             ← Quick reference
```

---

## Key Topics

| Topic | Doc |
|---|---|
| AI dùng AI để quản lý memory | MEMORY-SYSTEM.md |
| Forked subagent pattern | MULTI-AGENT.md + SERVICE-LAYER.md |
| 4-layer error recovery | QUERY-ENGINE.md |
| 40+ tools architecture | TOOL-SYSTEM.md |
| Closure factory pattern | SERVICE-LAYER.md |
| Persistent memory system | MEMORY-SYSTEM.md |
| Multi-agent orchestration | MULTI-AGENT.md |
| Feature flags (GrowthBook) | OVERVIEW.md |
