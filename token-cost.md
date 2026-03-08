# Token Cost Guide

## The Evolution

| Version | LOAD cost @ 50 memories | Strategy |
|---------|------------------------|----------|
| v2 (eager, verbose) | ~3,250 tokens | All memories, all fields |
| v3 (compact + hot/cold) | ~1,250 tokens | Compact format, only hot memories |
| **v4 (lazy, summary)** | **~300 tokens** | Summary paragraph + session list only |

v4 is **10x cheaper** than v2 to start a session.

## How v4 Lazy Loading Works

```
Layer 1 — Always loaded (~300 tok):
  memory.json → summary paragraph + session index + stats

Layer 2 — On demand (~25 tok/memory):
  memories/global.json → hot memories, fetched on first QUERY
  memories/archive.json → cold memories, fetched on SEARCH ARCHIVE

Layer 3 — On resume (~500-1,500 tok):
  sessions/{name}/ → context + progress + session memories
```

Most conversations only need Layer 1. Claude reads the summary, knows who you
are, and can have a useful conversation. Details are fetched only when someone
asks "what do you know about X?" or "cite the specific memory about Y."

## Cost Per Operation (v4)

| Operation | Files read | Tokens |
|-----------|-----------|--------|
| **LOAD** | memory.json | ~300 (fixed) |
| **QUERY** (first) | + memories/global.json | +25 × hot_count |
| **QUERY** (cached) | nothing | 0 |
| **RESUME** | + 3 session files | +500-1,500 |
| **REMEMBER** | Read detail + index, write both | ~500 |
| **SAVE SESSION** | Read 4, write 3 | ~1,000-2,000 |
| **CONSOLIDATE** | Read all, write all | ~2,000-5,000 |
| **SEARCH ARCHIVE** | memories/archive.json | +25 × cold_count |

## Typical Session Costs

| Pattern | API calls | Total tokens |
|---------|-----------|-------------|
| Load + chat (no queries) | 1 | ~300 |
| Load + 2 queries | 2 | ~800 |
| Load + resume + work + save | 5-6 | ~2,500 |
| Load + consolidate | 4 | ~3,000 |

## Scaling

| Memory count | Summary (always) | Hot details (on demand) | Archive (explicit) |
|---|---|---|---|
| 10 | ~300 | ~250 | ~0 |
| 50 | ~300 | ~750 | ~500 |
| 100 | ~300 | ~1,250 | ~1,250 |
| 200 | ~300 | ~2,500 | ~2,500 |
| 500 | ~300 | ~6,250 | ~6,250 |

Note: summary stays ~300 tokens regardless of memory count. That's the point.

## Real Money Cost

For claude.ai Pro/Max: included in subscription.

For API users (Sonnet pricing):

| Usage | Est. monthly cost |
|-------|-------------------|
| Light (1 session/day, lazy) | ~$0.05 |
| Medium (3 sessions/day, queries) | ~$0.25 |
| Heavy (10 sessions/day, full) | ~$1.50 |

## The Summary Trick

The summary is a natural language paragraph (~50 tokens) that captures ALL memories:

> "Enterprise architect based in Luxembourg. Works with TOGAF and ArchiMate.
> GitHub: hdjebar. Building an Always On Memory system for Claude. Prefers KISS
> architecture and GitHub as storage backend."

Claude can answer most questions from this alone. Only when someone asks for
specific citations ([Memory #N]) does Claude need to fetch the detail files.

The summary is regenerated on every REMEMBER, FORGET, SAVE SESSION, and CONSOLIDATE.
