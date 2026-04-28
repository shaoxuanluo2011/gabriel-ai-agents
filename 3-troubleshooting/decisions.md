# Troubleshooting and Key Decisions

This document captures real issues encountered during development and deployment, and the reasoning behind major decisions.

---

## OpenClaw Issues

### Issue: API Key Not Saving via UI
**Symptom:** Entered API key in OpenClaw UI, clicked Save, but got HTTP 401 errors.
**Root cause:** The UI's Save button had a bug — it appeared to save but the key was not persisted.
**Fix:** `openclaw onboard`
**Lesson:** Always use CLI for critical configuration. UI is convenient but CLI is authoritative.

---

### Issue: LLM Timeouts
**Symptom:** Agent requests to Anthropic API timing out randomly.
**Root cause:** macOS was trying IPv6 connections first. The Anthropic API was slow to respond on IPv6.
**Fix:** Set IPv6 to "Link-local only" in Mac Mini network settings.
**Lesson:** IPv6 issues are subtle and hard to diagnose. Check network configuration when seeing intermittent timeouts.

---

### Issue: Error 1006 on Dashboard
**Symptom:** Browser console showed WebSocket Error 1006.
**Root cause:** Normal reconnection behavior after gateway restart.
**Fix:** No action needed — reconnects automatically.
**Lesson:** Not every error requires action. Understand what's normal behavior.

---

### Issue: `pairing required` After Entering Token
**Symptom:** Entered correct token but still seeing "pairing required".
**Root cause:** Token and device pairing are two separate security checks.
- Token = proves you know the gateway password (Door 1)
- Pairing = proves OpenClaw recognises your device (Door 2)
**Fix:** `openclaw devices list` then `openclaw devices approve <ID>`
**Lesson:** Passing one security check does not automatically pass the other.

---

### Issue: Agent Forgets Fixes After a Few Days
**Symptom:** Agent repeats mistakes that were previously fixed.
**Root cause:** Fixes were discussed in conversation but never written to MEMORY.md or SKILL.md. Agents wake up fresh every session.
**Fix:** After every significant fix, immediately update the relevant memory file.
**Lesson:** If it's not written down, the agent doesn't know it happened. Treat MEMORY.md like a logbook.

---

### Issue: Overconfident Agent Assertions
**Symptom:** Agent confidently stated things that were wrong — wrong dates, wrong code behavior, wrong DB state.
**Root cause:** Agent was asserting from memory rather than verifying from source.
**Fix:** Added Source of Truth rule to SOUL.md requiring agents to always cite their source before asserting.
**Lesson:** Require evidence before accepting agent assertions. "Show me the query result" is always valid.

---

## Database Issues

### Issue: team_abbr Missing for 2024-25 Season
**Symptom:** API endpoints returning empty team data despite data existing in database.
**Root cause:** Data was inserted before `team_abbr` column was added. `INSERT OR IGNORE` silently skipped updates for existing rows.
**Investigation:**
```sql
SELECT COUNT(*) FROM team_advanced_stats
WHERE season = '2024-25' AND team_abbr != '';
-- Result: 0
```
**Fix:**
```sql
UPDATE team_advanced_stats
SET team_abbr = (
    SELECT DISTINCT team_abbr
    FROM team_game_logs
    WHERE team_game_logs.team_id = team_advanced_stats.team_id
    AND team_game_logs.season = '2024-25'
    LIMIT 1
)
WHERE season = '2024-25' AND (team_abbr IS NULL OR team_abbr = '');
```
Also changed `INSERT OR IGNORE` to `INSERT OR REPLACE` at line 403 of `historical_scraper.py`.
**Lesson:** `INSERT OR IGNORE` is appropriate for immutable data. For stats that get recalculated, use `INSERT OR REPLACE`.

---

### Issue: tov_pct Stored as Decimal Not Percentage
**Symptom:** Team turnover percentage showing as 0.1 instead of 10%.
**Root cause:** NBA API returns tov_pct as decimal (0.10 = 10%). Frontend was not multiplying.
**Fix:** Multiply by 100 in frontend display.
**Lesson:** Always verify units when working with external APIs. Document unit conventions explicitly.

---

### Issue: Missing Packages on Hetzner Server
**Symptom:** `ModuleNotFoundError` for anthropic, openai, pdfplumber after deploying.
**Root cause:** Packages installed on Mac Mini over months were never added to `requirements.txt`.
**Fix:** Install missing packages then update requirements.txt:
```bash
venv/bin/pip freeze > requirements.txt
```
**Lesson:** Always update requirements.txt when installing new packages.

---

## Security Issues Fixed

### Issue: Hardcoded Gmail Credentials in daily_briefing.sh
**Symptom:** Gmail app password visible in plain text in shell script.
**Risk:** Anyone with access to the script can read the password. If committed to GitHub, publicly exposed.
**Fix:** Moved to `.env` file. Script now reads `$GMAIL_APP_PASSWORD` from environment.
**Lesson:** Never hardcode credentials in scripts. Always use environment variables.

---

### Issue: Guardian API Key Hardcoded as Fallback in scraper.py
**Symptom:** API key hardcoded as default value in Python code.
**Risk:** If code is shared or committed to GitHub, API key is exposed.
**Fix:** Removed hardcoded fallback. Key must now be set in `.env` or scraper skips Guardian.
**Lesson:** Hardcoded fallback values for API keys are dangerous. Fail loudly instead.

---

### Issue: SECRET_KEY Hardcoded as Fallback in auth.py
**Symptom:** JWT signing key hardcoded as a weak default value.
**Risk:** Anyone knowing the default key could forge JWT tokens and bypass authentication.
**Fix:** Removed fallback. Server now raises `RuntimeError` if SECRET_KEY is not set in `.env`.
**Lesson:** Never provide weak fallbacks for security-critical values. Force explicit configuration.

---

### Issue: Bots Scanning Server Immediately After Deployment
**Symptom:** Server logs showing hundreds of requests to `/api/.env`, `/api/config` within minutes.
**Root cause:** Automated bots scan every new public IP address within minutes of it going live.
**What was seen:**
```
GET /api/.env → 404
GET /api/config → 404
GET /api/stripe/config → 404
```
**Fix:** Set up Cloudflare Bot Fight Mode.
**Lesson:** Any public IP will be scanned immediately. This is normal. Cloudflare stops these before they reach your server.

---

## Deployment Issues

### Issue: Nginx 500 Error (Permission Denied)
**Symptom:** 500 Internal Server Error after deploying frontend.
**Error log:** `stat() failed (13: Permission denied)`
**Root cause:** Nginx runs as `www-data` user but files were owned by `root`. Parent directory also needed permissions.
**Fix:**
```bash
chmod -R 755 /opt
chmod -R 755 /opt/financial-dashboard/frontend/dist
chown -R www-data:www-data /opt/financial-dashboard/frontend/dist
systemctl restart nginx
```
**Lesson:** Always check entire directory path permissions, not just the target folder.

---

### Issue: IPv6 vs IPv4 on Hetzner
**Decision:** Add IPv4 even though it costs €0.50/month extra.
**Why:** Many home and mobile internet connections don't support IPv6 yet. Guests on IPv4-only connections would fail to access the dashboard.
**Lesson:** IPv6-only is technically correct but practically problematic for consumer-facing services.

---

## Architecture Decisions

### Decision: SQLite over PostgreSQL
**Reasoning:**
- Single user with read-heavy dashboard queries
- Batch writes only during nightly jobs (no concurrent writers)
- Simple backup (copy single file)
- No server to manage
**When this breaks:** Multiple simultaneous writers, or >100 concurrent users.

---

### Decision: Mac Mini as Primary Server with Minimal Cloud
**Reasoning:**
- Agents need access to local files and databases — easier on Mac Mini
- Dashboard needs to be publicly accessible — needs cloud
- Mac Mini already running 24/7 — free compute
- Hetzner costs only €3.79/month for public-facing piece
- Sensitive financial data stays at home, never in cloud

---

### Decision: Two AI Models (Sonnet + Haiku)
**Reasoning:** Haiku costs ~20x less than Sonnet per token. Cron jobs (health checks, syncs, scraping) don't need Sonnet quality. Interactive sessions still use Sonnet.
**Savings:** Significant reduction in daily API costs.
**Rule:** Never change cron job models without explicit approval.

---

### Decision: Mock API Approach vs Cloud Deployment (Lesson Learned)
**Context:** Wanted to share dashboard demo without exposing real data.
**Approach tried:** Build `mockApi.js` to intercept all API calls with static data.
**Problem:** Dashboard has very specific API response formats accumulated over months. Replicating them perfectly without studying every component first led to format mismatches and blank screens.
**Approach adopted:** Deploy real dashboard to Hetzner with guest mode (fakified amounts).
**Why it won:** Real dashboard already works perfectly. Guest mode already built in. No risk of format mismatches. Real data is more impressive.
**Lesson:** A working system is your best asset. Don't over-engineer alternatives when the real thing can be deployed cheaply.

---

### Decision: Workspace Trimming (80% Reduction)
**Context:** Original workspace was 5,514 words — expensive in tokens loaded every session.
**Solution:** Trimmed to ~1,095 words. Verbose content moved to `docs/` folder (load on demand only).
**Impact:** Significant reduction in per-session API cost.
**Trade-off:** Agents need to explicitly load reference docs. Slightly more friction for complex tasks.

---

### Decision: Separate Bruno from Financial Dashboard
**Reasoning:**
- Bruno's data pipeline, models, and scheduling are complex enough to warrant isolation
- Financial dashboard works without Bruno running
- Different startup commands, different ports, different databases
- Easier to debug issues in isolation
**Trade-off:** Two separate servers to manage. Two separate startup processes.

---

## Commodity Sentiment Override (Miya)

**Problem:** FinBERT (the sentiment AI model) reads phrases like "oil surges" or "crude tops $110" as **positive** sentiment because words like "surges" and "tops" sound linguistically positive.

**Reality:** Rising oil prices are economically negative — they cause inflation, higher costs for businesses and consumers.

**Solution:** Added pattern matching in `analyzer.py` to detect commodity price rise patterns and flip the sentiment from positive to negative:

```python
_PRICE_RISE_PATTERNS = [
    r'oil\b.{0,60}(surge|soar|jump|rise|spike)',
    r'crude.{0,60}(surge|soar|jump)',
    r'gas price.{0,60}(surge|soar|jump)',
]

if commodity_price_rise and compound > 0:
    compound = -abs(compound)
```

**Lesson:** Domain-specific AI models still need domain-specific post-processing. Financial context requires understanding economic impact, not just linguistic sentiment.
