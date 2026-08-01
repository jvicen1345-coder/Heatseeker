---
name: heatseeker
description: Runs the Heatseeker 0DTE gamma-exposure options strategy for SPY/QQQ/SPX/AAPL/GOOGL on Robinhood using the live HOOD_AGENT tools — scans the watchlist, scores setups with the Trinity-vote framework, and manages open positions. Use when the user asks to run Heatseeker, scan for 0DTE/gamma setups, or run "the trading agent" / "the autopilot" for any of the watchlist tickers on Robinhood. Never places or closes an order without the user explicitly confirming that specific trade first — this is a decision-support and human-confirmed execution agent, not an unattended bot.
---

# Heatseeker — Robinhood 0DTE options agent

This skill runs the same strategy as the `index.html` browser dashboard in this
repo, but for real: it pulls live quotes, chains, and positions through the
`HOOD_AGENT` MCP tools instead of simulating them. `index.html` stays a
paper-trading visualizer only — it is not connected to this skill and its
"live mode" does not actually place trades.

**Hard rule: never call `place_option_order` (or any order-placing/closing
tool) without first calling `review_option_order` and getting the user's
explicit go-ahead on that specific contract, quantity, and price.** A generic
"run heatseeker" or "yes scan" is not that confirmation — only a reply that
confirms the specific proposed trade is. This applies to every entry and every
exit, with no exceptions.

## Procedure

1. **Account check.** Call `get_accounts`. Ask the user which account to use if
   there's more than one and it isn't obvious from context. Confirm it has
   `agentic_allowed: true` and `option_level_2` or `option_level_3`. If not,
   stop and explain what's missing (don't attempt any trade).

2. **Load the rules.** Read `reference/constants.json` (session windows, event
   days, risk defaults, sizing table) and keep `reference/strategy.md` (the
   full scoring algorithm) in mind — read it now if you haven't already this
   conversation.

3. **Session gates.** Check the current time (ET) against the session windows
   and `eodCloseMinuteEt`, check today's date against the event-day calendar,
   and resolve a VIX-like reading via `search` (`asset_type: market_index`,
   query "VIX") + `get_index_quotes` against `vixHaltLevel`. Report the gate
   state plainly — e.g. "Market closed for SPY/QQQ, SPX power-hour window
   opens at 3:05pm ET" or "Skipping — today is an FOMC day."

4. **Manage existing positions first.** Call `get_option_positions` for the
   account. For anything that looks like a Heatseeker position (naked long
   call/put, opened today), apply the management rules in
   `reference/strategy.md` (trailing stop, target, 5-minute stop break, EOD
   force-close). Propose any close with the reasoning and the real mark from
   `get_option_positions`, run `review_option_order`, and wait for
   confirmation before closing.

5. **Scan for new entries** (only if gates in step 3 pass and there's room
   under the position cap). For each eligible symbol in `constants.json` →
   `tickers`:
   - Get the underlying price (`get_equity_quotes` for SPY/QQQ,
     `get_index_quotes` for SPX).
   - Get today's 0DTE chain (`get_option_chains` → `get_option_instruments`
     filtered to today's expiration) for strikes near the money, with greeks
     and open interest.
   - Get recent candles (`get_equity_historicals` / index equivalent) for
     candle-direction confirmation.
   - Run the classification and scoring algorithm from
     `reference/strategy.md` (zone, regime, pattern, Trinity vote, score,
     stop/target nodes). Be explicit that GEX/DEX are proxies built from open
     interest and greeks, not a real dealer-exposure feed, and that
     VEX/flow votes are neutral unless the user has supplied that data in
     conversation.
   - If score ≥ `minScoreToTrade` and all confirmation conditions hold,
     propose the specific trade: symbol, direction, strike, 0DTE expiration,
     quantity (1 contract sized to the budget), estimated premium, stop
     level, target level, and the score breakdown. Run `review_option_order`
     and present its alerts/fees/collateral. Then stop and wait — do not
     place anything until the user confirms.
   - If score is below threshold or blocked (compression/transition/weak
     stop node/no confirmation), just report the read — no proposal needed.

6. **Session bookkeeping.** Track within this conversation which symbols have
   stopped out at a loss this session (drop the cap to 1 and block that
   symbol per `reference/strategy.md`), and the running realized P&L via
   `get_realized_pnl` against `dailyLossLimitUsd`. If the limit is breached,
   say so, stop proposing new entries, and offer to close anything still
   open.

7. **Kill switch.** If the user says stop/halt/kill at any point, immediately
   stop proposing new entries for the rest of the conversation; existing
   position management (step 4) still applies unless they say to leave
   positions alone too.

## Scope notes

- This skill only handles **single-leg long calls/puts** (Level 2), matching
  `place_option_order`'s current capability — no spreads.
- It never invents prices, marks, greeks, or exposure data. Every number in a
  proposal comes from a tool call made in that turn.
- If the user asks for a fully unattended/scheduled bot that fires trades
  without per-trade confirmation, explain that this conflicts with how the
  Robinhood order-placement tools are designed to be used (they require
  explicit confirmation) and offer the confirmed workflow above instead —
  don't build a workaround to bypass confirmation.
