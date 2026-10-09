# How to Use This Dataset

*Last updated: 2026-05-24*

This guide covers loading the data, understanding the schema, joining files, and common analytical queries. Examples are provided in Python and R.

---

## Revision History

| Date | Change |
|---|---|
| 2026-05-24 | v2.0: rewrote for three-group dataset (E&P, airline, refiner). New layout, master panel, crosswalk, status taxonomy. Old E&P-only examples updated to new file paths |
| 2026-03-28 | Added NGL hedge ratio columns to schema table. Updated null production count to 66 firm-years. NGL stats: 202 firm-years, 39 without NGL production |
| 2026-03-23 | Initial release |

---

## Prerequisites

**Python**
```bash
pip install pandas
```

**R**
```r
install.packages(c("jsonlite", "dplyr", "readr"))
```

---

## Dataset layout (v2.0)

```
data/
  ep/
    hedge_ratios.csv      E&P firm-year panel (2,963 rows)
    positions.csv         E&P instrument-level delta records (14,538 rows)
    filings.csv           E&P per-filing summary (3,446 rows)
    filing_index.csv      E&P parse superset, incl. non-hedgers (4,672 rows)
    verification_log.csv  Extraction verdict per filing
    extractions/          3,446 E&P JSON files
  airline/
    hedge_ratios.csv      Airline firm-year panel (601 rows, version 2)
    positions.csv         Forward coverage schedule, one row per coverage-year (344 rows)
    filings.csv           Airline per-filing summary (603 reports)
    extractions/          (v2: see v2_support/ and docs_v2/)
  refiner/
    hedge_ratios.csv      Refiner firm-year panel (235 rows)
    positions.csv         Un-aggregated notional positions (696 rows)
    filings.csv           Refiner per-filing summary (235 rows)
    extractions/          235 refiner JSON files
  crosswalk.csv           CIK <-> ticker <-> company crosswalk with era support
  master_panel.csv        Harmonized long-format cross-group panel (6,582 rows)
```

> **`cik` type note:** Store `cik` as a string in all files. Use `dtype={'cik': str}` in pandas
> and `col_types = cols(cik = col_character())` in R to prevent leading-zero truncation.

---

## 1. Start with the Master Panel

`data/master_panel.csv` provides one row per `(cik, fiscal_year, hedge_axis)` across all three
groups. It is the right starting point for cross-group comparisons and pooled analyses.

```python
import pandas as pd

panel = pd.read_csv("data/master_panel.csv", dtype={'cik': str})
print(f"Total rows: {len(panel)}")
print(panel.groupby(['group', 'hedge_axis']).size())
```

**CRITICAL:** `hedge_ratio_primary` values are **not comparable across groups**. Each group uses a
different denominator and a different ratio definition:

| group | hedge_axis | hedge_ratio_primary | denominator |
|---|---|---|---|
| ep | oil | delta-adjusted volume / oil production | oil_production_bbls (FY+1 lookup) |
| ep | gas | delta-adjusted volume / gas production | gas_production_mcf (FY+1 lookup) |
| airline | fuel | forward coverage pct | expected fuel consumption (gallons) |
| refiner | crude_net | net-short / crude throughput | throughput_mbbl |

Do not average or pool `hedge_ratio_primary` across groups without transforming to a common scale.
Within a group, ratios are defined consistently and can be used in panel regressions.

### Status taxonomy

The `status` column classifies every observation, including zeros and true missing values:

| status | hedge_ratio_primary | has_program | Meaning |
|---|---|---|---|
| `hedger` | > 0 | true | Active positions; ratio computed |
| `no_position` | 0.0 | true | Program exists; zero position this axis-year |
| `stopped` | 0.0 | false | Permanently exited hedging program |
| `never` | null | false | No derivative program disclosed |
| `pass_through` | null | false | Fuel cost passed to contracting carrier (airlines only) |
| `not_applicable` | null | — | Denominator missing; ratio cannot be computed |

**Why this matters:** `ratio = null` covers two economically distinct cases — `never` (company
does not hedge) and `not_applicable` (data gap). Collapsing both to zero or dropping both will
introduce bias. Use `status` to separate them before constructing a panel variable.

```python
# Build a panel variable that is: ratio for hedgers, 0 for non-hedgers, NaN for missing denom
panel['hr_clean'] = panel.apply(lambda r: (
    r['hedge_ratio_primary'] if r['status'] == 'hedger'
    else 0.0 if r['status'] in ('no_position', 'stopped', 'never')
    else float('nan')
), axis=1)
```

---

## 2. E&P Group

### Hedge ratios

```python
ep_hr = pd.read_csv("data/ep/hedge_ratios.csv", dtype={'cik': str})

# Fiscal year distribution
print(ep_hr['fiscal_year'].value_counts().sort_index())

# Oil hedgers (delta-adjusted)
oil_hedgers = ep_hr[ep_hr['hedge_ratio_oil_delta'].notna() & (ep_hr['hedge_ratio_oil_delta'] > 0)]
print(f"Oil hedgers: {len(oil_hedgers)} firm-years")
print(f"  Mean:   {oil_hedgers['hedge_ratio_oil_delta'].mean():.1%}")
print(f"  Median: {oil_hedgers['hedge_ratio_oil_delta'].median():.1%}")
```

**Key columns:**

| Column | Description |
|---|---|
| `cik`, `company`, `fiscal_year`, `filing_date` | Identifiers |
| `uses_derivatives` | True / False — whether company disclosed commodity derivatives |
| `hedge_ratio_oil_delta` | Oil: delta-adjusted hedged volume / oil production (FY+1) |
| `hedge_ratio_gas_delta` | Gas: delta-adjusted hedged volume / gas production (FY+1) |
| `hedge_ratio_oil_notional` | Oil: simple notional (no delta adjustment) |
| `hedge_ratio_gas_notional` | Gas: simple notional |
| `hedge_ratio_ngl_delta` | NGL: delta-adjusted / NGL production (same-filing denominator) |
| `hedge_ratio_oil_reserve` | Oil: delta-adjusted / proved oil reserves |
| `hedge_ratio_gas_reserve` | Gas: delta-adjusted / proved gas reserves |
| `oil_delta_bbls` | Delta-adjusted oil volume (Bbls) |
| `gas_delta_mcf` | Delta-adjusted gas volume (Mcf) |
| `oil_production_bbls` | Annual oil production used as denominator |
| `gas_production_mcf` | Annual gas production used as denominator |
| `n_oil_instruments` | Number of oil instruments in this filing |
| `n_gas_instruments` | Number of gas instruments |
| `n_ngl_instruments` | Number of NGL instruments |
| `delta_method` | `notional_only` / `linear` / `simplified` |
| `warnings` | Pipe-separated quality flags (empty = no issues) |

**Note on null hedge ratios:** A null ratio means the production denominator is unavailable, not
that the company did not hedge. Always check `uses_derivatives` and `n_oil_instruments` /
`n_gas_instruments` alongside a null ratio.

```python
# Null oil ratio: distinguish "no production data" from "no instruments"
ep_hr['null_reason'] = ep_hr.apply(lambda r: (
    'no_denominator' if r['hedge_ratio_oil_delta'] != r['hedge_ratio_oil_delta']
    and r['n_oil_instruments'] > 0
    else 'not_hedged' if r['hedge_ratio_oil_delta'] != r['hedge_ratio_oil_delta']
    else 'ok'
), axis=1)
```

### Positions (instrument-level)

```python
ep_pos = pd.read_csv("data/ep/positions.csv", dtype={'cik': str})
print(ep_pos['instrument_type'].value_counts())
print(ep_pos['commodity'].value_counts())
```

### E&P extraction JSONs

```python
import json, os

ext_dir = "data/ep/extractions"
records = []
for fname in os.listdir(ext_dir):
    if fname.endswith(".json"):
        with open(os.path.join(ext_dir, fname)) as f:
            records.append(json.load(f))

# Flatten instruments
rows = []
for rec in records:
    instrs = rec.get('instruments', {}).get('instruments', [])
    for instr in instrs:
        row = {'cik': rec['cik'], 'company': rec['company'], 'filing_date': rec['filing_date']}
        row.update(instr)
        rows.append(row)

instruments_df = pd.DataFrame(rows)
print(f"Total instruments: {len(instruments_df)}")
```

### Instrument schema (E&P)

| Field | Type | Description |
|---|---|---|
| `instrument_type` | str | swap / collar / three_way_collar / basis_swap / swaption / put_option / call_option |
| `commodity` | str | crude_oil / natural_gas / ngl |
| `direction` | str | short / collar / long |
| `volume_per_day` | float | Daily rate (Bbls/d or Mcf/d) |
| `volume_total` | float | Total for the period |
| `volume_unit` | str | Bbls / Mcf / Mmbtu / Boe |
| `fixed_price` | float | Swap fixed price ($/Bbl or $/Mmbtu) |
| `floor_price` | float | Put strike for collars |
| `ceiling_price` | float | Call strike for collars |
| `sub_floor_price` | float | Three-way collar short-put strike |
| `period_start` | str YYYY-MM | Coverage start |
| `period_end` | str YYYY-MM | Coverage end |

### Verification log (E&P only)

```python
vlog = pd.read_csv("data/ep/verification_log.csv",
                   names=['cik','company','filing_date','instrument_count',
                          'verdict','issue_count','issues_summary','verified_at'])
print(vlog['verdict'].value_counts())
# PASS     2766
# FIXED     638
# WARN       42
```

---

## 3. Airline Group

### Hedge ratios

```python
air_hr = pd.read_csv("data/airline/hedge_ratios.csv", dtype={'cik': str})

# Coverage rate by carrier
summary = (air_hr.groupby('company')
    .agg(n=('fiscal_year','count'),
         mean_next=('hedge_ratio_next','mean'),
         median_next=('hedge_ratio_next','median'))
    .round(3))
print(summary)
```

**Key columns:**

| Column | Description |
|---|---|
| `cik`, `company`, `ticker`, `fiscal_year`, `filing_date` | Identifiers |
| `hedge_ratio_next` | Forward coverage pct for FY+1 (primary ratio) |
| `hedge_ratio_current` | Forward coverage pct for the current fiscal year |
| `headline_coverage` | Which year drove the headline: `next` / `current` / `none` |
| `fuel_consumption_gallons_mm` | Actual fuel consumed in fiscal year (millions of gallons) |
| `expected_gallons_millions` | Expected fuel consumption disclosed for FY+1 (denominator) |
| `net_fair_value_mm` | Fair value of hedge portfolio ($ millions) |
| `has_program` | True / False |
| `stopped_hedging` | True if carrier permanently exited hedging |
| `stopped_hedging_year` | Year of permanent exit (if applicable) |
| `owns_refinery` | True for Delta (Monroe Energy) |
| `status` | hedger / no_position / stopped / never / pass_through / not_applicable |
| `warnings` | Pipe-separated quality flags |

**Hedge ratio interpretation:** `hedge_ratio_next` is a fraction of the next fiscal year's
expected fuel consumption that is covered by derivatives as of the filing date. A value of 0.60
means 60% of next-year consumption is hedged. Values above 1.0 are possible (e.g., if coverage
volume was estimated against a stale consumption forecast) but unusual.

```python
# Stopped hedgers: AAL (2013+) and UAL (2016+) have status='stopped', ratio=0.0
stopped = air_hr[air_hr['status'] == 'stopped']
print(stopped[['company', 'fiscal_year', 'hedge_ratio_next', 'status']])

# Pass-through carriers: ratio is null (not structural zero)
pass_thru = air_hr[air_hr['status'] == 'pass_through']
print(pass_thru['company'].unique())
```

### Positions (forward schedule)

```python
air_pos = pd.read_csv("data/airline/positions.csv", dtype={'cik': str})
# One row per (cik, fiscal_year, coverage_year)
# coverage_year > fiscal_year for forward hedges

# Southwest's full forward schedule in FY2005
sw = air_pos[(air_pos['ticker'] == 'LUV') & (air_pos['fiscal_year'] == 2005)]
print(sw[['coverage_year','volume_pct_of_consumption','volume_gallons_millions',
          'expected_gallons_millions','instrument_types']])
```

---

## 4. Refiner Group

### Hedge ratios

```python
ref_hr = pd.read_csv("data/refiner/hedge_ratios.csv", dtype={'cik': str})

# Net-short ratios
hedgers = ref_hr[ref_hr['status'] == 'hedger']
print(f"Refiner hedgers: {len(hedgers)} firm-years")
print(f"Mean net_short_ratio: {hedgers['net_short_ratio'].mean():.3f}")
print(f"Mean abs_ratio:       {hedgers['abs_ratio'].mean():.3f}")
```

**Key columns:**

| Column | Description |
|---|---|
| `cik`, `company`, `ticker`, `fiscal_year`, `filing_date` | Identifiers |
| `throughput_mbbl` | Annual crude throughput (thousands of barrels) |
| `crude_throughput_mmbbl` | Annual crude throughput (millions of barrels) |
| `total_long_mbbl` | Sum of long positions (MBbl) |
| `total_short_mbbl` | Sum of short positions (MBbl) |
| `net_short_mbbl` | `total_short_mbbl - total_long_mbbl` (positive = net short) |
| `total_abs_mbbl` | `total_long_mbbl + total_short_mbbl` |
| `net_short_ratio` | `net_short_mbbl / throughput_mbbl` (primary ratio) |
| `abs_ratio` | `total_abs_mbbl / throughput_mbbl` (secondary; may be more appropriate for some analyses) |
| `primary_objective` | Disclosed purpose: `cash_flow` / `economic` / `mixed` / unknown |
| `has_cash_flow_hedges` | True if ASC 815 cash flow hedge designation disclosed |
| `has_economic_hedges` | True if economic hedge disclosed |
| `status` | hedger / no_position / never / not_applicable |
| `warnings` | Pipe-separated flags |

**Ratio caveat:** `net_short_ratio` conflates crude inventory hedges with crack-spread hedges
(long crude / short product). A refiner that is long crude and short products will show a low or
negative net-short ratio even if it is substantially hedged. `abs_ratio` avoids this netting.
See `METHODOLOGY.md §7.2` for discussion.

### Positions (un-aggregated)

```python
ref_pos = pd.read_csv("data/refiner/positions.csv", dtype={'cik': str})
# One row per notional position (un-aggregated)
print(ref_pos[['cik','company','fiscal_year','maturity_year','commodity',
               'instrument','direction','volume_mbbl']].head(20))
```

---

## 5. Crosswalk

`data/crosswalk.csv` resolves identities across time. One row per `(cik, era)`.

```python
cw = pd.read_csv("data/crosswalk.csv", dtype={'cik': str})

# Join crosswalk onto master panel by cik and fiscal year
def crosswalk_merge(df, cw):
    merged = df.merge(cw, on='cik', how='left')
    # Keep only rows where fiscal_year falls in [era_start_fy, era_end_fy]
    mask = (
        (merged['fiscal_year'] >= merged['era_start_fy']) &
        (merged['fiscal_year'] <= merged['era_end_fy'])
    )
    return merged[mask].drop(columns=['era_start_fy','era_end_fy'])

panel_cw = crosswalk_merge(panel, cw)
```

**Key columns:**

| Column | Description |
|---|---|
| `cik` | SEC CIK (string with leading zeros) |
| `era_start_fy` | First fiscal year this row applies to |
| `era_end_fy` | Last fiscal year this row applies to (9999 = present) |
| `canonical_ticker` | Primary exchange ticker for this era |
| `company_name` | Canonical company name for this era |
| `group` | `ep` / `airline` / `refiner` |
| `entity_id` | Stable identity string across CIK changes (e.g., Frontier family) |
| `lineage` | Free-text note on M&A or name changes |
| `business_model` | `standard` / `pass_through` / `royalty_trust` / etc. |

**Multi-CIK entities:** Frontier Airlines went through three bankruptcy reorganizations under
different CIKs. All three share the same `entity_id = frontier_family_*` string. Use `entity_id`
to group the family for time-series analysis.

**CIK 821483 (Delta Petroleum → Par Pacific):** This CIK has two crosswalk rows — one E&P era
and one refiner era. `group` resolves per row; always join crosswalk on `cik` AND fiscal year,
not on `cik` alone.

---

## 6. Common Queries

### E&P: firm time series

```python
# Vital Energy (CIK 1528129) — oil hedge ratio time series
ep_hr_ve = ep_hr[ep_hr['cik'] == '1528129'].sort_values('fiscal_year')
print(ep_hr_ve[['fiscal_year', 'hedge_ratio_oil_delta', 'hedge_ratio_gas_delta']])
```

### E&P: industry average hedge ratio by year

```python
yearly = (ep_hr[ep_hr['hedge_ratio_oil_delta'] > 0]
    .groupby('fiscal_year')['hedge_ratio_oil_delta']
    .agg(['mean', 'median', 'count'])
    .reset_index())
print(yearly)
```

### Airline: Southwest coverage over time

```python
sw = air_hr[air_hr['ticker'] == 'LUV'].sort_values('fiscal_year')
print(sw[['fiscal_year', 'hedge_ratio_next', 'expected_gallons_millions', 'status']])
```

### Airline: fraction hedging by year

```python
air_trend = (air_hr[~air_hr['status'].isin(['pass_through'])]
    .groupby('fiscal_year')
    .apply(lambda g: (g['status'] == 'hedger').mean())
    .reset_index(name='frac_hedging'))
```

### Refiner: Valero through time

```python
val = ref_hr[ref_hr['ticker'] == 'VLO'].sort_values('fiscal_year')
print(val[['fiscal_year', 'net_short_ratio', 'abs_ratio', 'throughput_mbbl', 'status']])
```

### Master panel: E&P oil only, delta method, high confidence

```python
ep_oil = panel[
    (panel['group'] == 'ep') &
    (panel['hedge_axis'] == 'oil') &
    (panel['ratio_method'] == 'delta') &
    (panel['status'].isin(['hedger', 'no_position', 'never']))
].copy()
```

### Overhedged E&P firms (ratio > 100%)

```python
overhedged = ep_hr[ep_hr['hedge_ratio_oil_delta'] > 1.0][
    ['cik', 'company', 'fiscal_year', 'hedge_ratio_oil_delta', 'warnings']
].sort_values('hedge_ratio_oil_delta', ascending=False)
# Many > 100% cases are legitimate (production decline, trust structures)
# See DATA_QUALITY.md §6 for full discussion

# Exclude known non-E&P CIKs
NON_EP_CIKS = {'1338613', '1166036'}
ep_only = ep_hr[~ep_hr['cik'].isin(NON_EP_CIKS)]
```

### Cross-group: fraction of firm-years with active hedging

```python
# Use status to classify; exclude pass_through and not_applicable from denominator
active_statuses = {'hedger', 'no_position', 'stopped', 'never'}
frac = (panel[panel['status'].isin(active_statuses)]
    .groupby(['group', 'fiscal_year'])
    .apply(lambda g: (g['status'] == 'hedger').mean())
    .reset_index(name='frac_hedging'))
```

---

## 7. R Code Examples

```r
library(readr)
library(dplyr)
library(jsonlite)

# Load E&P hedge ratios
ep_hr <- read_csv("data/ep/hedge_ratios.csv", col_types = cols(cik = col_character()))

# Oil hedgers summary
ep_hr %>%
  filter(!is.na(hedge_ratio_oil_delta), hedge_ratio_oil_delta > 0) %>%
  summarise(
    n = n(),
    mean_hr = mean(hedge_ratio_oil_delta),
    median_hr = median(hedge_ratio_oil_delta)
  )

# Load master panel
panel <- read_csv("data/master_panel.csv", col_types = cols(cik = col_character()))

# E&P oil panel
ep_oil <- panel %>%
  filter(group == "ep", hedge_axis == "oil",
         status %in% c("hedger","no_position","never")) %>%
  mutate(hr_clean = case_when(
    status == "hedger"     ~ hedge_ratio_primary,
    status %in% c("no_position","never") ~ 0,
    TRUE ~ NA_real_
  ))

# Airline hedging trend
air_hr <- read_csv("data/airline/hedge_ratios.csv", col_types = cols(cik = col_character()))
air_hr %>%
  filter(!status %in% c("pass_through")) %>%
  group_by(fiscal_year) %>%
  summarise(frac_hedging = mean(status == "hedger", na.rm = TRUE))

# Load a single E&P extraction JSON
rec <- fromJSON("data/ep/extractions/1528129_2019-02-14.json")
instruments <- as.data.frame(rec$instruments$instruments)
print(instruments[, c("instrument_type","commodity","volume_total","fixed_price","period_start","period_end")])
```

---

## 8. Joining Files

### E&P: hedge ratios + filing confidence

```python
ep_fi = pd.read_csv("data/ep/filings.csv", dtype={'cik': str})

ep_merged = ep_hr.merge(
    ep_fi[['cik', 'filing_date', 'hedging_confidence', 'has_production_data']],
    on=['cik', 'filing_date'],
    how='left'
)

# High-confidence filings only
ep_high = ep_merged[ep_merged['hedging_confidence'] == 'high']
```

### Master panel + crosswalk

```python
# Full identity enrichment for master panel
cw = pd.read_csv("data/crosswalk.csv", dtype={'cik': str})

panel_enriched = (panel
    .merge(cw, on='cik', how='left')
    .query("fiscal_year >= era_start_fy and fiscal_year <= era_end_fy or era_start_fy.isna()")
    .drop(columns=['era_start_fy','era_end_fy']))
```

---

## 9. Notes on Data Interpretation

**Fiscal year vs. filing year.** E&P and refiner hedge ratios are indexed to the fiscal year of
production or throughput being hedged. A 10-K filed in March 2019 for FY2018 typically discloses
hedges for calendar year 2019 — those are indexed to fiscal_year=2019 (the production year being
hedged), not 2018 (the reporting year). Airlines are indexed to the fiscal year of the filing
(the year the forward coverage applies to FY+1).

**Basis swaps (E&P).** `basis_swap` instruments appear in `data/ep/extractions/` and
`data/ep/positions.csv` but are excluded from `data/ep/hedge_ratios.csv`. They hedge
location/quality price differentials, not outright commodity price exposure.

**Swaptions (E&P).** Excluded from hedge ratios. They represent the right to enter a future
swap, not a current hedge position.

**NGL hedge ratios (E&P).** `hedge_ratio_ngl_delta` uses the same-filing NGL production
denominator (not the FY+1 cross-lookup used for oil and gas) because the FY+1 lookup introduced
measurement errors for midstream entities. 13 of 202 NGL firm-years are flagged `ngl_delta_ratio>150%`
in `warnings`. For 39 firm-years with NGL instruments but no NGL production reported, ratio is null.

**Three-way collars (E&P).** Three price levels: `sub_floor_price` < `floor_price` < `ceiling_price`.
Delta approximated at −0.65 per Haushalter (2000).

**Refiner net-short vs. abs ratio.** `net_short_ratio` nets long and short positions. A refiner
with offsetting long-crude / short-product positions will show a near-zero net-short ratio even
if its total hedged volume is large. For total hedging scale, prefer `abs_ratio`.

**Pass-through carriers (airline).** Mesa Air, SkyWest Airlines, Republic Airways, and ATSG
operate under capacity purchase agreements; jet fuel is purchased and reimbursed by the
contracting major. These carriers do not hedge fuel on their own account. Their `status =
pass_through` and all ratio columns are null. This is not a data gap; it is the economically
correct representation.

**Stopped hedging (airline).** American Airlines (AAL) permanently exited commodity hedging
after FY2012; United Airlines (UAL) after FY2015. Their subsequent firm-years have `status =
stopped`, `hedge_ratio_primary = 0.0`, `has_program = false`. Do not confuse this structural
zero with `no_position` (program exists, zero coverage this year).

**Missing production denominators (E&P).** 66 E&P firm-years have no oil or gas production
denominator; their hedge ratios are null. Most are midstream non-producers, royalty trusts, or
most-recent fiscal years where the FY+1 production filing is not yet available. Instrument
details remain in `data/ep/extractions/` and `data/ep/positions.csv`.

**Cross-group pooling.** `master_panel.csv` puts all groups in one file for convenience.
When pooling across groups, always control for `group` and `hedge_axis`. The ratios are not
on a common scale; a 50% airline fuel coverage and a 50% E&P oil delta ratio are produced
by different economic behaviors and are not directly comparable.
