# Airline group: version 2 (audited), October 2026

This folder replaced the unverified version 1 airline group (421 filings) on 2026-10-09.

**Required credit:** any use of the airline data must credit *The AI Plus Research Lab at the University of Northern Iowa*.
**Airline-group licence: CC BY-NC 4.0 (non-commercial)**, which is stricter than the CC BY 4.0 licence on the rest of this repository. See `docs_v2/LICENSE.txt` and `docs_v2/ATTRIBUTION.md`. Contact: zhengyang.chen@uni.edu.

## What is here
| File | Content |
|---|---|
| `hedge_ratios.csv` | 601 firm-years (one report per company and fiscal year), 52 columns |
| `filings.csv` | all 603 annual reports (same columns) |
| `positions.csv` | 692 hedge coverage periods, each with the quote from the filing |
| `v2_support/` | company_summary, universe_register, corrections_vs_original (row-by-row v1 vs v2), original_rows_not_carried_over, excluded_filings, adjudications, reread_results (raw reader output) |
| `docs_v2/` | CODEBOOK, METHODOLOGY, LIMITATIONS_AND_FAQ, GLOSSARY, VERSION_HISTORY, validation reports |

## Differences from version 1 (breaking changes)
* **Schema changed.** The old columns (`hedge_ratio_next` as a fraction, `has_program`, `stopped_hedging`, `warnings`, `ticker`...) are gone. The headline is now `hedge_pct_next` in **percent** (43 = 43%), with `hedge_pct_next_basis` and `hedge_pct_next_asof`. See `docs_v2/CODEBOOK.md`.
* **New status taxonomy** (`hedger`, `hedger_no_next_year_figure`, `no_open_positions`, `no_hedging`, `no_hedging_after_hedging`, `no_hedging_inferred`, `stopped_this_year`, `refinery_hedges_only`, `not_discussed`). The v1 "stopped" label that stuck to every later year no longer exists.
* **Key is `cik` + `accession`**; every row carries the accession number.
* 603 reports from 51 registrants (v1: 421 from 26); 26 registrants added, 18 duplicate rows removed.
* `extractions/` (v1 per-filing JSON) removed; the raw reader output is in `v2_support/reread_results/`.
* `master_panel.csv` and `crosswalk.csv` were regenerated for the airline group (mapping below). `ticker` is filled only for registrants that had one in v1.

## Mapping of v2 `status` to the master-panel status
| v2 status | master_panel status | hedge_ratio_primary |
|---|---|---|
| hedger, hedger_no_next_year_figure | hedger | hedge_pct_next/100 where a headline exists, else null |
| no_open_positions | no_position | 0.0 |
| no_hedging_after_hedging, stopped_this_year | stopped | 0.0 |
| no_hedging | never | null |
| no_hedging_inferred | pass_through (regional carriers) or never | null |
| not_discussed, refinery_hedges_only | not_applicable | null |

`master_panel` airline rows use the v2 `fuel_gallons_consumed_mm` as `exposure_base` and now carry `accession_number`.

## v1 backup
The unverified v1 files, master panel and docs as they were are in `../../../archive/airline_v1_before_v2_2026-10-09/` (outside the repository; also recoverable from git history).
