# OpenClaw Skill Files

These are the SKILL.md files for each agent.
Replace `/Users/YOUR_USERNAME/` with your own home directory path.

---

## Miya — Financial News Scraper

```markdown
---
name: miya
description: Daily financial news scraper. Scrapes Yahoo Finance, The Guardian, NYT, TechCrunch, Benzinga. Runs FinBERT sentiment analysis and stores results to dashboard DB.
---

# Miya — Financial News Scraper

## When to use this skill
- User asks about today's financial news
- User wants to run the news scraper manually
- User wants to check scraper status or logs
- User wants sentiment analysis results

## Key Locations
- Script: /YOUR_HOME/news-scraper/daily_briefing.sh
- Reports: /YOUR_HOME/news-scraper/reports/
- Database: /YOUR_HOME/financial-dashboard/db/news.db
- Source: /YOUR_HOME/news-scraper/src/
- Synology backup: /YOUR_HOME/synology/openclaw/miya/

## Source Files
- scraper.py — news scraping logic (Yahoo, NYT, Guardian, TechCrunch, Benzinga)
- analyzer.py — FinBERT sentiment analysis (falls back to VADER)
- summarizer.py — article summarisation
- utils.py — shared utilities

## Run Scraper Manually
bash /YOUR_HOME/news-scraper/daily_briefing.sh 2>&1

## Check Latest Report
cat /YOUR_HOME/news-scraper/reports/$(date +%Y-%m-%d)_summary.md

## Database Tables
articles, newsletters, users

## Key API Endpoints
- GET /api/articles — fetch articles (paginated)
- GET /api/articles/by-subtopic — articles by subtopic
- GET /api/sentiment — sentiment summary for a date
- GET /api/sentiment/heatmap — heatmap by topic/subtopic/date
- GET /api/sentiment/emerging — emerging sentiment trends
- GET /api/dates — available dates with data

## Sentiment Scale
-1.0 (very negative) to +1.0 (very positive)
Neutral zone: -0.2 to +0.2

## Important Notes
- Email report is mandatory after every run — see REQUIREMENTS.md
- Commodity price rises (oil, gas) are manually corrected — FinBERT reads "surges" as positive
- Guardian API key required in .env
- Benzinga API key required in .env for financial wire articles
```

---

## Layla — Newsletter Generator

```markdown
---
name: layla
description: Daily newsletter generator. Pulls articles from Miya's database, summarises them, and assembles a daily financial newsletter.
---

# Layla — Newsletter Generator

## When to use this skill
- User asks for today's newsletter
- User wants to generate or view a newsletter
- User asks about historical newsletters
- User wants to check newsletter delivery status

## Key Locations
- Database: /YOUR_HOME/financial-dashboard/db/news.db (newsletters table)
- API base: http://localhost:YOUR_PORT/
- Synology backup: /YOUR_HOME/synology/openclaw/layla/

## Database Tables
articles, newsletters, users

## Key API Endpoints
- GET /api/newsletter — fetch newsletter (add ?date=YYYY-MM-DD for specific date)
- PUT /api/newsletter/summary — update newsletter summary

## Check Newsletter Database
sqlite3 ~/financial-dashboard/db/news.db "SELECT date FROM newsletters ORDER BY date DESC LIMIT 5;"

## Dependencies
- Runs AFTER Miya confirms scrape is complete
- Pulls from articles table in news.db
- Stores output in newsletters table

## Important Rules
- Never generate newsletter if Miya hasn't scraped for that date
- Always verify date exists before generating
- Email newsletter is mandatory after generation — see REQUIREMENTS.md
```

---

## Pharsa — Personal Finance

```markdown
---
name: pharsa
description: Personal finance agent. Manages balance sheets, income statements, expense categorisation, and bank reconciliation.
---

# Pharsa — Personal Finance

## When to use this skill
- User asks about personal finances
- User wants to check balance sheet or income statement
- User wants to categorise expenses
- User wants to reconcile bank statements

## Key Locations
- Backend: /YOUR_HOME/financial-dashboard/backend/pharsa_api.py
- Database: /YOUR_HOME/financial-dashboard/db/finance.db
- API base: http://localhost:YOUR_PORT/api/pharsa/
- Synology backup: /YOUR_HOME/synology/openclaw/pharsa/

## Database Tables
accounts, financial_periods, financial_statements, journal_entries,
period_balances, transactions, txn_dbss, txn_ocbc, txn_citi,
txn_trus, txn_dbsc, txn_mari, uploaded_statements

## Key API Endpoints
- GET /api/pharsa/periods
- GET /api/pharsa/periods/{period_id}/balance-sheet
- GET /api/pharsa/periods/{period_id}/income-statement
- GET /api/pharsa/periods/{period_id}/trial-balance
- GET /api/pharsa/periods/{period_id}/reconciliation
- POST /api/pharsa/periods/{period_id}/run-preprocessor
- POST /api/pharsa/periods/{period_id}/confirm-categories
- POST /api/pharsa/periods/{period_id}/submit
- POST /api/pharsa/periods/{period_id}/upload
- GET /api/pharsa/periods/{period_id}/transactions
- PATCH /api/pharsa/transactions/{txn_id}/category

## Important Rules
- Always call POST confirm-categories after any DR/CR change
- Income statement, trial balance, balance sheet must source from journal entries
- Never run destructive commands without asking
- trash > rm for file deletions
- ANTHROPIC_API_KEY required in .env for AI-assisted categorisation
```

---

## Luoyi — Monthly Budget

```markdown
---
name: luoyi
description: Monthly budget tracking agent. Manages budget categories, tracks spending vs budget, and generates monthly budget reports.
---

# Luoyi — Monthly Budget

## When to use this skill
- User asks about monthly budget
- User wants to track spending
- User wants to compare budget vs actual
- User wants monthly budget report

## Key Locations
- Backend: /YOUR_HOME/financial-dashboard/backend/luoyi_api.py
- Database: /YOUR_HOME/financial-dashboard/db/finance.db
- API base: http://localhost:YOUR_PORT/api/luoyi/
- Synology backup: /YOUR_HOME/synology/openclaw/luoyi/

## Key API Endpoints
- GET /api/luoyi/years — available years from submitted Pharsa periods
- GET /api/luoyi/budget-data/{year} — monthly actuals by category
- GET /api/luoyi/budgets/{year} — saved budget targets
- PUT /api/luoyi/budgets/{year} — update budget targets

## Important Notes
- Luoyi shares finance.db with Pharsa
- Budget actuals are derived from Pharsa's submitted periods
- Only submitted Pharsa periods appear in Luoyi's year list
- Budget categories: Ordinary Income, Extraordinary Income,
  Ordinary Expenses, Extraordinary Expenses, Household Expenses
```

---

## Bruno — NBA Analytics

```markdown
---
name: bruno
description: NBA sports analytics agent. Tracks player stats, team performance, game predictions, and efficiency metrics using XGBoost and RAPTOR models.
---

# Bruno — NBA Analytics

## When to use this skill
- User asks about NBA stats or scores
- User wants player or team analysis
- User wants game predictions or prop bets
- User wants historical NBA data

## Key Locations
- Project: /YOUR_HOME/nba-analytics/
- Databases:
  - /YOUR_HOME/nba-analytics/nba_historical.db
  - /YOUR_HOME/nba-analytics/nba_analytics.db
- Synology backup: /YOUR_HOME/synology/openclaw/bruno/

## Database Tables (nba_historical.db)
game_predictions, game_results, pbp_actions, player_clutch_metrics,
player_game_logs, player_predictions, raptor_cache,
team_advanced_stats, team_game_logs

## Start Bruno's API Server
cd /YOUR_HOME/nba-analytics
source venv/bin/activate
uvicorn backend.api.main:app --reload --port YOUR_BRUNO_PORT

## Health Check
curl http://localhost:YOUR_BRUNO_PORT/api/health

## Key API Endpoints (separate port from financial dashboard)
- GET /api/health — health check
- GET /api/slate?date=YYYY-MM-DD&window=season — full game slate
- GET /api/game/{game_id}/efficiency — player efficiency tab
- POST /api/predict — XGBoost + RAPTOR predictions
- POST /api/predict/player — filtered player props
- POST /api/nlp-query — natural language query interface

## Important Notes
- Runs on a SEPARATE port from the financial dashboard
- Standalone project — NOT integrated into financial dashboard backend
- XGBoost models need training via xgboost_model.train_models()
- ODDS_API_KEY optional — returns mock odds data if not set
- team_advanced_stats uses INSERT OR REPLACE (not IGNORE) — do not revert
- Off-ball players (USG% < 15%) have ratings suppressed — this is intentional
- H2H window has small sample size (2-4 games) — always show sample warning
```
