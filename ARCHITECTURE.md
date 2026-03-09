# Architecture: Session Isolation and the Event-Driven Question

*Why MemGitHub uses poll-on-read instead of events, what's possible today, and what would need to change.*

---

## The Fundamental Constraint

Every Claude session — web, Code, API, Cowork — starts with a **blank context window**.
There is no shared memory bus, no inter-session messaging, no event subscription between
sessions. Session A cannot know that Session B just wrote a memory.

This is by design (security, privacy, multi-tenancy) and is the single most important
architectural constraint shaping MemGitHub.

```
❌ Not possible today:
┌─────────────┐    event     ┌─────────────┐
│ Claude Web   │───"saved"──→│ Claude Code  │
│ Session A    │             │ Monitor      │
└─────────────┘             └─────────────┘

No Claude session can:
  • Subscribe to events from another session
  • Push notifications to another session
  • Poll for changes in another session in real time
  • Share in-memory state with another session
```

This rules out a true event-driven architecture where a dedicated Claude session
monitors other sessions' memory activity.

---

## What MemGitHub Does Instead: GitHub as Shared State

GitHub is a **shared state layer** that all sessions can independently read and write.
The "event" is a git commit. The "subscription" is a read at session start.

```
✅ What works today:
┌─────────────┐  write   ┌──────────┐  read    ┌─────────────┐
│ Claude Web   │────────→│  GitHub   │←────────│ Claude Code  │
│ Session A    │  commit  │  Repo     │  on load │ Session B    │
└─────────────┘          └──────────┘          └─────────────┘
```

This is **poll-on-read**, not event-driven. The `write_count` field in `memory.json`
is the primitive for change detection: if the count you read at session start differs
from the count at write time, something changed between your reads.

It's not real-time, but it's consistent, auditable, and requires zero infrastructure.

---

## Intra-Session Automation (What `/loop` and Hooks Get You)

Claude Code 2.1+ introduced features that provide daemon-like behavior **within a
single session**:

```
✅ Within one Claude Code session:

┌──────────────────────────────────────────────────┐
│  Claude Code Session                             │
│                                                  │
│  SessionStart hook → auto-load memories          │
│  /loop 30m         → periodic consolidation      │
│  Background agent  → non-blocking consolidation  │
│  Stop hook         → auto-save reminder          │
│                                                  │
│  All of this dies when the session closes.        │
└──────────────────────────────────────────────────┘
```

This is powerful — it mirrors the original Google Always On Memory Agent's timer-based
consolidation. But it's **intra-session**, not **inter-session**. The moment you close
that terminal or browser tab, the loop stops. There is no persistent background process
that survives session end.

### Cowork (macOS Desktop)

Cowork runs Claude in a local VM with file system access and can schedule recurring
tasks. It gets closer to always-on, but is still session-scoped and macOS-only.

---

## The Gap: What True Event-Driven Would Require

A real event-driven MemGitHub would look like this:

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│ Claude Web   │ commit →│ GitHub       │ webhook →│ Relay       │
│ Session      │         │ (memory.json)│         │ Service     │
└─────────────┘         └──────────────┘         └──────┬──────┘
                                                        │
                                                        ▼
                                                ┌──────────────┐
                                                │ Claude API   │
                                                │ (consolidate)│
                                                └──────────────┘
```

This requires components that exist today but live **outside Claude**:

### Option 1: GitHub Webhooks → External Relay → Claude API

GitHub can fire a webhook on every push. An external service (AWS Lambda, Cloudflare
Worker, etc.) receives it, parses the commit, and calls the Claude API to trigger
consolidation or notifications. This works today but violates the KISS principle —
you're now maintaining infrastructure.

### Option 2: Custom MCP Server as Event Bridge

Build an MCP server that:
- Polls the GitHub repo (or receives webhooks)
- Exposes a `check_for_updates` tool
- Claude sessions query it: "Has anything changed since my last read?"

This is feasible with the [MCP SDK](https://github.com/modelcontextprotocol) and
could run on any hosting platform. It adds a dependency but keeps the Claude-side
logic unchanged.

### Option 3: HTTP Hooks as Outbound Notification

Claude Code supports HTTP hooks — POST JSON to a URL on session events. You could
configure a Stop hook that POSTs to an endpoint when a session ends, signaling that
memories may have been updated. But this is **outbound only** — Claude Code can push
events out, not receive them in.

### Option 4: Anthropic Adds Inter-Session Primitives

The cleanest solution would be native Anthropic infrastructure:
- A shared key-value store accessible across sessions
- An event bus or pub/sub between sessions
- Persistent background tasks that survive session close

None of these exist today. They would fundamentally change Claude's architecture
from stateless sessions to stateful agents.

---

## Why Poll-On-Read Is the Right Choice (For Now)

Given the constraints, MemGitHub's approach is deliberately simple:

1. **Every session reads `memory.json` at start** (~300 tokens, always fresh)
2. **`write_count` detects drift** ("something changed since I last read")
3. **GitHub is the source of truth** (not any single session's memory)
4. **No external infrastructure required** (just a repo and an MCP connector)

The tradeoffs are explicit:

| | Event-Driven | Poll-On-Read (MemGitHub) |
|---|---|---|
| **Latency** | Real-time | Session-boundary (minutes to hours) |
| **Infrastructure** | Webhooks + relay + API | None (just GitHub) |
| **Complexity** | High | Low |
| **Reliability** | Depends on relay uptime | Depends on GitHub uptime |
| **Cost** | Relay hosting + API calls | Zero (included in subscription) |
| **KISS** | No | Yes |

For most use cases — resume a project tomorrow, remember decisions across weeks,
consolidate when prompted — poll-on-read is sufficient. Real-time sync between
concurrent sessions is the one thing it doesn't handle well, and that's what
`write_count` conflict detection is for.

---

## Future: What Could Change This

**Short-term (possible today with effort):**
- Custom MCP server as a change-detection bridge
- GitHub Actions that auto-trigger consolidation on push
- Cowork scheduled tasks for periodic repo monitoring

**Medium-term (requires Anthropic platform changes):**
- Persistent background tasks in Claude Code that survive session close
- Shared state between claude.ai and Claude Code sessions
- Inbound webhook support in Claude Code (HTTP hooks are currently outbound-only)

**Long-term (architectural shift):**
- Inter-session event bus or pub/sub
- Native persistent memory layer with structured schema support
- Agent-to-agent communication across sessions

MemGitHub is designed to evolve with these capabilities. The skill commands
(LOAD, SAVE, CONSOLIDATE) are the stable interface — the automation layer
around them can change without touching the core architecture.

---

## Summary

```
Today's reality:
  Claude sessions = isolated
  GitHub repo     = shared state (poll-on-read)
  /loop + hooks   = intra-session automation only
  write_count     = conflict detection, not real-time sync

What's missing:
  Inter-session events
  Persistent background processes
  Inbound webhook listeners in Claude

Design choice:
  KISS > infrastructure
  Poll-on-read > event-driven
  Good enough for 95% of use cases
  write_count catches the other 5%
```

See also: [article.md](article.md) for the full story, [setup.md Step 7](setup.md)
for Claude Code automation examples.
