# Miya — Financial News Scraper

## What Miya Does

Miya is a daily financial news scraper that:
- Fetches articles from Yahoo Finance, NY Times, TechCrunch, The Guardian
- Runs FinBERT sentiment analysis on every article
- Classifies articles by topic and subtopic
- Stores results in SQLite database
- Triggers Layla to generate the daily newsletter

Miya runs automatically every day at 10PM SGT.

---

## Architecture

```
Yahoo Finance / NY Times / TechCrunch / The Guardian
        ↓
   scraper.py — fetches raw articles
        ↓
   analyzer.py — FinBERT sentiment scoring
        ↓
   summarizer.py — article summarisation
        ↓
   topic_classifier.py — classifies into topics
        ↓
   news.db — stores everything
        ↓
   Dashboard — visualises sentiment data
```

---

## Key Files

| File | Purpose |
|---|---|
| `/YOUR_HOME/news-scraper/daily_briefing.sh` | Main entry point |
| `/YOUR_HOME/news-scraper/src/scraper.py` | News fetching logic |
| `/YOUR_HOME/news-scraper/src/analyzer.py` | Sentiment analysis |
| `/YOUR_HOME/news-scraper/src/summarizer.py` | Article summarisation |
| `/YOUR_HOME/financial-dashboard/db/news.db` | SQLite database |

---

## Database Schema

```sql
-- Articles table
CREATE TABLE articles (
    id INTEGER PRIMARY KEY,
    date TEXT NOT NULL,
    title TEXT NOT NULL,
    url TEXT NOT NULL,
    published TEXT,
    source TEXT,
    sub_source TEXT,
    description TEXT,
    sentiment_compound REAL,  -- -1.0 to +1.0
    sentiment_pos REAL,
    sentiment_neg REAL,
    sentiment_neu REAL,
    topic TEXT,               -- Broad category
    subtopic TEXT,            -- Specific subtopic
    summary TEXT,
    created_at TEXT DEFAULT datetime('now')
)

-- Newsletters table
CREATE TABLE newsletters (
    date TEXT PRIMARY KEY,
    content TEXT,             -- Full markdown newsletter
    summary TEXT              -- One paragraph summary
)
```

---

## Topics and Subtopics

**Broad Categories:**
- Geopolitics & War
- Energy & Commodities
- Markets & Trading
- Banking & Finance
- Tech & AI
- Others

**Sentiment Scale:**
- +0.5 to +1.0 → Strongly positive (green)
- +0.2 to +0.5 → Positive
- -0.2 to +0.2 → Neutral
- -0.5 to -0.2 → Negative
- -1.0 to -0.5 → Strongly negative (red)

---

## API Endpoints

```
GET /api/dates                    # Available dates with data
GET /api/articles                 # Fetch articles (paginated)
GET /api/articles/by-subtopic     # Articles filtered by subtopic
GET /api/sentiment                # Sentiment summary for a date
GET /api/sentiment/heatmap        # Heatmap data by topic/subtopic/date
GET /api/sentiment/emerging       # Emerging sentiment trends
```

---

## Running Miya Manually

```bash
# Run the full pipeline
bash /YOUR_HOME/news-scraper/daily_briefing.sh 2>&1

# Check latest report
cat /YOUR_HOME/news-scraper/reports/$(date +%Y-%m-%d)_summary.md

# Check database
sqlite3 ~/financial-dashboard/db/news.db \
  "SELECT COUNT(*) FROM articles WHERE date = '$(date +%Y-%m-%d)';"
```

---

## OpenClaw Skill File

```markdown
---
name: miya
description: Daily financial news scraper. Scrapes Yahoo Finance, The Guardian, NYT. Runs FinBERT sentiment analysis and stores results to dashboard DB.
---

# Miya — Financial News Scraper

## When to use this skill
- User asks about today's financial news
- User wants to run the news scraper manually
- User wants to check scraper status or logs
- User wants sentiment analysis results

## Key Locations
- Script: /path/to/news-scraper/daily_briefing.sh
- Database: /path/to/financial-dashboard/db/news.db

## Run Scraper Manually
bash /path/to/news-scraper/daily_briefing.sh 2>&1

## Database Tables
articles, newsletters, users

## Sentiment Scale
-1.0 (very negative) to +1.0 (very positive)
Neutral zone: -0.2 to +0.2
```

---

## Prompt for Claude Code / OpenClaw

Use this prompt to recreate Miya from scratch:

```
Build a daily financial news scraper called Miya with these requirements:

1. SOURCES
   - Yahoo Finance (RSS or scraping)
   - NY Times Business section
   - TechCrunch
   - The Guardian Business section

2. DATA COLLECTION
   - Fetch articles published in last 24 hours
   - Extract: title, url, published date, source, description
   - Deduplicate by URL
   - Store in SQLite database

3. SENTIMENT ANALYSIS
   - Use NLTK VADER for sentiment scoring
   - Score each article: compound, positive, negative, neutral
   - Compound score range: -1.0 to +1.0

4. TOPIC CLASSIFICATION
   - Classify into broad topics:
     - Geopolitics & War
     - Energy & Commodities
     - Markets & Trading
     - Banking & Finance
     - Tech & AI
     - Others
   - Use keyword matching for classification

5. DATABASE SCHEMA
   Create SQLite table: articles
   Columns: id, date, title, url, published, source, sub_source,
   description, sentiment_compound, sentiment_pos, sentiment_neg,
   sentiment_neu, topic, subtopic, summary, created_at

6. NEWSLETTER GENERATION
   After scraping, generate a daily newsletter in markdown format
   grouped by topic, stored in newsletters table

7. SCHEDULING
   Run automatically daily at 10PM local time via cron

8. API ENDPOINTS (FastAPI)
   - GET /api/dates — available dates
   - GET /api/articles — paginated articles
   - GET /api/sentiment/heatmap — sentiment by topic/date
   - GET /api/sentiment/emerging — trending topics

The scraper should be robust with retry logic and error handling.
Log all activity to a daily report file.
```

---

## Thought Process and Decisions

### Decision: NLTK VADER over FinBERT
**Why:** FinBERT is a finance-specific BERT model that would give more accurate financial sentiment. However, VADER is:
- Already part of NLTK (no separate download)
- Fast enough for daily batch processing
- Accurate enough for our use case (broad sentiment trends)
- Free with no API costs

**Trade-off:** Slightly less accurate financial sentiment scoring, but much simpler to deploy and maintain.

### Decision: SQLite over PostgreSQL
**Why:** For a single-user personal dashboard with daily batch inserts and read-heavy dashboard queries, SQLite is:
- Simpler to set up (no server needed)
- Fast enough (database fits in memory)
- Easy to backup (single file)
- Portable (copy file to sync)

**Trade-off:** Not suitable for multiple concurrent writers, but we only have one writer (the nightly scraper).

### Decision: Separate Miya and Layla
**Why:** Originally Miya did both scraping and newsletter generation. Splitting them:
- Makes each agent's responsibility clear
- Allows newsletter regeneration without re-scraping
- Easier to debug when something goes wrong
- Follows single responsibility principle

### Issue: Scraper breaking on article structure changes
**Problem:** News sites occasionally change their HTML structure, breaking scrapers.
**Solution:** Use RSS feeds where available (more stable than HTML scraping). Fall back to HTML scraping with multiple CSS selector options.
