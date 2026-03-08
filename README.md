# MemGitHub

Persistent cross-session memory for Claude using a GitHub repo as storage.

**One repo = Claude's brain.** Every memory is a git commit. Works in claude.ai and Claude Code.

## Why?

Claude has built-in memory, but it's a black box. You can't see what it remembers, organize it by project, or resume a work session from last week. MemGitHub gives you structured, transparent, cross-session memory that you control.

Inspired by [Google's Always On Memory Agent](https://github.com/GoogleCloudPlatform/generative-ai/tree/main/gemini/agents/always-on-memory-agent) — adopted the three-agent architecture, adapted it for Claude with GitHub as storage instead of SQLite. Read the full story: **[article.md](article.md)**

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

## Docs

- **[setup.md](setup.md)** — 5-minute install guide (fork → token → Smithery → test)
- **[article.md](article.md)** — The full story: from Google's Always On Memory to MemGitHub
- **[skill/SKILL.md](skill/SKILL.md)** — The skill Claude reads (all commands and workflows)

## KISS Principles

- **One JSON file** for global state — no database, no vector store
- **Git history = changelog** — every memory update is a commit
- **Works from claude.ai AND Claude Code** — same repo, same memories
- **Human-readable** — browse your memories on github.com
- **User-triggered** — you control when memories are read/written
- **Session isolation** — per-project context in separate folders

## Credits

Architecture inspired by Google's [Always On Memory Agent](https://github.com/GoogleCloudPlatform/generative-ai/tree/main/gemini/agents/always-on-memory-agent). The three-agent pattern (Ingest → Consolidate → Query) is theirs. The GitHub-as-storage adaptation, session folders, and KISS implementation are new.

## License

MIT
