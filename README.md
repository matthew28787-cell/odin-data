# odin-data — the shared data root for Project ODIN

Every ODIN module that needs bulk data reads from here. It sits beside the
module repos rather than inside any one of them, because data outlives the
module that first needed it: the trading module wants Kraken candles today,
and the finance and research modules will want their own bulk sources later.
Putting market data inside `odin-trading/` would mean a second copy the day a
second module wants it.

**This became a git repository on 2026-08-01**, reversing the original
design on purpose. "Not a git repo" was written when the contents were
"large, re-downloadable" *and* OneDrive was the live backup. OneDrive was
retired (platform REVIEW_LOG R-23 F-3), which would have left the archive
with no backup but a re-download — and the MANIFEST provenance chain with
no history at all. At 27MB the "large" premise no longer holds; revisit
with git-lfs if a future module's data actually gets big. `_incoming/`
stays untracked by rule: staging is emptied after extraction, never
committed.

## Layout

```
odin-data/
├── market/                  Price and volume data
│   └── kraken/
│       ├── ohlcvt/          Extracted CSVs, ready to load
│       │   ├── XBTUSD_15.csv
│       │   └── MANIFEST.md  Provenance — what, when, from where
│       └── _incoming/       Staging for downloads. Emptied after extraction.
├── finance/                 Statements, ledgers, tax documents (future)
├── properties/              Deeds, valuations, rent rolls (future)
├── research/                Scraped and downloaded research corpora (future)
├── assets/                  Inventories, valuations (future)
└── _schemas/                Column definitions for anything whose format
                             isn't self-describing
```

## Rules

**1. Raw data is never edited in place.** A downloaded file stays exactly as it
arrived. Cleaning, gap-filling, and reformatting happen in code at load time —
`odin-trading/src/odin_trading/data/` is the example. An edited raw file cannot
be re-derived, and a backtest run against a hand-patched CSV is not reproducible.

**2. Every directory that holds real data carries a `MANIFEST.md`.** What the
file is, where it came from, when it was fetched, and what it covers. Without
that, a CSV found in six months is a file of numbers with no provenance, and
nothing computed from it can be trusted or reproduced.

**3. `_incoming/` is staging, not storage.** Download there, extract what's
needed, delete the archive. The full Kraken ZIP is tens of gigabytes; the one
file the trading module actually uses is about 27 MB.

**4. Nothing here is a secret.** Credentials live in each module's `.env`.
If something sensitive ever needs storing, it does not go in a directory whose
whole purpose is being shared across modules.

## A note on OneDrive

This tree sits inside a OneDrive-synced folder. That is fine for the working
files — the BTC/USD 15-minute archive is ~27 MB — but **do not leave multi-GB
downloads in `_incoming/`**. Sync them once and OneDrive will keep doing it.

Download bulk archives somewhere outside OneDrive (`C:\Temp\` is fine), extract
only what is needed, and copy just those files in.
