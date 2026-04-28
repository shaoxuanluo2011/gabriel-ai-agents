# Luoyi — Monthly Budget Agent

## What Luoyi Does

Luoyi is a monthly budget tracking agent that:
- Reads submitted financial periods from Pharsa
- Compares actual spending vs budget targets
- Visualises budget vs actual by category
- Allows setting and updating budget targets
- Tracks trends across months

---

## Architecture

```
Pharsa submits a period
        ↓
   finance.db (journal entries)
        ↓
   Luoyi reads submitted periods
        ↓
   Calculates actuals by category
        ↓
   Compares against budget targets
        ↓
   Dashboard visualises budget vs actual
```

---

## Key Concepts

### Budget Categories

Luoyi uses broad categories inherited from Pharsa:
- **Ordinary Income** — regular salary, freelance income
- **Extraordinary Income** — bonuses, windfalls, asset sales
- **Ordinary Expenses** — rent, food, transport, utilities
- **Extraordinary Expenses** — travel, medical, one-off purchases
- **Household Expenses** — insurance, maintenance

Each broad category has sub-categories for detailed tracking.

### Dependency on Pharsa
Luoyi only shows data for months where Pharsa has a **submitted** period. If a period is still in "preprocessor" or "expense categorisation" stage, it won't appear in Luoyi.

This ensures Luoyi only shows clean, validated data.

---

## Database

Luoyi shares `finance.db` with Pharsa. Key data sources:
- `financial_periods` — to find submitted periods
- `journal_entries` — to calculate actuals
- `period_balances` — for opening/closing balances

Budget targets are stored separately:
```sql
-- Budget targets (set by user)
budgets table:
  year TEXT
  broad_category TEXT
  sub_category TEXT
  monthly_target REAL
```

---

## API Endpoints

```
GET /api/luoyi/years                  # Available years (submitted periods)
GET /api/luoyi/budget-data/{year}     # Monthly actuals by category
GET /api/luoyi/budgets/{year}         # Saved budget targets
PUT /api/luoyi/budgets/{year}         # Update budget targets
```

---

## Dashboard Visualisation

The Luoyi page shows:
1. **Year selector** — choose which year to view
2. **Budget vs Actual chart** — composed chart showing budget line and actual bars
3. **Category breakdown** — expandable sections per broad category
4. **Monthly columns** — Jan through Dec with actuals

**Chart colours:**
- Ordinary Income → Emerald
- Extraordinary Income → Teal
- Ordinary Expenses → Red
- Extraordinary Expenses → Orange
- Household Expenses → Yellow

---

## OpenClaw Skill File

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
- Backend: /path/to/financial-dashboard/backend/luoyi_api.py
- Database: /path/to/financial-dashboard/db/finance.db
- API base: http://localhost:8000/api/luoyi/

## Key API Endpoints
- GET /api/luoyi/years
- GET /api/luoyi/budget-data/{year}
- GET /api/luoyi/budgets/{year}
- PUT /api/luoyi/budgets/{year}

## Important Notes
- Luoyi shares finance.db with Pharsa
- Budget actuals are derived from Pharsa's submitted periods
- Only submitted Pharsa periods appear in Luoyi's year list
```

---

## Prompt for Claude Code / OpenClaw

```
Build a monthly budget tracking agent called Luoyi with these requirements:

1. DATA SOURCE
   - Read from SQLite: finance.db (shared with Pharsa)
   - Only use data from SUBMITTED financial periods
   - Calculate actuals from journal entries

2. BUDGET CATEGORIES
   Broad categories with sub-categories:
   - Ordinary Income (Salary, Freelance, Rental Income)
   - Extraordinary Income (Bonus, Capital Gains, Gifts)
   - Ordinary Expenses (Rent, Food, Transport, Utilities, Phone)
   - Extraordinary Expenses (Travel, Medical, Electronics)
   - Household Expenses (Insurance, Maintenance, Subscriptions)

3. BUDGET TARGETS
   - Allow user to set monthly targets per sub-category
   - Store targets in budgets table
   - Targets apply to all months in that year (can override per month)

4. VISUALISATION DATA
   Return data structured for a composed chart:
   - X axis: months (Jan-Dec)
   - Budget line: target amount
   - Actual bars: real spending
   - Variance: actual - budget

5. API ENDPOINTS (FastAPI)
   Under /api/luoyi/ prefix:
   - GET /years — years with submitted data
   - GET /budget-data/{year} — monthly actuals
   - GET /budgets/{year} — saved targets
   - PUT /budgets/{year} — update targets

6. GUEST MODE
   All amounts fakified for guest users
   Same fakification formula as Pharsa

7. FRONTEND COMPONENT
   React composed chart using Recharts:
   - Line for budget target
   - Area/Bar for actuals
   - Color coding by category type
   - Expandable category breakdown

8. DATABASE
   Share finance.db with Pharsa.
   Add budgets table for target storage.
```

---

## Thought Process and Decisions

### Decision: Share database with Pharsa
**Why:** Luoyi is inherently downstream of Pharsa. Sharing the database avoids:
- Data synchronisation complexity
- Duplicate data storage
- API calls between services

**Trade-off:** Luoyi depends on Pharsa's database schema. Schema changes in Pharsa affect Luoyi.

### Decision: Only submitted periods
**Why:** Showing unfinished periods would show incomplete data that could mislead budget analysis. Requiring submission ensures:
- All transactions are categorised
- Journal entries are correct
- Reconciliation is done

### Decision: Recharts ComposedChart
**Why:** Budget vs actual visualisation needs both a line (budget target) and bars (actuals) on the same chart. ComposedChart allows mixing chart types in Recharts.

### Issue: Budget targets applying to wrong year
**Problem:** When user sets budgets for 2026, they shouldn't carry over to 2025 data.
**Solution:** Budget targets are stored per year. Each year has independent targets.

### Issue: Categories not matching between Pharsa and Luoyi
**Problem:** Pharsa uses detailed sub-categories. Luoyi needs to aggregate them into broad categories for meaningful budget tracking.
**Solution:** Defined a mapping between Pharsa's detailed categories and Luoyi's broad categories. The `BROAD_ORDER` constant in the frontend defines the display order.
