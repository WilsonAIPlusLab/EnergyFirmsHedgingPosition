# Methodology: How This Dataset Was Constructed

*Last updated: 2026-05-24*

This document describes the full pipeline used to build the dataset: from raw SEC filings to extracted instruments to hedge ratio calculations.

---

## Revision History

| Date | Change |
|---|---|
| 2026-05-24 | v2.0 — Added §§7-9 covering airline/refiner extraction, master panel, crosswalk, and status taxonomy. Sections 7-9 (new) describe group-specific methodologies. §5 "Excluded instruments" now references `positions.csv` (renamed from `instrument_deltas.csv`). |
| 2026-03-28 | §4 post-processing: added note on 23 FY+1 production denomination patches. §5 NGL denominator note: updated to 202 firm-years |
| 2026-03-23 | Initial release. Added NGL hedge ratio variant to §5 output table. Corrected company universe description (midstream retained, not excluded). Added NGL denominator design note |

---

## 1. Source Data

**Research context:** This dataset was constructed to support research on whether U.S. oil and gas firms use stock market signals about their commodity price exposure to dynamically adjust hedging strategies, and whether failure to adjust predicts financial distress. The project is funded by the UNI Capacity Building Grant 2025-26, Wilson College of Business, University of Northern Iowa (PI: Zhongdong (Ronnie) Chen; Co-PI: Zhengyang (Robin) Chen). The original grant targeted ~100 firms; the pipeline was extended to the full SIC 1311 universe (453 companies).

**Filing source:** SEC EDGAR full-text search database (https://efts.sec.gov). Annual 10-K filings for companies with SIC code 1311 (Crude Petroleum and Natural Gas) were downloaded. Filing dates in this dataset range from 2010 to 2025, corresponding to fiscal years 2007–2025 (fiscal year is typically the calendar year preceding the filing date).

**Filing selection:** Only annual 10-K filings were used. 10-K/A (amendments), 8-K, and quarterly 10-Q filings are not included.

**Company universe:** All SEC registrants with SIC 1311 that filed 10-Ks during the period. This includes E&P companies, royalty trusts, limited partnerships, and related entities. Four non-E&P registrants (midstream companies) were identified during quality audit; they are retained in `hedge_ratios.csv` with their instruments but their ratios are null due to no oil/gas production (see `DATA_QUALITY.md`).

**Raw filing count:** 4,672 filings were parsed from the raw EDGAR download. The full parsed filing index is in `data/filing_index.csv`.

**Extraction count:** 3,446 filings have extraction JSON files. The gap (4,672 − 3,446 = 1,226) reflects filings identified but not yet fully processed in the extraction pipeline, primarily smaller companies with limited hedging disclosures and filings where section parsing returned empty text.

---

## 2. Stage 1: Deterministic Parsing

Each raw 10-K file (SEC SGML format with embedded HTML) was parsed by a deterministic Python script (`pipeline/stage1_parser.py`) that:

1. **Strips SGML wrappers** to extract the embedded HTML or plain-text filing.
2. **Identifies sections** using keyword matching on section headers: Item 7 (MD&A), Item 7A (Quantitative and Qualitative Disclosures About Market Risk), Item 8 (Financial Statements), and Notes to Financial Statements.
3. **Extracts section text** for each identified section into separate files:
   - `item7a.txt` — Item 7A: Quantitative disclosures (primary source for hedging tables)
   - `item8_notes.txt` — Notes to financial statements (secondary source for instrument details)
   - `item7.txt` — MD&A narrative (context)
   - `full_text.txt` — Complete filing text (fallback)
4. **Keyword classification:** Each filing is scored on the presence of hedging-related keywords (e.g., "derivative", "swap", "collar", "hedge ratio", "fixed price", "floor price"). A keyword hit count is recorded in `filing_index.csv`.

**Output:** `parsed/{CIK}_{Company}/{date}/sections/` directories, one per filing.

---

## 3. Stage 2: AI Extraction

Parsed section text was passed to a large language model (Claude Sonnet, model `claude-sonnet-4-5`) via the Anthropic Batches API for structured data extraction.

### What was extracted

For each filing, the model extracted:

**Hedging instruments** (from Item 7A and Item 8 Notes):
- Instrument type, commodity, direction
- Volume per day and total volume
- Volume unit (Bbls, Mcf, Mmbtu, Boe)
- Price fields: fixed_price, floor_price, ceiling_price, sub_floor_price
- Coverage period (period_start, period_end in YYYY-MM format)
- Counterparty (if named)
- Fair value in $ millions (if disclosed)

**Production data** (from MD&A and financial statements):
- Annual production: oil (Bbls), gas (Mcf), NGL (Bbls), total (Boe)
- Proved reserves: oil, gas, NGL, total, developed/undeveloped
- Selected economics: oil/gas price used in reserve valuation, lifting costs, DD&A, production taxes

### Extraction priority

The model was instructed to prioritize sources in this order:
1. Item 7A (`item7a.txt`) — Quantitative risk disclosure, contains the most complete hedging tables
2. Item 8 Notes (`item8_notes.txt`) — Detailed footnote tables
3. Item 7 MD&A (`item7.txt`) — Narrative context
4. Full text (`full_text.txt`) — Fallback for HTML-rendered filings

### Instrument identification rules

- All derivative positions disclosed as of the filing date (or a stated "as-of" date) are included.
- Expired or settled positions are excluded.
- Instruments covering periods already past the fiscal year end but not yet the filing date are included if disclosed as current.
- Instruments from subsequent events (added after fiscal year end but before filing) are included when clearly labeled.
- Each table row becomes one instrument object. Rows at different price levels or different periods are kept as separate instruments; they are not aggregated.

### Volume unit handling

All volume values in the JSON files are in the units specified by `volume_unit`. No pre-scaling was applied during extraction. The following units appear in the data:
- **Bbls** — Barrels (oil and NGL)
- **Mcf** — Thousand cubic feet (gas)
- **Mmbtu** — Million British thermal units (gas, treated as ≈ Mcf in hedge ratio calculations)
- **Boe** — Barrels of oil equivalent

Some older filings disclosed volumes in MBbls (thousands of barrels) or Bcf (billions of cubic feet). A post-processing normalization step (`pipeline/normalize_volume_units.py`) corrected these to canonical units by applying the appropriate multiplier (MBbls × 1,000 = Bbls; Bcf × 1,000,000 = Mcf).

---

## 4. Post-Processing

After AI extraction, three normalization passes were applied:

**Schema normalization** (`hedge_ratio/fix_schema.py`): 89 extraction files contained non-standard field names (e.g., `volume_bbls` instead of `volume_total`, `year` instead of `period_start`). These were renamed to the canonical schema.

**Unit standardization** (`pipeline/normalize_volume_units.py`): Volume units were standardized to canonical values. This step also populated `volume_total` from `volume_per_day × period_days` where `volume_total` was null, and vice versa.

**Production data backfill** (`hedge_ratio/patch_production.py`): For 106 filings where production data was missing or partial in the primary extraction, production figures were recovered from a supplementary re-extraction pass and merged in. An additional 23 FY+1 filing JSONs had production data recovered to serve as hedge ratio denominators for E&P firm-years whose FY+1 production was missing.

---

## 5. Hedge Ratio Calculation

Hedge ratios follow the framework of Jin & Jorion (2006) and Haushalter (2000). The theoretical motivation for corporate hedging derives from Smith & Stulz (1985), who show that hedging increases firm value by reducing expected tax liabilities, financial distress costs, and underinvestment. Froot, Scharfstein & Stein (1993) extend this to show that hedging coordinates internal cash flows with investment opportunities, a mechanism particularly relevant for capital-intensive E&P firms facing lumpy drilling expenditures.

### Framework

For each firm-year, two ratios are computed for oil and gas separately:

```
Notional hedge ratio  = Total hedged volume (Bbls or Mcf) / Annual production
Delta hedge ratio     = Total delta-adjusted volume / Annual production
```

Reserve-based ratios also use proved reserves as denominator:

```
Reserve hedge ratio = Delta-adjusted volume / Proved reserves
```

All ratios use the absolute value of the delta (producer hedges are short, yielding negative deltas; the absolute value gives the fraction hedged).

### Delta equivalents

Every instrument is converted to a delta equivalent — the effective number of units of commodity exposure hedged.

**Linear instruments** (fully lock in price; delta = −1.00 per unit):
- `swap`, `fixed_price_contract`, `forward`, `futures`, `volumetric_production_payment`

**Nonlinear instruments** (simplified approximations, following Haushalter 2000):

| Instrument type | Delta per unit | Rationale |
|---|---|---|
| collar | −0.80 | Long put − short call; partial hedge |
| three_way_collar | −0.65 | Collar − additional short put at sub_floor |
| put_option | −0.50 | At-the-money approximation |
| call_option | +0.50 | Long call (typically held by counterparty) |

Black's (1976) commodity option pricing model can be applied if market data (front-month futures price, implied volatility) is provided via `--market-data`. All records in this dataset use `delta_method = "simplified"` because no external market data file was supplied during the calculation.

### Volume allocation to fiscal year

Instruments often span multiple calendar years. Volume is allocated to the relevant fiscal year proportionally by calendar days:

```
allocated_volume = volume_total × (overlap_days / total_period_days)
```

where `overlap_days` = days the instrument period overlaps with the fiscal year.

### Fiscal year determination

The fiscal year for a filing is taken from `production.production.production_year` in the extraction JSON. If that field is null, it is inferred from the filing date:
- Filing date Jan–Jun → fiscal year = filing year − 1
- Filing date Jul–Dec → fiscal year = filing year

### FY+1 denominator

The instruments disclosed in a 10-K typically hedge the *next* year's production, not the year just ended. For example, a 10-K filed in March 2019 reporting FY2018 results will list instruments covering 2019. The denominator used is therefore the next year's production (FY+1).

Implementation: A production lookup table is built across all filings for each company. For a filing with fiscal year Y, the calculator looks up production for year Y+1 from any filing by the same CIK that reports `production_year = Y+1`. If FY+1 production is unavailable, it falls back to the filing's own reported production (year Y).

### Excluded instruments

The following instrument types are tracked in `instrument_deltas.csv` but excluded from the primary hedge ratio:
- **basis_swap** (1,731 instruments): Hedge geographic/quality basis differentials, not outright price risk.
- **swaption** (134 instruments): Options on future swaps; represent optionality, not a current hedge.

Instruments are also excluded if:
- `volume_total` and `volume_per_day` are both null or zero
- The commodity is unknown or mixed
- The unit cannot be converted to canonical units

### Output: seven hedge ratio variants

| Column | Numerator | Denominator | Denominator source |
|---|---|---|---|
| `hedge_ratio_oil_notional` | oil_notional_bbls | oil_production_bbls | FY+1 cross-lookup |
| `hedge_ratio_gas_notional` | gas_notional_mcf | gas_production_mcf | FY+1 cross-lookup |
| `hedge_ratio_oil_delta` | oil_delta_bbls | oil_production_bbls | FY+1 cross-lookup |
| `hedge_ratio_gas_delta` | gas_delta_mcf | gas_production_mcf | FY+1 cross-lookup |
| `hedge_ratio_ngl_delta` | ngl_delta_bbls | ngl_production_bbls | Same-filing data |
| `hedge_ratio_oil_reserve` | oil_delta_bbls | proved_oil_reserves_bbls | Same-filing data |
| `hedge_ratio_gas_reserve` | gas_delta_mcf | proved_gas_reserves_mcf | Same-filing data |

**Note on NGL denominator:** NGL production uses the same-filing reported value rather than the FY+1 cross-lookup. The FY+1 cross-lookup is restricted to records that have oil or gas production to avoid contaminating the oil/gas lookup with NGL-only midstream records. `hedge_ratio_ngl_delta` is non-null for 202 firm-years; 13 are flagged as `ngl_delta_ratio>150%`.

When the denominator is null or zero, the ratio is set to null (not zero, not infinity).

---

## 6. Verification

After initial extraction, all 3,446 filings were verified through a two-tier process.

### Automated checks (6 checks per filing)

1. **Volume math:** `volume_per_day × days_in_period ≈ volume_total` (±20% tolerance)
2. **Period overlap:** No two instruments of the same type and commodity for the same period should overlap implausibly
3. **Direction consistency:** Producer hedges should be "short"; collars should be "collar"
4. **Summary counts:** `hedging_summary.total_instruments` must match `len(instruments)`
5. **Schema completeness:** Required fields present; `sub_floor_price` populated for `three_way_collar`
6. **Price plausibility:** Fixed prices in plausible ranges for the period; no negative volumes

### Manual re-extraction (4 rounds)

Filings that failed automated checks (FAIL) or flagged potential issues (WARN) were re-extracted by re-reading the source filing text and performing a fresh extraction. Four rounds of re-extraction were conducted:

| Round | Filings | FAIL before | FAIL after |
|---|---|---|---|
| 1 | 31 | 144 | 111 |
| 2 | 31 | 111 | 80 |
| 3 | 44 | 80 | 49 |
| 4 | 69+ | 49 | 0 |

Final verification results: **PASS 2,766 / FIXED 638 / WARN 42 / FAIL 0** out of 3,446.

See `DATA_QUALITY.md` for details on remaining WARNs, known corrections, and audit findings.

---

## 7. Airline and Refiner Extraction (Groups 2 and 3)

Groups 2 (airlines) and 3 (refiners) were extracted using the same two-stage pipeline as the E&P
group (Stage 1 deterministic parsing, Stage 2 AI extraction), but with industry-specific extraction
prompts and schema definitions.

### Source data
- **Airlines:** 421 10-K filings from ~26 airline/air cargo entities. Filing dates 1999–2025.
  Parsed into `parsed_extension/{CIK}_{ticker}/{fiscal_year_end}/sections/`.
- **Refiners:** 235 10-K filings from 18 refiner entities. Filing dates 2001–2025.

### Airline extraction schema
The airline prompt instructs the model to extract from Item 8 Notes and related MD&A:
- `fuel_consumption` — actual gallons consumed (MM gallons) and expected gallons for next year
- `hedge_summary` — program existence, estimated % hedged, net fair value
- `annual_hedge_positions[]` — one entry per coverage year: volume in MM gallons,
  `volume_pct_of_consumption` (decimal fraction, e.g. 0.40 = 40%), commodity list,
  instrument type list, fair value
- `carrier_notes` — stopped_hedging flag, owns_refinery flag (Delta/Monroe Energy case)

Prices are intentionally not extracted for airlines: airlines rarely disclose fixed prices for
their collars (only floor/cap ranges), and including unreliable price data would add noise.

**Delta/Monroe Energy:** Delta Air Lines owns Monroe Energy (a refinery). Monroe's refiner hedges
are explicitly excluded from Delta's airline `annual_hedge_positions`; they are NOT fuel hedges.

### Refiner extraction schema
The refiner prompt instructs the model to extract from Item 7A and Item 8 Notes:
- `throughput` — crude oil processed (million barrels), year, and source quote
- `hedge_program` — program existence, primary objective (inventory, crack_spread, mixed),
  designation (cash-flow hedge vs. economic/undesignated)
- `notional_positions[]` — one entry per disclosed instrument × direction × commodity × maturity:
  maturity year, commodity, instrument type, direction (long/short), designation, volume in MBbl
- `fair_values` — total derivative assets and liabilities

Natural gas and precious-metal positions (e.g., Calumet's platinum catalyst hedges) are included
in the raw JSON but excluded from hedge ratio calculations.

### Airline hedge ratio — definition and denominator
```
Airline hedge ratio = volume_gallons_millions (coverage_year) / expected_gallons (coverage_year)
```
Primary ratio: `hedge_ratio_next` — the fraction of coverage_year+1 expected fuel consumption
that is hedged, as of the filing date. This matches the FY+1 convention used for E&P: both groups
report hedges for the next operating year. However, the mechanism differs:
- E&P uses a cross-filing lookup to find next year's actual production (reported in the following year's 10-K)
- Airlines disclose both the hedge volumes AND the expected consumption estimate in the same filing

When `volume_pct_of_consumption` is directly stated in the filing, it is used as-is (preferred).
When only raw volume and expected consumption are disclosed, `computed_pct` = volume / expected.
When expected consumption is not disclosed but actual consumption is, actual consumption serves as
the denominator — an approximation documented in the `warnings` column of affected filings.

### Refiner hedge ratio — definition and caveats
```
net_short_ratio = (total_short_mbbl - total_long_mbbl) / throughput_mbbl
abs_ratio       = (total_short_mbbl + total_long_mbbl) / throughput_mbbl
```
Both ratios exclude natural gas and platinum positions. Natural gas and platinum are included in
the source JSON for completeness but excluded from the ratio because they represent energy input
cost hedges and catalyst procurement hedges, respectively — distinct from the crude-oil and
refined-product price exposure that the ratio is meant to capture.

**Open methodological issue:** The `net_short_ratio` conflates two economically distinct hedging
strategies: (a) inventory hedges — short crude or products futures to protect the value of physical
inventory; (b) crack-spread hedges — a paired long crude / short product position that locks in a
processing margin. These can partially offset each other in the net calculation. `abs_ratio`
(gross turnover) avoids this netting but may double-count the paired legs. Researchers studying
refiner hedging should read the `crack_spread_hedging` field in the source JSON and consider whether
a crack-spread-specific subset of positions is more appropriate for their question.

A positive `net_short_ratio` means the refiner is net short (typical inventory hedge protecting
against a price decline). A negative value means net long (uncommon; indicates the long legs exceed
the short legs, e.g., Par Pacific buying crude forward).

### Airline and refiner verification
Airline and refiner extractions were verified through **manual company-by-company review**: for
each of the 45 entities, the extracted JSON was compared against the source 10-K text (from the
parsed sections) and corrected where errors were found. Known corrections made during verification
are documented in `DATA_QUALITY.md §8`. This verification was less systematic than the E&P automated
pipeline; some errors in less-reviewed filings may remain.

---

## 8. Master Panel and Crosswalk Construction

### master_panel.csv
The master panel is assembled from the three per-group `hedge_ratios.csv` files plus the crosswalk.
It uses a **long format** with one row per `(cik, fiscal_year, hedge_axis)`. The `group` column
for each row is resolved per-row from the crosswalk era lookup (not a static cik→group mapping),
which correctly handles CIK 821483 (Delta Petroleum E&P years vs. Par Pacific refiner years).

Cross-group `hedge_ratio_primary` mapping:

| Group | hedge_axis | hedge_ratio_primary | ratio_method |
|---|---|---|---|
| ep | oil | hedge_ratio_oil_delta | delta |
| ep | gas | hedge_ratio_gas_delta | delta |
| airline | fuel | hedge_ratio_next (or hedge_ratio_current as fallback) | notional_volume |
| refiner | crude_net | net_short_ratio | net_short |

### crosswalk.csv
The crosswalk uses an **era-based grain** — one row per (cik, era_start_fy, era_end_fy) — to handle
companies that changed identity or industry group over time. Multiple era rows for the same CIK
can have different `group` values (e.g., Delta Petroleum → Par Pacific: ep then refiner).

The `entity_id` column groups CIKs that represent the same legal entity across corporate
reorganizations (e.g., the three CIKs associated with Frontier Airlines across its two bankruptcies
share `entity_id = frontier_family_*`). This grouping is based on curated manual research; it is
not comprehensive for the full E&P universe.

---

## 9. Status Taxonomy

The `status` field appears in all three group `hedge_ratios.csv` files and in the master panel.
It encodes the reason a firm-year has a non-null or null ratio — distinguishing structural zeros
from informative missing values:

| Status | Economic meaning | hedge_ratio | has_program |
|---|---|---|---|
| hedger | Firm holds open positions; ratio computed | non-null | true |
| no_position | Program exists; no open positions at year-end | 0.0 | true |
| stopped | Firm has permanently exited hedging | 0.0 | false |
| never | No hedging program; firm does not hedge | null | false |
| pass_through | Fuel cost passed to airline partner (CPA carriers) | null | false |
| not_applicable | Denominator unavailable; ratio uncomputable | null | varies |

This distinction matters for empirical research: a `no_position` firm has a program and chose
zero coverage; a `never` firm has no infrastructure to hedge at all. Pooling these categories
as "zero hedgers" would confound two economically distinct behaviors.

---

## 10. References

- Adam, T.R., & Fernando, C.S. (2006). Hedging, speculation, and shareholder value. *Journal of Financial Economics*, 81(2), 283–309.
- Black, F. (1976). The pricing of commodity contracts. *Journal of Financial Economics*, 3(1–2), 167–179.
- Bond, P., Edmans, A., & Goldstein, I. (2012). The real effects of financial markets. *Annual Review of Financial Economics*, 4, 339–360.
- Chen, Z., Craig, K.A., & Karpovics, M. (2020). Once bitten twice shy? Evidence from the US banking industry during the crash of the energy market. *Energy Economics*, 92, 104981.
- Chen, Z., & Chen, Z. (2025). Learning from the stock market: Abnormal oil price risk exposure, management responses, and consequences. Working paper, Wilson College of Business, University of Northern Iowa.
- Dionne, G., Gueyie, J.-P., & Mnasri, M. (2018). Dynamic corporate risk management: Motivations and real implications. *Journal of Banking & Finance*, 95, 97–111.
- Froot, K.A., Scharfstein, D.S., & Stein, J.C. (1993). Risk management: Coordinating corporate investment and financing policies. *Journal of Finance*, 48(5), 1629–1658.
- Haushalter, G.D. (2000). Financing policy, basis risk, and corporate hedging: Evidence from oil and gas producers. *Journal of Finance*, 55(1), 107–152.
- Jin, Y., & Jorion, P. (2006). Firm value and hedging: Evidence from U.S. oil and gas producers. *Journal of Finance*, 61(2), 893–919.
- Smith, C.W., & Stulz, R.M. (1985). The determinants of firms' hedging policies. *Journal of Financial and Quantitative Analysis*, 20(4), 391–405.
