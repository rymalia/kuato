# Claude Session Memory

Search and query your Claude Code session history. Find what you discussed, what decisions you made, and where you left off.

## The Problem

Claude Code forgets everything between sessions. You're deep in a feature, close the tab, and the next day ask "where were we?" only to get a blank stare.

This tool solves that by making your session history searchable.

## Two Versions

| Feature | File-Based | PostgreSQL |
|---------|------------|------------|
| **Setup time** | 0 minutes | 5 minutes |
| **Dependencies** | Bun only | Bun + Postgres |
| **Search speed** | ~1-5 seconds | <100ms |
| **Full-text search** | Basic matching | Weighted, ranked |
| **Fuzzy matching** | No | Yes (trigram) |
| **API server** | No | Yes |
| **Token tracking** | Yes | Yes + per-model |
| **Best for** | Quick lookups | Daily use |

**Start with file-based.** Upgrade to PostgreSQL when you're running 10+ sessions a day and want faster, smarter search.

## Quick Start: File-Based

Zero setup. Works directly with Claude Code's JSONL files.

```bash
# Clone the repo
git clone https://github.com/alexknowshtml/claude-session-memory.git
cd claude-session-memory

# Search your sessions
bun run file-based/search.ts --query "email system" --days 7

# Filter by tools used
bun run file-based/search.ts --tools Edit,Bash --limit 10

# Filter by file patterns
bun run file-based/search.ts --file-pattern "components/"
```

Output is JSON with all session metadata:

```json
[
  {
    "id": "abc123-def456",
    "startedAt": "2025-01-15T10:30:00Z",
    "endedAt": "2025-01-15T11:45:00Z",
    "messageCount": 42,
    "toolsUsed": ["Edit", "Bash", "Read"],
    "filesFromToolCalls": ["src/api/email.ts", "src/utils/filter.ts"],
    "userMessages": [
      "Let's build an email filtering system",
      "Yes, use that approach",
      "Commit this and we'll continue tomorrow"
    ],
    "relevance": 23
  }
]
```

## Quick Start: PostgreSQL

One-command database setup, then sync and search.

```bash
cd claude-session-memory/postgres

# Start PostgreSQL (creates database + schema)
bun run db:up

# Install dependencies
bun install

# Sync your sessions
DATABASE_URL="postgres://claude:sessions@localhost:5433/claude_sessions" bun run sync

# Start API server
DATABASE_URL="postgres://claude:sessions@localhost:5433/claude_sessions" bun run serve
```

Then query the API:

```bash
# Search
curl "http://localhost:3847/sessions?search=email+filtering&days=7"

# Get single session
curl "http://localhost:3847/sessions/abc123-def456"

# With full transcript
curl "http://localhost:3847/sessions/abc123-def456?with_transcript=true"

# Statistics
curl "http://localhost:3847/sessions/stats?days=30"
```

## Using with Claude Code

The real power is teaching Claude how to search your history. Create a skill file:

```markdown
# Session Search

When the user asks "where did we leave off" or "what did we discuss about X":

1. Search sessions:
   curl "http://localhost:3847/sessions?search=TOPIC&days=14"

2. The results include `user_messages` - these tell the whole story
   (requests, confirmations, decisions)

3. Summarize what happened and offer to continue
```

See `shared/claude-skill.md` for a complete skill template.

## How It Works

### The Key Insight: User Messages

You don't need the full transcript to understand what happened. User messages are the signal:

- "Let's build an email filtering system" → Request
- "Yes, use that approach" → Decision
- "Actually, make it async" → Correction
- "Commit this" → Completion

Combined with `files_touched` and `tools_used`, you can reconstruct the session without loading 50k tokens.

### File-Based Search

1. Scans `~/.claude/projects/*/` for JSONL files
2. Parses each file, extracts metadata
3. Scores relevance against search query
4. Returns sorted results

Simple but effective. Scales to hundreds of sessions.

### PostgreSQL Search

1. Sync job parses JSONL → database
2. PostgreSQL tsvector for weighted full-text search
3. Trigram index for fuzzy matching
4. Sub-100ms queries across thousands of sessions

Worth the setup if you're a heavy user.

## API Reference (PostgreSQL)

### GET /sessions

Search sessions with filters.

| Param | Description |
|-------|-------------|
| `search` | Full-text search (weighted: summary > messages > tools) |
| `days` | Last N days only |
| `since` | Sessions after date (YYYY-MM-DD) |
| `until` | Sessions before date |
| `tools` | Filter by tool names (comma-separated) |
| `file_pattern` | Filter by file path pattern |
| `limit` | Max results (default 20, max 100) |

### GET /sessions/:id

Get single session. Add `?with_transcript=true` for full messages.

### GET /sessions/stats

Usage statistics. Add `?days=30` to customize window.

Returns: session count, token totals, breakdown by model.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CLAUDE_SESSIONS_DIR` | `~/.claude/projects` | Where Claude Code stores sessions |
| `DATABASE_URL` | `postgres://localhost/claude_sessions` | PostgreSQL connection |
| `PORT` | `3847` | API server port |

## Tips

### Scheduled Sync

Run sync every 15 minutes to keep the database current:

```bash
# crontab -e
*/15 * * * * cd /path/to/claude-session-memory/postgres && DATABASE_URL="..." bun run sync >> /var/log/session-sync.log 2>&1
```

### Search Strategies

**By topic:**
```bash
curl "http://localhost:3847/sessions?search=authentication"
```

**By file:**
```bash
curl "http://localhost:3847/sessions?file_pattern=src/auth"
```

**Recent activity:**
```bash
curl "http://localhost:3847/sessions?days=3&limit=50"
```

**Combined:**
```bash
curl "http://localhost:3847/sessions?search=refactor&tools=Edit&days=7"
```

## Contributing

Issues and PRs welcome. This started as a personal tool - feedback on what's useful appreciated.

## License

MIT
