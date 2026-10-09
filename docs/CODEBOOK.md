# Codebook: U.S. Commodity Hedging Dataset

*Last updated: 2026-05-24*

This codebook documents every column of every delivered CSV file. For each variable: name, type, units, allowed values, source, and missing-value code.

**Abbreviations:** str = string, int = integer, float = floating-point number, bool = True/False, null = missing value.

---

## Table of Contents

1. [data/master_panel.csv](#1-datamasterpanelcsv)
2. [data/crosswalk.csv](#2-datacrosswalkcsv)
3. [data/ep/hedge_ratios.csv](#3-dataephedgeratiosscsv)
4. [data/ep/positions.csv](#4-dataeppositionscsv)
5. [data/ep/filings.csv](#5-dataepfilingscsv)
6. [data/ep/filing_index.csv](#6-dataepfilingindexcsv)
7. [data/ep/verification_log.csv](#7-dataepverificationlogcsv)
8. [data/airline/hedge_ratios.csv](#8-dataairlinehedgeratioscsv)
9. [data/airline/positions.csv](#9-dataairlinepositionscsv)
10. [data/airline/filings.csv](#10-dataairlinefilingscsv)
11. [data/refiner/hedge_ratios.csv](#11-datarefinerhedgeratioscsv)
12. [data/refiner/positions.csv](#12-datarefinerpositionscsv)
13. [data/refiner/filings.csv](#13-datarefinerfilingscsv)

---

## 1. data/master_panel.csv

Grain: one row per `(cik, fiscal_year, hedge_axis)`. Long format across all three groups.
6,772 rows total (5,936 ep + 601 airline + 235 refiner).

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK, leading-zero-padded | never null |
| `group` | str | — | `ep` / `airline` / `refiner` | never null |
| `fiscal_year` | int | year | Calendar year the hedge ratio applies to | never null |
| `hedge_axis` | str | — | `oil` / `gas` / `fuel` / `crude_net` | never null |
| `company` | str | — | Company name as of filing | blank if unavailable |
| `ticker` | str | — | Exchange ticker; from crosswalk or SEC cache | blank if unavailable |
| `hedge_ratio_primary` | float | fraction | See table below; NOT comparable across groups | null if status ∈ {never, pass_through, not_applicable} |
| `ratio_method` | str | — | `delta` / `notional_volume` / `net_short` | never null |
| `hedge_volume` | float | see volume_unit | Numerator of hedge ratio (delta-adjusted or notional) | null for airline (pct already computed) |
| `volume_unit` | str | — | `bbl` / `mcf` / `gallon` / `mbbl` | never null |
| `exposure_base` | float | see exposure_unit | Denominator of hedge ratio (production / consumption / throughput) | null if not_applicable |
| `exposure_unit` | str | — | `bbl` / `mcf` / `gallon_mm` / `mbbl` | never null |
| `has_program` | str | — | `True` / `False` / blank | blank if unknown |
| `status` | str | — | `hedger` / `no_position` / `stopped` / `never` / `pass_through` / `not_applicable` | never null |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | blank for some airline/refiner rows |
| `accession_number` | str | — | SEC accession number (ep only) | blank for airline/refiner |

**hedge_ratio_primary by group/axis:**

| group | hedge_axis | ratio_method | numerator | denominator |
|---|---|---|---|---|
| ep | oil | delta | `oil_delta_bbls` (Bbls) | `oil_production_bbls` (Bbls, FY+1) |
| ep | gas | delta | `gas_delta_mcf` (Mcf) | `gas_production_mcf` (Mcf, FY+1) |
| airline | fuel | notional_volume | implied by `volume_pct_of_consumption` | `expected_gallons_millions` (M gallons) |
| refiner | crude_net | net_short | `net_short_mbbl` (MBbl) | `throughput_mbbl` (MBbl) |

---

## 2. data/crosswalk.csv

Grain: one row per `(cik, era_start_fy, era_end_fy)`. 50 era rows, covering all airline and refiner entities plus a subset of E&P entities that changed identity or group.

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK, leading-zero-padded | never null |
| `era_start_fy` | int | year | First fiscal year this row applies to (inclusive) | never null |
| `era_end_fy` | int | year | Last fiscal year this row applies to (inclusive); 9999 = present | never null |
| `canonical_ticker` | str | — | Primary exchange ticker for this era | blank if delisted / no ticker |
| `company_name` | str | — | Canonical name for this era | never null |
| `group` | str | — | `ep` / `airline` / `refiner` | never null |
| `entity_id` | str | — | Stable identity string across CIK changes (e.g., `frontier_family_mid`) | blank if no known family grouping |
| `lineage` | str | — | Free-text M&A or name-change annotation | blank if none |
| `business_model` | str | — | `standard` / `pass_through` / `royalty_trust` / `lcc` / `ulcc` / `regional` | `standard` if unspecified |

---

## 3. data/ep/hedge_ratios.csv

Grain: one row per `(cik, fiscal_year)`. E&P firm-year panel. 2,963 rows.

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK | never null |
| `company` | str | — | Company name as of filing | blank if unavailable |
| `fiscal_year` | int | year | Production year being hedged (FY+1 from filing) | never null |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | never null |
| `accession_number` | str | — | SEC accession number | never null |
| `uses_derivatives` | str | — | `True` / `False` | never null |
| `hedge_ratio_oil_notional` | float | fraction | Oil: notional volume / oil production | null if no oil production or no instruments |
| `hedge_ratio_gas_notional` | float | fraction | Gas: notional volume / gas production | null if no gas production or no instruments |
| `hedge_ratio_oil_delta` | float | fraction | Oil: delta-adjusted volume / oil production | null if no oil production |
| `hedge_ratio_gas_delta` | float | fraction | Gas: delta-adjusted volume / gas production | null if no gas production |
| `hedge_ratio_ngl_delta` | float | fraction | NGL: delta-adjusted / NGL production (same-filing denominator) | null if no NGL production reported |
| `hedge_ratio_oil_reserve` | float | fraction | Oil: delta-adjusted / proved oil reserves | null if reserves not available |
| `hedge_ratio_gas_reserve` | float | fraction | Gas: delta-adjusted / proved gas reserves | null if reserves not available |
| `oil_notional_bbls` | float | Bbls | Total oil hedged volume (notional) | 0.0 if none; null if unavailable |
| `gas_notional_mcf` | float | Mcf | Total gas hedged volume (notional) | 0.0 if none; null if unavailable |
| `ngl_notional_bbls` | float | Bbls | Total NGL hedged volume (notional) | 0.0 if none; null if unavailable |
| `oil_delta_bbls` | float | Bbls | Oil delta-adjusted volume | 0.0 if none; null if unavailable |
| `gas_delta_mcf` | float | Mcf | Gas delta-adjusted volume | 0.0 if none; null if unavailable |
| `ngl_delta_bbls` | float | Bbls | NGL delta-adjusted volume | 0.0 if none; null if unavailable |
| `oil_production_bbls` | float | Bbls | Annual oil production (FY+1 denominator) | null if not available |
| `gas_production_mcf` | float | Mcf | Annual gas production (FY+1 denominator) | null if not available |
| `ngl_production_bbls` | float | Bbls | Annual NGL production (same-filing denominator) | null if not reported |
| `oil_proved_reserves_bbls` | float | Bbls | Proved oil reserves | null if not available |
| `gas_proved_reserves_mcf` | float | Mcf | Proved gas reserves | null if not available |
| `n_oil_instruments` | int | count | Number of oil instruments | 0 if none |
| `n_gas_instruments` | int | count | Number of gas instruments | 0 if none |
| `n_ngl_instruments` | int | count | Number of NGL instruments | 0 if none |
| `delta_method` | str | — | `notional_only` / `linear` / `simplified` | never null |
| `warnings` | str | — | Pipe-separated quality flags; empty string = no issues | empty string |

**delta_method values:**

| Value | Meaning |
|---|---|
| `notional_only` | No qualifying instruments; hedge ratio is zero or null by definition |
| `linear` | All instruments are swaps (delta = −1.00); no approximation |
| `simplified` | At least one non-linear instrument; delta from Haushalter (2000) fixed approximations |

---

## 4. data/ep/positions.csv

Grain: one row per instrument per filing. Instrument-level delta records for all E&P filings. 14,538 rows. Formerly `instrument_deltas.csv` in v1.

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK | never null |
| `company` | str | — | Company name | blank if unavailable |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | never null |
| `fiscal_year` | int | year | Production year this instrument covers | never null |
| `instrument_type` | str | — | swap / collar / three_way_collar / basis_swap / swaption / put_option / call_option / fixed_price_contract / forward | never null |
| `commodity` | str | — | crude_oil / natural_gas / ngl | never null |
| `direction` | str | — | short / collar / long | never null |
| `volume_total` | float | see volume_unit | Total hedged volume for the period | null if not extractable |
| `volume_unit` | str | — | Bbls / Mcf / Mmbtu / Boe | never null |
| `delta` | float | — | Delta equivalent (−1.0 for swaps; −0.5 for collars; −0.65 for three-way collars) | null for basis_swap, swaption (excluded) |
| `delta_volume` | float | see volume_unit | `volume_total × |delta|` | null if excluded |
| `included` | bool | — | True if included in hedge ratio calculation; False for basis_swap and swaption | never null |
| `period_start` | str | YYYY-MM | Coverage start month | null if not in source |
| `period_end` | str | YYYY-MM | Coverage end month | null if not in source |
| `fixed_price` | float | $/Bbl or $/Mmbtu | Swap fixed price | null if not swap |
| `floor_price` | float | $/Bbl or $/Mmbtu | Put strike for collars | null if not collar |
| `ceiling_price` | float | $/Bbl or $/Mmbtu | Call strike for collars | null if not collar |
| `sub_floor_price` | float | $/Bbl or $/Mmbtu | Short put strike for three-way collars | null if not three-way |

---

## 5. data/ep/filings.csv

Grain: one row per extraction file (one per 10-K filing). Per-filing summary for E&P group. 3,446 rows. Formerly `instrument_index.csv` in v1.

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK | never null |
| `company` | str | — | Company name | blank if unavailable |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | never null |
| `accession_number` | str | — | SEC accession number | never null |
| `uses_derivatives` | str | — | `True` / `False` | never null |
| `instrument_count` | int | count | Total instruments extracted | 0 if none |
| `total_oil_instruments` | int | count | Oil instrument count | 0 if none |
| `total_gas_instruments` | int | count | Gas instrument count | 0 if none |
| `total_ngl_instruments` | int | count | NGL instrument count | 0 if none |
| `hedging_confidence` | str | — | `high` / `medium` / `low` / `none` | never null |
| `has_production_data` | str | — | `True` / `False` | never null |

**hedging_confidence note:** Assigned by Stage 1 keyword scanner before AI extraction. Measures
clarity of source disclosure, not extraction quality. 798 of 799 "low" filings are confirmed
non-hedgers. See `DATA_QUALITY.md §8`.

---

## 6. data/ep/filing_index.csv

Grain: one row per parsed section. E&P parse superset — every 10-K from which an Item 7A section was parsed, including non-hedgers. 4,672 rows. This file is E&P-specific; there is no equivalent for airline or refiner groups.

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK | never null |
| `company` | str | — | Company name | blank if unavailable |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | never null |
| `accession_number` | str | — | SEC accession number | never null |
| `section_path` | str | — | Relative path to parsed section text file | never null |
| `hedging_score` | int | — | Keyword relevance score (0–42+); higher = more likely hedger | 0 if no keywords |
| `hedging_confidence` | str | — | `high` / `medium` / `low` / `none` | never null |
| `selected_for_extraction` | bool | — | True if included in Stage 2 AI extraction | never null |

---

## 7. data/ep/verification_log.csv

Grain: one row per E&P extraction file. 3,446 rows. No header row — use the column names below when loading.

| Position | Column | Type | Allowed values / notes |
|---|---|---|---|
| 1 | `cik` | str | SEC CIK |
| 2 | `company` | str | Company name |
| 3 | `filing_date` | str YYYY-MM-DD | 10-K filing date |
| 4 | `instrument_count` | int | Instruments after verification |
| 5 | `verdict` | str | `PASS` / `FIXED` / `WARN` / `FAIL` |
| 6 | `issue_count` | int | Number of issues found |
| 7 | `issues_summary` | str | Pipe-separated issue descriptions; empty if none |
| 8 | `verified_at` | str | Timestamp of verification run |

```python
import pandas as pd
vlog = pd.read_csv("data/ep/verification_log.csv",
                   names=['cik','company','filing_date','instrument_count',
                          'verdict','issue_count','issues_summary','verified_at'])
```

---

## 8. data/airline/hedge_ratios.csv

Grain: one row per `(cik, fiscal_year)`. Airline firm-year panel. 601 rows. **Version 2 schema: see `data/airline/docs_v2/CODEBOOK.md` (the columns below describe version 1).**

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK | never null |
| `company` | str | — | Company name | blank if unavailable |
| `ticker` | str | — | Exchange ticker | blank if unavailable |
| `fiscal_year` | int | year | Fiscal year of the 10-K filing | never null |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | never null |
| `fuel_consumption_gallons_mm` | float | M gallons | Actual fuel consumed in fiscal year | null if not disclosed |
| `expected_gallons_millions` | float | M gallons | Expected fuel consumption for FY+1 (denominator) | null if not disclosed |
| `hedge_ratio_next` | float | fraction | Forward coverage pct for FY+1 (primary ratio) | null if not_applicable or status ∈ {never, stopped, pass_through} |
| `hedge_ratio_current` | float | fraction | Forward coverage pct for current fiscal year | null if not disclosed |
| `headline_coverage` | str | — | `next` / `current` / `none` — which year drove the headline ratio | never null |
| `net_fair_value_mm` | float | USD M | Fair value of hedge portfolio ($ millions) | null if not disclosed |
| `has_program` | str | — | `True` / `False` | never null |
| `stopped_hedging` | str | — | `True` / `False` | never null |
| `stopped_hedging_year` | float | year | Fiscal year of permanent exit (if applicable) | null if stopped_hedging = False |
| `owns_refinery` | str | — | `True` / `False`; True only for Delta (Monroe Energy) | never null |
| `status` | str | — | `hedger` / `no_position` / `stopped` / `never` / `pass_through` / `not_applicable` | never null |
| `warnings` | str | — | Pipe-separated quality flags; empty string = no issues | empty string |

**hedge_ratio_next units:** Decimal fraction (0.60 = 60% of expected consumption hedged).
Values above 1.0 indicate over-hedging relative to forecast consumption (uncommon but not impossible).
Values derived from `volume_pct_of_consumption` (preferred) or computed as
`volume_gallons_millions / expected_gallons_millions` (fallback).

---

## 9. data/airline/positions.csv

Grain: one row per `(cik, fiscal_year, coverage_year)`. Full forward coverage schedule as disclosed. 344 rows.

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK | never null |
| `company` | str | — | Company name | blank if unavailable |
| `ticker` | str | — | Exchange ticker | blank if unavailable |
| `fiscal_year` | int | year | Fiscal year of the 10-K disclosure | never null |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | never null |
| `coverage_year` | int | year | Year of fuel consumption the hedge covers | never null |
| `volume_pct_of_consumption` | float | fraction | Coverage as fraction of consumption (preferred if available) | null if not disclosed |
| `volume_gallons_millions` | float | M gallons | Hedged volume (nominal, not pct) | null if not disclosed |
| `expected_gallons_millions` | float | M gallons | Expected consumption for coverage_year (denominator) | null if not disclosed |
| `computed_pct` | float | fraction | `volume_gallons_millions / expected_gallons_millions` (fallback) | null if either component missing |
| `instrument_types` | str | — | Comma-separated instrument types disclosed (e.g., "collar, swap") | blank if not itemized |
| `notes` | str | — | Annotation from extraction (e.g., source table header text) | blank if none |

---

## 10. data/airline/filings.csv

Grain: one row per 10-K filing. Airline per-filing summary. 603 rows. **Version 2 schema: see `data/airline/docs_v2/CODEBOOK.md`.**

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK | never null |
| `company` | str | — | Company name | blank if unavailable |
| `ticker` | str | — | Exchange ticker | blank if unavailable |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | never null |
| `fiscal_year` | int | year | Fiscal year | never null |
| `has_program` | str | — | `True` / `False` | never null |
| `n_coverage_years` | int | count | Number of distinct coverage years disclosed | 0 if no positions |
| `coverage_years_range` | str | — | E.g., `2006-2008` (first to last coverage year) | blank if no positions |
| `stopped_hedging` | str | — | `True` / `False` | never null |
| `owns_refinery` | str | — | `True` / `False` | never null |
| `hedging_confidence` | str | — | `high` / `medium` / `low` — based on disclosure clarity | never null |

---

## 11. data/refiner/hedge_ratios.csv

Grain: one row per `(cik, fiscal_year)`. Refiner firm-year panel. 235 rows.

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK | never null |
| `company` | str | — | Company name | blank if unavailable |
| `ticker` | str | — | Exchange ticker | blank if unavailable |
| `fiscal_year` | int | year | Fiscal year | never null |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | never null |
| `crude_throughput_mmbbl` | float | MMBbl | Annual crude throughput (millions of barrels) | null if not disclosed |
| `throughput_mbbl` | float | MBbl | Annual crude throughput (thousands of barrels) | null if not disclosed |
| `total_long_mbbl` | float | MBbl | Sum of long crude positions | 0.0 if no long positions |
| `total_short_mbbl` | float | MBbl | Sum of short crude positions | 0.0 if no short positions |
| `net_short_mbbl` | float | MBbl | `total_short_mbbl − total_long_mbbl` (positive = net short) | null if throughput unknown |
| `total_abs_mbbl` | float | MBbl | `total_long_mbbl + total_short_mbbl` | 0.0 if no positions |
| `net_short_ratio` | float | fraction | `net_short_mbbl / throughput_mbbl` (primary ratio) | null if throughput unknown or no positions |
| `abs_ratio` | float | fraction | `total_abs_mbbl / throughput_mbbl` (secondary ratio) | null if throughput unknown |
| `primary_objective` | str | — | `cash_flow` / `economic` / `mixed` / blank | blank if not disclosed |
| `has_cash_flow_hedges` | str | — | `True` / `False` | never null |
| `has_economic_hedges` | str | — | `True` / `False` | never null |
| `status` | str | — | `hedger` / `no_position` / `never` / `not_applicable` | never null |
| `warnings` | str | — | Pipe-separated quality flags; empty string = no issues | empty string |

**net_short_ratio sign convention:** Positive values indicate net-short positions (more short crude
than long crude — the typical refiner hedge posture). Negative values indicate net-long positions
(long crude exceeds short crude). A negative net-short ratio does not mean the refiner is
unhedged; it may reflect crack-spread hedging with a long-crude / short-product structure.

**Excluded commodities:** `natural_gas` and `platinum` positions are excluded from the volume
aggregation. Natural gas hedges manage feedstock costs, not crude-to-product margin. Platinum is
a catalyst, not a feedstock. All other commodities (crude oil, fuel oil, gasoline, distillate,
blendstocks) are included.

---

## 12. data/refiner/positions.csv

Grain: one row per notional position as extracted (un-aggregated). 696 rows.

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK | never null |
| `company` | str | — | Company name | blank if unavailable |
| `ticker` | str | — | Exchange ticker | blank if unavailable |
| `fiscal_year` | int | year | Fiscal year | never null |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | never null |
| `maturity_year` | int | year | Year the position matures or expires | null if not disclosed |
| `commodity` | str | — | crude_oil / gasoline / distillate / fuel_oil / blendstock / natural_gas / platinum / other | never null |
| `instrument` | str | — | swap / collar / option / forward / futures / other | never null |
| `direction` | str | — | long / short / unknown | never null |
| `designation` | str | — | `cash_flow` / `economic` / `unknown` | never null |
| `volume_mbbl` | float | MBbl | Position volume (thousands of barrels) | null if not extractable |
| `volume_unit` | str | — | Unit as reported in source; `mbbl` if converted | never null |
| `volume_raw` | float | — | Volume in original units before conversion | null if not applicable |
| `notes` | str | — | Annotation from extraction | blank if none |

---

## 13. data/refiner/filings.csv

Grain: one row per 10-K filing. Refiner per-filing summary. 235 rows.

| Column | Type | Units | Allowed values / notes | Missing |
|---|---|---|---|---|
| `cik` | str | — | SEC CIK | never null |
| `company` | str | — | Company name | blank if unavailable |
| `ticker` | str | — | Exchange ticker | blank if unavailable |
| `filing_date` | str | YYYY-MM-DD | 10-K filing date | never null |
| `fiscal_year` | int | year | Fiscal year | never null |
| `has_program` | str | — | `True` / `False` | never null |
| `n_positions` | int | count | Number of notional positions extracted | 0 if none |
| `commodities` | str | — | Comma-separated list of commodities present | blank if no positions |
| `has_cash_flow_hedges` | str | — | `True` / `False` | never null |
| `has_economic_hedges` | str | — | `True` / `False` | never null |
| `hedging_confidence` | str | — | `high` / `medium` / `low` — based on disclosure clarity | never null |
