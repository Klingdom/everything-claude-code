# TradingAgents Integration

This fork bundles the [TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents) multi-agent LLM trading framework alongside Everything Claude Code (ECC).

The integration is **additive** — none of ECC's existing agents, skills, commands, or hooks are modified. The TradingAgents code is vendored under `tradingagents/` and its 12 agent personas are mirrored as ECC subagents under `agents/trading-*.md`.

## What was added

| Path | Purpose |
|---|---|
| `tradingagents/` | Vendored copy of TradingAgents source (Apache-2.0). Original `LICENSE` preserved at `tradingagents/LICENSE`. |
| `docker-compose.yml` | Local-development compose. Builds from `./tradingagents` and uses `.env` for credentials. |
| `compose.hostinger.yml` | Remote-build compose for the Hostinger Docker panel. Builds context directly from this GitHub repo. |
| `.env.example` (appended) | TradingAgents-related variables. The existing `ANTHROPIC_API_KEY` is reused. |
| `agents/trading-*.md` | 12 ECC-native subagent mirrors of the TradingAgents personas. Pure Claude Code, no Python runtime needed to invoke them. |
| `commands/trade-analysis.md` | Slash command that orchestrates the 12 subagents end-to-end. |

## Two ways to use it

### 1. Run the full Python framework (real data fetching, LangGraph orchestration)

This is what TradingAgents was built for — agents call real data tools (Yahoo Finance, StockTwits, Reddit, FinnHub) and collaborate through LangGraph.

```bash
cp .env.example .env             # fill in ANTHROPIC_API_KEY
docker compose build
docker compose run --rm tradingagents
```

For Hostinger panel deploys, see `compose.hostinger.yml` and the [Hostinger deploy notes](#hostinger-deploy) below.

### 2. Use the ECC-native subagent mirrors (no Python, no Docker)

Each TradingAgents persona has a counterpart in `agents/trading-*.md`. Invoke them directly via Claude Code's subagent system or via the `/trade-analysis <TICKER> <DATE>` slash command. They use Claude Code's built-in tool use (WebFetch, MCP servers) for any data fetching — much less feature-rich than the Python framework, but instantly usable in a fresh Claude Code session.

## Hostinger deploy

In the Hostinger Docker Compose panel:

1. Source = the raw URL of `compose.hostinger.yml` on this branch:
   `https://raw.githubusercontent.com/Klingdom/everything-claude-code/feature/tradingagents/compose.hostinger.yml`
2. Set environment variables in the panel:
   - `ANTHROPIC_API_KEY` (required)
   - `TRADINGAGENTS_LLM_PROVIDER` (optional, default: `anthropic`)
   - `TRADINGAGENTS_DEEP_THINK_LLM` (optional, default: `claude-opus-4-7`)
   - `TRADINGAGENTS_QUICK_THINK_LLM` (optional, default: `claude-haiku-4-5`)
3. Save & deploy. The container builds directly from the GitHub repo — no source code is copied onto the VPS.
4. SSH to the VPS and run the interactive CLI:
   ```bash
   docker compose -f /path/to/compose.hostinger.yml exec tradingagents tradingagents
   ```

The container stays alive with `sleep infinity` so you can `exec` into it on demand. Decision logs and checkpoints persist in the `tradingagents_data` named volume.

## Disclaimers

- TradingAgents is a **research framework**. From its README: *"not intended as financial, investment, or trading advice."*
- This integration vendors the upstream source under its original Apache-2.0 license. ECC remains MIT-licensed. The combined repository must respect both — attribution for `tradingagents/` lives at `tradingagents/LICENSE`.
- Costs add up fast. Each ticker analysis with multi-round debate can use $0.50–$3 of API credits. Set spend limits in the Anthropic console before unattended use.
