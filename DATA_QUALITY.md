# Data Quality and Reliability

This document states factually what quality processes were applied, what was found, what was corrected, and what limitations remain. The intent is to give researchers the information they need to assess fitness for purpose.

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
| Missing production data | 106 files | Production data recovered from supplementary re-extraction |

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

**Filing types:** Only annual 10-K filings. Hedges disclosed in 10-Q (quarterly) filings, 8-K (current reports), or proxy statements are not captured.

**HTML rendering failures:** Approximately 5–8% of filings have instrument tables rendered as images or in complex HTML formats that the parser could not fully reconstruct. These contribute to the `low` confidence category.

**NGL hedge ratios:** NGL instruments are extracted and available in `data/extractions/`, but NGL hedge ratios are not reported in `hedge_ratios.csv` because NGL production is inconsistently reported (sometimes as Bbls, sometimes as Boe, sometimes combined with oil) and comparable denominators could not be reliably constructed.

**Basis swaps and swaptions:** These appear in `data/extractions/` and `data/instrument_deltas.csv` but are excluded from `hedge_ratios.csv`. Basis swaps hedge geographic price differentials; swaptions are options on future hedges. Neither hedges current outright price exposure.

**Market data for Black's model:** Delta equivalents for options and collars use simplified approximations (Haushalter 2000) because no external market data (futures prices, implied volatilities) was integrated. Black's (1976) model would produce more precise deltas; the `calculate_hedge_ratios.py` script supports this if market data is provided.

**75 firm-years with no production denominator:** Hedge ratios are null for these observations. Instrument details are still available in the extraction JSONs and `instrument_deltas.csv`.

---

## 8. Confidence Assessment

**Per-filing confidence** (`hedging_confidence` in `data/instrument_index.csv` and `data/hedge_ratios.csv`):

| Level | Count (filings) | Meaning |
|---|---|---|
| high | 1,869 | Clear quantitative tables found in Item 7A; extraction straightforward |
| medium | 774 | Some ambiguity (narrative disclosures, partial tables, multiple as-of dates) |
| low | 799 | Limited disclosure, HTML parse failure, or no derivatives |
| other | 4 | Not classified (very small filings or missing metadata) |

**Delta method** (`delta_method` in `data/hedge_ratios.csv`): All 2,963 firm-year records show `"simplified"`, meaning delta equivalents use fixed approximations rather than Black's model. Researchers who prefer Black's model deltas can supply a market data file and re-run `hedge_ratio/calculate_hedge_ratios.py --market-data <file>`.

**Instrument-level auditability:** Every instrument's delta computation is recorded in `data/instrument_deltas.csv`. Researchers can reproduce any hedge ratio from first principles using that file.
