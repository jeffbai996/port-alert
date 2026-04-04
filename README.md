# port-alert

A lightweight portfolio alert engine for IBKR accounts. Runs on cron during market hours, checks account health against configurable thresholds, and fires alerts via webhook, email, or file log.

**Zero LLM calls. Every check is deterministic threshold logic.**

## What it monitors

- **Margin health** — excess liquidity, utilization %, maintenance margin spikes
- **Position risk** — concentration, unrealized loss, daily drawdown per position
- **Portfolio drawdown** — total portfolio daily P&L
- **Price levels** — user-defined support/resistance crossings per ticker
- **Earnings proximity** — warns N days before held positions report
- **Gateway health** — IBKR connection status, checked first before anything else

## Architecture

```
cron (every 5 min, market hours)
  └─ engine.py
       ├─ checks/         individual check modules (margin, positions, prices, earnings, connection)
       ├─ alerts/         delivery backends (webhook, email, file)
       ├─ ibkr.py         MCP client — connects to ibkr-terminal MCP server
       ├─ config.py       threshold + routing config from YAML + .env
       └─ state.py        SQLite dedup — prevents alert spam, per-severity cooldowns
```

Data flow: `cron → engine.py → ibkr MCP → check modules → dedup → alert backends`

## Alert delivery

| Backend | Use case |
|---|---|
| `file` | Always-on fallback, append to `logs/alerts.log` |
| `webhook` | Slack, Discord, ntfy.sh, or any HTTP endpoint |
| `email` | Gmail via app password (SMTP) |

Routing is configurable per severity: CRITICAL can go to all three, INFO can be file-only.

## Dedup & cooldown

Alerts are deduplicated via SQLite. Re-fire intervals:
- **CRITICAL**: every 15 minutes (keeps nagging while condition persists)
- **WARNING**: every 2 hours
- **INFO**: every 24 hours

Cooldown resets if the condition clears and re-triggers.

## Setup

### Requirements

- Python 3.11+
- [ibkr-terminal-core](https://github.com/jeffbai996/ibkr-terminal-core) MCP server running locally
- IBKR TWS or IB Gateway running with API access enabled

### Install

```bash
git clone https://github.com/jeffbai996/port-alert
cd port-alert
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Configure

```bash
cp env.example .env
# edit .env with your MCP URL, webhook URL, email credentials
```

```bash
cp config.example.yaml config.yaml
# edit config.yaml — thresholds, price alerts, alert routing
```

Key `.env` vars:

```bash
IBKR_MCP_URL=http://localhost:8000/mcp
ALERT_WEBHOOK_URL=https://ntfy.sh/your-topic
GMAIL_USER=your@gmail.com
GMAIL_APP_PASSWORD=xxxx-xxxx-xxxx-xxxx
ALERT_EMAIL_TO=your@gmail.com
```

### Run manually

```bash
# full check cycle
python engine.py

# dry run — all checks, no alerts fired
python engine.py --dry-run

# single check only
python engine.py --check connection
python engine.py --check margin
```

### Cron (market hours ET)

```cron
# Every 5 min during market hours (Mon-Fri, 9:25 AM – 4:05 PM ET)
*/5 9-16 * * 1-5 cd /path/to/port-alert && venv/bin/python engine.py >> logs/cron.log 2>&1

# Pre-market at 9:00 AM
0 9 * * 1-5 cd /path/to/port-alert && venv/bin/python engine.py >> logs/cron.log 2>&1

# Gateway health every 30 min outside market hours
*/30 0-8,17-23 * * 1-5 cd /path/to/port-alert && venv/bin/python engine.py --check connection >> logs/cron.log 2>&1
```

Adjust times to your server's local timezone.

## Dependencies

```
httpx
mcp
pyyaml
python-dotenv
yfinance        # earnings dates only
```

No web server, no frontend, no message queue. Headless daemon designed to run on a home server or VPS.

## Related

- [ibkr-terminal](https://github.com/jeffbai996/ibkr-terminal) — terminal UI for IBKR accounts
- [ibkr-terminal-core](https://github.com/jeffbai996/ibkr-terminal-core) — MCP server that port-alert connects to

## License

MIT