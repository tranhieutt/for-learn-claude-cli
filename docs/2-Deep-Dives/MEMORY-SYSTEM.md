# Deep Dive: Memory System — 5-Layer Persistent Memory Architecture

> Source: `src/memdir/` + `src/services/extractMemories/` + `src/services/autoDream/` + `src/services/teamMemorySync/`

---

## Mục lục

- [1. Tổng quan](#1-tổng-quan)
- [2. Disk Layout & Storage](#2-disk-layout--storage)
- [3. Data Format](#3-data-format)
- [4. Taxonomy: 4 Memory Types](#4-taxonomy-4-memory-types)
- [5. Lifecycle — 5 Subsystems](#5-lifecycle--5-subsystems)
- [6. Memory Agent Permissions](#6-memory-agent-permissions)
- [7. Security](#7-security)
- [8. Feature Flags & Configuration](#8-feature-flags--configuration)

---

## 1. Tổng quan

Claude Code quản lý memory qua **5 subsystem phối hợp**, lưu trữ dưới dạng Markdown files trên disk. Pattern đặc biệt nhất: **AI dùng AI để quản lý memory** — thay vì hardcode extraction logic, Claude spawn forked sub-agent để tự quyết định gì cần nhớ.

```
┌──────────────────────────────────────────────────────────┐
│ TURN END                                                 │
│                                                          │
│ extractMemories ──writes──► topic files                  │
│      │                       │    │                     │
│      │ (skip if main wrote)   │    │                     │
│      ▼                       ▼    │                     │
│ autoDream ──reads──► topic files + logs                 │
│      │ ──writes──► MEMORY.md                             │
│      │                                                 │
│      ▼                                                 │
│ teamMemorySync ──watch──► delta upload ──► server      │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ QUERY START                                              │
│                                                          │
│ system prompt ◄──── MEMORY.md (luôn inject)             │
│ user context ◄──── relevant topic files (≤5, on-demand)│
└──────────────────────────────────────────────────────────┘
```

---

## 2. Disk Layout & Storage

```
~/.claude/projects/{sanitized-git-root}/memory/
├── MEMORY.md                 ← Index file (tối đa 200 dòng, 25KB)
├── user_role.md              ← Topic: thông tin về user
├── feedback_testing.md        ← Topic: quy tắc làm việc
├── project_deadline.md       ← Topic: context dự án
├── reference_linear.md        ← Topic: pointer đến external systems
├── team/                     ← Team-shared memories
│   ├── MEMORY.md             ← Team index
│   └── team_policy.md
└── logs/                     ← KAIROS/Assistant mode only
    └── 2026/
        └── 03/
            └── 2026-03-31.md ← Daily append-only log
```

### Path Resolution (thứ tự ưu tiên)

| Priority | Source | Example |
|---|---|---|
| 1 | Env var | `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE=/full/path` |
| 2 | settings.json | `settings.json → autoMemoryDirectory` |
| 3 | Env var | `CLAUDE_CODE_REMOTE_MEMORY_DIR=~/base/path` |
| 4 | Default | `~/.claude/projects/{canonical-git-root}/memory/` |

**Isolation:** Mỗi git repository có thư mục memory riêng. Tất cả worktrees của cùng một repo **chia sẻ** một memory directory.

---

## 3. Data Format

### MEMORY.md (Index)

```markdown
# Memory Index

- [User Role](user_role.md) — Senior backend engineer, Go expertise
- [Testing Feedback](feedback_testing.md) — Don't mock database in tests
- [Project Deadline](project_deadline.md) — Mobile release freeze 2026-04-05
- [Linear Reference](reference_linear.md) — Bugs tracked in Linear INGEST project
```

**Rules:**
- Không có frontmatter
- Mỗi dòng = pointer: `- [Title](file.md) — one-line hook`
- Giới hạn **200 dòng** — dòng 201+ bị cắt + warning
- Giới hạn **25KB** tổng content

### Topic Files (YAML frontmatter bắt buộc)

```markdown
---
name: Testing Feedback
description: Don't mock database — caused prod incident last quarter
type: feedback
---

Don't mock the database in integration tests.

**Why:** Prior incident where mock/prod divergence masked a broken migration.

**How to apply:** All integration tests must hit a real database, never mocks.
```

| Field | Mô tả |
|---|---|
| `name` | Tên memory |
| `description` | 1 dòng mô tả — dùng để quyết định relevance |
| `type` | `user` / `feedback` / `project` / `reference` |

---

## 4. Taxonomy: 4 Memory Types

| Type | Nội dung | Privacy | Khi nào lưu |
|---|---|---|---|
| **user** | Vai trò, kỹ năng, sở thích của user | Luôn private | Khi biết thông tin về user |
| **feedback** | Quy tắc làm việc (do/don't) | Private hoặc team | Khi user correct/confirm approach |
| **project** | Deadline, quyết định, incidents | Bias toward team | Khi biết ai làm gì, tại sao, deadline nào |
| **reference** | Pointer đến Linear, Grafana, Slack... | Thường team | Khi biết external resource quan trọng |

---

## 5. Lifecycle — 5 Subsystems

### 5.1 System Prompt Injection (Startup)

```
Session bắt đầu
     │
     ▼
loadMemoryPrompt() [memdir/memdir.ts]
     │
     ▼
Đọc MEMORY.md → truncate tại 200 dòng / 25KB
     │
     ▼
Inject toàn bộ vào system prompt section 'memory'
     │
     ▼
Claude luôn thấy MEMORY.md index ngay từ đầu session
```

### 5.2 Query-Time Recall (Selective — ≤5 files)

```
User gửi query
     │
     ▼
findRelevantMemories(query) [memdir/findRelevantMemories.ts]
     │
     ▼
Scan frontmatter của tất cả .md files
     │
     ▼
Sonnet side-query: "File nào match query này?"
     │
     ▼
Chọn ≤ 5 files → đọc full content
     │
     ▼
Inject vào context + freshness warning nếu age > 1 ngày
```

### 5.3 Extract Memories (Auto-write sau mỗi turn)

```
Turn kết thúc → handleStopHooks() [query/stopHooks.ts]
     │
     ▼
executeExtractMemories(stopHookContext)
     │
     ▼
Gates kiểm tra (theo thứ tự, từ rẻ → đắt):
  1. !isBareMode()
  2. feature('EXTRACT_MEMORIES') bật
  3. isAutoMemoryEnabled() = true
  4. Throttle: mỗi N turns (default N=1)
  5. Main agent only (skip nếu là subagent)
  6. !getIsRemoteMode()
     │
     ▼
runForkedAgent() — chạy nền, KHÔNG block user
     │
     ▼
Forked agent (read-only + write-only-memory-dir):
  → Đọc turn history
  → Quyết định gì cần ghi nhớ lâu dài
  → Write/Edit files trong memory directory
  → Cập nhật MEMORY.md index
```

**Mutual exclusion:** Nếu main agent tự viết memory → extract skip.
**Stashing:** Nếu extract đang chạy và user gửi message → stash, chạy lại sau.

### 5.4 Auto-Dream (Nightly Consolidation)

```
Turn kết thúc (check mỗi turn)
     │
     ▼
Gate 1 (Time): hours_since_last_consolidation >= minHours (default 24h)
Gate 2 (Session): sessions_since_last_consolidation >= minSessions (default 5)
Gate 3 (Lock): Không process nào đang dream (file mtime lock)
Gate 4 (Throttle): SESSION_SCAN_INTERVAL_MS = 10 phút
     │
     ▼
runForkedAgent() — chạy /dream skill
     │
     ▼
Đọc tất cả logs + topic files
  → Xóa obsolete memories
  → Merge duplicates
  → Viết lại MEMORY.md
  → Cập nhật lastConsolidatedAt
```

### 5.5 Team Memory Sync

```
Local write → File watcher detect [services/teamMemorySync/watcher.ts]
     │
     ▼
Calculate delta (hash comparison với last-known state)
     │
     ▼
secretScanner.ts — scan tìm credit card, API key patterns
     │
     ▼
PUT /api/claude_code/team_memory?repo={owner/repo}
     │
     ▼
Server lưu → team members pull về khi bắt đầu session
```

---

## 6. Memory Agent Permissions

Forked memory agents bị **sandbox chặt**:

| Tool | Allowed | Notes |
|---|---|---|
| Read, Grep, Glob | ✅ Không giới hạn path | Read all workspace |
| Bash (read-only) | ✅ ls, find, grep, cat, stat, wc, head, tail | Read-only commands |
| Write, Edit | ✅ **CHỉ** trong memory directory | Auto-scoped |
| All other tools | ❌ Deny | Tuyệt đối |

---

## 7. Security

### Path Validation

- ❌ Reject: relative paths
- ❌ Reject: root/near-root (`/`, `/home`, `C:\`)
- ❌ Reject: Windows drive roots (`C:\`)
- ❌ Reject: UNC paths (`\\server\share`)
- ❌ Reject: paths với null bytes
- ✅ Accept: normal `~` expansion từ settings.json

### Team Memory

| Protection | Limit |
|---|---|
| Secret scan trước upload | credit cards, API keys, private keys |
| File size cap | **250KB per entry** |
| Upload body cap | **200KB** (gateway) |
| Server enforcement | max_entries tự học từ 413 response |

---

## 8. Feature Flags & Configuration

### GrowthBook Flags

| Flag | Mục đích | Default |
|---|---|---|
| `tengu_passport_quail` | Enable extract memories | `false` (ANT internal) |
| `tengu_slate_thimble` | Extract trong non-interactive session | `false` |
| `tengu_bramble_lintel` | Throttle: extract mỗi N turns | `1` |
| `tengu_onyx_plover` | Auto-dream config | `24h, 5 sessions` |
| `tengu_moth_copse` | Skip MEMORY.md index writing | `false` |
| `tengu_herring_clock` | Enable team memory | `false` |
| `KAIROS` | Assistant mode (daily logs) | feature gate |

### Environment Variables

| Var | Mục đích |
|---|---|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | Tắt toàn bộ auto memory |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Custom memory base (CCR) |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | Full path override |
| `CLAUDE_CODE_SIMPLE` / `--bare` | Skip tất cả memory ops |
| `CLAUDE_CODE_REMOTE` | Remote mode — disable memory |

### settings.json

```json
{
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "~/custom/memory/path"
}
```

---

## Key Files Reference

| File | Trách nhiệm |
|---|---|
| `memdir/paths.ts` | Path resolution, config gates |
| `memdir/memoryTypes.ts` | Type taxonomy, frontmatter format |
| `memdir/memdir.ts` | System prompt building, MEMORY.md truncation |
| `memdir/memoryScan.ts` | Scan directory, parse frontmatter, build manifest |
| `memdir/findRelevantMemories.ts` | Query-time recall, Sonnet selection |
| `memdir/memoryAge.ts` | Staleness calculation, freshness warnings |
| `services/extractMemories/extractMemories.ts` | Background extraction forked agent |
| `services/extractMemories/prompts.ts` | Extraction agent instructions |
| `services/autoDream/autoDream.ts` | Consolidation scheduling, lock management |
| `services/autoDream/consolidationLock.ts` | Distributed lock via file mtime |
| `services/SessionMemory/sessionMemory.ts` | Context window summary |
| `services/teamMemorySync/index.ts` | Server sync, hash comparison |
| `services/teamMemorySync/watcher.ts` | File watcher cho team memory |
| `services/teamMemorySync/secretScanner.ts` | Secret detection |
| `commands/memory/memory.tsx` | `/memory` slash command UI |
| `utils/backgroundHousekeeping.ts` | Init extract + dream subsystems |
| `query/stopHooks.ts` | Turn-end triggers cho memory operations |

---

> **Nguồn**: Source maps từ npm package ngày 2026-03-31. Bản quyền thuộc về Anthropic.
