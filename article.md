# From Google's Always On Memory Agent to MemGitHub: Giving Claude a Brain with Just Git

*An adopt-and-adapt story: how a Google engineer's open-source memory system inspired a KISS alternative for Claude, using nothing but a GitHub repo.*

---

## The Problem

Claude has built-in memory now. But it's a black box. You can't see what it remembers, organize it by project, or resume a work session from last week. Every conversation starts with re-explaining your stack, your preferences, your project context.

I wanted structured, transparent, cross-session memory. So I looked at how others had solved it.

## The Inspiration: Google's Always On Memory Agent

In early 2026, Google Cloud Platform published an open-source project called the [Always On Memory Agent](https://github.com/GoogleCloudPlatform/generative-ai/tree/main/gemini/agents/always-on-memory-agent), built by a Google PM. It's a single-file multi-agent system that runs 24/7 as a background process, giving Gemini persistent memory.

The architecture is elegant: three specialized agents operating on a shared SQLite database.

### The Three Agents

**IngestAgent** watches an inbox folder. Drop a file in — text, PDF, image, audio — and within seconds it extracts structured memories: a summary, entities, topics, and an importance score (0–1). It handles 27+ file types using Gemini's multimodal capabilities.

**ConsolidateAgent** runs on a 30-minute timer, like a brain's sleep cycle. It reviews all unconsolidated memories, finds cross-references, generates insights, and compresses related information. The metaphor is intentional: replay, connect, compress.

**QueryAgent** reads all memories and synthesizes answers with source citations referencing specific memory IDs.

An orchestrator routes requests to the right specialist. The whole thing runs as a continuous process with a file watcher, consolidation timer, and HTTP API.

### The Provocative Bet: No Vector Database

The most interesting design choice: the system uses no embeddings, no vector database, no knowledge graph. Just SQLite. The reasoning is that the LLM itself acts as both the extraction engine and the retrieval engine — it reads all memories in context and reasons over them directly.

This works because Gemini Flash-Lite is cheap and fast enough that loading all memories into context remains practical at moderate scale. It breaks down at large scale — which is exactly what Google's production-grade Vertex AI Memory Bank solves with embedding-based semantic search.

### What Vertex AI Memory Bank Adds

Google's managed Memory Bank service implements the same three-phase pattern but adds production infrastructure. Configurable memory topics filter extraction. Consolidation classifies every new fact as CREATED, UPDATED, or DELETED against existing memories. Retrieval uses embedding-based semantic similarity. Each memory includes revision history and optional TTL.

## What I Adopted

The three-agent architecture is the core insight. Separating ingest, consolidation, and query into distinct concerns produces clean, maintainable logic:

- **Ingest**: Extract atomic, self-contained facts from conversation context
- **Consolidate**: Deduplicate, merge, resolve contradictions, generate cross-cutting insights
- **Query**: Retrieve relevant memories with cited sources

I also adopted:

- **Atomic facts** — each memory is one self-contained sentence that makes sense without the original conversation
- **Importance scoring** — 0.0–1.0 scale for prioritizing what to load into context
- **Memory types** — fact, preference, event, procedure, goal
- **Soft deletion** — `is_active: false` rather than removing, preserving audit trail
- **Consolidation as maintenance** — periodic cleanup rather than real-time
- **LLM-as-retriever** for small memory stores — let Claude read everything and reason over it

## What I Adapted

The original system assumes a long-running server with file watchers and timers. That doesn't exist in claude.ai. So everything had to change at the infrastructure level while preserving the agent logic.

### SQLite → GitHub Repo

The original uses SQLite. I replaced it with a GitHub repo:

| Original (SQLite) | MemGitHub (GitHub) |
|---|---|
| `memory.db` file on disk | `memory.json` in a repo |
| SQL queries | `GitHubMCP:get_file_contents` |
| INSERT/UPDATE | `GitHubMCP:create_or_update_file` |
| No history | Every change = a git commit |
| Local only | Accessible from anywhere |
| Needs a running process | Works from any Claude session |

GitHub gives you version control for free. Every memory update is a commit. You can browse your memories on github.com, roll back changes, and audit what Claude wrote.

### Three Files → One File

The original has separate database tables for memories and consolidations. I started with multiple files (memories.json, MEMORY.md, config.json, sessions/index.json) but quickly realized that violated KISS. Every extra file means an extra API call.

Final design: **one `memory.json`** that holds everything — global memories, session index, and config. One read gives Claude the complete picture.

### Background Daemon → User-Triggered Commands (with Automation Options)

The original runs a file watcher, consolidation timer, and HTTP API as concurrent background services. In claude.ai, there's no background process — so the system is entirely user-triggered:

- "Load my memories" → QueryAgent reads memory.json
- "Remember that..." → IngestAgent extracts and writes back
- "Save session" → IngestAgent + progress update
- "Consolidate" → ConsolidateAgent deduplicates and merges

For claude.ai, manual triggers remain the right choice. But Claude Code has closed this gap significantly since March 2026. With **`/loop`** (recurring prompts on a timer), **cron scheduling**, **SessionStart/Stop hooks**, and **background agents**, MemGitHub can now run in a near-daemon mode inside Claude Code:

- **SessionStart hook** auto-loads memories when you open a session
- **`/loop 30m consolidate`** mirrors the original's 30-minute consolidation timer
- **Background agents** run consolidation without blocking your main workflow
- **Agent Teams** (experimental) let one agent handle memory while another codes

The KISS principle still applies: use manual triggers for simple sessions, layer on automation when your workflow demands it.

### Single Context → Session Folders

The original system has one flat memory store per user. All memories live in the same database.

I added **session folders** to isolate per-project context:

```
sessions/
  api-redesign/
    context.md       ← what this project is
    progress.md      ← done / next steps
    memories.json    ← project-specific facts and decisions
  ml-pipeline/
    context.md
    progress.md
    memories.json
```

Global memories (who you are, your preferences) live in the root `memory.json`. Project-specific memories (architecture decisions, blockers, progress) live in session folders. This separation means Claude can resume any project by reading 4 files: global memory + that session's context, progress, and memories.

### The Session Resume Flow

This is the feature the original doesn't have, and it's the one that matters most for real work:

```
Tuesday:
  "New session for api-redesign" → creates folder + files
  ... work together for 2 hours ...
  "Save session" → extracts decisions, updates progress, commits

Thursday (new conversation):
  "Load my memories" → Claude reads memory.json, sees sessions
  "Resume api-redesign" → reads context + progress + session memories
  Claude: "Last time we decided on REST over GraphQL [Memory #3],
           finished the auth endpoints [progress], and the next step
           is the rate limiting middleware. Want to continue there?"
```

Cross-session continuity with cited sources. That's the whole point.

## The Skill: Making It One-Line

All three agents and their workflows are encapsulated in a single `skill/SKILL.md` file that Claude reads at the start of each session. It defines:

- **LOAD** — read memory.json, summarize, list sessions
- **RESUME** — load a session's context + progress + memories
- **NEW SESSION** — create a session folder
- **REMEMBER** — extract and save a memory (global vs session)
- **SAVE SESSION** — update progress, extract facts, commit
- **FORGET** — soft-delete a memory
- **CONSOLIDATE** — deduplicate, merge, resolve conflicts
- **QUERY** — answer questions citing memory IDs

To start any session:

```
Read {OWNER}/{REPO}/skill/SKILL.md via GitHub then load my memories
```

Claude reads the skill, knows all the commands, loads your context. One line.

Since January 2026, the SKILL.md follows the [Agent Skills open standard](https://github.com/anthropics/skills), which means it works the same way across claude.ai, Claude Code, and the Claude Agent SDK. Claude Code merged slash commands and skills in the same release, so the skill also works as a `/mem-github` command if placed in the right directory.

## What I Learned

**The three-agent pattern is the real insight.** Not the implementation details. SQLite vs GitHub vs Postgres doesn't matter. What matters is: extract atomic facts, periodically consolidate, and retrieve with citations. Those three operations, separated cleanly, produce emergent intelligence.

**KISS beats infrastructure.** The original needs a running daemon, aiohttp, filesystem watchers, and SQLite. MemGitHub needs a JSON file in a GitHub repo. Both achieve the same goal. The simpler version is easier to debug, easier to understand, and works from any Claude session without setup.

**Session isolation is crucial for real work.** A flat memory store becomes noise quickly. Project-specific context in separate folders keeps things focused. This was my addition to the original pattern.

**Git history is underrated as a memory feature.** Every memory update is a commit with a message. You can audit what Claude remembered, when, and why. Roll back mistakes. Browse on github.com. It's version control for your AI's brain.

**User-triggered beats always-on for most use cases.** The "always on" in the original name implies a running process. But most users don't need real-time memory extraction from a file watcher. They need: "Remember this. Now save my progress. Tomorrow, resume where I left off." Explicit triggers give you control. That said, Claude Code's `/loop` and hooks now let you opt into always-on behavior when you want it — the best of both worlds.

## Try It

The whole system is open source and ready to fork:

👉 **[github.com/hdjebar/MemGitHub](https://github.com/hdjebar/MemGitHub)**

Fork it. Make it private. Connect the GitHub MCP. Give Claude structured, transparent, cross-session memory.

Full setup takes 5 minutes. Instructions in [setup.md](setup.md).

---

*Built with Claude. Stored in Git. Inspired by Google.*

---

## Postscript: March 2026 — Claude's Memory Landscape Has Changed

*Added March 9, 2026*

A lot has happened since I wrote this article. Claude's own memory ecosystem has expanded significantly, and it's worth explaining how MemGitHub fits into the new landscape.

### What Claude Now Offers Natively

**Claude.ai built-in memory** (now free for all users, March 2026) automatically summarizes your conversations and creates a synthesis of key insights. It supports project-level memory isolation, memory import from other AI providers (ChatGPT, Gemini, etc.), and export. You can view and edit memories in Settings.

**Claude Code auto-memory** records build commands, debugging insights, architecture notes, and code style preferences locally in `~/.claude/projects/<project>/memory/MEMORY.md`. It's on by default and accumulates knowledge without you writing anything.

**Claude API memory tool** (client-side) lets Claude create, read, update, and delete files in a `/memories` directory that persists between API sessions. It's designed for agentic workflows where context needs to survive across executions.

### Claude Code's Near-Daemon Capabilities

The biggest change since I wrote the main article: **Claude Code now supports the daemon-like automation** that I originally said wasn't possible. Key features from the 2.1.x releases:

- **`/loop`** — run any prompt on a recurring interval (e.g., `/loop 30m consolidate`), directly mirroring the original's 30-minute ConsolidateAgent timer
- **Cron scheduling** — schedule recurring prompts within a session for even more precise timing control
- **SessionStart/Stop hooks** — auto-trigger memory load at session start and save reminders at session end, no manual typing needed
- **HTTP hooks** — POST JSON to a URL on events, enabling webhook-based triggers for memory operations
- **Background agents** — run memory consolidation in the background without blocking your coding workflow
- **Agent Teams** (experimental, `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) — run a memory-management agent in parallel alongside your coding agent
- **`/batch`** — batch multiple memory operations in a single command
- **Merged slash commands and skills** — SKILL.md files now also function as slash commands, so `/mem-github` works directly

This means MemGitHub in Claude Code can now operate on a spectrum from fully manual ("load my memories") to fully automated (hooks + loop + background agents). The architecture didn't need to change — the same skill, the same commands — it's the tooling around it that caught up.

See [setup.md Step 7](setup.md) for configuration examples.

### Why MemGitHub Still Matters

All three native systems share a common trait: **opacity**. Built-in memory is a managed summary you can view but not deeply structure. Auto-memory is local `.md` files you can read but that lack schema, scoring, or session isolation. The API memory tool is powerful but requires you to build the structure yourself.

MemGitHub provides what none of them do:

- **Git-native auditability** — every memory change is a commit with a message, diffable and revertable
- **Structured schema** — typed memories with importance scoring, hot/cold tiering, and compact format
- **Session folders** — full project context (goals, progress, decisions) isolated per workstream
- **Cross-tool portability** — the same memory repo works from claude.ai, Claude Code, the Agent SDK, or any tool with GitHub MCP access
- **User control** — you decide what gets remembered, when it's consolidated, and what gets archived

Think of it this way: Claude's built-in memory is like your phone's autocomplete learning your typing patterns. MemGitHub is like a structured notebook with chapters, an index, and version history. Both are useful. They serve different purposes.

The best setup in 2026 is to **use both**: let Claude's built-in memory handle casual preferences and conversational context, and use MemGitHub for structured project work, decision trails, and anything you want to audit, version, or share across tools.
