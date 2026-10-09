# Version history: what changed between version 1 and version 2

*Data work by The AI Plus Research Lab at the University of Northern Iowa (attribution required; CC BY-NC 4.0, see ../LICENSE.txt and ../ATTRIBUTION.md).*

## Version 1 (May 2026)
421 annual reports from 26 airline and cargo registrants, fiscal years 1999-2025. Built with a prompt-based extraction and hand-entered data for some carriers. **No systematic verification** was done. Kept for comparison in `previous_version_v1/`.

## Version 2 (October 2026, this dataset)
603 reports from 51 registrants, fiscal years 1999-2026. Every report re-read from the SEC source, every value stored with its quote, tables rebuilt with fixed rules, 28 automated checks, hand audit and independent second reading. Version 1 values were not reused.

### Problems found in version 1 and how version 2 handles them
| # | Problem in version 1 | Evidence | Version 2 |
|---|---|---|---|
| 1 | **26 registrants missing**: Atlas Air Worldwide, ExpressJet, Pinnacle, MAIR/Mesaba, Great Lakes, Kitty Hawk, World Air, ATA, FLYi, Vanguard, Midway, Airborne, AirNet, Evergreen, Alpine Air, Fine Air, Air T, Harbor Diversified (Air Wisconsin), Baltia, Gulfstream International, Global Aviation, Global Crossing Airlines, Hawaiian Airlines Inc (2000-02), TWA (FY1999), America West Holdings, Atlas Air Inc | EDGAR industry-code lists | added: 200 reports |
| 2 | **Reports filed on form 10-K405 (2000-2002) were skipped**, plus two 2025 transition reports | EDGAR filing lists | all 616 original annual reports read |
| 3 | **Double counting**: one combined report counted under two registrant numbers (United 16 times, America West / US Airways 2) | same accession number | carried once; 18 rows listed in `data/original_rows_not_carried_over.csv` |
| 4 | **"Stopped" label stuck** to every later year (129 reports); 9 of those actually showed active hedging (for example Alaska FY2023 with 324 million gallons hedged) | the reports | status derived year by year |
| 5 | **"No hedging" assumed** without a statement (31 reports; SkyWest FY2010 never uses the word "hedge") | full-text search | `not_discussed`; inferred absences labelled |
| 6 | **Delta** from FY2018 coded "no programme" although the report prints a 219-million-gallon Monroe refinery position; Delta FY2013-14 ratios (105%, 239%) were refinery positions divided by airline consumption | answer key, report | `refinery_hedges_only`; position kept, no airline ratio |
| 7 | **Mixed bases** in ratios (stated %, computed %, current versus next year); a report coded both "stopped" and "2% hedged" (Allegiant FY2007); no company number or accession inside the JSON files; model and date not recorded | v1 files | one definition, accession on every row, evidence and agent output stored |
| 8 | **Non-airline filings**: CIK 899394 was a drug developer until 2019 | report text | 13 filings excluded |

### Problems found by version 2's own audits, and fixed
(See `validation/manual_audit_sample.md` and `validation/second_reader.md`.)
* part-year figures used as a full-year headline (16 reports); explicit "no futures or swaps" statements labelled inferred (17); "we use options" without any position counted as hedging (7); quarterly average mixing Alaska and Hawaiian rows; "stopped this year" used for stops in earlier years (5 reports); two headlines with the wrong scope (America West FY2005 68% became 20%; AMTRAN FY2001); six missing full-year headlines filled; Virgin America's fixed forward price agreements added; Alaska FY2004 quarterly gallons rejected; consumption filled for 70 more reports.

### Numbers, version 1 versus version 2 (the 403 reports both hold)
* 276 reports changed status label (mostly sticky "stopped", assumed "no hedging", and Delta).
* Of the 150 that version 1 labelled as hedging with a ratio, 148 still show hedging; 34 that version 1 labelled otherwise are hedging.
* For the 122 reports where both versions have a next-year percentage, 113 agree within 3 percentage points.
* `data/corrections_vs_original.csv` has the row-by-row comparison.

### What stayed the same
The idea (coverage of next year's fuel by hedges), the airline universe starting point, the two hand-made answer keys (Southwest FY2013, Delta FY2018) and the principle that interest-rate and currency contracts are excluded.
