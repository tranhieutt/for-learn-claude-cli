# Quick Reference — Claude Code Commands & Tools

---

## Slash Commands

| Command | Mô tả |
|---|---|
| `/commit` | Git commit với smart message |
| `/review [files...]` | Code review |
| `/compact` | Nén context |
| `/memory` | Quản lý memory |
| `/tasks` | Task list |
| `/mcp` | MCP server management |
| `/config` | Settings |
| `/doctor` | Diagnostics |
| `/diff` | Git diff |
| `/cost` | Token usage |
| `/context` | Visualize context |
| `/resume` | Resume session |
| `/share` | Share session |
| `/skills` | Skill management |
| `/vim` | Vim mode |
| `/theme` | Theme |
| `/pr_comments` | PR comments |
| `/desktop` | Desktop app |
| `/mobile` | Mobile app |

---

## Agent Tool

```
/agent <prompt> [options]

Options:
  --model <opus|sonnet|haiku>
  --subagent <type>          e.g., code-reviewer, planner
  --background               Async execution
  --isolation <mode>         worktree, remote, in-process
  --tools <tool1,tool2,...>  Restrict available tools
  --system-prompt <text>     Override system prompt
```

---

## Worktree Commands

```
/worktree create [name] --base <branch>
/worktree list
/worktree exit --action <keep|remove>
```

---

## Permission Modes

```
--dangerously-enable-all-commands   Full bypass
--auto                             Auto-approve safe ops
--plan                             Plan mode only
```

---

## Key File Patterns

```
MEMORY.md              → Index (max 200 lines)
*.md (in memory/)      → Topic files (YAML frontmatter required)
CLAUDE.md              → Project-level instructions (imperative style)
.claude/               → Project-level config & memory
~/.claude/            → User-level config & memory
```

---

## Error Recovery

```
Layer 1: API error        → Retry (exponential backoff)
Layer 2: Context overflow → Auto-compact
Layer 3: Token limit      → Escalate output limit
Layer 4: Model fail       → Fallback (Opus → Sonnet → Haiku)
```

---

## Token Budget

```
< 60%    → OK
60-70%   → Trigger /compact
> 80%    → Emergency compact
> 90%    → Prompt too long (recovery kicks in)
```
