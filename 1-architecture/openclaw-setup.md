# OpenClaw Setup Guide on Mac Mini

This guide documents the exact steps to set up OpenClaw on a Mac Mini, including all issues encountered and how they were resolved.

---

## Personalise Before You Start

Replace these placeholders throughout this guide with your own values:

| Placeholder | What It Is | How To Find It |
|---|---|---|
| `YOUR_MAC_MINI_IP` | Mac Mini's local IP | `ipconfig getifaddr en0` |
| `YOUR_TOKEN` | OpenClaw gateway token | `openclaw dashboard` |
| `YOUR_OPENCLAW_PORT` | Gateway port (default 18789) | `openclaw config show` |
| `YOUR_WHATSAPP_NUMBER` | Your WhatsApp number | Your phone |

---

## Prerequisites

- Mac Mini (M1/M2/M4 or any always-on Mac)
- Anthropic API key (get one at console.anthropic.com)
- WhatsApp account
- Terminal access

---

## Step 1 — Install OpenClaw

Visit [openclaw.ai](https://openclaw.ai) and download the installer for macOS.

After installation, verify it works:
```bash
openclaw --version
```

---

## Step 2 — Onboard with API Key

**Critical:** Use `openclaw onboard` — NOT the UI config editor.

```bash
openclaw onboard
```

You will be prompted to enter your Anthropic API key. Paste it and press Enter.

**Why this matters:**
Early on, we tried editing the config via the UI — the Save button appeared to work but the API key was never actually saved. The `openclaw onboard` command is the correct and reliable way to configure your API key.

---

## Step 3 — Install and Start the Gateway

The gateway is the server that connects OpenClaw to your devices and WhatsApp.

```bash
openclaw gateway install
openclaw gateway start
```

**Verify it's running:**
```bash
openclaw gateway status
```

**Configure the gateway:**
- Default port: `18789` (check yours with `openclaw config show`)
- Bind: LAN (not localhost) — allows access from other devices on your network

**Gateway URL:**
```
ws://YOUR_MAC_MINI_IP:YOUR_OPENCLAW_PORT
```

To find your Mac Mini's IP:
```bash
ipconfig getifaddr en0
```

---

## Step 4 — Fix IPv6 Issue (Important!)

If you experience LLM timeouts or slow responses, this is likely caused by IPv6 taking priority over IPv4.

**Fix:**
1. Open **System Settings** on Mac Mini
2. Go to **Network** → **Wi-Fi** (or Ethernet) → **Details**
3. Click **TCP/IP** tab
4. Set **Configure IPv6** to **Link-local only**
5. Click **OK**

**Why this happens:**
macOS tries IPv6 first. If the Anthropic API doesn't respond quickly on IPv6, requests time out before falling back to IPv4. Setting IPv6 to Link-local forces all outbound traffic to use IPv4.

---

## Step 5 — Access the Dashboard

There are two ways to access the OpenClaw dashboard:

**From Mac Mini itself:**
```
http://127.0.0.1:YOUR_OPENCLAW_PORT/#token=YOUR_TOKEN
```

**From other devices on the same network (phone, laptop, iPad):**
```
http://YOUR_MAC_MINI_IP:YOUR_OPENCLAW_PORT/#token=YOUR_TOKEN
```

To get your tokenized URL:
```bash
openclaw dashboard
```

**Important distinction:**
- `127.0.0.1` = "this computer talking to itself" — only works on Mac Mini itself
- `YOUR_MAC_MINI_IP` (e.g. `192.168.x.x`) = works from any device on your home network

---

## Step 6 — Device Pairing

When accessing from a different device (phone, laptop), OpenClaw shows "pairing required". This is a security feature — every new device must be explicitly approved.

**Why pairing is separate from the token:**
- The token proves you know the gateway password (Door 1)
- Device pairing proves OpenClaw recognises your specific device (Door 2)
- Both are required — passing Door 1 does not automatically open Door 2

**Fix:**
```bash
openclaw devices list
openclaw devices approve <DEVICE_ID>
```

Once approved, that device is remembered permanently — unless you switch browsers, clear cookies, or use a new device.

**Multiple paired devices accumulating:**
Each new browser session or device creates a new pairing entry. Clean up old ones periodically:
```bash
openclaw devices list
openclaw devices remove <DEVICE_ID>
```

---

## Step 7 — Connect WhatsApp

1. In the OpenClaw dashboard, go to **Devices**
2. Click **Add Device** → **WhatsApp**
3. Scan the QR code with your WhatsApp
4. Once connected, approve the device:

```bash
openclaw devices list
openclaw devices approve <DEVICE_ID>
```

---

## Step 8 — Configure Models

Two models are used for different purposes:

**Main sessions (interactive):**
```
anthropic/claude-sonnet-4-6
```

**Subagents and cron jobs (cost optimised):**
```
anthropic/claude-haiku-4-5-20251001
```

**Why two models?**
Haiku costs significantly less than Sonnet. For routine automated tasks (news scraping, daily briefings, syncs), Haiku is more than capable. Sonnet is reserved for interactive sessions where quality matters.

**Important:** Never change cron job models without considering the cost and quality tradeoffs. Document any model changes in MEMORY.md.

---

## Step 9 — Set Up Workspace

The workspace is a folder of markdown files that agents read on startup. Think of it as shared memory.

**Default location:**
```
~/.openclaw/workspace/
```

**Key files:**

| File | Purpose |
|---|---|
| `AGENTS.md` | How agents should behave — session startup order |
| `SOUL.md` | Agent personality, values, and source of truth rules |
| `USER.md` | Information about you (name, timezone, interests) |
| `REQUIREMENTS.md` | Non-negotiable standing requirements |
| `HEARTBEAT.md` | Periodic check instructions |
| `MEMORY.md` | Long-term memory (main sessions only) |
| `TOOLS.md` | Environment-specific tool notes |

**Important:** Keep workspace files lean. Every word costs tokens on every session start. Target under 2,000 words total across all workspace files.

**Reference docs** (load on demand only — put in `docs/` subfolder):
- Project details
- Detailed task specifications
- Historical decisions

---

## Step 10 — REQUIREMENTS.md (Critical)

REQUIREMENTS.md is a **contract** between you and your agent. It contains non-negotiable standing requirements that must never be forgotten.

Example entries:
```markdown
## Standing Requirements

- Daily news briefing MUST be emailed after every run
- Never change cron job models without explicit approval
- Never change cron schedules without asking first
- Always verify source before making assertions
```

**When to add to REQUIREMENTS.md:**
Any time you find yourself repeating the same instruction more than once, add it here permanently.

---

## Step 11 — SOUL.md — Source of Truth Rule

One critical rule added to SOUL.md after repeated issues with overconfident assertions:

```markdown
## Source of Truth
Always cite where an answer comes from. Never assert confidently without showing the source.
- Time/date → show the message timestamp or run `date` command
- Code behaviour → quote the actual line of code
- DB state → show the query result
- Memory → cite the file and line
```

**Why this matters:**
AI agents can confidently state things that are wrong. Requiring the agent to always show its source catches errors before they cause problems.

---

## Step 12 — Create Agent Skills

Skills are markdown files that teach agents how to use specific tools or perform specific tasks.

**Skills location:**
```
~/.openclaw/skills/YOUR_SKILL_NAME/SKILL.md
```

**Example skill structure:**
```markdown
---
name: skill-name
description: One line description of what this skill does
---

# Skill Name

## When to use this skill
- Use case 1
- Use case 2

## Key Locations
- Database: /path/to/database.db
- Script: /path/to/script.sh

## Key API Endpoints
- GET /api/endpoint — description

## Important Rules
- Rule 1
- Rule 2
```

---

## Step 13 — Set Up Cron Jobs

Cron jobs are scheduled tasks that run automatically.

**View existing cron jobs:**
```bash
openclaw cron list
```

**View a specific cron job's configuration:**
```bash
openclaw cron edit <JOB_ID>
```

**Example cron schedule for this setup:**

| Job | Schedule (SGT) | Model |
|---|---|---|
| Miya & Layla News Pipeline | 10:00 PM | Sonnet |
| Daily Hetzner Sync | 10:30 PM | Haiku |
| NBA Analytics Report | 11:00 PM | Haiku |
| Daily Synology Sync | 11:00 PM | Haiku |
| NBA Predictions vs Actuals | 1:30 PM | Haiku |

**Important:** Never change cron job models, schedules, or payloads without explicit approval. Document all changes in MEMORY.md.

---

## Step 14 — Cost Optimisation Settings

Apply these settings to reduce API costs:

```yaml
cacheRetention: "long"
heartbeat:
  every: "60m"
contextPruning:
  ttl: "30m"
maxConcurrent: 2
subagents:
  maxConcurrent: 3
  model: "anthropic/claude-haiku-4-5-20251001"
```

---

## Step 15 — Prevent Agents Forgetting Fixes

Agents wake up fresh every session. If a fix is not written down, it never happened from the agent's perspective.

**After every significant fix, update MEMORY.md:**
```markdown
## Known Issues and Fixes Applied

### [Issue Name] — [Date Fixed]
- Root cause: [what caused it]
- Fix applied: [what was done]
- Prevention: [how to avoid recurrence]
```

**Four places to document fixes:**

| Where | What to Write |
|---|---|
| `MEMORY.md` | Important fixes that must never be forgotten |
| Agent `SKILL.md` | Rules specific to that agent |
| `memory/YYYY-MM-DD.md` | Recent daily fixes |
| `REQUIREMENTS.md` | Standing rules that must always apply |

**The golden rule: if it's not written down, the agent doesn't know it happened.**

---

## Step 16 — Set Up Synology NAS Backup

If you have a Synology NAS, set up nightly backups:

**Create `~/synology/sync_to_synology.sh`:**
```bash
#!/bin/bash
rsync -avz ~/financial-dashboard/db/news.db ~/synology/openclaw/miya/
rsync -avz ~/financial-dashboard/db/finance.db ~/synology/openclaw/pharsa/
rsync -avz ~/nba-analytics/nba_historical.db ~/synology/openclaw/bruno/
```

Add as OpenClaw cron job at 11PM SGT with `--no-deliver` flag.

---

## Common Issues and Fixes

### Issue: HTTP 401 invalid x-api-key
**Cause:** API key not properly saved via UI
**Fix:** Run `openclaw onboard` and re-enter your API key

### Issue: Error 1006 disconnected
**Cause:** Normal behavior on gateway restart
**Fix:** No action needed — reconnects automatically

### Issue: Gateway not loading
**Fix:**
```bash
openclaw gateway install
openclaw gateway start
```

### Issue: LLM timeouts
**Cause:** IPv6 priority over IPv4
**Fix:** Set Configure IPv6 to "Link-local only" in Mac Mini network settings

### Issue: Save button not working in UI
**Fix:** Use CLI commands instead of UI for configuration changes

### Issue: `pairing required`
**Fix:**
```bash
openclaw devices list
openclaw devices approve <DEVICE_ID>
```

### Issue: Agent forgets fixes after a few days
**Cause:** Fixes were never written to MEMORY.md or SKILL.md
**Fix:** After every significant fix, immediately update the relevant memory file
