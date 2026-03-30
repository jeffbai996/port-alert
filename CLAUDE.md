# port-alert

Portfolio sentinel — threshold-based monitoring and alerting for IBKR positions. Runs on cron, zero AI tokens, pure Python logic.

## What This Is

A lightweight daemon that periodically checks portfolio health via the IBKR MCP server and fires alerts when thresholds are breached. Designed to run on fragserv (WSL2, Ryzen 9 5900X) via cron during market hours.

**Key constraint**: No LLM calls. Every check is deterministic threshold logic. This must be cheap to run (CPU + one MCP round-trip per cycle) and reliable enough to trust with real money alerts.

## Architecture

```
cron (every 5 min, market hours)
  └─ sentinel.py (main entry point)
       ├─ checks/         ← individual check modules
       │   ├─ margin.py        — excess liquidity, margin utilization, cushion
       │   ├─ positions.py     — position P&L %, concentration drift
       │   ├─ prices.py        — price level alerts (support/resistance)
       │   ├─ earnings.py      — earnings proximity warning
       │   └─ connection.py    — gateway health / connectivity
       ├─ alerts/         ← alert delivery backends
       │   ├─ email.py         — Gmail via SMTP (python smtplib, not MCP)
       │   ├─ webhook.py       — generic webhook (Slack, Discord, ntfy.sh)
       │   └─ file.py          — append to local log (always-on fallback)
       ├─ config.py       ← thresholds + alert routing from .env / YAML
       ├─ ibkr.py         ← thin MCP client (reuse pattern from ticker-tape/ibkr_client.py)
       └─ state.py        ← last-alert dedup + cooldown tracking (SQLite)
```

## Data Flow

1. `sentinel.py` loads config, inits MCP client
2. Calls `ibkr_get_account_summary`, `ibkr_get_positions`, `ibkr_get_pnl_by_position` via MCP
3. Passes raw data through each check module
4. Each check returns `list[Alert]` (dataclass: severity, check_name, message, data)
5. Alerts are deduped against `state.db` (don't re-fire same alert within cooldown window)
6. Surviving alerts dispatched through configured backends

## MCP Client

Reuse the streamable HTTP MCP client pattern from `ticker-tape/ibkr_client.py`:
- Connect to IBKR MCP server at `IBKR_MCP_URL` (env var, default `http://localhost:8000/mcp`)
- Supports multi-account via `IBKR_MCP_URL_2`
- Each sentinel run opens one session, makes all tool calls, closes. No persistent connection needed.

The ibkr-terminal MCP server exposes these tools we'll use:
- `ibkr_get_account_summary` — NLV, excess liquidity, margin used, cushion
- `ibkr_get_positions` — all positions with market value, unrealized P&L, weight
- `ibkr_get_pnl_by_position` — per-position daily/unrealized P&L
- `ibkr_margin` — detailed margin breakdown (detail="headroom" for specific symbols)
- `ibkr_cushion_alert` — pre-built cushion monitoring (but we'll build our own for flexibility)
- `ibkr_connection_status` — gateway health check
- `ibkr_trades` — today's fills (view="fills")

## Check Modules — Detail

### margin.py
| Check | Threshold | Severity |
|---|---|---|
| Excess liquidity < $X | configurable, default $50K | CRITICAL |
| Excess liquidity < $Y | configurable, default $100K | WARNING |
| Margin utilization > Z% | configurable, default 70% | WARNING |
| Margin utilization > Z% | configurable, default 85% | CRITICAL |
| Cushion < N | configurable, default 0.10 | CRITICAL |
| Maintenance margin spike > X% from last check | configurable, default 15% | WARNING |

### positions.py
| Check | Threshold | Severity |
|---|---|---|
| Single position > X% of NLV | configurable, default 30% | WARNING |
| Single position unrealized loss > $X | configurable, default -$20K | WARNING |
| Single position daily loss > Y% | configurable, default -8% | WARNING |
| Total portfolio daily loss > Z% | configurable, default -5% | CRITICAL |

### prices.py
User-defined price alerts in config file:
```yaml
price_alerts:
  NVDA:
    above: [160, 180]
    below: [120, 100]
  AVGO:
    above: [250, 280]
    below: [180, 160]
```
Triggered once per level crossing, cooldown prevents re-fire.

### earnings.py
- Warn when a held position has earnings within N trading days (default 5)
- Data source: yfinance `.get_earnings_dates()` or calendar, cached daily
- Severity: INFO at 5 days, WARNING at 2 days

### connection.py
- Gateway not responding → CRITICAL
- Gateway responding but no positions returned → WARNING (possible disconnect)
- Run this check FIRST — skip other checks if gateway is down

## Alert Dedup & Cooldown (state.py)

SQLite `state.db` with table:
```sql
CREATE TABLE alert_history (
    id INTEGER PRIMARY KEY,
    check_name TEXT NOT NULL,
    alert_key TEXT NOT NULL,      -- unique identifier (e.g., "margin_util_warning")
    severity TEXT NOT NULL,
    message TEXT,
    fired_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(check_name, alert_key)  -- for cooldown lookups
);
```

Cooldown rules:
- CRITICAL: re-fire every 15 minutes (things are bad, keep nagging)
- WARNING: re-fire every 2 hours
- INFO: re-fire every 24 hours
- Cooldown resets if the condition clears and re-triggers

## Config (config.py)

Load from `config.yaml` with `.env` overrides for secrets:
```yaml
# config.yaml
ibkr:
  mcp_url: "${IBKR_MCP_URL}"       # from .env
  mcp_url_2: "${IBKR_MCP_URL_2}"   # optional second account

thresholds:
  margin:
    excess_liquidity_warn: 100000
    excess_liquidity_crit: 50000
    utilization_warn: 0.70
    utilization_crit: 0.85
    cushion_crit: 0.10
  positions:
    max_concentration: 0.30
    max_unrealized_loss: -20000
    max_daily_loss_pct: -0.08
    max_portfolio_daily_loss_pct: -0.05
  earnings:
    warn_days: 5
    alert_days: 2

price_alerts:
  # populated per user

alerts:
  backends:
    - type: file
      path: logs/alerts.log
    - type: webhook
      url: "${ALERT_WEBHOOK_URL}"
    - type: email
      smtp_host: smtp.gmail.com
      smtp_port: 587
      smtp_user: "${GMAIL_USER}"
      smtp_pass: "${GMAIL_APP_PASSWORD}"
      to: "${ALERT_EMAIL_TO}"

  routing:
    CRITICAL: [file, webhook, email]
    WARNING: [file, webhook]
    INFO: [file]
```

## Cron Setup

```bash
# Market hours only (ET): 9:25 AM - 4:05 PM, Mon-Fri
# Converted to server local time as needed
*/5 9-16 * * 1-5 cd /home/jbai/repos/port-alert && venv/bin/python sentinel.py >> logs/cron.log 2>&1

# Pre-market check at 9:00 AM ET
0 9 * * 1-5 cd /home/jbai/repos/port-alert && venv/bin/python sentinel.py >> logs/cron.log 2>&1

# Gateway health check every 30 min outside market hours (catch overnight disconnects)
*/30 0-8,17-23 * * 1-5 cd /home/jbai/repos/port-alert && venv/bin/python sentinel.py --check connection >> logs/cron.log 2>&1
```

## Dependencies

```
httpx>=0.27
mcp>=1.0.0
pyyaml>=6.0
python-dotenv>=1.0.0
yfinance>=0.2.30   # earnings dates only
```

No Flask, no web server, no frontend. This is a headless sentinel.

## Implementation Order

1. **ibkr.py** — MCP client (port from ticker-tape/ibkr_client.py, adapt for sync-only usage)
2. **config.py** — YAML + env loading
3. **state.py** — SQLite alert history + cooldown logic
4. **checks/connection.py** — gateway health (simplest check, validates MCP client works)
5. **checks/margin.py** — margin thresholds (highest value check)
6. **checks/positions.py** — position-level alerts
7. **alerts/file.py** — file logger (always-on, test other checks against this)
8. **alerts/webhook.py** — webhook delivery
9. **alerts/email.py** — Gmail SMTP
10. **sentinel.py** — orchestrator, CLI args, cron entry point
11. **checks/prices.py** — price level alerts
12. **checks/earnings.py** — earnings proximity
13. Cron setup + testing

## Testing Strategy

- TDD per CLAUDE.md global rules
- Mock the MCP client responses (known account data → expected alerts)
- Test cooldown/dedup logic with time mocking
- Integration test: run against live IBKR MCP server, verify no crashes
- `sentinel.py --dry-run` flag that runs all checks but doesn't fire alerts (logs to stdout)

## env.example

```bash
IBKR_MCP_URL=http://localhost:8000/mcp
# IBKR_MCP_URL_2=http://localhost:8001/mcp
ALERT_WEBHOOK_URL=https://ntfy.sh/your-topic
GMAIL_USER=your@gmail.com
GMAIL_APP_PASSWORD=xxxx-xxxx-xxxx-xxxx
ALERT_EMAIL_TO=your@gmail.com
```
