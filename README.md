# Multi-Venue Automated Trading Fleet

Architecture overview of an automated trading system managing positions across 6 concurrent venue APIs with real-time monitoring, automated safety systems, and multi-model AI code review.

**This repo documents the engineering design. No trading logic, venue names, or sensitive code is included.**

## System Overview

```
135 scripts → refactored to 27
99.9% uptime over 20+ operational sessions
6 venue API integrations (REST + WebSocket)
10 custom MCP diagnostic tools
19 issues found via multi-model AI code review
```

## Architecture

### Orchestration Layer

PM2 manages 27 Python processes across three categories:

- **Trading bots** — Market making, grid strategies, funding rate capture. Each bot is a standalone async Python process with its own state file and error handling.
- **Verification suite** — Queries all venue APIs independently of bots to verify positions, balances, and hedge ratios match expected state. Runs on cron and on-demand.
- **Safety systems** — Automated killswitch triggers on position drift, unhedged exposure, or venue API failure. Position limits enforced at the process level, not just the venue level.

```
PM2 Ecosystem
├── Trading Bots (market making, grid, funding carry)
├── Fleet Verify (cross-venue position audit)
├── Parity Audit (state file vs on-chain comparison)
├── Watchdog (cron */30, Telegram alerts on failure)
└── Safety (killswitch, hedge guard, position limits)
```

### Venue Integration

6 venues connected simultaneously via async HTTP and WebSocket:

- **REST APIs** for order placement, position queries, balance checks
- **WebSocket feeds** for real-time price data and order fill notifications
- **Rate limiting** handled per-venue with exponential backoff
- **Empty response detection** — distinguishes API failure from genuine zero-position state (a lesson learned from a real incident where "no data" was treated as "no positions")

### MCP Diagnostic Server

A custom [Model Context Protocol](https://modelcontextprotocol.io/) server exposes 10 tools to Claude Code for live fleet diagnostics:

| Tool | Purpose |
|------|---------|
| `query_positions` | Current positions across all venues |
| `check_balances` | Wallet balances with USD equivalents |
| `hedge_status` | Cross-venue hedge ratio verification |
| `bot_health` | PM2 process status + last heartbeat |
| `cancel_orders` | Emergency order cancellation |
| `venue_status` | API connectivity and latency |
| `pnl_snapshot` | Realized + unrealized P&L |
| `risk_check` | Exposure limits and margin usage |
| `log_tail` | Recent log entries per bot |
| `fleet_summary` | Aggregate dashboard data |

This allows an AI assistant to investigate issues, verify deployments, and diagnose problems using the same tools a human operator would use — but faster.

### Multi-Model Code Review Pipeline

Every code change runs through 6 AI models in parallel, each reviewing from a specialist angle:

| Reviewer | Focus Area |
|----------|-----------|
| Security | OWASP Top 10, injection, auth, secrets |
| Architecture | Coupling, cohesion, abstraction quality |
| Performance | Complexity, allocations, N+1, caching |
| Edge Cases | Null handling, boundaries, off-by-one |
| Deep Reasoning | Logic correctness, invariant verification |
| Code Quality | DRY, idioms, readability, dead code |

Findings are deduplicated across models (fuzzy title/description matching), ranked by severity, and reported with which models agreed on each issue. In production use, this pipeline identified 19 issues across 6 review sessions — including a race condition that would have caused a $79 loss (and had already caused one in a prior session before the review pipeline existed).

### Monitoring & Alerting

- **Telegram bot** — Real-time alerts for: position drift, hedge breaks, bot crashes, watchdog failures, killswitch triggers
- **Structured logging** — PM2 log aggregation with rotation, searchable by bot name and severity
- **Watchdog cron** — Runs every 30 minutes, queries all venues, compares against expected state, alerts on any mismatch

### Infrastructure & Security

- **VPS deployment** with SSH key-only auth, fail2ban, UFW firewall (only required ports open)
- **Tailscale mesh VPN** for secure remote management — no public SSH port
- **Per-venue API keys** with minimum required permissions (a policy adopted after a security incident where a single compromised key could access all venues)
- **Git-based deployment** — all changes version controlled, no manual file edits on production

## Engineering Principles

These were learned through real incidents, not theory:

1. **Verify on-chain, not bot output.** Bot state files can diverge from reality. Always query the venue API after any action.
2. **Cancel orders before stopping bots.** PM2 stop does not cancel on-exchange orders. Orphaned orders fill at stale prices.
3. **Empty response ≠ zero positions.** An API failure returns empty data. Treating it as "no positions" leads to incorrect hedge calculations.
4. **Fail small, learn fast.** Test with $1 orders before deploying $50. Test one pair before deploying ten.
5. **Automated safety is not optional.** Every position must have a hedge. Every bot must have a killswitch. Every deployment must be verified.

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Python 3.11, async/await |
| Process management | PM2 |
| Venue APIs | aiohttp, websockets |
| Diagnostics | MCP Protocol (custom server) |
| Code review | LiteLLM + 6 free-tier AI models |
| Alerting | Telegram Bot API |
| Security | SSH, UFW, Tailscale, per-venue key isolation |
| Version control | Git, GitHub |

## Metrics

| Metric | Value |
|--------|-------|
| Codebase reduction | 135 → 27 scripts (80% reduction) |
| Uptime | 99.9% across 20+ sessions |
| Concurrent API connections | 6 venues |
| MCP diagnostic tools | 10 |
| AI review findings | 19 issues caught |
| Incident response time | < 30 seconds (automated killswitch) |

## License

This architecture documentation is MIT licensed. The trading system implementation is private.
