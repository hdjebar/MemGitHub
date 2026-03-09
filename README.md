# MemGitHub

Persistent cross-session memory for Claude using a GitHub repo as storage.

**One repo = Claude's brain.** Every memory is a git commit. Works in claude.ai, Claude Code, and the Claude Agent SDK.

## Why MemGitHub When Claude Has Built-In Memory?

As of March 2026, Claude has free built-in memory for all users — it summarizes conversations,
learns preferences, and even lets you import memories from other AI providers. Claude Code
has auto-memory that records build commands and code patterns locally.

So why MemGitHub? Because built-in memory is a **managed black box**, and sometimes you
need a **transparent, structured system you control**.

| | Claude Built-In Memory | Claude Code Auto-Memory | MemGitHub |
|---|---|---|---|
| **Transparency** | View/edit in settings | Local MEMORY.md files | Every change is a git commit |
| **Structure** | Free-text summaries | Free-text notes | Typed, scored, hot/cold tiered |
| **Project isolation** | Project summaries | Per-project directories | Full session folders (context + progress + decisions) |
| **Version history** | No rollback | No rollback | Full git history, diff, revert |
| **Portability** | Export as text blob | Machine-local only | Standard JSON, works anywhere |
| **Cross-tool** | claude.ai only | Claude Code only | Any tool with GitHubMCP |
| **Auditable** | Not really | Partially (local files) | Fully (git log, blame, PR review) |

MemGitHub is for people who want to **see, version, and control** what their AI remembers.

Inspired by [Google's Always On Memory Agent](https://github.com/GoogleCloudPlatform/generative-ai/tree/main/gemini/agents/always-on-memory-agent) — adopted the three-agent architecture, adapted for Claude with GitHub as storage. Full story: **[article.md](article.md)**

## How It Works — Lazy Loading

```
Session start: read memory.json (~300 tokens)
    │
    ├── Claude reads the SUMMARY paragraph
    │       → knows who you are immediately
    │       → sees available sessions
    │       → costs ~300 tokens, always
    │
    ├── "What do you know about my TOGAF work?"
    │       → NOW fetches memories/global.json (on demand)
    │       → answers with [Memory #N] citations
    │
    ├── "Resume session X"
    │       → fetches sessions/X/ files (on demand)
    │
    └── "Remember that..." / "Save session"
        → writes to detail file + regenerates summary
```

**The summary is the trick.** A natural language paragraph that captures the essence of all your memories in ~50 tokens. Claude works from it most of the time. Details are only fetched when citing specific memories or doing a targeted query.

## Structure

```
your-memory-repo/
├── memory.json                ← INDEX: summary + sessions + config (~300 tok)
├── memories/
│   ├── global.json            ← HOT details (importance >= 0.7, on demand)
│   └── archive.json           ← COLD details (< 0.7, explicit search only)
├── sessions/
│   └── {project-name}/
│       ├── context.md
│       ├── progress.md
│       └── memories.json
└── skill/
    └── SKILL.md
```

## Token Cost Comparison

| Version | LOAD cost @ 50 memories | How |
|---------|------------------------|-----|
| v2 (eager, verbose) | ~3,250 tokens | Loads all memories, all fields |
| v3 (compact + hot/cold) | ~1,250 tokens | Compact format, only hot |
| **v4 (lazy, summary)** | **~300 tokens** | Summary paragraph only, details on demand |

Full math: **[token-cost.md](token-cost.md)**

## Commands

| Say | Reads | Cost |
|-----|-------|------|
| Load my memories | memory.json only | ~300 tok |
| What do you know about X? | + memories/global.json | +~25/memory |
| Resume session X | + sessions/X/ files | +~500-1,500 tok |
| Remember that... | Write detail + update summary | ~500 tok |
| Save session | Write session + update summary | ~1,000 tok |
| Consolidate | Read all + write all | ~2,000-5,000 tok |
| Search archive | + memories/archive.json | +~25/memory |

## Quick Start

See **[setup.md](setup.md)** for full installation instructions.

## Docs

- **[setup.md](setup.md)** — Install guide (fork → token → Smithery → system prompt → test)
- **[article.md](article.md)** — From Google's Always On Memory to MemGitHub
- **[token-cost.md](token-cost.md)** — Token math & optimization guide
- **[skill/SKILL.md](skill/SKILL.md)** — The skill Claude reads

## Robustness Features

- **First-run guard** — Claude refuses to proceed if OWNER/REPO are still unconfigured placeholders
- **Hot memory cap** — Auto-archives lowest-importance memory when hot count hits 20, keeping summary regeneration cost bounded
- **Sync drift detection** — `write_count` field detects concurrent writes from multiple sessions before any write
- **System prompt automation** — Optional claude.ai system prompt to auto-load memory at session start (see setup.md Step 6)
- **Agent Skills standard** — Follows the [open standard](https://github.com/anthropics/skills), works across claude.ai, Claude Code, and the Claude Agent SDK

## Credits

Architecture inspired by Google's [Always On Memory Agent](https://github.com/GoogleCloudPlatform/generative-ai/tree/main/gemini/agents/always-on-memory-agent).

## License

MIT
