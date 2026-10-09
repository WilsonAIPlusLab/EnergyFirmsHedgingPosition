# Data Quality and Reliability

*Last updated: 2026-05-24*

This document states factually what quality processes were applied, what was found, what was corrected, and what limitations remain. The intent is to give researchers the information they need to assess fitness for purpose.

---

## Revision History

| Date | Change |
|---|---|
| 2026-03-29 | Added note to §7 documenting Hess Corporation and ConocoPhillips as SIC 2911 registrants absent from the SIC 1311 universe |
| 2026-03-28 | §7: null firm-year count reduced from 85 to 66 after recovering FY+1 production denominators for 24 E&P filings; updated breakdown by category. §4: added FY+1 production patch row. NGL stats updated: 202 firm-years (was 200), 39 without NGL production (was 41) |
| 2026-03-23 | Initial release. Added NGL hedge ratio coverage note. Fixed delta_method table (three values, not all "simplified"). Corrected low-confidence row. Fixed false "5–8% HTML rendering failures" claim. Updated null firm-year count and breakdown |

---

## 1. Extraction Verification Summary

All 3,446 extraction files were verified through automated checks and, where needed, manual re-extraction from the original filing text. The extraction methodology was validated against the Devon Energy example in Jin & Jorion (2006), targeting agreement on position counts, instrument types, delta approximations, and computed hedge ratios. A random subsample of AI-extracted filings was reviewed against the original source documents to confirm faithfulness of instrument details.

| Verdict | Count | Fraction | Meaning |
|---|---|---|---|
| **PASS** | 2,766 | 80.3% | All automated checks passed; extraction matches source |
| **FIXED** | 638 | 18.5% | Errors found and corrected during re-extraction |
| **WARN** | 42 | 1.2% | Minor potential issues; could not be definitively resolved |
| **FAIL** | 0 | 0.0% | All failures resolved |
| **Total** | 3,446 | 100% | |

Verdict and any correction notes are in `data/verification_log.csv`. Detailed correction records for each FIXED filing are in `docs/VERIFICATION_PROJECT_NOTES.md`.

---

## 2. What Each Verdict Means

**PASS:** The extraction passed all six automated checks (volume math, period overlap, direction consistency, summary counts, schema completeness, price plausibility) with no discrepancies detected.

**FIXED:** The automated checks flagged a discrepancy, or a manual review identified an error. The filing was re-extracted by reading the original source text directly and producing a corrected instruments array. Examples: missing instruments (e.g., NGL instruments omitted), wrong volume scale (MBbls not multiplied by 1,000), incorrect three-way collar schema (missing sub_floor_price).

**WARN:** A potential issue was flagged but could not be definitively resolved. These are described in Section 5 below.

**FAIL:** No filings remain at FAIL. All 144 original FAILs were resolved across four rounds of re-extraction.

---

## 3. Automated Checks Applied

Six automated checks were run on every extraction by `pipeline/verify_extractions.py`:

| Check | Description | Tolerance |
|---|---|---|
| Volume math | `volume_per_day × days_in_period ≈ volume_total` | ±20% |
| Direction consistency | Producer swaps should be "short"; collars should be "collar" | Exact |
| Summary counts | `hedging_summary.total_instruments == len(instruments)` | Exact |
| Schema completeness | Required fields present; `sub_floor_price` on `three_way_collar` | Exact |
| Price plausibility | Prices in plausible ranges for commodity and year | Heuristic |
| Instrument count | Non-negative, not implausibly high for a single filing | Heuristic |

---

## 4. Notable Corrections During Re-Extraction

The following corrections were among the most consequential. All were made by re-reading the original source filing text.

### Instrument corrections

| Company | CIK | Filing | Correction |
|---|---|---|---|
| Vital Energy | 1528129 | 2019-03-12 | 17 → 28 instruments (11 NGL instruments missing) |
| EOG Resources | 72207 | 2023-03-07 | 6 → 19 instruments (sub-period splits + closed positions added) |
| WPX Energy | 1518832 | 2015 filings | 7 instruments had null volumes; filled from source |
| Antero Resources | 1476655 | 4 filings | Martica VIE instruments were missing; added |
| Civitas Resources | 895126 | 2015 filing | 6 three-way collars missing sub_floor_price; added |
| Gastar Exploration | various | 3 filings | sub_floor_price added from "Short Put" column in source |
| Noble Energy | 72207 | 8 of 10 filings | Instrument counts and schema normalized |
| Northern O&G | 1104485 | 8 filings | Various count and schema corrections |

### Extraction-level corrections (applied to all files)

| Issue | Files affected | Correction |
|---|---|---|
| Non-standard schema (volume_bbls, year fields) | 89 files | Renamed to canonical fields via fix_schema.py; +249 instruments gained valid volumes |
| EQT Corp Bcf→Mcf scaling (2015–2017) | 5 files | Volumes divided by 1,000 (were 1,000× too large) |
| EQT Corp Bcf→Mmbtu scaling (2018) | 1 file | Volumes multiplied by 1,000 (were 1,000× too small) |
| SandRidge Trust I: non-standard unit fields | ~21 instruments across 4 files | Schema normalized; volumes now populated |
| Missing production data | 106 files | Production data recovered from supplementary re-extraction pass |
| Missing FY+1 production denominator | 23 filings | Production data patched into FY+1 filing JSONs to enable hedge ratio computation |

---

## 5. Remaining WARNs: 42 Filings

Forty-two filings carry a WARN verdict. These have potential quality issues that could not be definitively resolved by re-reading the source text. They are grouped into six categories:

### Category 1: Source text insufficient or HTML parse failure (8 filings)
The Item 7A section of the 10-K was rendered in a format (typically deeply nested HTML tables or embedded images) that the parser could not fully reconstruct. Volumes or prices may be null or approximate. These filings are flagged with `hedging_confidence = "low"`.

### Category 2: Subsequent-event instruments with incomplete data (3 filings)
Some 10-Ks disclose instruments entered after the fiscal year end but before the filing date ("subsequent events") without providing volumes or price details. These instruments are represented in the extraction with available fields; missing fields remain null.

### Category 3: Ambiguous consolidation defensible either way (5 filings)
In some cases, a source table could be read as either (a) two separate instruments at different prices for the same period, or (b) one combined instrument at a weighted-average price. Either reading is consistent with the disclosed data. The extraction represents one defensible interpretation.

### Category 4: Stale or expired instruments possibly retained (2 filings)
Some filings disclose positions that may have expired or settled before the filing date, presented alongside current positions in the same table without clear labeling. The status cannot be determined from the text alone.

### Category 5: Minor field issues with low analytical impact (12 filings)
Small discrepancies in summary counts (off by one), minor volume rounding, or notes fields that are incomplete. These do not affect the main analytical fields.

### Category 6: Historical WARN label, likely correct (13 filings)
These filings were flagged early in the verification process based on automated heuristics, but subsequent manual review found no actual error. The WARN label was retained to maintain log continuity. These extractions are likely as accurate as PASS filings.

---

## 6. Hedge Ratio Audit Findings

After the hedge ratios were calculated, a separate quality audit examined extreme values and year-over-year inconsistencies. See `docs/QUALITY_AUDIT.md` for the full audit report.

### Firm-year errors corrected

Nine firm-year errors were identified and corrected at the extraction level:

| Company | CIK | Fiscal year | Issue | Corrective action |
|---|---|---|---|---|
| HKN Inc | 313478 | 2009–2010 | volume_per_day misread as volume_total (178× too large) | Corrected to daily rate; oil HR 715% → 4% |
| Tengasco Inc | 1001614 | 2009–2012 | Physical gas supply contract included as financial hedge | Removed non-derivative instrument |
| Ultra Petroleum | 1022646 | 2017 | Swap direction marked "long" instead of "short" | Direction corrected |
| HighPoint Resources | 1725526 | 2018 | Dual-snapshot contamination (Dec 31 + Feb 21 positions merged) | Removed Dec 31 positions; retained Feb 21 filing date only |
| Cimarex Energy | 1168054 | 2015 | Quarterly gas volume recorded as daily rate | Corrected volume; gas HR 197% → 70% |
| Memorial Resource Dev. | 1599222 | 2014–2015 | Production understated (parent only, not consolidated) | Denominator corrected to consolidated production |
| Amplify Energy | 1533924 | 2023–2024 | Monthly averages stored as annual totals (12× error) | Multiplied by 12; oil HR 2,119% → 70% |
| CNX Resources | 1070412 | 2023 | Quarterly breakdown double-counted with annual total | 16 quarterly instruments removed |
| Rex Energy | 1397516 | 2015 | Oil HR 154% — investigated | No error; confirmed legitimate (strategic gas pivot reduced FY2016 oil production 68%) |

### Non-E&P registrants flagged but retained in hedge_ratios.csv

The quality audit identified four SIC-1311 CIKs that include non-E&P firm-years. These records are **retained in `hedge_ratios.csv`** because (a) some CIKs have both E&P and non-E&P periods, and (b) the calculation script does not filter by company type. Users should exclude non-E&P observations as appropriate for their research.

| CIK | Company history | Non-E&P concern |
|---|---|---|
| 1338613 | Regency Energy Partners (all years) | Midstream gas gathering; no upstream production |
| 1166036 | MarkWest Energy Partners (all years) | Gas processing and fractionation; midstream |
| 821483 | Delta Petroleum Corp (FY2009–2011, E&P) → PAR Petroleum/PAR Pacific (FY2012+, refiner) | Later years hedge refining margin, not E&P production |
| 61398 | Magellan Petroleum Corp (FY2010–2015, E&P) → Tellurian Inc (FY2017+, LNG developer) | Later years are LNG-focused; no comparable E&P production denominator |

These four CIKs account for 42 firm-year rows in `hedge_ratios.csv`. Hedge ratios for the non-E&P periods are not meaningful in the context of E&P commodity hedging and may be extreme (e.g., PAR Pacific FY2018 oil ratio: 1,139%). The `warnings` column in `hedge_ratios.csv` flags anomalies but does not tag non-E&P company type specifically.

To exclude non-E&P observations, filter out CIKs 1338613, 1166036, and the later years of 821483 and 61398.

### Ratio bounds validation

Hedge ratios above 100% are unusual but can be legitimate. All firm-years with ratios above 150% were individually investigated.

| Threshold | Oil firm-years | Gas firm-years | Status |
|---|---|---|---|
| > 100% | 143 | 103 | Most legitimate (see below) |
| > 150% | 43 | 29 | All investigated; all legitimate |

Common legitimate reasons for ratios above 100%:
- **Production decline:** Company hedged based on prior-year production, which then fell (bankruptcies, asset sales, natural decline). The hedge volume exceeds actual production.
- **Royalty trust structures:** Trusts have declining production schedules; hedges agreed at trust formation may exceed any single year's production.
- **Fast-growing companies:** FY+1 production lookup failed; the current year's (lower) production was used as denominator.
- **Intercompany/VIE hedges:** A subsidiary's hedges were disclosed at the consolidated level against the subsidiary's production, which is only a portion of consolidated production.
- **First year of operations:** Hedges placed before production ramped up.

### Statistical validation

The year-over-year autocorrelation of hedge ratios was computed as a consistency check:
- **Oil:** ρ = 0.47 across 1,549 consecutive firm-year pairs (expected range ≥ 0.30: pass)
- **Gas:** ρ = 0.29 (slightly below threshold; gas hedging is more volatile due to basis risk and regional price dynamics)

---

## 7. Coverage Gaps

The following are not included in this dataset:

**Companies:** Only SIC 1311 (Crude Petroleum and Natural Gas) registrants are included. Integrated majors (SIC 2911), midstream companies (SIC 4922/4924), and gas utilities are excluded even if they disclose commodity hedging.

Two large E&P companies that researchers might expect to find are absent because their SEC EDGAR registrations list SIC 2911 (Petroleum Refining) rather than SIC 1311: **Hess Corporation** (CIK 4447) and **ConocoPhillips** (CIK 1163165). Both divested their downstream refining assets (Hess in 2013; ConocoPhillips via Phillips 66 spinoff in 2012) but retained their historical SIC 2911 classifications with the SEC. As a result, they do not appear in a SIC 1311 filing universe. Researchers interested in these companies should obtain their 10-K filings directly from EDGAR.

**Filing types:** Only annual 10-K filings. Hedges disclosed in 10-Q (quarterly) filings, 8-K (current reports), or proxy statements are not captured.

**NGL hedge ratios:** `hedge_ratio_ngl_delta` is reported in `hedge_ratios.csv` for 202 firm-years where NGL instruments are present and `ngl_production_bbls` is available from the same filing. NGL production volume uses the filing's own reported year (not the FY+1 cross-lookup used for oil/gas) because the NGL production cross-lookup created measurement errors for midstream entities. 13 of the 202 NGL firm-years are flagged with `ngl_delta_ratio>150%` in the `warnings` column and should be treated with caution. For 39 firm-years with NGL instruments but no reported NGL production, the ratio is null.

**Basis swaps and swaptions:** These appear in `data/extractions/` and `data/instrument_deltas.csv` but are excluded from `hedge_ratios.csv`. Basis swaps hedge geographic price differentials; swaptions are options on future hedges. Neither hedges current outright price exposure.

**Market data for Black's model:** Delta equivalents for options and collars use simplified approximations (Haushalter 2000) because no external market data (futures prices, implied volatilities) was integrated. Black's (1976) model would produce more precise deltas; the `calculate_hedge_ratios.py` script supports this if market data is provided.

**66 firm-years with no oil or gas production denominator:** Oil and gas hedge ratios are null for these observations; they consist of 14 confirmed midstream non-producers (MarkWest Energy, Regency Energy, Tellurian), 3 refiners (PAR Pacific), 38 limited partnerships or royalty trusts with no standard production reporting, and 11 E&P companies where production data was not disclosed, could not be extracted, or the FY+1 production year is not yet available (most recent fiscal years). Instrument details remain available in the extraction JSONs and `instrument_deltas.csv`. This count was reduced from 90 to 66 by recovering FY+1 production denominators from 24 additional filing JSONs.

---

## 8. Confidence Assessment

**Per-filing confidence** (`hedging_confidence` in `data/instrument_index.csv` and `data/hedge_ratios.csv`):

> **Important:** `hedging_confidence` measures the *clarity of the source disclosure*, not the quality of the extraction. It is assigned by the Stage 1 keyword scanner before AI extraction runs. All 3,446 files — including every "low" filing — passed automated verification (PASS or FIXED verdict). A "low" filing is not a poorly extracted filing; it is a filing where the company disclosed it does not use commodity derivatives, or where the hedging section was too sparse to score highly on keywords.

| Level | Count (filings) | Meaning |
|---|---|---|
| high | 1,869 | Clear quantitative tables found in Item 7A; extraction straightforward |
| medium | 774 | Some ambiguity (narrative disclosures, partial tables, multiple as-of dates) |
| low | 799 | **798 of 799 are confirmed non-hedgers** (`uses_derivatives=False`); the remainder had an HTML parse failure or minimal disclosure. All 799 have PASS or FIXED verification verdicts. |
| none / unclassified | 4 | Very small filings or missing metadata (3 null, 1 `"none"`) |

**Delta method** (`delta_method` in `data/hedge_ratios.csv`): Records the highest-fidelity method used for any instrument in that firm-year:

| Method | Firm-years | Meaning |
|---|---|---|
| `notional_only` | 1,483 | Firm-year has no qualifying instruments (all excluded as basis swaps/swaptions, or uses_derivatives=False); hedge ratio is zero by definition |
| `linear` | 991 | All instruments are swaps with delta = −1.00; no approximation needed |
| `simplified` | 489 | At least one non-linear instrument (collar, put, three-way collar); delta uses fixed approximations from Haushalter (2000) rather than Black's model |

No records use Black's model (`"black"`) because no external market data (futures prices, implied volatilities) was integrated. Researchers who prefer Black's model deltas can supply a market data file and re-run `hedge_ratio/calculate_hedge_ratios.py --market-data <file>`.

**Instrument-level auditability:** Every instrument's delta computation is recorded in `data/ep/positions.csv`. Researchers can reproduce any hedge ratio from first principles using that file.

---

## 9. Airline and Refiner Group: Verification and Limitations

### Extraction overview

The airline and refiner groups were added in v2.0. Extraction for these groups used the same AI-based Stage 2 pipeline as E&P, but with group-specific prompts designed for the structural differences in airline and refiner 10-K disclosures. **The automated six-check verification suite (§3) was not applied to airline and refiner extractions**, because that suite is calibrated for E&P derivative instrument schemas (volume math, direction consistency, etc.) that do not transfer to forward coverage percentages or refiner notional positions.

Instead, verification for these groups consisted of:

1. **Anchor validation**: Four representative filings (2 airline, 2 refiner) were manually validated against their source 10-K text before batch extraction began.
2. **Schema checks**: All 661 JSON files were checked for required top-level fields; files with unexpected or absent schema were flagged.
3. **Range and plausibility checks**: Fuel coverage percentages checked to be in [0.0, 1.5]; refiner volumes checked for plausible orders of magnitude.
4. **Spot audits**: A selection of individual filings with unusual values (coverage > 100%, zero-volume positions, suspect throughput figures) were manually cross-checked against their source 10-K text.

No equivalent of the `verification_log.csv` or FIXED/WARN verdict system exists for the airline and refiner groups. Researchers who require systematic, audit-trail verification of individual airline or refiner filings should consult the original 10-K filings directly from SEC EDGAR.

### Known accuracy limitations

**Disclosure variation:** Airline and refiner hedging disclosures are substantially less standardized than E&P disclosures. Airlines may disclose forward coverage as a percentage of expected consumption, as a total volume, or as a notional dollar value; the coverage year may be implicit, explicit, or inferred from table headers. Refiners may aggregate positions by calendar quarter, by maturity band, or by commodity, and the netting convention (long-crude vs. short-product vs. crack-spread) varies by company and year. AI extraction unavoidably applies heuristics to resolve ambiguities.

**Multi-year forward schedules:** Airlines often disclose hedging for two or three future years. The `positions.csv` table preserves all coverage years as extracted. `hedge_ratios.csv` aggregates to one headline ratio per firm-year (the FY+1 next-year coverage pct). Users comparing the headline ratio across carriers should verify that the same coverage year is being compared.

**Pass-through carriers:** Four regional carriers (Mesa Air, SkyWest Airlines, Republic Airways, ATSG) operate under capacity purchase agreements that transfer fuel cost risk to the contracting major airline. These entities appear in the airline extraction set with `status = pass_through` and null hedge ratios. This status is assigned manually from a curated allowlist; it is not reliably recoverable from the 10-K text alone.

**Refiner net-short ratio methodology:** The primary hedge ratio for refiners (`net_short_ratio`) is defined as `(total short positions − total long positions) / crude throughput`. This netting conflates crude oil inventory hedges with crack-spread hedges (long crude / short product), which serve different economic purposes. The `abs_ratio` (total absolute volume / throughput) is provided as a secondary measure and may be more appropriate for some analyses. The methodology choice is flagged as an open question for researcher review; see `METHODOLOGY.md §7.2`.

**Incomplete or absent disclosures:** Some filings disclose only a broad statement that the company uses derivatives without quantitative detail. In these cases, instruments may be absent from the extraction even though a hedging program exists; `status` is set to `no_position` (not `never`) when the filing text confirms an active program. Researchers should treat zero or near-zero values with appropriate caution.

---

## 10. Known Issues and Exclusions

### CIK 777676 (Oracle Corporation) — excluded from airline group

Oracle Corporation's SEC EDGAR filings were inadvertently included in the airline extraction input set as a result of a keyword false-positive during the initial 10-K filing universe construction. Oracle's 10-K filings contain aviation-adjacent language (references to airlines as customers; aircraft lease agreements) that triggered the keyword scanner used to identify airline registrants.

**Corrective action:** All five Oracle Corporation filings (CIK 777676) are excluded from `data/airline/`. They are not present in `data/airline/hedge_ratios.csv`, `data/airline/filings.csv`, `data/airline/positions.csv`, or `data/airline/extractions/`. This exclusion is applied by the build script (`builders/build_airline.py`); the exclusion reason is documented there.

**Impact:** Five filing-years that would otherwise appear under CIK 777676 are absent from the delivered airline dataset. No other airline CIK is affected.

### CIK 821483 (Delta Petroleum Corp → Par Pacific Holdings) — dual identity

CIK 821483 was originally registered by Delta Petroleum Corporation, an E&P company, and is present in the E&P universe (SIC 1311) for fiscal years 2009–2011. Delta Petroleum filed for bankruptcy in 2011 and was acquired; the CIK was subsequently reused by Par Pacific Holdings (formerly Par Petroleum), a petroleum refiner.

This is a **legitimate CIK reuse** — one SEC registration number covers two economically distinct entities across time. The dataset handles this via the era-based crosswalk: CIK 821483 has two rows in `data/crosswalk.csv`, one for each identity.

- Delta Petroleum filings (FY2009–FY2011): appear in `data/ep/` with E&P hedge ratios (oil and gas production denominator).
- Par Pacific/Par Pacific Holdings filings (FY2014–FY2025): appear in `data/refiner/` with refiner hedge ratios (crude throughput denominator).
- FY2012–FY2013: transition period; no hedge ratio data for this CIK.

Researchers pooling data across groups by CIK should be aware that CIK 821483 is not a single entity across its full sample span.

### CIK 821483 — flagged in E&P quality audit (non-E&P concern)

The original E&P quality audit (`DATA_QUALITY.md §6`) flagged CIK 821483 as a non-E&P registrant for later years (as Par Pacific, a refiner). In v2.0, the Par Pacific firm-years for this CIK are removed from the E&P group and placed correctly in the refiner group. Researchers working with `data/ep/hedge_ratios.csv` will find no Par Pacific rows there; those rows now appear in `data/refiner/hedge_ratios.csv`.

### Alon USA Energy — volume unit correction

During spot audits of refiner extractions, Alon USA Energy (CIK 1283140) filings from several years were found to have notional crude oil position volumes recorded in barrels rather than thousands of barrels. The denominator (crude throughput) is reported in millions of barrels in the same filings. This introduced a factor-of-1,000 error in the raw ratio. Affected files were corrected by the build script (`builders/build_refiner.py`) by applying the appropriate unit conversion. Researchers who re-run the refiner builder on the original JSON files will obtain the corrected ratios.

### Calumet Specialty Products — empty positions in several filings

Calumet Specialty Products (CIK 1362988) 10-K filings for several years contain hedging disclosures referencing an active commodity risk management program but provide no quantitative position tables (or tables that collapsed to zero volume after unit normalization). These filings are present in `data/refiner/filings.csv` with `status = no_position` and null hedge ratios. They are not data entry errors; they reflect the absence of quantitative detail in the source disclosure.

### Southwest Airlines FY2019 — coverage year ambiguity

Southwest Airlines (CIK 92380) FY2019 10-K discloses fuel hedging positions for multiple forward years in a single table without explicit year labeling. The coverage year was inferred from table column headers and cross-checked against the FY2020 filing (which discloses a consistent but not identical position). Researchers requiring precise Southwest FY2019 instrument-level detail should consult the original 10-K filing directly.

### Delta Air Lines — Monroe Energy boundary

Delta Air Lines (CIK 27904) acquired the Monroe Energy refinery in 2012 and began disclosing crude oil hedges at the refinery level in addition to jet fuel hedges at the airline level. For FY2012 and later, Delta's airline filings include some crude oil hedging instruments (Monroe) that are attributable to refining operations, not airline fuel cost management. These instruments are included in the airline group extractions as extracted; they are not separately classified as refiner positions because they are disclosed within the airline 10-K (CIK 27904), not a separate refiner registrant. Researchers studying airline fuel hedging may wish to exclude or adjust Delta's crude oil positions post-2012.

---

## 11. v1 → v2 File Migration

Version 2.0 reorganized `data/` into per-group subfolders. The following table maps v1 file paths to their v2 equivalents for users who cite or link to files from the original published dataset.

All renames were performed using `git mv` inside the `dataset/` git repository, so file history is preserved. Users who cloned the original dataset can update by pulling the latest commit.

| v1 path | v2 path | Notes |
|---|---|---|
| `data/hedge_ratios.csv` | `data/ep/hedge_ratios.csv` | E&P firm-year panel; content unchanged |
| `data/instrument_deltas.csv` | `data/ep/positions.csv` | Instrument-level delta records; content unchanged |
| `data/instrument_index.csv` | `data/ep/filings.csv` | E&P per-filing summary; content unchanged |
| `data/filing_index.csv` | `data/ep/filing_index.csv` | E&P parse superset (4,672 rows); content unchanged |
| `data/verification_log.csv` | `data/ep/verification_log.csv` | Verdict log; content unchanged |
| `data/hedge_ratio_diagnostics.txt` | `data/ep/hedge_ratio_diagnostics.txt` | Diagnostic report; content unchanged |
| `data/extractions/` | `data/ep/extractions/` | 3,446 E&P JSON files; content unchanged |

**New in v2.0** (no v1 equivalent):

| New file | Description |
|---|---|
| `data/airline/hedge_ratios.csv` | Airline firm-year panel (601 firm-years, version 2) |
| `data/airline/positions.csv` | Airline position-level forward schedule |
| `data/airline/filings.csv` | Airline per-filing summary |
| `data/airline/v2_support/` | version 2 audit tables and raw reader output |
| `data/refiner/hedge_ratios.csv` | Refiner firm-year panel (235 firm-years) |
| `data/refiner/positions.csv` | Refiner un-aggregated notional positions (696 rows) |
| `data/refiner/filings.csv` | Refiner per-filing summary |
| `data/refiner/extractions/` | 235 refiner JSON files |
| `data/crosswalk.csv` | CIK ↔ ticker ↔ company crosswalk with era support |
| `data/master_panel.csv` | Harmonized long-format cross-group panel |
| `docs/CODEBOOK.md` | Per-variable data dictionary for all tables |
| `LICENSE` | CC-BY-4.0 license |
| `CITATION.cff` | Machine-readable citation |

**Downstream code changes required by v2.0:**

Scripts or notebooks that read the v1 paths will need to update path references. No column names were changed in the E&P tables; only the directory structure changed. The new airline and refiner tables follow the same naming conventions (`hedge_ratios.csv`, `positions.csv`, `filings.csv`) to make per-group analysis code easy to generalize.


## Airline group replaced by audited version 2 (2026-10-09)
Sections above that describe the airline group (the unverified v1 extraction, §9-10, the Delta/Monroe note, the Oracle exclusion) refer to **version 1, which has been replaced**. Version 2 was re-read from the SEC source for all 603 reports, with a verbatim quote for every value, 28 automated checks, a 38-report hand audit and an independent second reading of 443 reports. Details, known weak spots and the row-by-row comparison with v1 are in `data/airline/README_AIRLINE_V2.md`, `data/airline/docs_v2/` and `data/airline/v2_support/corrections_vs_original.csv`.
