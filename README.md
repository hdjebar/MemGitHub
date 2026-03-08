# MemGitHub

Persistent cross-session memory for Claude using a GitHub repo as storage.

**One repo = Claude's brain.** Every memory is a git commit. Works in claude.ai and Claude Code.

## Why?

Claude forgets everything between conversations. This system gives Claude persistent memory by storing facts, preferences, and project context in a GitHub repo that Claude reads/writes via the GitHub MCP connector.

## How It Works

```
Any claude.ai or Claude Code session
    │
    ├── Read memory.json (one file = everything)
    │       → global memories (who you are)
    │       → session index (what projects exist)
    │       → config
    │
    ├── "resume session X"
    │       → Read sessions/X/context.md + progress.md + memories.json
    │       → Pick up where you left off
    │
    ├── "remember that..." / "save session"
    │       → Extract facts → write back → git commit
    │
    └── Start any session with:
        "Read {OWNER}/{REPO}/skill/SKILL.md via GitHub then load my memories"
```

## Structure

```
your-memory-repo/
├── memory.json                ← ONE FILE: memories + sessions + config
├── sessions/
│   └── {project-name}/
│       ├── context.md         ← What the project is about
│       ├── progress.md        ← Done / next steps
│       └── memories.json      ← Project-specific memories
└── skill/
    └── SKILL.md               ← Skill: all commands and workflows
```

## Commands

| Say | Does |
|-----|------|
| "Load my memories" | Reads memory.json, summarizes what Claude knows |
| "Resume session X" | Loads session context + progress + memories |
| "New session for X" | Creates session folder + updates memory.json |
| "Remember that..." | Extracts fact → saves to global or session |
| "Save session" | Updates progress, extracts memories, commits |
| "Forget memory #N" | Soft-deletes a memory |
| "Consolidate" | Deduplicates, merges, cleans up |
| "List sessions" | Shows all sessions from memory.json |

## Quick Start

See **[setup.md](setup.md)** for full installation instructions.

## KISS Principles

- **One JSON file** for global state — no database, no vector store
- **Git history = changelog** — every memory update is a commit
- **Works from claude.ai AND Claude Code** — same repo, same memories
- **Human-readable** — browse your memories on github.com
- **User-triggered** — you control when memories are read/written
- **Session isolation** — per-project context in separate folders

## License

MIT
