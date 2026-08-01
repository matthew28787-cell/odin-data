# Kraken quarterly incremental updates

The "Complete Data" ZIP is cut once and then supplemented quarterly. The base
archive in `../ohlcvt/` ends **2025-12-31**; these fill the gap from there to
the present.

Source: https://drive.google.com/drive/folders/15RSlNuW_h0kVM8or8McOGOMfHeBFvFGI
(linked from Kraken's OHLCVT support page, "Incremental Updates")

## Why these are kept separate rather than appended to the base file

`odin-data/README.md` rule 1: **raw data is never edited in place.** Appending
a quarter onto `XBTUSD_15.csv` would produce a file that no longer corresponds
to anything Kraken published, cannot be re-derived, and cannot be checked
against the source. Every backtest run against it afterwards would be
unreproducible.

Merging is a **code** operation — `data/cache.py` stitches base + quarterlies at
load time and validates the seam. That keeps each downloaded file exactly as it
arrived and makes the merge inspectable rather than baked in.

The seam is the part worth checking: overlapping bars between the base archive
and a quarterly must agree, and a mismatch means Kraken restated history. That
check is REVIEW_LOG R-3's recommendation, finally implementable now that both
sources exist.

**Result, 2026-07-31 (verified by `merge_segments`, not by eye):** the base ends
`2025-12-31 23:45` and 2026Q1 starts `2026-01-01 00:00` — exactly one bar apart,
**zero overlap, zero restatements**, merging to **370,222 bars** spanning
2013-10-06 → 2026-03-31 with nothing lost or duplicated. Run
`load_merged("XBTUSD", 15)` and read the returned `MergeReport.describe()` to
reproduce it.

## What the merge refuses to do

Three things raise rather than proceeding, because each would produce a
plausible-looking backtest over data that isn't what it claims to be:

1. **Restated bars.** A timestamp present in two files with different values.
   Pass `restatements=RestatementPolicy.PREFER_NEWER` to accept Kraken's
   correction deliberately — the report still lists every changed bar.
2. **A missing quarter in the middle.** `2026Q1` + `2026Q3` with no `2026Q2`
   raises and names the missing quarter. This is the important one: `gaps.py`
   treats a missing interval as "no trades occurred" per Kraken's docs, so
   without this check a three-month hole would be skipped **silently**.
3. **An oversized gap at a file boundary.** Defaults to the same 96-bar ceiling
   as `gaps.py` but is a separate setting, because a hole at a seam is far more
   likely to be a file nobody downloaded than a quiet market.

Staleness is deliberately **not** an error. How fresh the data must be depends
on the run — a decade-long stability test doesn't care that the last few months
are missing, a paper-trading comparison does. `MergeReport.staleness_days()`
reports it and the caller decides. As of 2026-07-31 the merged series is **121
days stale**.

## Layout

```
quarterly/
├── 2026Q1/          XBTUSD_15.csv, XBTUSD_60.csv, XBTUSD_1440.csv
└── MANIFEST.md      this file
```

**Only 2026Q1 has been published** as of 2026-07-31. Q2 was expected at the end
of June but is not in Kraken's Drive folder. **Re-checked 2026-08-01: still
absent from both the OHLCVT and the trade-data quarterly folders** (both end at
Q1 2026). Check again before assuming the data reaches further than it does.

One directory per quarter, same filenames as the base archive. Do not rename —
the loader keys off Kraken's convention.

## Quarters held

| Quarter | Files | Rows (15m) | Covers | Downloaded | Seam checked |
|---|---|---|---|---|---|
| 2026Q1 | XBTUSD_15/60/1440 | 8,639 | 2026-01-01 00:00 → 2026-03-31 23:45 | 2026-07-31 | 2026-07-31 by hand; 2026-08-01 by `load_merged` |

The 2026Q1 15-minute series has one missing bar (2026-02-04 11:15) — no trades
that interval, per Kraken's convention; `gaps.py` handles it as normal.

## Coverage after merge

| Source | Covers |
|---|---|
| Base archive | 2013-10-06 → 2025-12-31 |
| 2026Q1 | 2026-01-01 → 2026-03-31 |
| 2026Q2 | **not published** (re-checked 2026-08-01) |
| REST `/public/OHLC` | trailing ~7.5 days (720 bars) |
| **Gap remaining** | **2026-04-01 → ~2026-07-24** (roughly four months) |

The gap is larger than expected because Q2 has not been released. Four months of
2026 have no archive source. `/public/OHLC` covers only the trailing 720 bars,
so it closes the last week and nothing before it.

**Consequence for Phase 4e:** a backtest can run continuously through
2026-03-31, then must either stop or accept a four-month hole. It should stop,
and say where it stopped. Quietly ending early is the failure mode ADR-0019's
gap ceiling exists to prevent — and a four-month gap is 11,500 bars, well past
any sane `max_gap_bars`, so the loader will refuse it rather than paper over it.
That refusal is correct.
