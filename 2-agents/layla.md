# Layla — Newsletter Generator

## What Layla Does

Layla is a daily newsletter generator that:
- Pulls articles from Miya's database
- Groups them by topic and sentiment
- Generates a structured markdown newsletter
- Stores it in the newsletters table
- Makes it available via the dashboard

Layla runs after Miya completes each evening.

---

## Architecture

```
news.db (articles table)
        ↓
   Layla reads articles for the day
        ↓
   Groups by topic
        ↓
   Generates markdown newsletter
        ↓
   news.db (newsletters table)
        ↓
   Dashboard displays newsletter
```

---

## Database Schema

```sql
-- Newsletters table
CREATE TABLE newsletters (
    date TEXT PRIMARY KEY,
    content TEXT,    -- Full markdown newsletter
    summary TEXT     -- One paragraph executive summary
)
```

---

## Newsletter Format

Each newsletter follows this structure:

```markdown
# 📰 Daily Financial News Summary
**Date:** YYYY-MM-DD | **Total articles:** N

## 📊 Sources
- **Yahoo Finance**: N articles
- **NY Times**: N articles
- **TechCrunch**: N articles
- **The Guardian**: N articles

## 🟡 Geopolitics & War
*N articles*
- **[Article Title](url)** — *Source* — timestamp
  > Description

## 🟠 Energy & Commodities
...

## 🟢 Markets & Trading
...

## 🔵 Banking & Finance
...

## 🟣 Tech & AI
...

## 📌 Others
...
```

---

## API Endpoints

```
GET /api/newsletter?date=YYYY-MM-DD    # Fetch newsletter for date
PUT /api/newsletter/summary            # Update newsletter summary
```

---

## Key Rules

1. **Never generate newsletter if Miya hasn't scraped for that date**
2. **Always verify date exists in articles table before generating**
3. **Dependency on Miya** — Layla must run after Miya completes

```bash
# Verify Miya has scraped for today
sqlite3 ~/financial-dashboard/db/news.db \
  "SELECT COUNT(*) FROM articles WHERE date = '$(date +%Y-%m-%d)';"

# Only run Layla if count > 0
```

---

## OpenClaw Skill File

```markdown
---
name: layla
description: Daily newsletter generator. Pulls articles from Miya's database, summarizes them, and assembles a daily financial newsletter.
---

# Layla — Newsletter Generator

## When to use this skill
- User asks for today's newsletter
- User wants to generate or view a newsletter
- User asks about historical newsletters

## Key Locations
- Database: /path/to/financial-dashboard/db/news.db
- API base: http://localhost:8000/

## Key API Endpoints
- GET /api/newsletter?date=YYYY-MM-DD
- PUT /api/newsletter/summary

## Important Rules
- Never generate newsletter if Miya hasn't scraped for that date
- Always verify date exists before generating
```

---

## Prompt for Claude Code / OpenClaw

```
Build a daily newsletter generator called Layla with these requirements:

1. DATA SOURCE
   - Read from SQLite database: news.db
   - Table: articles
   - Filter by today's date

2. NEWSLETTER STRUCTURE
   Generate a markdown newsletter with:
   - Header: date, total article count, source breakdown
   - Sections by topic: Geopolitics & War, Energy & Commodities,
     Markets & Trading, Banking & Finance, Tech & AI, Others
   - Each article: title (linked), source, timestamp, description

3. TOPIC ICONS
   - 🟡 Geopolitics & War
   - 🟠 Energy & Commodities
   - 🟢 Markets & Trading
   - 🔵 Banking & Finance
   - 🟣 Tech & AI
   - 📌 Others

4. EXECUTIVE SUMMARY
   Generate a 2-3 sentence summary of the day's key themes
   Store separately in summary column

5. STORAGE
   Store in newsletters table:
   - date (PRIMARY KEY)
   - content (full markdown)
   - summary (executive summary)

6. DEPENDENCY CHECK
   Before running, verify articles exist for the date.
   If no articles found, send WhatsApp alert and abort.

7. API ENDPOINTS (FastAPI)
   - GET /api/newsletter?date=YYYY-MM-DD
   - PUT /api/newsletter/summary

8. SCHEDULING
   Run automatically after Miya completes (10:30PM SGT)
```

---

## Thought Process and Decisions

### Decision: Separate from Miya
**Why:** Separating newsletter generation from scraping means:
- Newsletter can be regenerated without re-scraping
- Different scheduling flexibility
- Cleaner separation of concerns

### Decision: Markdown format
**Why:** Markdown renders beautifully in the dashboard's NewsletterViewer component. It's also readable as plain text in WhatsApp.

### Decision: Store full content + summary separately
**Why:** The full newsletter is needed for the dashboard viewer. The summary is useful for WhatsApp notifications and quick previews without loading the full content.

### Issue: Newsletter generated before scraping complete
**Problem:** If Layla runs too early, it generates an incomplete newsletter.
**Solution:** Layla checks article count before generating. If count is below threshold (e.g. < 10 articles), it waits and retries. Added a watchdog cron at 11:30PM to verify both Miya and Layla completed successfully.
