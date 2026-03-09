# Setup Guide — MemGitHub

Give Claude persistent memory across sessions in 3 steps.

---

## Step 1: Create Your Private Memory Repo

Your memories are personal — keep them in a private repo.

### Option A: Fork this repo (recommended)

1. Click **Fork** on [github.com/hdjebar/MemGitHub](https://github.com/hdjebar/MemGitHub)
2. Name it anything you want (e.g. `my-memory`, `claude-brain`)
3. **Set to Private** ✔️
4. Click **Create Fork**

### Option B: Clone manually

```bash
git clone https://github.com/hdjebar/MemGitHub.git my-memory
cd my-memory
git remote remove origin
git remote add origin https://github.com/YOUR_USERNAME/my-memory.git
git push -u origin main
```

Then go to repo Settings → Danger Zone → Change visibility → **Private**.

---

## Step 2: Create a GitHub Token Scoped to Your Repo

1. Go to [github.com/settings/tokens?type=beta](https://github.com/settings/tokens?type=beta)  
   *(Fine-grained personal access tokens)*
2. Click **Generate new token**
3. Configure:
   - **Token name:** `MemGitHub` (or anything)
   - **Expiration:** 90 days (or longer)
   - **Repository access:** Select **"Only select repositories"**
     → Pick your private memory repo (e.g. `my-memory`)
   - **Permissions:**
     - **Contents:** Read and Write
     - **Metadata:** Read-only (required)
   - Everything else: No access
4. Click **Generate token**
5. **Copy the token** — you'll need it in the next step

⚠️ **Minimal scope = maximum security.** This token can ONLY read/write files in your memory repo. Nothing else.

---

## Step 3: Connect GitHub MCP in Claude

### For claude.ai

1. Go to [smithery.ai/server/@smithery-ai/github](https://smithery.ai/server/@smithery-ai/github)
2. Click **Connect** (or **Install** if first time)
3. When prompted for **Personal Access Token**, paste the token from Step 2
4. Click **Connect to Claude** → this adds the GitHub MCP to your claude.ai

### For Claude Code

Add to your Claude Code MCP config (`.claude/settings.json` or via `claude mcp add`):

```json
{
  "mcpServers": {
    "github": {
      "type": "url",
      "url": "https://server.smithery.ai/@smithery-ai/github/mcp?api_key=YOUR_SMITHERY_KEY"
    }
  }
}
```

Or use the official GitHub MCP server directly:
```bash
claude mcp add github -- npx -y @modelcontextprotocol/server-github
```
Set `GITHUB_PERSONAL_ACCESS_TOKEN` env var to your token from Step 2.

**Tip:** If you already have GitHub MCP connected in claude.ai, Claude Code can
reuse those connectors automatically. They appear as `claude.ai connectors` in
your session. To opt out, set `ENABLE_CLAUDEAI_MCP_SERVERS=false`.

---

## Step 4: Configure the Skill

Open `skill/SKILL.md` in your forked repo and update the owner/repo:

```
OWNER = "YOUR_GITHUB_USERNAME"
REPO  = "YOUR_REPO_NAME"
```

For example, if your fork is `alice/claude-brain`:
```
OWNER = "alice"
REPO  = "claude-brain"
```

Commit this change.

> ⚠️ **This step is required.** The skill has a first-run guard that will refuse to proceed
> until OWNER and REPO are set to real values. Claude will tell you if they are still placeholders.

---

## Step 5: Test It

Open a new claude.ai conversation and type:

```
Read YOUR_USERNAME/YOUR_REPO/skill/SKILL.md via GitHub then load my memories
```

Claude should:
1. Read the skill → understand all commands
2. Read `memory.json` → see it's empty
3. Say: "No memories yet. Want to start a new session or tell me about yourself?"

Then try:
```
Remember that I'm a software developer based in Berlin who prefers TypeScript
```

Claude will extract the facts and commit them to your repo. Check github.com to see the commit!

---

## Step 6: Automate the Memory Load (Optional but Recommended)

Without automation, you must type the skill path every new conversation. To avoid this,
add a **system prompt** in claude.ai:

1. Go to **claude.ai → Settings → Profile → Custom Instructions**
2. Add the following, replacing the placeholders:

```
At the start of every conversation, silently read
YOUR_USERNAME/YOUR_REPO/skill/SKILL.md via GitHubMCP, then run the LOAD command
from that skill (read memory.json, present the summary, list sessions).
Do this before responding to the user's first message.
```

With this in place, Claude will automatically load your memory at the start of every
conversation — no trigger phrase needed.

### Alternative: Upload as a Custom Skill in claude.ai

Since January 2026, claude.ai supports **custom skills** on Pro, Max, Team, and
Enterprise plans. You can upload the `skill/` folder as a zip file:

1. Go to **Settings → Features → Skills**
2. Click **Upload Skill** and select a zip of the `skill/` directory
3. Claude will discover and invoke it automatically when you mention memory commands

This avoids the system prompt entirely — Claude will auto-detect when the skill is relevant.

### For Claude Code

Add to your project's `CLAUDE.md` or `~/.claude/CLAUDE.md`:

```markdown
## Memory
At session start, read YOUR_USERNAME/YOUR_REPO/skill/SKILL.md via GitHub
then execute the LOAD command.
```

Alternatively, copy the `skill/` folder into `.claude/skills/mem-github/` in your
project or `~/.claude/skills/mem-github/` globally. Claude Code auto-discovers
skills in these directories and via `--add-dir`.

### How MemGitHub Relates to Claude Code Auto-Memory

Claude Code has its own **auto-memory** system (`~/.claude/projects/<project>/memory/MEMORY.md`)
that records build commands, debugging insights, and code style preferences locally.
The two systems are complementary:

| | Claude Code Auto-Memory | MemGitHub |
|---|---|---|
| **Scope** | One machine, one project | Cross-device, cross-tool |
| **Storage** | Local filesystem | GitHub (versioned, auditable) |
| **Content** | Code patterns, build commands | Identity, decisions, project context |
| **Written by** | Claude automatically | Claude on your command |
| **Survives** | Machine wipes? No | Machine wipes? Yes |

Use auto-memory for code-level patterns. Use MemGitHub for everything else.

---

## Step 7: Advanced Claude Code Automation (Optional)

Claude Code 2.1+ introduced features that make MemGitHub's operations automatable —
closing the gap with the "always-on" daemon model that inspired this project.

### Auto-Load with SessionStart Hooks

Use a [hook](https://code.claude.com/docs/en/hooks) to auto-load memories when any
Claude Code session starts. Add to `.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "type": "command",
        "command": "echo 'Read YOUR_USERNAME/YOUR_REPO/skill/SKILL.md via GitHub then load my memories'"
      }
    ]
  }
}
```

This injects the load instruction at the start of every session without needing
a CLAUDE.md prompt.

### Auto-Save with Stop Hooks

Similarly, use a `Stop` hook to remind Claude to save session before ending:

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "command",
        "command": "echo 'If there are unsaved memories, run SAVE SESSION before stopping.'"
      }
    ]
  }
}
```

### Periodic Consolidation with `/loop`

The `/loop` command runs a prompt on a recurring interval. Use it for periodic
memory consolidation during long work sessions:

```
/loop 30m consolidate memories if write_count has increased
```

This mirrors Google's Always On Memory Agent's `ConsolidateAgent` timer — the
original ran every 30 minutes, and now Claude Code can do the same.

### Cron-Scheduled Memory Maintenance

Claude Code now supports **cron scheduling** for recurring prompts within a session.
You can schedule consolidation or archive checks on a timer, which gives you
daemon-like behavior without a separate process.

### Background Agents for Heavy Operations

For large memory stores, run consolidation as a **background agent** so it doesn't
block your main session:

```
Run consolidation in the background: read all memory files, deduplicate, and
write back. Notify me when done.
```

Background agents run autonomously and send completion notifications when finished.

### Agent Teams (Experimental)

With `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, you can run multiple agents in
parallel — for example, one agent working on code while another handles memory
operations in a separate worktree. This enables true separation of concerns
between your coding workflow and your memory management.

---

## How It Works After Setup

Every new conversation, just say:

```
Read YOUR_USERNAME/YOUR_REPO/skill/SKILL.md via GitHub then load my memories
```

Or shorter, once Claude knows the pattern:
```
Load my memories from YOUR_USERNAME/YOUR_REPO
```

### Session workflow

```
Session 1 (tuesday):
  "Load my memories" → Claude reads memory.json
  "New session for api-redesign" → creates session folder
  ... work together ...
  "Save session" → commits progress + memories

Session 2 (thursday, new conversation):
  "Load my memories" → Claude reads memory.json, sees sessions
  "Resume api-redesign" → reads context + progress + session memories
  ... picks up exactly where you left off ...
  "Save session" → commits updated progress
```

---

## Security Notes

- Your memory repo should be **private**
- The GitHub token is scoped to **only your memory repo**
- Claude never stores passwords, API keys, or secrets in memory (extraction rules filter these out)
- All data lives in YOUR GitHub account — you own it, you can delete it anytime
- Git history lets you audit every change Claude made

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Not Found" when reading files | Check token has `Contents: Read and Write` permission on the correct repo |
| "SHA required" when writing | Always read the file first to get its SHA, then write with that SHA |
| Token expired | Generate a new one at github.com/settings/tokens and update Smithery |
| Claude doesn't follow the skill | Make sure the skill path is correct: `{OWNER}/{REPO}/skill/SKILL.md` |
| Private repo not accessible | Token must be Fine-grained with "Only select repositories" pointing to your fork |
| Claude says OWNER/REPO not configured | Edit `skill/SKILL.md` in your fork, set the real values, commit |
| Two conversations wrote at the same time | Claude will warn you if `write_count` changed — confirm re-read before proceeding |
| Conflicts with Claude Code auto-memory | No conflict — they serve different purposes (see comparison table above) |
