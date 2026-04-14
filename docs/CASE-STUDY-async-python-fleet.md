# Case Study: 27-Script Async Python Fleet with MCP Diagnostics

## Problem Statement

A crypto trading operation had grown organically to 135 Python scripts across 6 venue APIs, creating a maintenance nightmare. Scripts had inconsistent error handling, duplicated logic, and no centralized monitoring. Debugging production issues required SSH-ing into multiple servers and grepping through scattered logs. The fleet needed:

- Consolidation without losing functionality
- Real-time diagnostics accessible from development environment
- Sub-30-second incident response capability
- 99.9% uptime for mission-critical trading operations

## Technical Approach

### Architecture Redesign
Refactored the entire codebase using a layered architecture:
1. **Orchestration Layer** — PM2 manages all processes with restart policies
2. **Venue Integration Layer** — Unified async clients for 6 exchanges (REST + WebSocket)
3. **Diagnostic Layer** — Custom MCP server exposing fleet state to Claude Code
4. **Monitoring Layer** — Telegram alerts + automated watchdog (cron */30)

### MCP Diagnostic Server
Built 10 custom MCP tools enabling real-time fleet inspection:
- `fleet_verify` — Query all 7 wallets, compute hedge status
- `fleet_parity_audit` — Compare documented state vs live reality
- `position_check` — Instant position snapshot across venues
- `bot_status` — PM2 process health with restart counts
- `margin_capacity` — Available margin per venue
- Plus 5 more specialized diagnostic tools

### Safety Systems
- Automated killswitch on position drift >$50
- Order cancellation before any bot shutdown (learned: PM2 stop doesn't cancel on-chain orders)
- Hedge verification: every position must exist on 2+ venues simultaneously

## Stack Used

| Category | Technologies |
|----------|-------------|
| Language | Python 3.11, async/await |
| Orchestration | PM2, ecosystem.config.js |
| Networking | aiohttp, websockets |
| Protocol | MCP (Model Context Protocol) |
| AI Integration | LiteLLM, Claude Code |
| Alerting | Telegram Bot API |
| Security | SSH keys, UFW, Tailscale VPN |
| Version Control | Git, GitHub |

## Key Metrics

| Metric | Before | After |
|--------|--------|-------|
| Script count | 135 | 27 (80% reduction) |
| Uptime | ~95% | 99.9% |
| Incident response | 15+ minutes | <30 seconds |
| Duplicate code | ~40% | <5% |
| MCP diagnostic tools | 0 | 10 |
| Venue integrations | 6 (fragmented) | 6 (unified) |

## Challenges Overcome

### 1. Orphaned Orders Problem
**Issue:** PM2 stop doesn't cancel on-chain orders. A $79 loss occurred from orphaned limit orders filling after bot shutdown.
**Solution:** Implemented mandatory `cancel_all_orders()` call in bot shutdown hooks. Added pre-flight check to verify no open orders before any deployment.

### 2. Venue Position Netting
**Issue:** Ethereal and Perpl merge same-asset positions. Opening a SHORT against an existing LONG reduces the LONG instead of creating a separate SHORT.
**Solution:** Added venue-specific position logic. Check existing positions before choosing hedge venue. Never assume separate long/short legs on netting venues.

### 3. Context Loss in Long Sessions
**Issue:** AI context gets compressed, losing critical fleet state mid-session.
**Solution:** Created 4-file system (FLEET_MEMORY.md, FLEET_STATE.md, FLEET_BUGS.md, CLAUDE.md) that survives context resets. Re-read all files at session start.

## GitHub Repository

[github.com/JustDreameritis/fleet-architecture](https://github.com/JustDreameritis/fleet-architecture)

## Lessons Learned

1. **Verify, don't trust** — "Order placed successfully" ≠ "Order filled". Always query venue for actual state after any action.

2. **Build killswitches early** — The automated killswitch saved thousands in potential losses. Safety systems pay for themselves on first incident.

3. **Document operational state** — Code documentation isn't enough. Live fleet state must be tracked separately and verified against reality.

4. **MCP for operations** — MCP isn't just for AI assistants. It's excellent for exposing production system state to any tooling.

5. **Consolidation > features** — 27 well-maintained scripts beat 135 scattered ones. Reduced cognitive load enables faster iteration.

---

*Built for a trading operation requiring institutional-grade reliability on retail infrastructure.*
