# Codebook: every column in every table

*Data work by The AI Plus Research Lab at the University of Northern Iowa (attribution required; CC BY-NC 4.0, see ../LICENSE.txt and ../ATTRIBUTION.md).*

Conventions: **text keys** (`cik`, `accession`) must be read as text, not numbers. Percentages are written as printed (43 means 43%).
Gallons are **millions of gallons** unless a column name says otherwise. Money is **$ millions**. Dates are `YYYY-MM-DD`.
Empty = not stated in the report (or not applicable), never "zero". Zero is a real zero.

Tables are in `data/`. The headline table is `hedge_ratios.csv`.

---
## 1. `hedge_ratios.csv` (601 rows) and `filings.csv` (603 rows)

Same columns. One row = one annual report. `filings.csv` has every report; `hedge_ratios.csv` keeps one report per company and fiscal year (the later one if a company filed two for the same year; see `is_primary_for_firm_year`).

### Identity
| Column | Meaning |
|---|---|
| `cik` | SEC registrant number (text). Where a parent and a subsidiary file one combined report, the report is stored once under the parent (for example United Airlines Holdings, CIK 100517). |
| `company_edgar` | Company name as written in the filing at that date (names change with mergers). |
| `carrier_type` | `passenger_scheduled`, `passenger_startup`, `passenger_charter`, `passenger_charter_scheduled`, `passenger_cargo_charter`, `regional_pass_through` (regional airline whose partners pay for fuel), `cargo`, `parcel` (FedEx, UPS). From `data/universe_register.csv`. |
| `fiscal_year` | Year in which the fiscal year **ends** (FY ended 31 March 2006 = 2006). Taken from the filing's own year-end date when it prints one. |
| `fiscal_year_end` | Date the fiscal year ended. |
| `filing_date` | Date the report was filed with the SEC. |
| `form` | `10-K`, `10-K405` (the form used until 2002) or `10-KT` (transition period). |
| `accession` | SEC accession number: the **unique key** of a report. Look it up at sec.gov/Archives/edgar/data/<cik>/<accession>.txt. |
| `fiscal_year_basis` | How the fiscal year was derived: `header_period`, `fye_code`, `filing_date_rule`, or `text_overrides_*` (the report text states a different year). |

### What the report says about fuel hedging
| Column | Meaning |
|---|---|
| `program_status` | The raw reading: `active_hedging` (fuel hedges outstanding at year-end), `program_no_open_positions` (a programme exists, nothing outstanding), `no_hedging` (the report says it does not hedge fuel, or enters no futures/swaps), `stopped_hedging` (programme ended), `refinery_margin_only` (derivatives relate only to a refinery the company owns), `not_discussed` (the report says nothing about fuel hedging). |
| `status` | **Analysis label**, derived from `program_status` plus history and the headline. Use this for analysis. Values in section 5. |
| `no_hedging_basis` | For `program_status = no_hedging`: `explicit_statement` (a sentence about fuel hedging, futures or swaps) or `inferred_from_pass_through_or_absence` (the report says partners pay for fuel or the company has no derivatives, but never says it does not hedge fuel). |
| `confidence` | The reading agent's own confidence: `high`, `medium`, `low`. `low` is also set by hand where the text conflicts. |
| `hedged_in_an_earlier_filing` | True if an earlier report of the same company in this dataset showed hedging. Drives `no_hedging_after_hedging`. |
| `weak_active_claim` | True when the report says the carrier "uses options / swaps" but quotes no position and no (or a zero) fair value. Such reports are labelled `no_open_positions`. |
| `stopped_hedging_year` | Year in which the report says hedging stopped, if stated. |
| `fuel_pass_through` | True if the report says partner airlines pay for fuel (capacity purchase agreements). `pass_through_carrier` is true for these and for regional carriers. |
| `owns_refinery` | True if the report says the company owns or operates a refinery (Delta, from 2012). |
| `hedge_accounting` | `designated`, `not_designated` or `mixed`, if stated. |

### The headline: share of next year's fuel that is hedged
| Column | Meaning |
|---|---|
| **`hedge_pct_next`** | Percent of **next fiscal year's** expected fuel consumption that is hedged. Only from a clean **annual** figure (or the average of four quarterly figures). Empty if the report prints only a quarter or part-year, or several conflicting figures. |
| `hedge_pct_next_basis` | How it was chosen: `annual_row`; `mean_of_4_quarters` (unweighted average, an approximation); `adjudicated_*` (corrected or filled by hand from the second reading, see `adjudications.csv`); `ambiguous_multiple_rows` (several instruments or figures that cannot be combined: min and max given instead); `partial_period_only` (only a quarter or some months printed: no headline). |
| `hedge_pct_next_asof` | Date basis of the figure: `year_end` (as of the fiscal year-end), `after_year_end` (as of a later date such as "January 17"), `mixed_or_unspecified` (the quote carries no date). Southwest FY2013 is 43% at year-end but about 20% in the January table: the date matters. |
| `hedge_pct_next_rows`, `hedge_pct_next_min`, `hedge_pct_next_max` | Number of next-year percentage rows found, and their smallest and largest value. |
| `hedged_gallons_next_mm` | Million gallons hedged for next year, only when one row states it. |
| `expected_gallons_mm`, `gallons_ratio_next` | Expected consumption for next year if the report states it, and hedged gallons divided by it. Sparse. |
| `net_fair_value_fuel_mm` | Net fair value of fuel derivatives at year-end, $ million (positive = net asset). Fuel only: interest-rate and currency contracts are excluded. |

### Gallons consumed in the fiscal year (the denominator)
| Column | Meaning |
|---|---|
| `fuel_gallons_consumed_mm` | Million gallons consumed. The most inclusive figure two sources agree on (company-wide rather than mainline-only, where both are printed). Empty for 266 reports that do not print it. |
| `fuel_gallons_source` | Which sources agree: `agent+table+reader2_agree`, `agent+reader2_agree`, `table+reader2_agree`, `agent+table_agree`, one source only (`agent_only`, `table_only`, `reader2_only`), `max_of_disagreeing_sources` (all differ: larger shown, flagged), `rejected_outlier`. |
| `fuel_gallons_candidates` | Every value found, for example `agent=2338.0; table=2669.0; reader2=2338.0`. |
| `fuel_gallons_other_source_differs` | True if one of the sources disagrees (often a mainline-only table). |
| `fuel_gallons_table_mm`, `fuel_gallons_table_quote`, `fuel_gallons_table_check` | Value read from the operating-statistics table by a parser, the table row as printed, and whether the prior-year cell matched the prior-year report (`confirmed`). |
| `fuel_gallons_reader2_mm`, `fuel_gallons_consumed_agent_mm` | The same figure from the independent second reader, and from the first reader. |
| `fuel_gallons_issue`, `fuel_gallons_caveat`, `fuel_gallons_rescaled` | Notes: a value rejected as an outlier; "pass_through_carrier: covers only fuel the carrier buys itself"; true when the filing printed thousands or units and the value was converted to millions. |

### Audit trail
| Column | Meaning |
|---|---|
| `is_primary_for_firm_year` | True for the report kept in `hedge_ratios.csv`. |
| `adjudication` | Text of any manual correction applied to this row (details in `adjudications.csv`). |
| `n_positions`, `n_positions_verified` | Rows in `positions.csv` for this report, and how many have a verified quote (all). |
| `program_quote_verified` | The sentence behind `program_status` was found verbatim in the evidence. |
| `identity_confirmed` (excluded table only) | The agent confirmed the company named in the text. |
| `in_original`, `original_status` | Whether version 1 (previous_version_v1) held this report and what status it gave. |
| `n_issues`, `issues`, `summary` | The reading agent's one-line summary and its own caveats, in plain English. Read `issues` before relying on a single number. |

---
## 2. `positions.csv` (692 rows)
One row per hedge coverage period disclosed in a report (a year, a quarter, an instrument).
`cik`, `accession`, `filing_date`, `fiscal_year` identify the report. `coverage_year` is the calendar (or fiscal) year the hedge **covers**, not when it was entered into. `period` is the label as printed (`2014`, `Q1`, `Full Year`, `through February 2009`...).
`pct` percent of expected fuel consumption hedged; `gallons_mm` million gallons hedged; `commodities` and `instruments` as printed (WTI crude, heating oil, collars, call options...); `pct_note` the reading agent's remark (for example "as of January 17, 2014"); `verified` true; **`quote`** the sentence or table row copied from the filing.
Several rows can share a coverage year (quarters, different instruments, price bands). No contract prices are collected.

## 3. Other tables
| File | Content |
|---|---|
| `company_summary.csv` | One row per registrant: reports, fiscal years, counts by status, mean next-year hedged % over reports with a headline. |
| `universe_register.csv` | All 96 candidate registrants EDGAR lists in the airline industry codes, with `decision` (include / exclude), the reason, and whether version 1 held them. |
| `corrections_vs_original.csv` | For the 403 reports version 1 also held: old and new status, ratio, gallons and fair value side by side. |
| `original_rows_not_carried_over.csv` | The 18 version-1 rows that were duplicates (one report counted under two registrant numbers). |
| `excluded_filings.csv` | 13 filings of CIK 899394 that are not airline reports (the CIK belonged to a drug developer until 2019). |
| `adjudications.csv` | Every manual correction: report (`accession`), `field`, new `value`, `reason`. 19 reports. |
| `reread_results/final_raw.json` | The first reader's complete output for each report. |
| `reread_results/second_reader_raw.json`, `second_reader_comparison.csv` | The independent second reader's output and its comparison with the dataset. |
| `reread_results/final_verified.csv`, `final_positions.csv` | Quote and number verification results. |
| `support/` | EDGAR filing lists used to define the universe. |
| `previous_version_v1/` (outside `data/`) | The first version of the dataset, for comparison only. Do not use it for analysis. |

## 4. Identifiers
* `cik` + `accession` identify a report. Two registrants can share one accession (a combined parent and subsidiary report); the report appears once.
* Fiscal years: some airlines do not end their year in December (Mesa September, Frontier and Air T March, FedEx May, Alpine October). "Next year" is the next fiscal year.

## 5. `status` values
| status | meaning | reports |
|---|---|---|
| `hedger` | open fuel hedges **and** a next-year figure (percentage or gallons) | 164 |
| `hedger_no_next_year_figure` | open fuel hedges, but no clean annual next-year figure | 52 |
| `no_open_positions` | a programme exists, nothing outstanding at year-end | 55 |
| `no_hedging` | explicit statement that the carrier does not hedge, and no earlier report showed hedging | 58 |
| `no_hedging_after_hedging` | explicit "no hedging" after an earlier report showed hedging, or a stop in an earlier year | 111 |
| `no_hedging_inferred` | no explicit fuel statement; absence of exposure is inferred from a pass-through or generic sentence | 69 |
| `stopped_this_year` | the report says the programme was stopped or terminated in that fiscal year | 6 |
| `refinery_hedges_only` | derivatives relate to a refinery the company owns (Delta, FY2018-FY2025) | 8 |
| `not_discussed` | the report says nothing about fuel hedging | 80 |

Do **not** treat `not_discussed` or `no_hedging_inferred` as a measured zero; they mean "no evidence of hedging in the report".
