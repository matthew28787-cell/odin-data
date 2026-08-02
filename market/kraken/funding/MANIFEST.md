# Funding rate history — provenance

## PF_XBTUSD-hourly-to-2026-08-02.json

**This is the WRONG VENUE on purpose, and the label is the point.**

PBTCUC (Kraken Derivatives US) publishes **no funding history anywhere
public** — not on the futures API (which serves the international venue
only), not on the spot API, not on any support page. This is consistent
with its margin figures, which also exist only in the platform panel
(R-14). The panel shows current and predicted rates only.

This file is therefore **PF_XBTUSD — Kraken's international BTC perp** —
pulled from `derivatives/api/v4/historicalfundingrates` on 2026-08-02:
8,888 hourly observations, 2025-07-27 → 2026-08-02. It is a proxy for
the *shape* of BTC perpetual funding (level, spread, sign mix), not a
record of PBTCUC's rates. Different venue, different regulatory regime,
same underlying.

What it is for: choosing the funding-rate grid a continuation backtest
sweeps, instead of treating one panel reading as "the" rate. What it is
not for: any claim about what PBTCUC charged on a given day.

Summary statistics and the puller script live in
`odin-trading/docs/results/` (`funding_history.py`,
`funding-history-2026-08-02.json`). Headline: median +0.0037%/8h,
p95 +0.0145%, 27.1% of hours negative — and the engine's single PBTCUC
observation (+0.0100%/8h, 2026-08-01) sits at the **85th percentile**
of this distribution.

Cadence, verified from Kraken's own pages 2026-08-02 (REVIEW_LOG R-26):
**8-hour accrual, settled as one daily cash adjustment at 3:00pm CT** —
45 minutes before the 3:45pm CT initial-margin evaluation.

Refresh: re-run the script. The endpoint returns roughly a trailing
year; keep old pulls rather than overwriting (raw data is never edited
in place — repo rule 1).
