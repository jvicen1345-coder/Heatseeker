# Heatseeker strategy — scoring algorithm

Ported from the `index.html` browser simulator (`classifyAndScore` and friends) and
adapted to run on **real data from HOOD_AGENT tools** instead of the simulator's
random walk / Unusual Whales feed. Read this file when you need the exact scoring
rules; `SKILL.md` only summarizes the procedure.

This is a same-day (0DTE) options momentum/gamma strategy, run against the watchlist
in `constants.json` (SPY, QQQ, SPX, AAPL, GOOGL). SPY/QQQ/SPX list daily expirations,
so 0DTE is normally available every session; AAPL/GOOGL only list same-day expirations
on their weekly (typically Friday) expiration days — on other days there's no 0DTE
chain to trade, so skip them rather than substituting a later expiration.
It looks for a strong directional consensus ("Trinity vote"), a nearby exposure
wall it can use as a hard stop, and a clean path to a target strike, then sizes
a small naked long call/put by conviction score.

## Data adaptation note (important)

The original simulator used Unusual Whales for real dealer gamma exposure (GEX),
delta exposure (DEX), vanna exposure (VEX), and options flow. **HOOD_AGENT has
none of those feeds.** Do not fabricate them. Build the closest honest proxy from
`get_option_chains` / `get_option_instruments` (open interest, greeks) and
`get_equity_historicals`:

- **GEX proxy** — for each nearby strike, `exposure = (call_open_interest * call_gamma) - (put_open_interest * put_gamma)`, scaled by 100 * spot. Positive = call-side wall (dealer long gamma / support), negative = put-side wall. This replaces the simulator's `strikes[].value`.
- **DEX proxy** — sign of `sum(call_delta * call_oi) + sum(put_delta * put_oi)` across near strikes: positive skew → `bull`, negative → `bear`, near zero → `neut`.
- **VEX / flow votes** — there is no reliable proxy without an options-flow feed. Treat both as `neut` (i.e. drop them from the vote) rather than guessing. Document this to the user when reporting the score: "Trinity vote is running 2-signal (GEX + DEX) — no flow/vanna feed available."
- Rescale the vote: with only 2 signals (GEX, DEX) instead of 4, `direction` requires **both** to agree (2/2), and `strength` is on a 2-point scale, not 4. Multiply the strength contribution in scoring by 2 to keep score magnitudes comparable to the thresholds below (a 2/2 agree ≈ old 4/4).
- If the user has independently supplied real UW-style flow/vanna context in the conversation, you may fold it in as `flowVote`/`vexVote` and use the original 4-vote math instead — say so explicitly.

## Session & risk gates (check ALL before scoring; see `constants.json`)

1. Not currently halted for the session (daily loss limit already hit, or user said kill/stop).
2. Not an event day (FOMC/CPI/NFP — `constants.json` → `eventDays2026`).
3. Volatility gate: resolve a VIX-like reading (search `asset_type: market_index`, query "VIX", then `get_index_quotes`). If VIX > `vixHaltLevel` (30 default), no new entries.
4. Within a trade window for the symbol (`sessionWindows`), and at least `openLockoutMins` past the window open.
5. Before `eodCloseMinuteEt` (955 = 3:55pm ET) — no new entries after that; existing positions get force-closed instead.
6. Not already holding a position in that symbol; under the position cap (start at 3, drop to 1 for the rest of the session the first time any position stops out at a loss).
7. Symbol not on this session's "blocked after stop-loss" list.

If any gate fails, do not score the symbol — just note why it's skipped.

## Per-symbol classification

For ticker `s` with a resolved `price` and nearby-strike exposure array `strikes` (each `{strike, value, testCount}`, `value` = GEX proxy above):

- `strikeStep(price)`: 10 if price>1000 (SPX), 5 if >400, 2.5 if >100, else 1.
- `callWall` = strike with the largest positive `value`; `putWall` = strike with the most negative `value`.
- `flipLevel` = zero-crossing of cumulative `value` walking strikes low→high (interpolate between the two strikes that bracket the sign flip); if no crossing, the strike with smallest `|value|`.
- `zone`: `above-cw` if price ≥ callWall; `pos-gex` if price ≥ flipLevel; `neg-gex` if price ≥ putWall; else `below-pw`.
- `regime`: if `|price - flipLevel| < strikeStep` → `transition`. Else if zone is `neg-gex`/`below-pw` and callWall or putWall moved ≥0.3 strikeStep since the last read → `expansion`; if zone is `neg-gex`/`below-pw` (no wall movement data available intraday — treat as `trending` when you can't compare to a prior snapshot) → `trending`; otherwise → `compression`.
- `pattern`:
  - `compression` if ≥4 strikes have `|value| > 60` within a span of `< 3*strikeStep`.
  - `flip` if a strike with `testCount ≥ 2` sits within `1.5*strikeStep` of price and price is now on the *opposite* side of that strike from its GEX sign (i.e. the wall got run through and retested).
  - `void` if the two nearest strikes above (for a call setup) or below (for a put setup) price are more than `2*strikeStep` apart — an air pocket.
  - `ladder` if the exposure values are monotonically increasing or decreasing across the strike ladder.
  - `king` if the single largest `|value|` strike is more than 2.2x the next largest.
  - else `none`.
- `candleDirection`: from `get_equity_historicals` (or index historicals for SPX), compare the close `candlePeriodTicks` bars ago to the latest close, ±0.01% noise band → `red`/`green`/`flat`.
- `deflectionConfirmed(direction, candle)`: only relevant outside neg-gex/below-pw zones — `call` needs a `red` candle (buying the dip into support), `put` needs `green` (selling the rip into resistance). Inside neg-gex/below-pw, momentum confirmation is the opposite: `call` needs `green`, `put` needs `red`.

## Trinity vote

- `gexVote`: `bull` if zone is `pos-gex`/`above-cw`, `bear` if `neg-gex`/`below-pw`, else `neut`.
- `dexVote`: the DEX proxy above.
- Count agreement between the two (or four, if using real flow data). `direction = call` if bulls strictly outnumber bears and meet the majority threshold for the vote-count in use; `put` for the reverse; otherwise no trade.

## Scoring (build up `score`, collect `reasons`)

Hard blocks (score forced to 0, no trade): `pattern === 'compression'`, `regime === 'transition'`, or vote strength below the majority threshold.

Additive, starting from the vote base:
- Full agreement (4/4, or 2/2 in the 2-signal mode): **+4**. 3/4: **+3**. 2/4 (4-signal mode only): **+2**.
- `regime === 'expansion'`: **+2**. `regime === 'trending'`: **+2**.
- `pattern === 'king'`: **+3**. `void`: **+2**. `ladder`: **+2**. `flip`: **+2**.
- Zone/direction alignment: neg-gex/below-pw + put: **+1**. pos-gex + call: **+1**. above-cw + put: **+1**.
- Stop node quality (the exposure strike on the *opposite* side of price from the trade direction — `gkBelow` for calls, `gkAbove` for puts): if `|value| < minNodeGex` (30) the node is **too weak — abort, no trade**. `testCount === 0` (fresh): **+1**. `testCount ≥ 2` (stale): **-1**.
- Target path (`selectVelocityTarget`: next strike in the trade direction where the gap to the strike after it is ≥ `2*strikeStep`, i.e. where price could air-pocket through): clean path (0 strikes between price and target): **+2**. One strike in the way: **+1**. Two or more: **-1**.
- Pin risk: another strike within `1.5*strikeStep` of price with `|value| > 120` that isn't the stop node: **-1**.
- Inside the last `charmWindowMins` (30) of *any* session window, and regime is `trending`/`expansion`: **+1**.

If `score < minScoreToTrade` (6), or there's no stop node, or the stop node is weak, or price action doesn't confirm (`deflectionConfirmed`) — **no trade, just report the read.**

## Sizing & strike selection

- `capital` = $75 if score ≥ 10, $50 if score ≥ 8, else $25 (cap at the user's configured max, default $75).
- Strike: walk OTM strikes in the trade direction, nearest first; pick the first one whose **real delta** (from `get_option_instruments`/chain greeks — do not estimate) falls in `[0.20, 0.45]` **and** whose ask/mid premium × 100 ≤ `capital`. If nothing in that band and budget fits, fall back to the first OTM strike.
- Stop level = the stop node's strike. Target level = the velocity target's strike (may be null).
- Expiration = today (0DTE) for SPY/QQQ/SPX where available; if 0DTE isn't listed for the symbol that trading day, say so and skip rather than substituting a different expiration silently.

## Position management (existing positions)

Pull marks from `get_option_positions` (never synthesize a P&L). For each open Heatseeker position:

- **Trailing stop**: once unrealized P&L ≥ `capital * trailTriggerPct` (100%), arm a trailing stop at `capital * trailExitPct` (90%). Once armed, exit the moment P&L drops to or below that trail level — propose a close.
- **Target hit**: if the underlying trades through the recorded target strike in the trade's favor, propose a close.
- **Stop break**: if the underlying crosses the recorded stop strike against the position, start a 5-minute clock (only if the trailing stop isn't armed yet). If it's still beyond the stop after `stopHoldMs` (5 min) of wall-clock time, propose a close. If price recrosses back before then, the clock resets — note that in the log, don't close.
- **EOD force-close**: at/after `eodCloseMinuteEt` (3:55pm ET), propose closing everything still open — 0DTE options go to zero at 4pm.
- After any stop-loss exit, add that symbol to this session's blocked list and drop the position cap to 1 for the rest of the session (matches the simulator's circuit breaker).
- After any exit, check `get_realized_pnl` / running day total against `dailyLossLimitUsd`; if breached, halt the session (no more new entries) and say so clearly, then propose closing anything still open.

Every proposed close, like every proposed entry, goes through `review_option_order` and explicit user confirmation before `place_option_order` is called.
