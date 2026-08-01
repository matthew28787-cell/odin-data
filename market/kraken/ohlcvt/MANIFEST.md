# Kraken OHLCVT archives — provenance

Fill this in when you extract files. A CSV found later with no record of where
it came from is a file of numbers nothing can be trusted against.

## Source

- **Kraken support page:** https://support.kraken.com/articles/360047124832-downloadable-historical-ohlcvt-open-high-low-close-volume-trades-data
- **Complete history (single ZIP, all pairs, all intervals):** https://drive.google.com/file/d/1ptNqWYidLkhb2VAKuLCxmp2OXEfGO-AP/view
- **Quarterly incremental updates:** https://drive.google.com/drive/folders/15RSlNuW_h0kVM8or8McOGOMfHeBFvFGI

Manual download only — there is no API for these (ADR-0005). The support page
confirms entries exist **only for intervals where trades occurred**, so missing
candles mean no trades, not missing data. That is what `data/gaps.py` handles
(ADR-0019).

## Format

Seven headerless columns, timestamp in **seconds**:

```
timestamp,open,high,low,close,volume,trades
1609459200,29000.0,29100.0,28900.0,29050.0,12.5,340
```

Kraken does not publish this in writing — it is the convention every
third-party implementation uses. `data/archive.py` therefore validates rather
than assumes: wrong column count, millisecond timestamps, and impossible OHLC
relationships all raise with the file and line number.

**If the loader rejects a real Kraken file, that is a finding, not a nuisance.**
Record it here and the expectation gets corrected.

## Files held

Downloaded 2026-07-31 from the "Complete Data" single ZIP (`Kraken_OHLCVT.zip`,
7.34 GB, 24,055 files, 25.69 GB extracted). Only these three were kept.

| File | Pair | Interval | Rows | Covers | Notes |
|---|---|---|---|---|---|
| `XBTUSD_15.csv` | BTC/USD | 15 min | 361,583 | 2013-10-06 → 2025-12-31 | **Primary backtest series** |
| `XBTUSD_60.csv` | BTC/USD | 60 min | 96,381 | same | Multi-timeframe context |
| `XBTUSD_1440.csv` | BTC/USD | daily | 4,457 | same | ADR sanity checks |

**Format confirmed.** The seven-headerless-column, seconds-timestamp assumption
in `data/archive.py` was correct — first row is
`1381095000,122.0,122.0,122.0,122.0,0.1,1`, BTC at $122 in October 2013. Prices
are sane across the span: $122 (2013) → $968 (2017-01) → $62,751 (2021-11) →
$87,500 (2025-12-31).

## Three findings that affect how this data can be used

### 1. The archive ends 2025-12-31 — it is 211 days stale

The "Complete Data" ZIP was cut at year-end 2025. Quarterly incremental updates
cover the rest. **Anything claiming to backtest "through today" using only this
file is wrong by seven months.** Fetch the quarterly updates before any result
is treated as current.

### 2. 2013–2015 cannot be loaded at the default gap ceiling

| From | Bars | Gaps | Missing | Largest gap | Loads at `max_gap_bars=96`? |
|---|---|---|---|---|---|
| 2013 | 361,583 | 11,655 | 67,451 | **229 bars (2.4 days)** | **No — raises** |
| 2016 | 345,190 | 3,095 | 5,481 | 95 bars | Yes |
| 2017 | 314,912 | 181 | 640 | 95 bars | Yes |
| 2019 | 245,200 | 72 | 272 | 26 bars | Yes |

Early Kraken was thin: 2014 alone has 29,511 missing 15-minute intervals across
3,234 gaps. That is not an error — per Kraken's documentation a missing candle
means no trades occurred — but a backtest over a period where the market barely
traded says nothing about a strategy meant for today's liquidity.

**Recommended window: 2016 onward** (345,190 bars, ~9.5 years). 2017 onward if
you want near-continuous data at the cost of 30,000 bars.

### 3. The default gap ceiling has a one-bar margin

`max_gap_bars` defaults to 96 — one day of 15-minute bars. The largest gap in
the modern data is **95 bars**: a 23.8-hour outage on 2018-01-11.

That is one bar of headroom. The default was chosen as "one day" on reasoning,
and the worst real-world Kraken outage in a decade is also almost exactly one
day. Worth knowing that a slightly longer outage in a future quarterly update
would start blocking loads — which is the intended behaviour, but it will look
like a regression if nobody remembers why.

## The seam with the quarterly updates — verified by code 2026-07-31

`data/cache.py` reproduces the hand check made when the files were placed:

| | |
|---|---|
| Base ends | `2025-12-31 23:45` |
| 2026Q1 starts | `2026-01-01 00:00` |
| Distance | **exactly one bar** |
| Overlapping bars | **0** |
| Restated bars | **0** |
| Merged total | **370,222** (361,583 + 8,639, nothing lost or doubled) |
| Merged span | 2013-10-06 21:30 → 2026-03-31 23:45 |

Kraken has not restated any history across this boundary. If a future
quarterly does restate bars, `merge_segments` raises and names them — it
will not absorb the change quietly, because that would make a backtest
run today disagree with the same backtest run last month for no recorded
reason.

**A missing quarter is a hard error, not a gap.** `gaps.py` reads a
missing interval as "no trades occurred", per Kraken's own documentation,
and `GapPolicy.SKIP` drops it as normal. A quarter that was never
downloaded is absent *data*, and letting it through would mean a backtest
silently skipping three months. `merge_segments` therefore refuses to
join `2026Q1` to `2026Q3` and names `2026Q2` in the error.

## Verifying an extraction

From `odin-trading/` with the venv active:

```powershell
python -c "from odin_trading.data.archive import load_archive; from odin_trading.data.gaps import find_gaps, gap_summary; c = load_archive(r'..\odin-data\market\kraken\ohlcvt\XBTUSD_15.csv'); print(f'{len(c):,} candles'); print(c[0].time, '->', c[-1].time); print(gap_summary(find_gaps(c, 15)))"
```

That prints the row count, the date range, and the gap profile. Record all three
in the table above — the gap summary especially, because it is the thing that
determines whether a backtest over a given period means anything.
