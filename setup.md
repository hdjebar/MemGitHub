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
