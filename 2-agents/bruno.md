# Bruno — NBA Analytics Agent

## What Bruno Does

Bruno is a standalone NBA analytics agent that:
- Pulls daily game data from the official NBA API
- Calculates advanced player and team metrics
- Runs XGBoost models to predict game outcomes
- Computes RAPTOR player ratings
- Sends WhatsApp alerts for key insights
- Provides a natural language query interface

Bruno runs independently from the financial dashboard — it has its own FastAPI server on port 8001.

---

## Architecture

```
NBA Stats API (stats.nba.com)
        ↓
   historical_scraper.py
        ↓
   nba_historical.db
   ├── game_results
   ├── team_game_logs
   ├── player_game_logs
   ├── team_advanced_stats
   ├── raptor_cache
   ├── game_predictions
   ├── player_predictions
   ├── player_clutch_metrics
   └── pbp_actions
        ↓
   ┌────────────────────────┐
   │   Models                │
   │  XGBoost (predictions) │
   │  RAPTOR (ratings)      │
   │  Player Advanced       │
   └────────────────────────┘
        ↓
   FastAPI (port 8001)
        ↓
   Dashboard + WhatsApp alerts
```

---

## Data Sources

All data comes from the official NBA API via the `nba_api` Python library:

```python
from nba_api.stats.endpoints import LeagueDashTeamStats
from nba_api.stats.endpoints import PlayerGameLog

# Example: fetch team advanced stats
result = LeagueDashTeamStats(
    season="2025-26",
    measure_type_detailed_defense="Advanced",
).get_data_frames()[0]
```

**Why official NBA API?**
- Free, no API key required
- Comprehensive stats going back decades
- Same data used by NBA.com
- Reliable and well-documented

---

## Database Tables

### `team_advanced_stats`
Season-level advanced stats for all 30 teams:
- `ortg` — Offensive Rating (points per 100 possessions)
- `drtg` — Defensive Rating (opponent points per 100 possessions)
- `net_rtg` — Net Rating (ortg - drtg)
- `pace` — Possessions per 48 minutes
- `efg_pct` — Effective Field Goal %
- `tov_pct` — Turnover %

**Important note:** `team_abbr` was initially missing for 2024-25 season due to `INSERT OR IGNORE` skipping updates. Fixed by:
1. Running SQL update from `team_game_logs`
2. Changing to `INSERT OR REPLACE` in scraper

### `player_game_logs`
Per-game stats for every player:
- Standard: pts, reb, ast, stl, blk, tov, min
- Shooting: fgm, fga, fg3m, fg3a, ftm, fta
- Advanced: plus_minus, oreb, dreb

### `raptor_cache`
Pre-computed RAPTOR ratings per player:
- `raptor_off` — offensive contribution
- `raptor_def` — defensive contribution
- `raptor_total` — net rating
- Supports windows: season, L10, L20

---

## Advanced Metrics (player_advanced.py)

12 metrics calculated from raw game logs:

**Offensive:**
| Metric | Formula | Meaning |
|---|---|---|
| TS% | pts / (2 × (fga + 0.44 × fta)) × 100 | True shooting efficiency |
| eFG% | (fgm + 0.5 × fg3m) / fga | Shooting with 3pt credit |
| TOV% | tov / (fga + 0.44 × fta + ast + tov) × 100 | Turnover rate |
| USG% | (fga + fta/2 + tov) × 48 / (min × team_totals) × 100 | Usage rate |
| AST% | ast / team_fgm × time_share × 100 | Assist rate |
| REB% | reb × 48 / (min × total_reb) × 100 | Rebound rate |
| OFF_RATING | pts / total_possessions × 100 | Points per 100 possessions |

**Defensive:**
| Metric | Formula | Meaning |
|---|---|---|
| FORCED_MISSES | stl + 0.5 × blk | Defensive disruptions |
| PLAYER_STOPS | stl + 0.5×blk + 0.7×dreb + forced_misses | Total stops |
| POINTS_PREVENTED | stops × opp_pts_per_poss | Defensive value |
| DEF_RATING | opp_pts_allowed / opp_poss × 100 | Defensive efficiency |
| NET_RATING | OFF_RATING - DEF_RATING | Overall impact |

**Off-ball player detection:**
Players with USG% < 15% have ratings suppressed. Possession-based formulas produce misleading values for spot-up shooters and screeners who don't initiate plays.

---

## Scheduler

Bruno runs scheduled jobs automatically:

| Job | Frequency | What it does |
|---|---|---|
| Injuries & Odds | Every 15 min | Updates injury reports and betting odds |
| Efficiency Stats | Every 60 min | Refreshes player/team efficiency |
| Box Score Ingest | Daily 10AM | Downloads previous day's box scores |
| Game Predictions | Daily 11AM | Runs XGBoost predictions |
| Watchdog | Daily 11:30AM | Verifies pipeline completed |
| Model Retraining | Daily 3AM | Retrains XGBoost models |
| Annual Retrain | July 1st 4AM | Full off-season retraining |

---

## API Endpoints

All endpoints on port 8001:

```
GET  /api/health                              # Health check
GET  /api/slate?date=YYYY-MM-DD              # Full game slate
GET  /api/game/{id}/efficiency               # Player efficiency tab
POST /api/predict                            # XGBoost + RAPTOR predictions
POST /api/nlp-query                          # Natural language query
GET  /api/accuracy                           # Model accuracy stats
```

---

## Query Builder

Bruno has a template-based query system for WhatsApp:

```
"Which home team is favored to win tonight?"
"Which game has the highest projected total?"
"Show me players projected for 25+ points tonight"
"Compare Warriors vs Lakers efficiency this season"
```

---

## OpenClaw Skill File

```markdown
---
name: bruno
description: NBA sports analytics agent. Tracks player stats, team performance, game predictions, and betting market analysis using XGBoost and RAPTOR models.
---

# Bruno — NBA Analytics

## When to use this skill
- User asks about NBA stats or scores
- User wants player or team analysis
- User wants game predictions or prop bets

## Start Bruno's API Server
cd /path/to/nba-analytics
source venv/bin/activate
uvicorn backend.api.main:app --reload --port 8001

## Health Check
curl http://localhost:8001/api/health

## Important Notes
- Runs on port 8001 (not 8000)
- Standalone project — NOT in financial dashboard
- XGBoost models need training before predictions work
```

---

## Prompt for Claude Code / OpenClaw

```
Build an NBA analytics agent called Bruno with these requirements:

1. DATA PIPELINE
   - Source: official NBA API (nba_api Python library)
   - Fetch: game logs, box scores, team stats, play-by-play
   - Schedule: daily ingestion, hourly efficiency refresh
   - Storage: SQLite database

2. DATABASE TABLES
   - game_results — final scores
   - team_game_logs — team stats per game
   - player_game_logs — player stats per game
   - team_advanced_stats — season-level advanced stats
   - raptor_cache — pre-computed player ratings
   - game_predictions — model predictions
   - player_predictions — prop predictions
   - player_clutch_metrics — clutch performance
   - pbp_actions — play-by-play

3. ADVANCED METRICS
   Calculate for top 8 players per team:
   Offensive: TS%, eFG%, TOV%, USG%, AST%, REB%, OFF_RATING
   Defensive: FORCED_MISSES, PLAYER_STOPS, DEF_RATING, NET_RATING
   Net: NET_RATING = OFF_RATING - DEF_RATING
   
   Suppress ratings for off-ball players (USG% < 15%)

4. PREDICTION MODELS
   XGBoost model predicting:
   - Win probability (home vs away)
   - Projected total score
   - Projected spread
   
   RAPTOR model:
   - Player offensive/defensive ratings
   - Team strength from player ratings
   - Injury-adjusted predictions

5. WINDOWS
   Support multiple time windows:
   - season — full season averages
   - L10 — last 10 games
   - L20 — last 20 games
   - H2H — head-to-head history

6. WHATSAPP ALERTS
   Send alerts for:
   - Daily prediction summary at 11AM
   - Injury updates affecting predictions
   - Model accuracy reports

7. NATURAL LANGUAGE QUERIES
   POST /api/nlp-query accepts plain English:
   "Who is favored tonight?"
   "Show me efficient scorers on the Lakers"

8. API ENDPOINTS (FastAPI on port 8001)
   - GET /api/slate — today's games with predictions
   - GET /api/game/{id}/efficiency — player efficiency tab
   - POST /api/predict — run predictions
   - POST /api/nlp-query — natural language interface
   - GET /api/accuracy — model performance stats

9. SCHEDULER
   - 15min: injuries and odds
   - 60min: efficiency stats
   - 10AM daily: box score ingest
   - 11AM daily: predictions
   - 3AM daily: model retraining
```

---

## Thought Process and Decisions

### Decision: Standalone project (port 8001)
**Why:** Bruno's data pipeline, models, and scheduling are complex enough to warrant isolation. Mixing NBA analytics with financial data in the same codebase would create confusion.

**Trade-off:** Two separate servers to manage. However, they can run independently — financial dashboard works without Bruno.

### Decision: XGBoost over neural networks
**Why:** XGBoost is:
- Interpretable — can explain predictions
- Fast to train and retrain nightly
- Works well with tabular sports data
- Doesn't need GPU

**Trade-off:** Less powerful than deep learning for complex patterns. But for structured sports data with clear features, XGBoost performs comparably.

### Decision: RAPTOR ratings from scratch
**Why:** FiveThirtyEight's original RAPTOR is proprietary. We implemented an approximation using publicly available box score data. While not identical, it captures the same concept — estimating player value from stats.

### Issue: team_abbr missing for 2024-25
**Problem:** The `team_advanced_stats` table had empty `team_abbr` for 2024-25 season. API endpoints that looked up teams by abbreviation returned empty results.

**Root cause:** Data was initially inserted with `INSERT OR IGNORE`. When we later added `team_abbr` to the schema and re-ran the scraper, existing rows were silently skipped.

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

Also changed `INSERT OR IGNORE` to `INSERT OR REPLACE` at line 403 of `historical_scraper.py` to prevent future occurrences.

### Issue: Off-ball player ratings inflated
**Problem:** Players like spot-up shooters had wildly high OFF_RATING (200+) because they score points but rarely "use possessions" in the traditional sense.

**Solution:** Detect off-ball players using USG% < 15% threshold. Suppress OFF_RATING, DEF_RATING, NET_RATING for these players and flag them with `off_ball: true`.

### Decision: Cache heatmap for 15 minutes
**Why:** Efficiency tab data requires joining multiple tables and computing averages. This is expensive to run for every request. 15 minutes is short enough to feel fresh but long enough to reduce database load.

### Issue: H2H window small sample size
**Problem:** Two teams only play each other 2-4 times per season. H2H stats based on 2 games are statistically unreliable.

**Solution:** Include `sample_size` in API response. Frontend displays "Small sample (N games)" warning when sample_size < 5.
