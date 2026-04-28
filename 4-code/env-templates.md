# Environment Variable Templates

Copy the relevant `.env.example` file to `.env` and fill in your values.
Never commit `.env` files to GitHub — only commit `.env.example` files.

---

## news-scraper/.env.example

```bash
# News Scraper Environment Variables
# Copy this file to .env and fill in your values

# OpenAI (used for article summarisation)
OPENAI_API_KEY=your_openai_key_here

# The Guardian API (free tier available at open-platform.theguardian.com)
GUARDIAN_API_KEY=your_guardian_api_key_here

# Benzinga (paid — for financial wire articles)
BENZINGA_API_KEY=your_benzinga_key_here

# Alpaca (optional — for watchlist-filtered news)
ALPACA_API_KEY=your_alpaca_key_here
ALPACA_SECRET_KEY=your_alpaca_secret_here

# Gmail (use App Password, not your real Gmail password)
# Create at: myaccount.google.com/apppasswords
GMAIL_ADDRESS=your_email@gmail.com
GMAIL_APP_PASSWORD=your_16_char_app_password
```

---

## financial-dashboard/backend/.env.example

```bash
# Financial Dashboard Backend Environment Variables
# Copy this file to .env and fill in your values

# JWT signing key — generate a long random string
# Example: python3 -c "import secrets; print(secrets.token_hex(32))"
SECRET_KEY=generate_a_long_random_string_here

# JWT settings (optional — these are the defaults)
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=1440

# Anthropic API key (for AI-assisted expense categorisation in Pharsa)
ANTHROPIC_API_KEY=your_anthropic_key_here
```

---

## nba-analytics/.env.example

```bash
# NBA Analytics Environment Variables
# Copy this file to .env and fill in your values

# Odds API (optional — mock odds returned if not set)
ODDS_API_KEY=your_odds_api_key_here

# WhatsApp notification target
WHATSAPP_NOTIFY_TARGET=+YOUR_COUNTRY_CODE_AND_NUMBER

# Anthropic API key (for NLP query interface)
ANTHROPIC_API_KEY=your_anthropic_key_here
```

---

## How to Generate a Secure SECRET_KEY

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

This generates a cryptographically secure random 64-character string.
Use this as your SECRET_KEY for JWT token signing.

---

## .gitignore Template

Add this to every project's `.gitignore`:

```
# Environment files — NEVER commit these
.env
*.env
.env.*
!.env.example

# Python
__pycache__/
*.pyc
*.pyo
venv/
.venv/

# Database files — contain personal data
*.db
*.db-wal
*.db-shm

# Node
node_modules/
dist/

# Logs
logs/
*.log

# macOS
.DS_Store
```
