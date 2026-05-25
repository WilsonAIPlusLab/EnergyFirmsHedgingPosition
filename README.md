# U.S. Commodity Hedging Dataset: E&P, Airlines, and Refiners

*Last updated: 2026-05-24*

A firm-level panel dataset of commodity hedging instruments and hedge ratios for three groups of U.S.
energy-sector companies, extracted from SEC 10-K annual filings: upstream E&P producers, commercial
airlines (jet fuel consumers), and petroleum refiners.

---

## Revision History

| Date | Change |
|---|---|
| 2026-05-24 | v2.0 — Added airline and refiner groups (661 filings); reorganized data/ into per-group subfolders (ep/, airline/, refiner/); added crosswalk.csv and master_panel.csv; E&P instrument_deltas.csv renamed positions.csv, instrument_index.csv renamed filings.csv. See DATA_QUALITY.md for name-migration table. |
| 2026-03-28 | NGL hedge ratio column count updated to 202 firm-years in schema table |
| 2026-03-23 | Initial release (E&P only). Added NGL columns to hedge_ratios.csv schema. Updated instrument count to 14,538 |

---

## Dataset at a Glance

| | E&P Producers | Airlines | Petroleum Refiners | All Groups |
|---|---|---|---|---|
| **Company group** | Upstream exploration & production | Jet fuel consumers | Crude oil / products processors | |
| **SEC filing type** | 10-K, SIC 1311 | 10-K, various SICs | 10-K, various SICs | |
| **Unique entities** | 453 | ~26 | 18 | ~497 |
| **Filings extracted** | 3,446 | 421 | 235 | 4,102 |
| **Filing period** | 2010–2025 | 1999–2025 | 2001–2025 | 1999–2025 |
| **Fiscal years covered** | 2007–2025 | 1999–2025 | 2001–2025 | 1999–2025 |
| **Hedging positions extracted** | 14,538 instruments | 344 coverage-year positions | 696 notional positions | 15,578 |
| **Hedge ratio denominator** | Annual oil/gas production | Expected fuel consumption (gallons) | Crude throughput (MBbl) | — (not unified across groups) |
| **Master panel rows** | 5,926 (oil + gas axes) | 421 | 235 | 6,582 |
| **Source** | SEC EDGAR, SIC 1311 | SEC EDGAR, individual CIKs | SEC EDGAR, individual CIKs | |
| **Extraction method** | Claude AI + automated verification | Claude AI + manual verification | Claude AI + manual verification | |

---

## Research Context

This dataset was developed as part of a research project funded by the **UNI Capacity Building Grant
2025-26**, Wilson College of Business, University of Northern Iowa.

**Principal Investigator:** Zhongdong (Ronnie) Chen, Ph.D., Associate Professor of Finance
**Co-Investigator:** Zhengyang (Robin) Chen, Ph.D., Assistant Professor of Economics

**Research focus:** The dataset is designed to support studies of corporate commodity risk management
across three economically distinct groups:
- **E&P producers** — How do oil and gas producers adjust hedging in response to price signals and
  financial stress? (Following Jin & Jorion 2006; Haushalter 2000)
- **Airlines** — How do jet fuel costs and hedging strategies affect airline profitability and survival?
- **Refiners** — How do refiners manage crude oil and refined product price risk (crack-spread risk)
  using inventory hedges and financial derivatives?

The methodology follows the academic hedging literature: Jin & Jorion (2006), Haushalter (2000),
Géczy, Minton & Schrand (1997), Carter, Rogers & Simkins (2006). The dataset was validated against
the Devon Energy example in Jin & Jorion (2006) as a benchmark.

---

## Dataset Groups

### Group 1: E&P Producers (`data/ep/`)
Upstream exploration and production companies (SIC 1311). Hedges are typically **short** derivatives
on crude oil, natural gas, and NGL — locking in a sale price for future production. Instruments
include swaps, collars, three-way collars, basis swaps, and options.

Hedge ratio definition: delta-adjusted hedged volume / annual production (barrel or Mcf basis,
separately for oil and gas).

### Group 2: Airlines (`data/airline/`)
Commercial airlines and air cargo carriers. Hedges are typically **long** derivatives on crude oil,
heating oil, or jet fuel kerosene — locking in a purchase price. Instruments include collars, call
options, swaps, and call spreads.

Key distinction: Four capacity-purchase agreement (CPA) carriers — Mesa Air, SkyWest, Republic
Airways, Air Transport Services Group — pass fuel costs to their major-airline partners and therefore
have no economic incentive to hedge. These are included in the dataset but their status is coded
`pass_through`; their hedge ratios are null.

Hedge ratio definition: gallons hedged for coverage_year / expected gallons to be consumed in
coverage_year. Reported as a decimal fraction (0.40 = 40% hedged).

### Group 3: Petroleum Refiners (`data/refiner/`)
Petroleum refiners and specialty products companies. Refiner hedges are structurally different from
producer hedges: refiners may hold **both long and short** positions (long crude / short products)
to lock in a processing margin ("crack spread"), or they may hedge inventory risk with short futures
on crude oil they hold in storage.

Hedge ratio definition: net short MBbl (short − long, excluding natural gas and platinum) /
crude throughput in MBbl. A positive net_short_ratio means the refiner is net short (classic
inventory hedge); a negative value means net long.

**Important:** The refiner hedge ratio definition is **an open methodological question**. The ratio
conflates crude and product hedges (which offset the crack spread) with outright inventory hedges.
Researchers using this data for refiner analysis should read `METHODOLOGY.md §7` and consider
whether `net_short_ratio` or `abs_ratio` (or a crack-spread-specific subset) is appropriate for
their question.

---

## File Structure

```
dataset/
├── README.md                    <- This file
├── METHODOLOGY.md               <- How the dataset was constructed
├── DATA_QUALITY.md              <- Known issues, limitations, verification results
├── HOW_TO_USE.md                <- Code examples, schema reference
├── LICENSE                      <- CC-BY-4.0
├── CITATION.cff                 <- Machine-readable citation
│
├── data/
│   ├── ep/                      <- E&P producer group
│   │   ├── hedge_ratios.csv     <- Firm-year panel (2,963 rows)
│   │   ├── positions.csv        <- Instrument-level detail (14,538 rows)
│   │   ├── filings.csv          <- Filing-level summary (3,446 rows)
│   │   ├── filing_index.csv     <- Full parse index incl. unextracted filings (4,672 rows) [E&P only]
│   │   ├── verification_log.csv <- Per-filing verification verdict (3,446 rows) [E&P only]
│   │   └── extractions/         <- Per-filing JSON source files (3,446 files)
│   │
│   ├── airline/                 <- Airline / air cargo group
│   │   ├── hedge_ratios.csv     <- Firm-year panel (421 rows)
│   │   ├── positions.csv        <- Coverage-year position detail (344 rows)
│   │   ├── filings.csv          <- Filing-level summary (421 rows)
│   │   └── extractions/         <- Per-filing JSON source files (421 files)
│   │
│   ├── refiner/                 <- Petroleum refiner group
│   │   ├── hedge_ratios.csv     <- Firm-year panel (235 rows)
│   │   ├── positions.csv        <- Notional position detail (696 rows)
│   │   ├── filings.csv          <- Filing-level summary (235 rows)
│   │   └── extractions/         <- Per-filing JSON source files (235 files)
│   │
│   ├── crosswalk.csv            <- CIK to ticker/entity mapping, era-based (50 rows)
│   └── master_panel.csv         <- Harmonized cross-group panel (6,582 rows)
│
└── docs/                        <- Supplemental technical documentation
    ├── CODEBOOK.md              <- Per-variable data dictionary (all tables)
    ├── PIPELINE_SUMMARY.md
    ├── VERIFICATION_PROJECT_NOTES.md
    ├── VERIFICATION_REPORT.txt
    ├── QUALITY_AUDIT.md
    └── project_review.md
```

**Note on v2.0 file renames (E&P group):** Users of the v1.0 dataset should note that three files
were moved and two were renamed. See `DATA_QUALITY.md §9` for a complete migration table.

---

## Per-Group Table Schemas

### Uniform core (all three groups)

Every group's `hedge_ratios.csv` contains:
`cik, company, ticker, fiscal_year, filing_date`
plus group-specific ratio columns (see below) and a `status` column.

Every group's `positions.csv` contains per-position records linkable back to the filing via
`(cik, filing_date)`.

Every group's `filings.csv` contains one row per filing with filing-level provenance.

### data/ep/hedge_ratios.csv — E&P firm-year panel

| Column | Description |
|---|---|
| `cik`, `company`, `fiscal_year`, `filing_date` | Identifiers |
| `n_oil_instruments`, `n_gas_instruments`, `n_ngl_instruments` | Instrument counts by commodity |
| `oil_notional_bbls`, `gas_notional_mcf`, `ngl_notional_bbls` | Total hedged volume (raw notional) |
| `oil_delta_bbls`, `gas_delta_mcf`, `ngl_delta_bbls` | Delta-adjusted hedged volume |
| `oil_production_bbls`, `gas_production_mcf`, `ngl_production_bbls` | Annual production (denominator) |
| `hedge_ratio_oil_notional` | Oil notional hedge ratio |
| `hedge_ratio_gas_notional` | Gas notional hedge ratio |
| `hedge_ratio_oil_delta` | Oil delta-adjusted hedge ratio |
| `hedge_ratio_gas_delta` | Gas delta-adjusted hedge ratio |
| `hedge_ratio_ngl_delta` | NGL delta-adjusted hedge ratio (202 firm-years; null where not reported) |
| `hedge_ratio_oil_reserve` | Oil delta / proved oil reserves |
| `hedge_ratio_gas_reserve` | Gas delta / proved gas reserves |
| `delta_method` | `simplified` (all records; no external market data) |
| `uses_derivatives` | Whether derivatives disclosed (boolean) |
| `hedging_confidence` | Extraction confidence: high / medium / low |
| `warnings` | Pipe-separated data quality flags |

### data/airline/hedge_ratios.csv — Airline firm-year panel

| Column | Description |
|---|---|
| `cik`, `company`, `ticker`, `fiscal_year`, `filing_date` | Identifiers |
| `fuel_consumption_gallons_mm` | Actual gallons consumed in fiscal_year (from 10-K; MM gallons) |
| `expected_gallons_millions` | Expected gallons for coverage_year (if disclosed separately) |
| `hedge_ratio_next` | Hedge coverage for fiscal_year+1 (decimal fraction; primary ratio) |
| `hedge_ratio_current` | Hedge coverage for fiscal_year itself (if disclosed) |
| `headline_coverage` | Which year the headline ratio came from: `next`, `current`, or `none` |
| `has_program`, `stopped_hedging`, `stopped_hedging_year`, `owns_refinery` | Program flags |
| `net_fair_value_mm` | Net fair value of derivatives as of fiscal year-end ($ millions) |
| `status` | See Status Taxonomy below |
| `warnings` | Pipe-separated data quality flags |

### data/refiner/hedge_ratios.csv — Refiner firm-year panel

| Column | Description |
|---|---|
| `cik`, `company`, `ticker`, `fiscal_year`, `filing_date` | Identifiers |
| `crude_throughput_mmbbl` | Crude oil processed in fiscal_year (million barrels) |
| `throughput_mbbl` | Same in thousands of barrels (= crude_throughput_mmbbl × 1,000) |
| `total_long_mbbl`, `total_short_mbbl` | Gross long and short positions (crude + products, MBbl) |
| `net_short_mbbl` | Net short position (total_short − total_long; positive = net short) |
| `total_abs_mbbl` | Total absolute notional (long + short; double-counts paired hedges) |
| `net_short_ratio` | net_short_mbbl / throughput_mbbl (primary ratio; see caveats in METHODOLOGY §7) |
| `abs_ratio` | total_abs_mbbl / throughput_mbbl (gross turnover ratio) |
| `primary_objective`, `has_cash_flow_hedges`, `has_economic_hedges` | Program classification |
| `status` | See Status Taxonomy below |
| `warnings` | Pipe-separated data quality flags |

### data/master_panel.csv — Harmonized cross-group panel

One row per `(cik, fiscal_year, hedge_axis)`. `hedge_axis ∈ {oil, gas, fuel, crude_net}`.

**CRITICAL CAVEAT:** `hedge_ratio_primary` is NOT comparable across groups. Different denominators
(production, fuel consumption, crude throughput) and different economic meanings make pooled
cross-group analysis of the ratio itself inappropriate without careful adjustment. The master panel
is designed for queries like "which firms hedged in a given year" or "how has hedging changed over
time within a group," not for directly comparing a 40%-hedged airline to a 4%-hedged refiner.

| Column | Description |
|---|---|
| `cik`, `group`, `fiscal_year`, `hedge_axis` | Key |
| `company`, `ticker` | Identity |
| `hedge_ratio_primary` | Group's primary ratio (see mapping in METHODOLOGY §7) |
| `ratio_method` | `delta`, `notional_volume`, or `net_short` — how ratio was computed |
| `hedge_volume` | Numerator (native units per group) |
| `volume_unit` | `bbl`, `mcf`, `gallon`, `mbbl` |
| `exposure_base` | Denominator (native units per group) |
| `exposure_unit` | `bbl`, `mcf`, `gallon_mm`, `mbbl` |
| `has_program`, `status` | Program presence and status code |
| `filing_date`, `accession_number` | Provenance |

### Status Taxonomy

The `status` column distinguishes structural zeros from true missing values — a critical distinction
for hedging-behavior research where zero-hedging and non-disclosed hedging are economically different.

| Status | Meaning | hedge_ratio_primary |
|---|---|---|
| `hedger` | Firm holds positions; ratio computed | non-null |
| `no_position` | Program exists; zero positions this axis-year | 0.0 |
| `stopped` | Firm permanently exited hedging (e.g. AAL ≥2013, UAL ≥2016) | 0.0 |
| `never` | No hedging program disclosed | null |
| `pass_through` | Fuel cost passed to partners; no exposure to hedge (airline CPA carriers) | null |
| `not_applicable` | Denominator missing; ratio uncomputable | null |

### data/crosswalk.csv — Entity crosswalk

| Column | Description |
|---|---|
| `cik` | SEC Central Index Key |
| `era_start_fy`, `era_end_fy` | Fiscal years this entity/group identity applies |
| `canonical_ticker` | Representative ticker for this era |
| `company_name` | Company name during this era |
| `group` | `ep`, `airline`, or `refiner` |
| `entity_id` | Stable ID grouping eras that are the same legal lineage (e.g. all three Frontier CIKs share an entity_id) |
| `lineage` | Free-text note on mergers, bankruptcies, and corporate identity changes |
| `business_model` | `standard` or `pass_through` |

---

## Limitations and Caveats

This section documents the dataset's most important limitations. We believe in transparency about
what the data can and cannot support — understating limitations misleads researchers.

### AI extraction accuracy
Hedging positions were extracted by a large language model (Claude Sonnet/Opus, Anthropic). Despite
careful prompt design and extensive verification (see `DATA_QUALITY.md`), AI extraction is not
infallible. The model can misread ambiguous tables, confuse fiscal-year versus calendar-year
coverage, conflate notional volumes with fair values, or miss positions disclosed in non-standard
formats. For the E&P group, an automated 6-check verification pipeline + 4 rounds of targeted
re-extraction corrected identified errors. For the airline and refiner groups, manual company-by-
company verification was performed, but not an automated check for every filing. Some extraction
errors may remain.

**Recommendation:** For any firm-year where the extracted ratio appears extreme or economically
implausible, consult the source `extractions/{cik}_{date}.json` and the underlying 10-K filing.
The `warnings` column in each panel file records known issues.

### Disclosure variation
Companies disclose hedging positions in widely varying levels of detail. Some provide complete
tabular summaries; others include only aggregate fair values; a few provide narrative descriptions
without quantitative volumes. Where quantitative data was unavailable, positions may be omitted or
marked null. This creates **systematic missing data** that correlates with firm size (smaller,
less complex filers tend to disclose less), which researchers should account for in analysis.

### Coverage gaps by group

**Airlines:**
- FedEx and UPS are air cargo carriers whose fuel exposure spans both aircraft and ground vehicles;
  no aircraft-specific fuel gallons are disclosed in their 10-Ks. Fuel_consumption is null for
  these 49 filings.
- Four CPA pass-through carriers (Mesa, SkyWest, Republic, ATSG) are included but are not expected
  to hedge; their status is `pass_through` and ratios are null.
- Some early-year filings (pre-2005) may lack quantitative coverage percentages.

**Refiners:**
- CIK 821483 (Delta Petroleum Corporation, pre-2014) is an E&P company included in the refiner
  set because it later became Par Pacific Holdings (a refiner). The Delta Petroleum era filings
  have null throughput and no hedge positions and are coded `never`.
- One entity (CIK 777676, Oracle Corporation) was detected by keyword search and incorrectly
  included in the airline raw extraction. It has been excluded from the delivered dataset and is
  documented in `DATA_QUALITY.md §10`.

**E&P:**
- 1,226 parsed filings do not have extraction JSON files — these are primarily low-keyword-score
  filings (little or no hedging disclosure).
- 42 E&P filings remain flagged with `WARN` in `verification_log.csv` (see `DATA_QUALITY.md §5`).

### Cross-group non-comparability
The three groups use fundamentally different hedge ratio definitions:
- E&P: delta-adjusted hedged volume / annual production (commodity-specific)
- Airlines: gallons hedged / expected gallons consumed (next year)
- Refiners: net short MBbl / crude throughput (ambiguous sign; see `METHODOLOGY.md §7`)

These ratios cannot be compared directly. A 40% airline hedge ratio and a 4% refiner net-short
ratio do not mean the same thing. The `master_panel.csv` facilitates cross-group queries on
hedging status but requires careful interpretation of the ratio columns.

### Entity identity over time
Companies change names, merge, emerge from bankruptcy under new CIKs, and delist. The `crosswalk.csv`
documents known identity changes, but it is based on curated manual research and may not be
complete. Researchers building long panels should verify entity continuity using the `entity_id`
and `lineage` columns before pooling across corporate-event boundaries.

### No GVKEY / PERMNO linkage
The dataset does not include Compustat GVKEY or CRSP PERMNO identifiers. Linking to Compustat or
CRSP requires a CIK-to-GVKEY crosswalk available from WRDS (the SEC-to-Compustat linking table).
Start with `cik` as the merge key; most modern Compustat records include the CIK.

---

## Quick Start

```python
import pandas as pd, json

# ── E&P Producers ──────────────────────────────────────────────────────────
ep = pd.read_csv("data/ep/hedge_ratios.csv")
print(f"E&P: {ep.fiscal_year.nunique()} years, {ep.cik.nunique()} firms")
print(f"Mean oil delta hedge ratio: {ep['hedge_ratio_oil_delta'].mean():.1%}")

# ── Airlines ───────────────────────────────────────────────────────────────
air = pd.read_csv("data/airline/hedge_ratios.csv")
hedgers = air[air.status == "hedger"]
print(f"Airline hedgers: {hedgers.cik.nunique()} carriers")
print(f"Mean next-year coverage: {hedgers['hedge_ratio_next'].mean():.1%}")

# ── Refiners ───────────────────────────────────────────────────────────────
ref = pd.read_csv("data/refiner/hedge_ratios.csv")
print(f"Refiner net-short ratio (median): {ref['net_short_ratio'].median():.3f}")

# ── Master panel ───────────────────────────────────────────────────────────
panel = pd.read_csv("data/master_panel.csv")
# Filter by group and hedge_axis
oil_ep = panel[(panel.group == "ep") & (panel.hedge_axis == "oil")]
print(f"E&P oil-axis firm-years: {len(oil_ep)}")
```

See `HOW_TO_USE.md` for complete examples including the position-level tables and crosswalk joins.

---

## Source Attribution

Source filings are SEC EDGAR 10-K annual reports in the public domain. Hedging positions were
extracted using AI (Anthropic Claude models) and verified manually. The dataset itself is research
output licensed CC-BY-4.0; see `LICENSE`.

> Chen, Z., & Chen, Z. (2026). *U.S. Commodity Hedging Dataset: E&P, Airlines, and Refiners*.
> Extracted from SEC EDGAR 10-K filings, fiscal years 1999–2025. Wilson College of Business,
> University of Northern Iowa. Developed under UNI Capacity Building Grant 2025-26.

---

## Related Documentation

| File | Contents |
|---|---|
| `METHODOLOGY.md` | Full pipeline description, hedge ratio formulas, limitations |
| `DATA_QUALITY.md` | Verification results, known corrections, v1→v2 migration table |
| `HOW_TO_USE.md` | Code examples, schema reference, common queries |
| `docs/CODEBOOK.md` | Per-variable data dictionary for all tables |
| `docs/QUALITY_AUDIT.md` | E&P hedge ratio audit findings |
| `docs/VERIFICATION_PROJECT_NOTES.md` | Detailed E&P verification process |
