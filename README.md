# U.S. Oil & Gas Commodity Hedging Dataset

*Last updated: 2026-03-23*

A firm-level dataset of commodity hedging instruments and hedge ratios for U.S. oil and gas producers, extracted from SEC 10-K annual filings.

---

## Dataset at a Glance

| | |
|---|---|
| **Companies** | 453 unique SEC registrants (SIC 1311) |
| **Annual filings** | 3,446 |
| **Filing dates** | 2010–2025 (fiscal years covered: 2007–2025) |
| **Hedging instruments extracted** | 14,538 |
| **Firm-years with hedging instruments** | 1,491 (of 2,963 unique firm-years) |
| **Firm-years without hedging** | 1,472 |
| **Hedge ratios computed** | 2,963 firm-years |
| **Source** | SEC EDGAR, SIC 1311 annual 10-K filings |
| **Extraction model** | Claude Sonnet (claude-sonnet-4-5) via Anthropic Batches API |
| **Verification** | Automated 6-check pipeline + 4 rounds of manual re-extraction |

---

## Research Context

This dataset was developed as part of a research project funded by the **UNI Capacity Building Grant 2025-26**, Wilson College of Business, University of Northern Iowa.

**Principal Investigator:** Zhongdong (Ronnie) Chen, Ph.D., Associate Professor of Finance
**Co-Investigator:** Zhengyang (Robin) Chen, Ph.D., Assistant Professor of Economics

**Research question:** Do upstream E&P firms use stock market signals about their oil price risk exposure to dynamically adjust their commodity hedging strategies? And does failure to adjust contribute to firm financial distress or bankruptcy?

The project examines this question around two commodity price crises: the 2014–2016 oil price collapse and the 2020 COVID-era oil price crash. The dataset supports panel regression analysis of hedge ratios on abnormal oil price exposure measures derived from equity returns, as well as firm survival analysis linking hedging behavior to financial outcomes.

The original grant proposal targeted approximately 100 firms and ~1,000 firm-years. The actual extraction pipeline covers **453 companies and 2,963 firm-years** — a substantially broader scope made possible by automated AI extraction across the full SIC 1311 universe. The methodology follows Jin & Jorion (2006) and Haushalter (2000) and was validated against the Devon Energy example in Jin & Jorion (2006) as a benchmark.

---

## What This Dataset Contains

This dataset captures the commodity hedging activity of U.S. oil and gas exploration and production (E&P) companies as disclosed in their annual 10-K filings. For each filing, it provides:

1. **Hedging instruments** — every derivative contract disclosed: swap, collar, three-way collar, basis swap, swaption, etc., with volumes, prices, and coverage periods.
2. **Hedge ratios** — firm-year summary ratios measuring the fraction of production covered by hedges, computed using the methodology of Jin & Jorion (2006).
3. **Production and reserves** — annual production, proved reserves, and selected economics as reported in the same 10-K.
4. **Verification records** — extraction verdict (PASS / FIXED / WARN) and any corrections applied for each filing.

The dataset covers the shale era through the most recent filings available (fiscal years 2007–2025), spanning the rise of horizontal drilling, the 2014–2016 oil price collapse, and the post-COVID commodity cycle. It is intended for academic research on corporate risk management, commodity price exposure, and firm value.

---

## File Structure

```
dataset/
├── README.md                    ← This file
├── HOW_TO_USE.md                ← Practical usage guide with code examples
├── METHODOLOGY.md               ← How the dataset was constructed
├── DATA_QUALITY.md              ← Verification results and known limitations
│
├── data/
│   ├── extractions/             ← 3,446 JSON files, one per filing
│   ├── instrument_index.csv     ← Filing-level summary index (3,446 rows)
│   ├── filing_index.csv         ← Full parsed filing index (4,672 rows)
│   ├── hedge_ratios.csv         ← Firm-year hedge ratios (2,963 rows)
│   ├── instrument_deltas.csv    ← Instrument-level delta calculations (14,538 rows)
│   └── verification_log.csv     ← Per-filing verification verdict (3,446 rows)
│
└── docs/                        ← Supplemental technical documentation
    ├── PIPELINE_SUMMARY.md
    ├── VERIFICATION_PROJECT_NOTES.md
    ├── VERIFICATION_REPORT.txt
    ├── QUALITY_AUDIT.md
    ├── item7_merge_summary.md
    └── project_review.md
```

### data/extractions/

3,446 JSON files named `{CIK}_{filing_date}.json`. Each contains:
- `cik`, `company`, `filing_date`, `accession_number`
- `instruments.instruments` — array of hedging instrument objects (empty array if none disclosed)
- `instruments.hedging_summary` — counts by commodity, hedging policy text
- `production.production` — annual production (oil Bbls, gas Mcf, NGL Bbls, BOE)
- `production.reserves` — proved reserves
- `production.economics` — unit costs, prices used in reserve valuation
- `metadata` — model, extraction timestamp, confidence level

### data/instrument_index.csv

One row per filing. Key columns:
| Column | Description |
|---|---|
| `cik` | SEC Central Index Key |
| `company` | Company name |
| `filing_date` | 10-K filing date (YYYY-MM-DD) |
| `accession_number` | SEC accession number |
| `hedging_confidence` | Extraction confidence: high / medium / low |
| `uses_derivatives` | Whether the company used derivatives (boolean) |
| `instrument_count` | Number of hedging instruments extracted |
| `total_oil_instruments` | Count of oil instruments |
| `total_gas_instruments` | Count of gas instruments |
| `total_ngl_instruments` | Count of NGL instruments |
| `has_production_data` | Whether production data was extracted |
| `output_file` | Path to extraction JSON |

### data/filing_index.csv

One row per parsed filing (4,672 rows, including filings not yet extracted). Key additional columns: `form_type`, `sic`, `has_item7a`, `has_derivatives`, `hedging_confidence`, `item7a_length`, `notes_length`.

### data/hedge_ratios.csv

One row per firm-year. See `METHODOLOGY.md` for calculation details. Key columns:
| Column | Description |
|---|---|
| `cik`, `company`, `fiscal_year`, `filing_date` | Identifiers |
| `n_oil_instruments`, `n_gas_instruments`, `n_ngl_instruments` | Instrument counts by commodity |
| `oil_notional_bbls`, `gas_notional_mcf`, `ngl_notional_bbls` | Total hedged volume (raw) |
| `oil_delta_bbls`, `gas_delta_mcf`, `ngl_delta_bbls` | Delta-adjusted hedged volume |
| `oil_production_bbls`, `gas_production_mcf`, `ngl_production_bbls` | Annual production (denominator) |
| `hedge_ratio_oil_notional` | Oil notional hedge ratio |
| `hedge_ratio_gas_notional` | Gas notional hedge ratio |
| `hedge_ratio_oil_delta` | Oil delta-adjusted hedge ratio |
| `hedge_ratio_gas_delta` | Gas delta-adjusted hedge ratio |
| `hedge_ratio_ngl_delta` | NGL delta-adjusted hedge ratio (200 firm-years; null where NGL production not reported) |
| `hedge_ratio_oil_reserve` | Oil delta / proved oil reserves |
| `hedge_ratio_gas_reserve` | Gas delta / proved gas reserves |
| `delta_method` | `notional_only` / `linear` / `simplified` — see DATA_QUALITY.md |
| `hedging_confidence` | Extraction confidence level |
| `warnings` | Pipe-separated data quality flags |

### data/instrument_deltas.csv

One row per instrument included in hedge ratio calculations (14,538 rows). Contains the instrument-level delta computation: allocated volume, delta per unit, delta total, fiscal year assignment. Useful for auditing or building custom aggregations.

### data/verification_log.csv

One row per filing. Columns: `cik`, `company`, `filing_date`, `instrument_count`, `verdict`, `issue_count`, `issues_summary`, `verified_at`. Verdicts: PASS / FIXED / WARN / FAIL. No filings remain at FAIL.

---

## Quick Start

```python
import pandas as pd
import json, os

# Load the instrument index
idx = pd.read_csv("data/instrument_index.csv")
print(f"{len(idx)} filings, {idx['instrument_count'].sum():.0f} total instruments")

# Load hedge ratios
hr = pd.read_csv("data/hedge_ratios.csv")
hedgers = hr[hr['hedge_ratio_oil_notional'].notna()]
print(f"Mean oil hedge ratio: {hedgers['hedge_ratio_oil_notional'].mean():.1%}")

# Load a single extraction JSON (Vital Energy FY2018 10-K)
with open("data/extractions/1528129_2019-02-14.json") as f:
    record = json.load(f)
for instr in record['instruments']['instruments']:
    print(instr['instrument_type'], instr['commodity'], instr['volume_total'])
```

See `HOW_TO_USE.md` for complete Python and R examples.

---

## Key Concepts

**Instrument types** captured: `swap`, `collar`, `three_way_collar`, `basis_swap`, `swaption`, `put_option`, `call_option`, `fixed_price_contract`, `forward`

**Commodities**: `crude_oil`, `natural_gas`, `ngl`

**Hedge ratio**: ratio of hedged volume (in delta equivalents) to annual production. Computed for oil, gas, and NGL separately. Swaps have delta = −1.00 (fully hedged); collars = −0.80; three-way collars = −0.65. See `METHODOLOGY.md`.

**Fiscal year**: determined from `production_year` field or inferred from filing date (filings in Jan–Jun report the prior year).

**FY+1 denominator**: instruments in a given 10-K typically cover the *next* production year. Hedge ratios use next-year production as the denominator where available.

---

## Source Attribution

Source filings are SEC EDGAR 10-K annual reports, which are in the public domain. Company-reported data was extracted using AI and verified manually. The dataset itself is research output; cite as:

> Chen, Z., & Chen, Z. (2026). *U.S. Oil & Gas Commodity Hedging Dataset*. Constructed from SEC EDGAR 10-K filings, SIC 1311, fiscal years 2007–2025. Wilson College of Business, University of Northern Iowa. Developed under UNI Capacity Building Grant 2025-26.

---

## Related Documentation

| File | Contents |
|---|---|
| `HOW_TO_USE.md` | Code examples, schema reference, common queries |
| `METHODOLOGY.md` | Full pipeline description, hedge ratio formulas |
| `DATA_QUALITY.md` | Verification results, known corrections, limitations |
| `docs/VERIFICATION_PROJECT_NOTES.md` | Detailed verification process notes |
| `docs/QUALITY_AUDIT.md` | Hedge ratio audit findings |
| `docs/PIPELINE_SUMMARY.md` | Technical pipeline documentation |
