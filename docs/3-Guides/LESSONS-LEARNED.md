# Lessons Learned — Using Claude Code Effectively

> Bài học rút ra từ việc đọc và phân tích source code Claude Code.
> Nguồn: `src/` + thực nghiệm

---

## Mục lục

- [1. Memory Files — Đầu tư cao nhất](#1-memory-files--đầu-tư-cao-nhất)
- [2. CLAUDE.md Writing](#2-claudemd-writing)
- [3. Multi-Agent — Parallel Decomposition](#3-multi-agent--parallel-decomposition)
- [4. Proactive `/compact`](#4-proactive-compact)
- [5. Worktree Isolation](#5-worktree-isolation)
- [6. Common Misconceptions](#6-common-misconceptions)
- [7. Context Management](#7-context-management)
- [8. Permission Modes](#8-permission-modes)

---

## 1. Memory Files — Đầu tư cao nhất

### Tại sao quan trọng

Claude Code **không nhớ** gì giữa các session. `MEMORY.md` là cách duy nhất để có persistent knowledge.

### Cách hoạt động

```
Session 1: "Tôi là senior backend engineer, thích TDD, không mock database"
Session 2: "Viết test cho auth module"
Claude: → Đọc MEMORY.md → biết preferred approach → apply TDD + real DB
```

### 4 loại Memory

| Type | Khi nào tạo | Ví dụ |
|---|---|---|
| **user** | Khi biết thông tin user | "User thích Go, hay quên viết tests" |
| **feedback** | Khi user correct/confirm | "User nói đừng dùng any type" |
| **project** | Khi biết project context | "Project deadline 2026-04-05, mobile release" |
| **reference** | Khi biết external systems | "Bugs được track ở Linear INGEST project" |

### Best Practices

```markdown
<!-- MEMORY.md index — tối đa 200 dòng -->
# Memory Index
- [Go Expertise](user_go_engineer.md) — Senior backend, Go + PostgreSQL
- [TDD Feedback](feedback_tdd.md) — Always write tests first, no mocks
- [Mobile Deadline](project_mobile_release.md) — Freeze 2026-04-05
- [Linear Tracking](reference_linear.md) — Bugs → Linear INGEST project
```

```markdown
<!-- Topic file — YAML frontmatter bắt buộc -->
---
name: TDD Feedback
description: Always write tests first, avoid mocking database
type: feedback
---
Write tests before implementation.
Don't mock the database — caused prod incident last quarter.
Use table-driven tests in Go.
```

### Giới hạn

- MEMORY.md: **200 dòng** max
- Topic files: **25KB** max mỗi file
- ≤ **5 topic files** được inject vào context mỗi query

---

## 2. CLAUDE.md Writing

### Sai lầm phổ biến

❌ Viết CLAUDE.md như documentation:
```markdown
# Claude Code Usage Guide

Claude Code is a CLI tool that...
```

✅ Viết như **imperative instructions** cho Claude:
```markdown
# This Project

- Backend: Go + PostgreSQL + Redis
- Tests: table-driven, no mocks for DB
- Commit format: conventional commits (feat/fix/docs)
- On PR: always run `make test && make lint` before requesting review
```

### Nguyên tắc

| Sai | Đúng |
|---|---|
| Mô tả tool làm gì | Chỉ rõ **bạn muốn** Claude làm gì |
| Liệt kê features | Nêu **quy tắc cụ thể** (do/don't) |
| Documentation style | **Imperative**, action-oriented |

### Điểm quan trọng

> **CLAUDE.md được đọc MỖI API call**, không phải chỉ đầu session.
> → Đặt những gì Claude **cần nhớ mỗi turn** ở đây.

---

## 3. Multi-Agent — Parallel Decomposition

### Nguyên tắc vàng

> **"Parallelism is your superpower. Workers are async. Launch independent workers concurrently."**

### Khi nào dùng

```
┌─────────────────────────────────────────────────────┐
│  DÙNG MULTI-AGENT                                    │
├─────────────────────────────────────────────────────┤
│  • Nhiều files cần analyze đồng thời                │
│  • Code review + test writing + refactoring         │
│  • Independent tasks không phụ thuộc nhau           │
│  • Heavy research tasks                              │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  KHÔNG DÙNG MULTI-AGENT                             │
├─────────────────────────────────────────────────────┤
│  • Task có dependency (A trước B sau)                │
│  • Simple CRUD, typos, single-file edits             │
│  • Khi bạn cần review từng bước                     │
└─────────────────────────────────────────────────────┘
```

### Cách dùng

```typescript
// Trong Claude Code CLI:
// Dùng Agent tool với run_in_background: true

/agent Analyze src/auth/ for security vulnerabilities
  --subagent code-reviewer
  --model sonnet
  --background

/agent Review src/api/ for REST conventions
  --subagent code-reviewer
  --model sonnet
  --background

// Kết quả được gửi về khi agent hoàn thành
```

### Với Worktree

```typescript
// Risky refactoring — cách ly trong worktree
/agent Migrate from REST to GraphQL
  --isolation worktree
  --description "Review changes before merging"
```

---

## 4. Proactive `/compact`

### Khi nào gọi

| Context usage | Hành động |
|---|---|
| < 60% | OK — tiếp tục |
| 60-70% | **Gọi `/compact`** — sớm hơn là tốt hơn |
| > 80% | Gọi ngay — có thể bị context overflow |
| `prompt_too_long` | Late — đã xảy ra rồi |

### Cách hoạt động

```
compact được gọi như một forked sub-agent:
  1. AI đọc full conversation history
  2. AI quyết định giữ lại / compress cái gì
  3. Output: condensed summary + key decisions
  4. Context được replace bằng summary
```

### Tip

```
# Thay vì đợi Claude tự compact, chủ động gọi khi:
- Session > 50 turns
- Bạn đã hoàn thành 1 feature lớn
- Trước khi bắt đầu task mới hoàn toàn
```

---

## 5. Worktree Isolation

### Khi nào dùng

| Scenario | Nên dùng Worktree? |
|---|---|
| Risky refactoring (renames, migrations) | ✅ Bắt buộc |
| Breaking changes có thể fail | ✅ Rất nên |
| Experimenting với new architecture | ✅ An toàn |
| Simple bug fixes | ❌ Không cần |
| Documentation updates | ❌ Không cần |
| Small, reversible changes | ❌ Không cần |

### Cách dùng

```
# Trong Claude Code CLI:

/worktree create migration-graphql
  --base main
# → Claude tạo .git/worktrees/migration-graphql/
# → Làm việc trong worktree
# → Review changes
# → Merge hoặc discard

/worktree exit
  --action keep  # giữ lại worktree để review thêm
# hoặc
/worktree exit
  --action remove  # xóa worktree, discard changes
```

---

## 6. Common Misconceptions

### ❌ "Claude nhớ across sessions"
```typescript
// THỰC TẾ:
Session memory: Chỉ tồn tại trong current context window
Persistent memory: Cần MEMORY.md + extractMemories subsystem
→ KHÔNG có cross-session memory nếu không có MEMORY.md
```

### ❌ "ESC dừng file writes ngay"
```typescript
// THỰC TẾ:
Write/Edit tools: BLOCK cho đến khi write hoàn tất
  → Data integrity: không muốn partial writes
Bash/Web tools: Cancel immediately
  → Interrupt được ngay lập tức
→ ESC không đảm bảo 100% stop trong mọi trường hợp
```

### ❌ "Prompt too long = restart session"
```typescript
// THỰC TẾ:
4-layer recovery tự động:
  Layer 1: Retry với exponential backoff
  Layer 2: Auto-compact (context collapse)
  Layer 3: Output token escalation
  Layer 4: Model fallback
→ KHÔNG cần restart — Claude tự phục hồi
```

### ❌ "Claude chậm = đang fail"
```typescript
// THỰC TẾ:
Có thể Claude đang trong recovery cycle:
  - Retry exponential backoff
  - Auto-compact đang chạy
  - Model fallback đang thử
→ KHÔNG interrupt khi thấy chậm — có thể đang tự fix
```

### ❌ "CLAUDE.md chỉ đọc đầu session"
```typescript
// THỰC TẾ:
CLAUDE.md được đọc MỖI API call
→ Có thể dùng để:
  - Inject thay đổi context real-time
  - Override behavior mid-session
  - Thay đổi approach theo tình huống
```

---

## 7. Context Management

### Priority order

| Priority | Content | Frequency |
|---|---|---|
| 🥇 | MEMORY.md index | Luôn luôn (200 dòng) |
| 🥈 | CLAUDE.md | Mỗi API call |
| 🥉 | Relevant topic files | ≤ 5, query-time (selective) |
| 4 | Recent file reads | Theo thứ tự sử dụng |
| 5 | Git diff | Khi có changes |
| 6 | MCP resources | On-demand |

### Diminishing Returns

Claude Code có **diminishing returns detection** — khi thêm context không còn useful:

```
Query: "Fix login bug"
Context: [relevant auth files] → Score: 9/10 ✅
Context: [auth files + unrelated config] → Score: 6/10 ⚠️
Context: [auth files + 50 unrelated files] → Score: 2/10 ❌
```

---

## 8. Permission Modes

| Mode | Use case |
|---|---|
| `default` | Interactive sessions — hỏi khi cần |
| `plan` | Planning mode — hỏi cho dangerous tools |
| `auto` | Trusted tasks — auto-approve safe ops |
| `bypass` | Sub-agents, CI — no prompts |
| `dangerously_bypass` | Full bypass — only for trusted automation |

---

## Priority Summary

```
🥇 1. MEMORY.md investment         → Persistent knowledge
🥈 2. CLAUDE.md writing            → Per-session guidance
🥉 3. Parallel task decomposition  → Speed up complex work
4️⃣ 4. Proactive /compact          → Avoid context overflow
5️⃣ 5. Worktree isolation          → Safe risky changes
```

---

> **Nguồn**: Phân tích source code Claude Code + thực nghiệm.
