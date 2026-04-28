# Pharsa — Personal Finance Agent

## What Pharsa Does

Pharsa is a personal finance agent that:
- Accepts bank statement uploads (PDF/CSV)
- Extracts and preprocesses transactions
- Categorises expenses using AI assistance
- Maintains double-entry bookkeeping journal entries
- Generates balance sheets and income statements
- Performs bank reconciliation

---

## Architecture

```
Bank Statement (PDF/CSV upload)
        ↓
   Upload endpoint stores raw file
        ↓
   Preprocessor extracts transactions
        ↓
   AI-assisted expense categorisation
        ↓
   Journal entries (double-entry bookkeeping)
        ↓
   Balance sheet / Income statement / Trial balance
        ↓
   Reconciliation report
```

---

## Key Concepts

### Financial Periods
Each month is a "period". Periods go through stages:
1. **Opening** — period created, awaiting data
2. **Preprocessor** — transactions extracted and staged
3. **Expense categorisation** — transactions categorised
4. **Submitted** — period finalised and locked

### Double-Entry Bookkeeping
Every transaction has two sides:
- **DR (Debit)** — money coming in or asset increasing
- **CR (Credit)** — money going out or liability increasing

**Important rule:** Always call `POST /confirm-categories` after any DR/CR change. This recalculates journal entries.

### Guest Mode
When logged in as a guest, all financial amounts are "fakified" — replaced with plausible but fake numbers. This allows demos without exposing real financial data.

```javascript
// Fakification formula
function fakify(real) {
  const magnitude = Math.abs(real)
  const seed = ((magnitude * 6364136223846793) % 999983 + 999983) % 999983
  const factor = 0.55 + (seed % 1000) / 2222
  const fake = Math.round(magnitude * factor * 100) / 100
  return real < 0 ? -fake : fake
}
```

---

## Database Schema

Key tables in `finance.db`:

```sql
financial_periods    -- Monthly periods with status
transactions         -- All transactions
journal_entries      -- Double-entry bookkeeping
period_balances      -- Opening/closing balances
txn_dbss             -- DBS savings transactions
txn_ocbc             -- OCBC transactions
txn_citi             -- Citibank transactions
uploaded_statements  -- Raw uploaded files
```

---

## API Endpoints

```
GET  /api/pharsa/periods
GET  /api/pharsa/periods/{id}
GET  /api/pharsa/periods/{id}/balance-sheet
GET  /api/pharsa/periods/{id}/income-statement
GET  /api/pharsa/periods/{id}/trial-balance
GET  /api/pharsa/periods/{id}/reconciliation
GET  /api/pharsa/periods/{id}/transactions
POST /api/pharsa/periods/{id}/upload
POST /api/pharsa/periods/{id}/run-preprocessor
POST /api/pharsa/periods/{id}/confirm-categories
POST /api/pharsa/periods/{id}/submit
PATCH /api/pharsa/transactions/{txn_id}/category
```

---

## OpenClaw Skill File

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
- Backend: /path/to/financial-dashboard/backend/pharsa_api.py
- Database: /path/to/financial-dashboard/db/finance.db
- API base: http://localhost:8000/api/pharsa/

## Important Rules
- Always call POST confirm-categories after any DR/CR change
- Income statement, trial balance, balance sheet must source from journal entries
- Never run destructive commands without asking
- trash > rm for file deletions
```

---

## Prompt for Claude Code / OpenClaw

```
Build a personal finance agent called Pharsa with these requirements:

1. BANK STATEMENT PROCESSING
   - Accept PDF and CSV uploads from multiple banks
   - Extract transactions: date, description, amount, balance
   - Support multiple banks (DBS, OCBC, Citibank etc.)
   - Store raw files and extracted transactions separately

2. EXPENSE CATEGORISATION
   - Automatically suggest categories based on transaction description
   - Categories: Income, Housing, Food, Transport, Entertainment, etc.
   - Allow manual override of categories
   - DR/CR classification for double-entry bookkeeping

3. DOUBLE-ENTRY BOOKKEEPING
   - Every transaction generates journal entries
   - Maintain chart of accounts
   - Recalculate journal entries when categories change
   - Validate that debits = credits

4. FINANCIAL STATEMENTS
   Generate from journal entries:
   - Balance sheet (assets, liabilities, equity)
   - Income statement (income vs expenses)
   - Trial balance (all accounts)

5. RECONCILIATION
   - Compare bank statement balance vs book balance
   - Identify unmatched transactions
   - Generate reconciliation report

6. PERIOD MANAGEMENT
   - Monthly periods with status workflow:
     opening → preprocessor → categorised → submitted
   - Only submitted periods flow to Luoyi budget tracking

7. GUEST MODE
   - Fakify all financial amounts for guest users
   - Same data structure, different values
   - Consistent fakification (same real value → same fake value)

8. API ENDPOINTS (FastAPI)
   All endpoints under /api/pharsa/ prefix:
   - CRUD for periods
   - Balance sheet, income statement, trial balance
   - Transaction management
   - Statement upload and preprocessing

9. DATABASE
   SQLite with tables for:
   - Financial periods
   - Transactions per bank
   - Journal entries
   - Period balances
   - Uploaded statements
```

---

## Thought Process and Decisions

### Decision: Double-entry bookkeeping
**Why:** Single-entry (just recording transactions) doesn't catch errors. Double-entry ensures the books always balance and makes it easy to generate proper financial statements.

**Trade-off:** More complex to implement. Required learning accounting concepts (debits, credits, chart of accounts).

### Decision: Period-based workflow
**Why:** Monthly periods with status progression ensures data integrity. You can't accidentally edit a submitted period. Data flows to Luoyi only when a period is properly submitted.

### Decision: Per-bank transaction tables
**Why:** Each bank has different CSV formats. Separate tables allow bank-specific parsing logic without complex polymorphism.

### Issue: PDF parsing inconsistency
**Problem:** Different banks format PDFs differently. Column positions, date formats, and amount formats vary widely.
**Solution:** Used `pdfplumber` which extracts text with position information. Wrote bank-specific parsers for each institution. Added validation to catch parsing errors before storing.

### Issue: Journal entry recalculation
**Problem:** When a user changes a transaction's category, all downstream journal entries need recalculation.
**Solution:** `POST /confirm-categories` endpoint recalculates everything. Made this a required step — the UI blocks progression until confirmed.

### Decision: Guest mode fakification
**Why:** Real financial data is sensitive. Rather than showing zeros or hiding data, fakified amounts show realistic-looking numbers that demonstrate the dashboard's capabilities without exposing real data.

The fakification formula is deterministic — the same real amount always produces the same fake amount within a session, so ratios and trends look consistent.
