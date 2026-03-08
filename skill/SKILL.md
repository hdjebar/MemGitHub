---
name: mem-github
description: >
  Persistent cross-session memory using a GitHub repo as storage.
  Handles loading, saving, querying, and consolidating memories via GitHubMCP.
  Use this skill ALWAYS when the user says "remember", "forget", "load my memories",
  "save session", "resume session", "new session", "consolidate", "what do you know",
  "what do you remember", "show my memories", "list sessions", or any reference to
  persistent memory, cross-session context, or the memory repo.
---

# MemGitHub — GitHub-Backed Persistent Memory (v4 Lazy Loading)

All memory lives in a GitHub repo. Every write is a git commit.

## Required Tool

**GitHubMCP** must be connected. Set for YOUR fork:

```
OWNER = "YOUR_GITHUB_USERNAME"
REPO  = "YOUR_REPO_NAME"
```

## Architecture — Lazy Loading

The key idea: **load a summary first, fetch details only when needed.**

```
Layer 1 — Index (always loaded, ~300 tokens):
  memory.json
    ├── summary        ← natural language paragraph, captures ALL memories
    ├── stats          ← counts by type
    ├── sessions[]     ← name + goal + status per session
    └── config

Layer 2 — Details (loaded on demand, per query):
  memories/global.json     ← hot memories (compact JSON, importance >= 0.7)
  memories/archive.json    ← cold memories (importance < 0.7)

Layer 3 — Session context (loaded only on resume):
  sessions/{name}/context.md
  sessions/{name}/progress.md
  sessions/{name}/memories.json
```

### Why This Is Better

| Approach | LOAD cost | What Claude knows |
|----------|-----------|-------------------|
| v2 (eager, verbose) | ~3,250 tok @ 50 memories | Everything |
| v3 (compact + hot/cold) | ~1,250 tok @ 50 memories | All hot memories |
| **v4 (lazy, summary)** | **~300 tok always** | Summary paragraph + session list |

The summary is a natural language paragraph that captures the *essence* of all
memories. Claude can work from it for most turns. Details are fetched only when
a specific question requires citing [Memory #N] or when saving/consolidating.

### Token Budget by Layer

| Layer | When loaded | Cost |
|-------|------------|------|
| memory.json (index) | Every session start | ~300 tokens (fixed) |
| memories/global.json | On first QUERY or explicit "load details" | ~25 tokens/memory |
| memories/archive.json | On "search archive" | ~25 tokens/memory |
| Session files | On RESUME | ~500-1,500 tokens |

---

## Repo Structure

```
{OWNER}/{REPO}/
├── memory.json                    ← INDEX: summary + sessions + config (~300 tok)
├── memories/
│   ├── global.json                ← HOT details (importance >= 0.7)
│   └── archive.json               ← COLD details (importance < 0.7)
└── sessions/
    └── {session-name}/
        ├── context.md
        ├── progress.md
        └── memories.json
```

## memory.json Schema (The Index)

```json
{
  "version": 4,
  "last_updated": "2026-03-08T14:00:00Z",
  "next_id": 7,
  "config": {
    "importance_threshold": 0.3,
    "hot_threshold": 0.7,
    "max_load_tokens": 4000
  },
  "stats": {
    "total": 6, "hot": 4, "cold": 2,
    "by_type": {"fact": 3, "goal": 1, "preference": 2}
  },
  "summary": "Enterprise architect based in Luxembourg. Works with TOGAF and ArchiMate. GitHub: hdjebar. Building an Always On Memory system for Claude. Prefers KISS architecture and GitHub as storage.",
  "sessions": [
    {"name": "ai-memory-system", "status": "active",
     "goal": "Build Always On Memory for Claude", "last_updated": "2026-03-08"}
  ],
  "detail_files": {
    "global_hot": "memories/global.json",
    "global_cold": "memories/archive.json"
  }
}
```

## Memory Detail Format (Compact, ~25 tokens each)

```json
{"i":1,"c":"User is based in Luxembourg","t":"fact","imp":0.9}
```

Fields: `i`=id, `c`=content, `t`=type, `imp`=importance.

---

## Commands

### 🔵 LOAD — Start of any conversation

**Triggers:** "load my memories", conversation start

1. Read `memory.json` only (`mode: "full"`) — ~300 tokens
2. Present the `summary` to the user
3. List available sessions from `sessions` array
4. Ask: "Resume a session or start fresh?"
5. **Do NOT read memories/global.json yet** — wait until needed

Claude now has enough context to have a useful conversation. The summary tells
Claude who the user is. Details are fetched lazily.

---

### 🟡 QUERY — Answer from memory (triggers detail fetch)

**Triggers:** "what do you know about...", "do you remember..."

1. If details not loaded yet → read `memories/global.json` (one API call)
2. Search loaded memories for relevant content
3. Answer citing `[Memory #N]`
4. If not found in hot: "Not in active memories. Want me to search the archive?"
5. If yes → read `memories/archive.json`

This is the **lazy trigger** — details are only fetched when Claude needs them.

---

### ⏯️ RESUME — Continue a previous session

**Triggers:** "resume session", "continue {name}"

1. If no name → show sessions from memory.json (already loaded)
2. Read 3 session files:
   - `sessions/{name}/context.md`
   - `sessions/{name}/progress.md`
   - `sessions/{name}/memories.json`
3. Synthesize: goal + done + next
4. **Do NOT load global details** unless the session needs cross-referencing

---

### 🆕 NEW SESSION — Create a session folder

**Triggers:** "new session", "start session for..."

1. Read `memory.json` + SHA (already loaded from LOAD)
2. Create via `push_files`: context.md, progress.md, memories.json (empty)
3. Append to `sessions` array in memory.json (with SHA!)
4. Commit: `session: init {name}`

---

### 🟢 REMEMBER — Save a new memory

**Triggers:** "remember that...", "note that..."

1. Classify: global or session? Hot or cold?
2. Read the **target detail file** + SHA:
   - Global hot → `memories/global.json`
   - Global cold → `memories/archive.json`
   - Session → `sessions/{name}/memories.json`
3. Append the new memory (compact format)
4. Read `memory.json` + SHA, update:
   - `summary` — regenerate the natural language paragraph
   - `stats` — increment counts
   - `next_id` — increment
   - `last_updated` — now
5. Write both files (detail + index) via `push_files`
6. Confirm: "Saved as Memory #{id}."

### Summary Regeneration

When updating the summary, condense ALL hot memories into a paragraph:
- One sentence per 2-3 related facts
- Include the most important details
- Keep under 100 words
- No memory IDs in summary (those are in the detail files)

---

### 🔴 FORGET — Remove a memory

**Triggers:** "forget memory #N"

1. Read target detail file + SHA
2. Remove the entry
3. Update memory.json summary + stats
4. Write both via `push_files`

---

### 💾 SAVE SESSION — Persist before closing

**Triggers:** "save session", "we're done"

1. Read session files + memory.json + SHAs
2. Update progress.md
3. Extract decisions → session memories.json
4. Promote user-level facts to global (hot or cold detail file)
5. Regenerate summary in memory.json
6. Update sessions[].last_updated + stats
7. Write all via `push_files`

---

### 📋 LIST SESSIONS

**Triggers:** "list sessions"

1. Already in memory.json — no extra read needed

---

### 🔄 CONSOLIDATE

**Triggers:** "consolidate", "clean up"

1. Read memory.json + memories/global.json + memories/archive.json
2. Deduplicate, merge, delete across both
3. Re-sort by hot_threshold
4. Regenerate summary paragraph
5. Update stats
6. Write all three files back

---

### 📦 SEARCH ARCHIVE

**Triggers:** "search archive", "load more", "check old memories"

1. Read `memories/archive.json`
2. Find relevant by keyword
3. Present, offer to promote to hot

---

## Rules

1. **Read before write.** Get SHA, write with SHA.
2. **`mode: "full"` always.**
3. **Lazy by default.** LOAD reads only memory.json. Details on demand.
4. **Summary is king.** Most turns only need the summary paragraph.
5. **Compact format** in detail files: `{"i","c","t","imp"}`.
6. **Regenerate summary** on every REMEMBER, FORGET, SAVE SESSION, CONSOLIDATE.
7. **Global vs session.** User identity → global. Project → session.
8. **Commit messages:** `memory: {action} — {description}`
