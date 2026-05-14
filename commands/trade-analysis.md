---
description: Run the TradingAgents multi-agent analysis flow on a ticker — analysts → bull/bear debate → research plan → trader proposal → risk debate → portfolio decision.
argument-hint: <TICKER> [YYYY-MM-DD]
---

# Trade Analysis

> Research workflow only. Not financial advice. See INTEGRATION-TRADINGAGENTS.md for the integration context.

**Input**: `$ARGUMENTS`

Parse arguments:
- First token = `TICKER` (uppercase, e.g. `NVDA`). Required.
- Second token = `DATE` in `YYYY-MM-DD`. Optional; default to today's date.

If `TICKER` is missing, ask the user for it before continuing.

---

## Flow

Execute the following phases. Where multiple agents can run independently, dispatch them **in parallel**.

### Phase 1 — Analysts (parallel, 4 agents)

Dispatch in a single message with four `Agent` tool calls:

1. `trading-fundamentals-analyst` — produces `fundamentals_report`
2. `trading-market-analyst` — produces `market_research_report`
3. `trading-news-analyst` — produces `news_report`
4. `trading-sentiment-analyst` — produces `sentiment_report`

Each agent receives `TICKER`, `DATE`, and any user-supplied instrument context.

**Stop condition for phase 1**: all four reports returned, or the user aborts.

### Phase 2 — Bull/bear debate (sequential, up to N rounds)

Default `N = 2` debate rounds. Set `N = 3` if the user passes `--deep` in `$ARGUMENTS`.

For round `i` in `1..N`:
1. Dispatch `trading-bull-researcher` with the four analyst reports + `debate_history` so far + `last_bear_argument` (empty in round 1).
2. Dispatch `trading-bear-researcher` with the same context + `last_bull_argument` from this round.
3. Append both to `debate_history`.

### Phase 3 — Research Manager (single call)

Dispatch `trading-research-manager` with the full `debate_history`. It returns the `investment_plan` containing a Rating (Buy / Overweight / Hold / Underweight / Sell) and supporting structure.

### Phase 4 — Trader (single call)

Dispatch `trading-trader` with the `investment_plan` and the four analyst reports. It returns the `trader_decision` with a concrete BUY / HOLD / SELL and a `FINAL TRANSACTION PROPOSAL` line.

### Phase 5 — Risk debate (sequential, up to M rounds)

Default `M = 1` round. Set `M = 2` if `--deep`.

For each round, dispatch in this fixed order:
1. `trading-risk-aggressive`
2. `trading-risk-conservative`
3. `trading-risk-neutral`

Each receives the `trader_decision`, the four analyst reports, and the `risk_debate_history` so far.

### Phase 6 — Portfolio Manager (final call)

Dispatch `trading-portfolio-manager` with the `investment_plan`, `trader_decision`, and the full `risk_debate_history`. It returns the `final_trade_decision` with Rating + Conviction + Specific Action + What-would-change-my-mind.

---

## Output to the User

Present results in this order:

1. **Final decision** — rating, conviction, specific action. Lead with it.
2. **One-line "Why not the adjacent rating"** — from the Portfolio Manager.
3. **Triggers to reassess** — from the Portfolio Manager.
4. **Show your work** — collapsible sections (or clearly headed) for:
   - Analyst reports (4)
   - Bull/bear debate transcript
   - Research Manager's investment plan
   - Trader's proposal
   - Risk debate transcript
5. **Disclaimer** — "Research output, not financial advice."

---

## Hard Rules

- Never trade or instruct another tool to trade on the user's behalf. This command produces *research*, period.
- If a phase's agent fails or returns nothing, report the partial state and stop — do not fabricate the missing output.
- Token costs are non-trivial. If `--deep` is set and the user hasn't confirmed, ask before proceeding past Phase 1.
- This command is a Claude-Code-native mirror of the Python TradingAgents framework in `tradingagents/`. For real market data fetching (StockTwits, FinnHub, etc.), run the Python framework via `docker compose run --rm tradingagents` and have the user paste the report back in.
