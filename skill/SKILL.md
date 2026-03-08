---
name: mem-github
description: >
  Persistent cross-session memory using a GitHub repo as storage.
  Handles loading, saving, querying, and consolidating memories via GitHubMCP.
  Use this skill ALWAYS when the user says "remember", "forget", "load my memories",
  "save session", "resume session", "new session", "consolidate", "what do you know",
  "what do you remember", "show my memories", "list sessions", or any reference to
  persistent memory, cross-session context, or the memory repo. Also trigger when
  starting any new conversation where the user might benefit from their stored context.
---

# MemGitHub — GitHub-Backed Persistent Memory

All memory lives in a GitHub repo. Every write is a git commit.

## Required Tool

**GitHubMCP** must be connected. Set these two values for YOUR fork:

```
OWNER = "YOUR_GITHUB_USERNAME"
REPO  = "YOUR_REPO_NAME"
```

GitHubMCP tools used:
- `GitHubMCP:get_file_contents` — read (always use `mode: "full"`)
- `GitHubMCP:create_or_update_file` — write single file (requires SHA!)
- `GitHubMCP:push_files` — write multiple files in one commit

## Repo Structure

```
{OWNER}/{REPO}/
├── memory.json                    ← ONE FILE: global memories + session index + config
└── sessions/
    └── {session-name}/
        ├── context.md             ← What this session is about
        ├── progress.md            ← Done / in-progress / next steps
        └── memories.json          ← Session-specific memories
```

### memory.json — Single Source of Truth

One read gives Claude everything: who you are, what sessions exist, and config.

---

## Commands

### 🔵 LOAD — Start of any conversation

**Triggers:** "load my memories", "what do you remember", conversation start

1. Read `memory.json` (`mode: "full"`) — one call gets everything
2. Filter memories to `is_active: true`, group by type
3. Note available sessions from the `sessions` array
4. Summarize: who the user is + list sessions + ask "resume or start fresh?"

---

### ⏯️ RESUME — Continue a previous session

**Triggers:** "resume session", "continue {name}", "load session {name}"

1. If no name → show sessions from `memory.json`, let user pick
2. Read 3 session files (`mode: "full"` for each):
   - `sessions/{name}/context.md`
   - `sessions/{name}/progress.md`
   - `sessions/{name}/memories.json`
3. Synthesize: goal + what's done + what's next + session memories
4. Continue working

---

### 🆕 NEW SESSION — Create a session folder

**Triggers:** "new session", "start session for..."

1. Read `memory.json` + note its SHA
2. Ask for session name if not provided (kebab-case)
3. Create session files via `GitHubMCP:push_files`:
   - `sessions/{name}/context.md` — goal + scope
   - `sessions/{name}/progress.md` — empty checklist
   - `sessions/{name}/memories.json` — `{"session_name":"{name}","next_id":1,"memories":[]}`
4. Update `memory.json` — append to `sessions` array (use SHA!)
5. Commit: `session: init {name}`

---

### 🟢 REMEMBER — Save a new memory

**Triggers:** "remember that...", "note that...", "save to memory"

1. Read target file + SHA (global `memory.json` or session `memories.json`)
2. Extract atomic fact(s):
   ```json
   {"id": N, "content": "User prefers X", "memory_type": "preference",
    "entities": ["X"], "topics": ["tools"], "importance": 0.7,
    "is_active": true, "created_at": "ISO timestamp"}
   ```
3. **Where does it go?**
   - About the USER (identity, preferences, skills) → `memory.json`
   - About THIS PROJECT (decisions, architecture) → `sessions/{name}/memories.json`
   - Rule: useful in a different project? → global. Only this project? → session.
4. Append, increment `next_id`, update `last_updated`, write with SHA
5. Confirm: "Saved as Memory #{id}."

**Types:** fact, preference, event, procedure, goal
**Importance:** 0.9+ critical · 0.7-0.8 strong · 0.5-0.6 moderate · 0.3-0.4 minor

---

### 🟡 QUERY — Answer from memory

**Triggers:** "what do you know about...", "do you remember..."

1. Use already-loaded memories (global + session)
2. Answer citing `[Memory #N]` — never fabricate
3. If nothing relevant, say so

---

### 🔴 FORGET — Remove a memory

**Triggers:** "forget memory #N", "delete memory about..."

1. Read target file + SHA
2. Set `is_active: false` (soft delete, never remove from file)
3. Write back with SHA
4. Confirm

---

### 💾 SAVE SESSION — Persist before closing

**Triggers:** "save session", "save progress", "we're done for now"

1. Read session files + `memory.json` + all SHAs
2. Update `progress.md` — check off done, add next steps
3. Extract key decisions → session `memories.json`
4. Promote user-level facts to global `memory.json`
5. Update `sessions[].last_updated` in `memory.json`
6. Write all via `GitHubMCP:push_files`
7. Commit: `session: save {name} — {summary}`

---

### 📋 LIST SESSIONS

**Triggers:** "list sessions", "show projects"

1. Read `memory.json` → `sessions` array
2. Display: name, status, goal, last updated

---

### 🔄 CONSOLIDATE

**Triggers:** "consolidate", "clean up memories"

1. Read `memory.json`
2. For each memory: KEEP, MERGE, UPDATE, or DELETE
3. Write cleaned `memory.json`
4. Report results

---

## Rules

1. **Read before write.** Always get current SHA, then write with that SHA.
2. **`mode: "full"` always.** Overview mode truncates content.
3. **One file for global state.** `memory.json` = memories + sessions + config.
4. **Atomic facts.** One self-contained sentence per memory. No dangling pronouns.
5. **Don't over-memorize.** Skip chitchat, passwords, transient details.
6. **Global vs session.** User identity → `memory.json`. Project details → session.
7. **Commit messages:** `memory: {action} — {description}`
