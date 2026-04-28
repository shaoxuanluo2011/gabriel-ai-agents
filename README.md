# Gabriel's AI Agents

A personal AI agent ecosystem built on [OpenClaw](https://openclaw.ai), running on a Mac Mini, integrating with WhatsApp, a financial dashboard, and NBA analytics.

## What This Is

This repository documents how I built a suite of 5 AI agents that run 24/7 on my Mac Mini, accessible via WhatsApp and a web dashboard hosted on Hetzner cloud.

Each agent has a specific job:

| Agent | Role |
|---|---|
| **Miya** | Scrapes financial news daily, runs FinBERT sentiment analysis |
| **Layla** | Generates daily financial newsletters from Miya's data |
| **Pharsa** | Manages personal finance — balance sheets, income statements, reconciliation |
| **Luoyi** | Tracks monthly budget vs actual spending |
| **Bruno** | NBA analytics — game predictions, player stats, efficiency metrics |

## Architecture Overview

```
                    ┌─────────────────────┐
                    │     Mac Mini M4     │
                    │  (Always On, SGT)   │
                    │                     │
                    │  ┌───────────────┐  │
                    │  │   OpenClaw    │  │
                    │  │   Gateway     │  │
                    │  └──────┬────────┘  │
                    │         │           │
                    │  ┌──────┴────────┐  │
                    │  │  5 AI Agents  │  │
                    │  │ Miya  Layla   │  │
                    │  │ Pharsa Luoyi  │  │
                    │  │    Bruno      │  │
                    │  └──────┬────────┘  │
                    │         │           │
                    │  ┌──────┴────────┐  │
                    │  │   Databases   │  │
                    │  │  news.db      │  │
                    │  │  finance.db   │  │
                    │  │  nba.db       │  │
                    │  └───────────────┘  │
                    └─────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
     ┌────────▼──────┐ ┌──────▼──────┐ ┌─────▼──────┐
     │   WhatsApp    │ │  Hetzner    │ │  Synology  │
     │  (Alerts &    │ │   Cloud     │ │    NAS     │
     │   Queries)    │ │  Dashboard  │ │  (Nightly  │
     └───────────────┘ │ sxluo-      │ │   Backup)  │
                       │ dashboard   │ └────────────┘
                       │ .com        │
                       └────────────┘
```

## Tech Stack

| Layer | Technology |
|---|---|
| AI Agent Platform | OpenClaw |
| AI Model | Claude Sonnet 4 (main sessions), Claude Haiku (cron jobs) |
| Backend | Python FastAPI |
| Frontend | React + Vite + Tailwind CSS |
| Database | SQLite |
| Web Server | Nginx |
| Cloud Hosting | Hetzner CX22 (€3.79/month) |
| DNS & Security | Cloudflare (free) |
| Notifications | WhatsApp via OpenClaw |
| Backup | Synology NAS |
| Hardware | Mac Mini M4 (always on) |

## Repository Structure

```
gabriel-ai-agents/
├── README.md
├── 1-architecture/
│   ├── overview.md           # Detailed architecture guide
│   ├── openclaw-setup.md     # OpenClaw installation guide
│   └── cheatsheet.md         # Terminal commands cheatsheet
├── 2-agents/
│   ├── miya.md               # Miya agent guide + prompt
│   ├── layla.md              # Layla agent guide + prompt
│   ├── pharsa.md             # Pharsa agent guide + prompt
│   ├── luoyi.md              # Luoyi agent guide + prompt
│   └── bruno.md              # Bruno agent guide + prompt
├── 3-troubleshooting/
│   └── decisions.md          # Issues encountered + key decisions
├── 4-code/
│   ├── skills/               # OpenClaw skill files
│   └── workspace/            # Sanitized workspace docs
└── 5-deployment/
    ├── hetzner-setup.md      # Cloud deployment guide
    └── cloudflare-setup.md   # Security setup guide
```

## Quick Start

1. Read [Architecture Overview](1-architecture/overview.md)
2. Follow [OpenClaw Setup Guide](1-architecture/openclaw-setup.md)
3. Set up agents one by one starting with [Miya](2-agents/miya.md)
4. Deploy dashboard following [Hetzner Setup](5-deployment/hetzner-setup.md)

## Prerequisites

- Mac Mini (or any always-on Mac/Linux machine)
- WhatsApp account
- Anthropic API key (get one at console.anthropic.com)
- Basic familiarity with terminal commands

## Cost Summary

| Item | Monthly Cost |
|---|---|
| Hetzner CX22 server | €3.29 |
| Hetzner IPv4 | €0.50 |
| Domain name | ~$1.20 |
| Cloudflare | Free |
| OpenClaw | Check openclaw.ai |
| **Total** | **~€5/month** |

## Security Note

This repository never contains real API keys, passwords or sensitive data.
All secrets are stored in `.env` files which are excluded from git via `.gitignore`.
See `.env.example` files for the required environment variables.

## License

MIT — feel free to use, modify and share.
