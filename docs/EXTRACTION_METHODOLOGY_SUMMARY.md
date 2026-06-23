# Extraction Methodology Summary
# Oil & Gas Hedging Dataset — Lessons for Pipeline Extension to Airlines and Refiners

*Last updated: 2026-05-22*

This document summarizes the full experience building the SIC 1311 (E&P) hedging dataset:
filing acquisition, AI extraction, validation, and hedge ratio calculation. It is written as
institutional knowledge to guide extending the same pipeline to airlines (SIC 4512) and
refiners (SIC 2911).

---

## 1. Downloading 10-K Filings

### Source and Query

All filings were obtained from the SEC EDGAR full-text search API:

```
https://efts.sec.gov/LATEST/search-index?q=%22%22&dateRange=custom&startdt=2010-01-01
    &enddt=2025-12-31&forms=10-K&category=form-type
```

The universe was defined by SIC code filter (`SIC=1311`), not by a pre-selected list of firms.
This yielded 4,761 raw `.txt` files covering 453 unique CIKs filing from 2005 through 2025.

### Filing format

EDGAR delivers 10-Ks in SGML format with embedded HTML inside `<DOCUMENT>` tags. Each file
also contains XBRL instance documents and PEM certificate blocks interleaved. The deterministic
parser (`pipeline/stage1_parser.py`) must:

1. Strip the SGML wrapper to isolate the primary document `<DOCUMENT>` block
2. Remove XBRL, PEM, and binary embedded attachments before HTML parsing
3. Use BeautifulSoup or regex to strip HTML tags from the human-readable document

**Lesson for airlines/refiners:** The SGML format is identical across SIC codes. The same
Stage 1 parser works unchanged — only the SIC filter in the EDGAR query changes.

### Section identification

After HTML stripping, sections are found by keyword matching on headers:

| Target section | Keywords matched |
|---|---|
| Item 7A (Market Risk) | `item 7a`, `quantitative and qualitative`, `market risk` |
| Item 8 Notes | `notes to consolidated financial statements`, `note [0-9]` |
| Item 7 (MD&A) | `item 7`, `management.*discussion`, `results of operations` |
| Oil & Gas Supplement | `supplemental.*oil`, `oil and gas.*supplement` |

238 keyword patterns across 11 categories were used to classify each filing's hedging
confidence: `high` (40%), `medium` (17%), `low` (17%), `none` (26%).

**Critical lesson:** 473 filings (10%) had an empty Item 7A but complete hedging tables in
Item 7 (MD&A). These were missed entirely in the first extraction pass. The fix was adding
`item7.txt` as a secondary context source with a 40K character budget. For any new industry,
assume that the section structure is irregular — always include MD&A as a fallback context.

---

## 2. AI Extraction of Hedging Positions

### Model and API

- **High-confidence batch (1,869 filings):** `claude-sonnet-4-5` via the Anthropic Batches API
  (8 chunks of 500 requests each). Cost-efficient for large runs.
- **Failed reruns, medium/low confidence (1,574 filings):** `claude-opus-4-6` via the
  interactive API. Used for filings where Sonnet failed or where more complex reasoning was
  needed.
- **Item 7 merge, warning verification, quality audit:** `claude-sonnet-4-6` agents running
  in parallel.

### Context assembly

Three source sections are assembled in priority order up to a 120K character budget:

1. `item7a.txt` — up to 50K chars (primary: quantitative risk disclosures, structured tables)
2. `item7.txt` — up to 40K chars (secondary: MD&A narrative, often contains hedging tables
   for smaller firms)
3. `item8_notes.txt` — remaining budget (tertiary: detailed footnotes)

### Extraction schema (per instrument)

```json
{
  "instrument_type":  "swap | collar | three_way_collar | put_option | call_option |
                       fixed_price_contract | basis_swap | swaption | forward | futures",
  "commodity":        "crude_oil | natural_gas | ngl | mixed | unknown",
  "direction":        "short | long | collar | unknown",
  "volume_per_day":   1100,
  "volume_total":     null,
  "volume_unit":      "Bbls | Mcf | Mmbtu | Boe",
  "fixed_price":      null,
  "floor_price":      55.0,
  "ceiling_price":    64.05,
  "sub_floor_price":  null,
  "price_unit":       "$/Bbl | $/Mcf | $/Mmbtu",
  "period_start":     "2015-07",
  "period_end":       "2015-12",
  "counterparty":     "Cargill, Incorporated",
  "fair_value_mm":    -0.109974,
  "notes":            "..."
}
```

Production data (oil/gas/NGL annual volumes, proved reserves, PV10) is extracted in the
same API call.

### Known extraction failure modes

| Failure | Cause | Fix |
|---|---|---|
| JSON truncated mid-array | `max_tokens=4096` too small for firms with 15–30+ instruments | Raise to 8192; re-run on Opus 4.6 |
| Instruments in Item 7 missed | `item7.txt` not included in context | Add as secondary source |
| Unit confusion (MBbls stored as Bbls) | Table headers say "(in thousands)" — AI ignores the multiplier | Post-processing normalization pass |
| Gas production off by 1,000x | Bcf stored as Mcf | Same normalization pass |
| Financial derivatives included | AI extracts interest rate swaps, FX forwards, etc. | Warning verification + source check to remove |
| Expired contracts included | AI extracts settled/matured positions | Extraction instruction: "as-of date positions only" |

**Lesson for airlines/refiners:** Instrument vocabulary and table structure differ by industry.
Do not reuse E&P extraction prompts verbatim — build industry-tagged prompt templates. See
Section 6 for the expected differences.

---

## 3. Validation of Hedging Positions

### Two-tier verification process

All 3,446 extraction files were verified through a systematic two-tier process. Total
outcomes: **PASS 2,766 (80.3%) / FIXED 638 (18.5%) / WARN 42 (1.2%) / FAIL 0 (0.0%)**.

### Tier 1: Automated checks (6 checks per filing)

Run by `pipeline/verify_extractions.py` on every extraction file:

| Check | Rule | Tolerance |
|---|---|---|
| Volume math | `volume_per_day × days_in_period ≈ volume_total` | ±20% |
| Direction consistency | Producer swaps → `"short"`; collars → `"collar"` | Exact |
| Summary counts | `hedging_summary.total_instruments == len(instruments)` | Exact |
| Schema completeness | All required fields present; `sub_floor_price` on `three_way_collar` | Exact |
| Price plausibility | Prices within commodity- and year-appropriate ranges | Heuristic |
| Instrument count | Non-negative; not implausibly high per filing | Heuristic |

### Tier 2: Agent-based source verification

Filings that failed Tier 1 (initially 144 FAILs) were re-extracted across four rounds by
AI agents that read the original parsed source text and compared it against the JSON. Each
round reduced the FAIL count:

| Round | Filings re-extracted | FAILs before | FAILs after |
|---|---|---|---|
| 1 | 31 | 144 | 111 |
| 2 | 31 | 111 | 80 |
| 3 | 44 | 80 | 49 |
| 4 | 69+ | 49 | 0 |

The 794 warnings generated after the Item 7 merge were separately verified: 235 unique
warning files were checked against source text. Results:
- ~212 instruments removed (financial derivatives misidentified as commodity hedges, expired)
- ~20 instruments recovered from source
- 338 warnings confirmed as genuine source-data limitations (no table disclosed, fair-value-only)

### Most impactful corrections found during verification

| Company | Issue | Impact |
|---|---|---|
| Vital Energy | 11 NGL instruments missing entirely | +11 instruments |
| EOG Resources | Sub-period splits and closed positions omitted | +13 instruments |
| Antero Resources | VIE subsidiary instruments absent | Multiple filings |
| Civitas Resources | `sub_floor_price` missing from 6 three-way collars | Schema errors |
| Multiple firms | MBbls/Bcf scale errors | Volumes off by 1,000x |

### Validation logic for hedge ratios (post-calculation)

After computing ratios, `data/hedge_ratio_diagnostics.txt` flags:
- Ratios > 150% (overhedged or denominator error)
- Ratios where FY+1 production was unavailable (fell back to same-year production)
- NGL ratios without NGL production data (13 flagged)
- Firm-years with instruments but null ratios (66, all documented)

---

## 4. Hedge Ratio Calculation

### Conceptual framework

Follows Jin & Jorion (2006) and Haushalter (2000). Two ratios for oil and gas separately:

```
Notional hedge ratio = Total hedged volume (Bbls or Mcf) / Next-year production
Delta hedge ratio    = Total delta-adjusted hedged volume / Next-year production
Reserve hedge ratio  = Delta-adjusted volume / Proved reserves
```

**Why FY+1 production as denominator:** Instruments disclosed in a 10-K filed in March 2024
cover calendar year 2024 production, not 2023 production reported in the same filing.
The numerator and denominator must be contemporaneous. The calculator looks up production for
year Y+1 from any other filing by the same CIK that reports `production_year = Y+1`.

### Delta equivalents by instrument type

| Instrument | Delta per unit | Rationale |
|---|---|---|
| `swap`, `fixed_price_contract`, `forward`, `futures` | −1.00 | Full price lock |
| `collar` | −0.80 | Long put minus short call; partial upside capture |
| `three_way_collar` | −0.65 | Collar with additional sold put at sub-floor price |
| `put_option` | −0.50 | At-the-money approximation (Black 1976) |
| `call_option` | +0.50 | Long call held by counterparty |

Black's (1976) model can replace simplified deltas if market data (futures price, implied vol)
is provided. All records in this dataset use `delta_method = "simplified"`.

### Excluded instrument types

| Type | Count | Reason |
|---|---|---|
| `basis_swap` | 1,731 | Hedges geographic/quality basis only, not outright price |
| `swaption` | 134 | Represents optionality on a future swap, not a current hedge |

### Seven output ratio variants (in `data/hedge_ratios.csv`)

| Column | Numerator | Denominator |
|---|---|---|
| `hedge_ratio_oil_notional` | oil_notional_bbls | FY+1 oil production |
| `hedge_ratio_gas_notional` | gas_notional_mcf | FY+1 gas production |
| `hedge_ratio_oil_delta` | oil_delta_bbls | FY+1 oil production |
| `hedge_ratio_gas_delta` | gas_delta_mcf | FY+1 gas production |
| `hedge_ratio_ngl_delta` | ngl_delta_bbls | Same-filing NGL production |
| `hedge_ratio_oil_reserve` | oil_delta_bbls | Proved oil reserves |
| `hedge_ratio_gas_reserve` | gas_delta_mcf | Proved gas reserves |

Ratio is set to `null` (not zero) when denominator is null or zero.

### Volume allocation across fiscal years

Instruments often span multiple calendar years. Volume is allocated proportionally:

```
allocated_volume = volume_total × (overlap_days / total_period_days)
```

where `overlap_days` is the intersection of the instrument's coverage period with the
fiscal year being computed.

---

## 5. Pipeline Performance Summary

| Stage | Files | Time | Notes |
|---|---|---|---|
| Stage 1 parse | 4,761 | ~2 hours | Deterministic, no API cost |
| Stage 2 extraction (Sonnet batch) | 1,869 | ~4 hours | Batches API, 8 chunks |
| Stage 2 re-runs (Opus interactive) | 1,574 | ~6 hours | Sequential |
| Item 7 merge | 2,917 | ~3 hours | Two-phase, Sonnet 4.6 agents |
| Warning verification | 235 | ~2 hours | Parallel Sonnet agents |
| Quality audit | 3,443 | ~3 hours | Three parallel Sonnet agents |
| Verification (4 rounds) | 638 | ~4 hours | Per-filing agent re-extraction |
| Hedge ratio calculation | 3,446 | <10 min | Pure Python, no API |

**Final dataset state:**
- 453 unique companies, filings spanning 2005–2025
- 3,446 extraction files; 961 with instrument-level data
- 10,638 individual instruments (after all cleaning)
- 0 errors, 322 warnings (all verified as source-data limitations)

---

## 6. Industry-Specific Notes for Extension

*Updated 2026-05-22 — anchor filing analysis completed for all four validation filings
(Southwest FY2013, Delta FY2018, Valero FY2022, Marathon FY2019). Notes below incorporate
findings from reading the actual source text.*

---

### Airlines (SIC 4512)

**Anchor filing findings (Southwest FY2013):**

The primary disclosure is in the **financial statement notes** (Note 10 for Southwest),
not Item 7A. Item 7A gives only a narrative summary and one aggregate fair value figure.
The note contains a single table — aggregate gallons hedged by coverage year, with the
proxy commodity type listed. Individual contract-level details (specific strike prices,
instrument breakdown by year) are **not disclosed**. This is the standard airline format
across the industry.

Southwest FY2013 actual table:

| Coverage year | Gallons hedged (MM) | Commodity proxies |
|---|---|---|
| 2014 | 764 | WTI crude, Brent crude, GC Jet Fuel |
| 2015 | 1,156 | WTI crude, Brent crude |
| 2016 | 977 | Brent crude |
| 2017 | 933 | WTI crude, Brent crude |

Expected consumption (2014): 1.8 billion gallons → 43% hedged for 2014.
Net fair value: +$180M (net asset). Gross: +$503M asset, -$380M liability.

**Anchor filing findings (Delta FY2018):**

Delta is a **special case** that must be handled separately.
Item 7A states explicitly: *"Our derivative contracts to hedge the financial risk from
changing fuel prices are primarily related to Monroe's refining margins."*
Monroe Energy is Delta's owned oil refinery (Trainer, PA, acquired 2012). Delta's
"fuel hedges" are crack-spread hedges at the refinery — crude-in vs. refined-products-out
positions — not traditional airline consumer fuel hedges.

Delta FY2018 actual data: 219M gallons, crude oil and refined products, maturity Dec 2019,
net fair value +$15M. NOT designated as accounting hedges. Single aggregate position; no
instrument breakdown; no by-year table (only one coverage year).

**Implication for research design:** Delta requires a separate flag in the dataset.
Its "fuel hedge" variable measures something different from other airlines. When computing
hedge ratios, use Delta fuel hedges cautiously — the denominator (consumption) and numerator
(refinery crack-spread position) are conceptually mismatched.

**Universe:** ~15–20 publicly listed US carriers over 1998–2025. Small enough to manually
validate 100% of filings as a calibration check.

**Disclosure format (standard airlines, non-Delta):**
- Aggregate table by coverage year: total gallons + commodity type
- Volume unit: gallons (millions). Never barrels — do NOT convert.
- `% of expected fuel consumption hedged` is often disclosed directly in Item 7A narrative
- Expected consumption for FY+1 is disclosed in Item 7A ("we expect to consume X billion gallons")
- No individual contract strike prices — airlines aggregate their portfolios
- Fair value appears in the balance sheet note and in Item 7A

**Instrument vocabulary (standard airlines):**
- "purchased call options", "collar structures" (bought call + sold put), "call spreads"
  (bought call + sold call), "fixed-price swap agreements"
- Proxy commodities: WTI crude, Brent crude, heating oil, RBOB gasoline, GC Jet Fuel
- "fuel hedge" is used generically — ask for the underlying instrument
- Southwest FY2013 discloses all four instrument types in the portfolio; exact splits not given per year

**Direction:** Airlines are consumers (net buyers), so hedges are long — swaps lock in a
ceiling, calls cap fuel costs. Delta convention is +1.00 for a fuel cost cap, not −1.00.

**Denominator for hedge ratio:** Expected fuel consumption (gallons) for the next year.
Airlines disclose this in the MD&A as "we expect to consume X billion gallons in FY+1."
Actual consumption is in the income statement (fuel expense ÷ average fuel price).

**Key extraction differences to build into prompts:**
1. Ask for gallons, not Bbls
2. Extract `% of consumption hedged` as a direct field alongside volume
3. The reference price is heating oil or Brent, not WTI — extract `benchmark_price` explicitly
4. Prompt must exclude interest rate swaps and FX hedges (airlines have many)
5. Several airlines hedge 0% of consumption in some years — this is real data, not missing

**Notable variation to preserve:** Southwest Airlines dramatically reduced hedging after
2014 losses. Delta is unique in that it owns the Trainer refinery (crack spread, not just
fuel). These structural differences are analytically important — flag them in the metadata.

**Schema additions needed:**
- `volume_unit`: add `"gallons"` and `"billion_gallons"` to the canonical list
- Add `pct_consumption_hedged` as an optional field (airlines often disclose only this)
- Add `benchmark_commodity` field: `"heating_oil" | "brent" | "wti" | "jet_fuel"`

---

### Refiners (SIC 2911)

**Anchor filing findings (Valero FY2022 + Marathon FY2019):**

Both filings confirmed the following structure, which differs substantially from what
was anticipated before reading the actual disclosures:

- **All positions are short-horizon inventory hedges.** Nearly all contracts expire within
  one quarter (Valero: all in 2023; Marathon: 85–99% expire in Q1 2020). This is not
  the multi-year directional hedging of E&P firms or airlines.
- **No prices are disclosed.** Neither Valero nor Marathon discloses individual strike
  prices or weighted average prices for any contract.
- **Long AND short positions coexist.** Refiners run simultaneous long (crude feedstock)
  and short (refined products) positions, roughly offsetting each other. The NET position
  is small relative to the gross positions.
- **Marathon does not designate any derivatives as accounting hedges.** All economic hedges.
  Valero uses both cash flow hedges and economic hedges.

Marathon FY2019 actual data (Note 18, "units in thousands of barrels"):

| Commodity | Direction | Designation | Volume (MBbl) | % expiring next quarter |
|---|---|---|---|---|
| Crude oil | Long | Economic | 26,287 | 85.7% |
| Crude oil | Short | Economic | 27,237 | 85.7% |
| Refined products | Long | Economic | 15,298 | 94.7% |
| Refined products | Short | Economic | 12,519 | 94.7% |
| Blending products | Long | Economic | 1,976 | 99.5% |
| Blending products | Short | Economic | 3,770 | 99.5% |

Net fair value at Dec 31, 2019: -$98M (gross asset $63M, gross liability $161M).
Net loss on derivatives for 2019: -$96M.

Valero FY2022 actual data (Note 19, "volumes in thousands of barrels"):

| Commodity | Direction | Designation | Volume (MBbl) |
|---|---|---|---|
| Refined products | Long | Cash flow hedge | 2,929 |
| Refined products | Short | Cash flow hedge | 7,589 |
| Crude oil + refined products | Long | Economic | 73,415 |
| Crude oil + refined products | Short | Economic | 68,973 |

**Implication for research design:** The refiner schema must capture BOTH long and short
positions separately, and must track designation (cash flow hedge vs. economic). The
analytically interesting variable is the NET position (long minus short), which reveals
whether the refiner is running a net long or short bias. The gross position mostly
reflects inventory hedging that nets to near-zero.

**Universe:** ~15–20 firms. Valero, Marathon Petroleum, Phillips 66, HF Sinclair,
PBF Energy, Delek, CVR Energy.

**Disclosure format:** More standardized than anticipated. Both anchor filings contain
a tabular notional volume table by instrument type and direction. Less structured than E&P
but more tabular than initially expected. Expect ~70% PASS rate vs. ~80% for E&P.

**Instrument vocabulary (actual, from anchor filings):**
- Valero: "Futures — long" and "Futures — short" for crude oil and refined products
- Marathon: "exchange-traded" futures with "long" and "short" positions; "spread contracts"
  nested within (calendar spreads, not outright price hedges — flag separately)
- Neither anchor uses "crack spread swap" language in the derivative table — crack spread
  exposure is managed implicitly through simultaneous long/short positions

**Direction:** Both long and short — capture separately, do not net in extraction.

**Denominator for hedge ratio:** Barrels of crude throughput per year. Disclosed in the
operations section as "we processed X million barrels." No "production" in the E&P sense.

**Key extraction differences (updated from anchor filings):**
1. Extract BOTH long and short positions for each commodity — do not aggregate or net
2. Distinguish "cash flow hedge" from "economic hedge" designations — they are materially
   different (only cash flow hedges create AOCI; economic hedges hit P&L immediately)
3. Note whether positions are "exchange-traded" (futures/options on NYMEX) or OTC (swaps)
4. Extract the "% expiring next quarter" figure if disclosed — it signals hedge horizon
5. Blending products (ethanol, RINs) should be extracted but flagged as non-crude
6. No prices will be found — the schema correctly has no price fields for refiners
7. Check if the company designates hedges for accounting purposes — some don't (Marathon)

**Schema additions needed:**
- `designation`: `"cash_flow_hedge" | "economic_hedge" | "undesignated"`
- `pct_expiring_next_quarter`: float (Marathon discloses this; helps measure hedge horizon)
- `is_spread_contract`: bool (calendar spreads are analytically distinct from outright hedges)
- Volume unit: `"MBbls"` (thousands of barrels) — most common refiner unit

---

## 7. Recommended Validation Benchmarks

When building extraction prompts for a new industry, always validate against a single
well-documented filing before running the batch. All four anchors below are now on disk
in `anchor_filings/` with Stage 1 parsed sections and manual ground truth JSON files.

| Industry | Primary anchor | Secondary anchor | Ground truth file |
|---|---|---|---|
| E&P (current) | Devon Energy (CIK 1090012), FY2003 10-K | Pioneer Natural Resources, FY2014 | — |
| Airlines (standard) | Southwest Airlines (CIK 92380), FY2013 10-K | Alaska Air Group, FY2017 | `SW_2013_ground_truth.json` |
| Airlines (refinery-owner) | Delta Air Lines (CIK 27904), FY2018 10-K | — (Delta is unique case) | `DAL_2018_ground_truth.json` |
| Refiners | Valero Energy (CIK 1035002), FY2022 10-K | Marathon Petroleum (CIK 1510295), FY2019 | `VLO_2022_ground_truth.json` / `MPC_2019_ground_truth.json` |

---

## 8. Cost Estimation for Extension

Based on the E&P experience (4,761 filings, ~$0.003/filing for Sonnet batch):

| Industry | Est. filings | Extraction cost | Note |
|---|---|---|---|
| Airlines | ~300 | < $5 | 15–20 firms × 15+ years × 10-K + re-runs |
| Refiners | ~250 | < $5 | 15–20 firms × 12+ years; more Opus re-runs expected |

The extraction cost is negligible. Time cost is in prompt design and validation (1–2 weeks
per industry). Prioritize getting the prompt right on 5–10 filings before running the batch.
