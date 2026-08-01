# Heatseeker

A 0DTE options gamma-scalping strategy for SPY, QQQ, and SPX ("Trinity vote"
scoring across gamma exposure, delta exposure, and price action), in two
parts:

- **`index.html`** — a self-contained browser dashboard that paper-trades the
  strategy against simulated (or, with an Unusual Whales API key, live) GEX
  data. It's a visualizer/backtester, not connected to a real broker — its UI
  has a "live mode" toggle but that path isn't wired to any working backend.
- **`.claude/skills/heatseeker/`** — a Claude Code skill that runs the same
  strategy for real, using live Robinhood account/quote/chain/position data
  through the `HOOD_AGENT` tools. It scans the watchlist, scores setups, and
  manages open positions, but **never places or closes an order without the
  user explicitly confirming that specific trade first**. Invoke it by asking
  Claude Code to "run Heatseeker" / "scan for 0DTE setups" with a Robinhood
  connector attached. See `.claude/skills/heatseeker/SKILL.md` for the full
  procedure and `reference/strategy.md` for the scoring algorithm.