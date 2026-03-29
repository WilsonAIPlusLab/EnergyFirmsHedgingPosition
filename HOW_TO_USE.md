# How to Use This Dataset

*Last updated: 2026-03-28*

This guide covers loading the data, understanding the schema, joining files, and common analytical queries. Examples are provided in Python and R.

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

## 1. The Instrument Index (Start Here)

`data/instrument_index.csv` is the main filing-level summary. Load it first to understand coverage.

See `METHODOLOGY.md` for the full pipeline description and the research context behind this dataset. The primary intended use cases are panel regressions of hedge ratios on firm-level variables (abnormal oil price exposure, leverage, growth, liquidity) and firm survival analysis linking hedging intensity to financial distress outcomes around the 2014–2016 oil price collapse and 2020 COVID crash.

> **Note on `cik`:** The `cik` column is stored as a string in all CSV files to preserve leading zeros (e.g., CIK "0000077877"). When joining files, ensure both sides are the same type. In pandas, use `dtype={'cik': str}` when reading.

```python
import pandas as pd

idx = pd.read_csv("data/instrument_index.csv", dtype={'cik': str})

# Basic coverage
print(f"Total filings:    {len(idx)}")
print(f"Filings with instruments: {(idx['instrument_count'] > 0).sum()}")
print(f"Total instruments: {idx['instrument_count'].sum():.0f}")
print(f"Companies (CIKs): {idx['cik'].nunique()}")
print(f"Date range: {idx['filing_date'].min()} to {idx['filing_date'].max()}")
```

**Key columns:**

| Column | Type | Description |
|---|---|---|
| `cik` | str | SEC Central Index Key (use as company identifier) |
| `company` | str | Company name (may vary across years for same CIK) |
| `filing_date` | str YYYY-MM-DD | 10-K filing date |
| `accession_number` | str | SEC accession number |
| `uses_derivatives` | bool | Whether the company used commodity derivatives |
| `instrument_count` | int | Total instruments extracted for this filing |
| `total_oil_instruments` | int | Oil instrument count |
| `total_gas_instruments` | int | Gas instrument count |
| `total_ngl_instruments` | int | NGL instrument count |
| `hedging_confidence` | str | "high" / "medium" / "low" |
| `has_production_data` | bool | Whether production data was extracted |

---

## 2. Loading Hedge Ratios

`data/hedge_ratios.csv` provides firm-year hedging intensity metrics.

```python
hr = pd.read_csv("data/hedge_ratios.csv", dtype={'cik': str})

# Fiscal year distribution
print(hr['fiscal_year'].value_counts().sort_index())

# Companies with oil hedge ratios
oil_hedgers = hr[hr['hedge_ratio_oil_notional'].notna() & (hr['hedge_ratio_oil_notional'] > 0)]
print(f"\nOil hedgers: {len(oil_hedgers)} firm-years")
print(f"  Mean:   {oil_hedgers['hedge_ratio_oil_notional'].mean():.1%}")
print(f"  Median: {oil_hedgers['hedge_ratio_oil_notional'].median():.1%}")

# Firms by hedging status
hr['hedges_oil'] = hr['hedge_ratio_oil_notional'].notna() & (hr['hedge_ratio_oil_notional'] > 0)
hr['hedges_gas'] = hr['hedge_ratio_gas_notional'].notna() & (hr['hedge_ratio_gas_notional'] > 0)
```

**Key hedge ratio columns:**

| Column | Description |
|---|---|
| `fiscal_year` | The production year being hedged (not the filing year) |
| `hedge_ratio_oil_notional` | Oil: hedged volume / oil production |
| `hedge_ratio_gas_notional` | Gas: hedged volume / gas production |
| `hedge_ratio_oil_delta` | Oil: delta-adjusted volume / oil production |
| `hedge_ratio_gas_delta` | Gas: delta-adjusted volume / gas production |
| `hedge_ratio_ngl_delta` | NGL: delta-adjusted volume / NGL production (null if no NGL production reported) |
| `hedge_ratio_oil_reserve` | Oil: delta-adjusted volume / proved oil reserves |
| `hedge_ratio_gas_reserve` | Gas: delta-adjusted volume / proved gas reserves |
| `oil_notional_bbls` | Total oil hedged volume (Bbls) |
| `gas_notional_mcf` | Total gas hedged volume (Mcf) |
| `ngl_notional_bbls` | Total NGL hedged volume (Bbls) |
| `oil_delta_bbls` | Oil delta-adjusted volume (Bbls) |
| `gas_delta_mcf` | Gas delta-adjusted volume (Mcf) |
| `ngl_delta_bbls` | NGL delta-adjusted volume (Bbls) |
| `oil_production_bbls` | Annual oil production (Bbls, FY+1 denominator) |
| `gas_production_mcf` | Annual gas production (Mcf, FY+1 denominator) |
| `ngl_production_bbls` | Annual NGL production (Bbls, same-filing denominator) |
| `n_oil_instruments` | Number of oil instruments |
| `n_gas_instruments` | Number of gas instruments |
| `n_ngl_instruments` | Number of NGL instruments |
| `delta_method` | `notional_only` / `linear` / `simplified` — see DATA_QUALITY.md §8 |
| `warnings` | Pipe-separated data quality flags (empty = no issues) |

**Note on null ratios:** Hedge ratios are `NaN` when production data is missing, not zero. A NaN hedge ratio means the denominator is unavailable, not that the company did not hedge.

---

## 3. Loading Individual Extraction JSONs

Each extraction JSON contains the full instrument detail for one filing.

```python
import json, os

# Load all extractions into a list
records = []
ext_dir = "data/extractions"
for fname in os.listdir(ext_dir):
    if fname.endswith(".json"):
        with open(os.path.join(ext_dir, fname)) as f:
            records.append(json.load(f))

print(f"Loaded {len(records)} records")
```

### Instrument schema

Each instrument object in `instruments.instruments` has these fields:

| Field | Type | Description |
|---|---|---|
| `instrument_type` | str | swap / collar / three_way_collar / basis_swap / swaption / put_option / call_option / fixed_price_contract / forward |
| `commodity` | str | crude_oil / natural_gas / ngl |
| `direction` | str | short (producer hedge) / collar / long (put buyer) |
| `volume_per_day` | float | Daily volume (Bbls/d or Mcf/d) |
| `volume_total` | float | Total volume for the period |
| `volume_unit` | str | Bbls / Mcf / Mmbtu / Boe |
| `fixed_price` | float or null | Fixed price for swaps ($/Bbl or $/Mmbtu) |
| `floor_price` | float or null | Put strike price for collars |
| `ceiling_price` | float or null | Call strike price for collars |
| `sub_floor_price` | float or null | Short put strike for three-way collars (sub_floor < floor < ceiling) |
| `price_unit` | str | $/Bbl or $/Mmbtu |
| `period_start` | str YYYY-MM | Hedge coverage start (month) |
| `period_end` | str YYYY-MM | Hedge coverage end (month) |
| `counterparty` | str or null | Counterparty name if disclosed |
| `fair_value_mm` | float or null | Fair value in $ millions if disclosed |
| `notes` | str or null | Source text annotation |

### Flatten instruments to a DataFrame

```python
rows = []
for rec in records:
    instrs = rec.get('instruments', {}).get('instruments', [])
    for instr in instrs:
        row = {
            'cik': rec['cik'],
            'company': rec['company'],
            'filing_date': rec['filing_date'],
        }
        row.update(instr)
        rows.append(row)

instruments_df = pd.DataFrame(rows)
print(f"Total instruments: {len(instruments_df)}")
print(instruments_df['instrument_type'].value_counts())
```

---

## 4. Joining Files

Join `hedge_ratios.csv` with `instrument_index.csv` on `cik` + `filing_date`:

```python
merged = hr.merge(
    idx[['cik', 'filing_date', 'hedging_confidence', 'has_production_data']],
    on=['cik', 'filing_date'],
    how='left'
)
```

Join hedge ratios with instruments DataFrame on `cik` + `filing_date`:

```python
# instruments_df was built above
instr_summary = (instruments_df
    .groupby(['cik', 'filing_date', 'instrument_type'])
    .size()
    .reset_index(name='count'))

analysis = hr.merge(instr_summary, on=['cik', 'filing_date'], how='left')
```

---

## 5. Common Queries

### By company (CIK)

```python
# Vital Energy (CIK 1528129) time series
cik = '1528129'
company_hr = hr[hr['cik'] == cik].sort_values('fiscal_year')
print(company_hr[['fiscal_year', 'hedge_ratio_oil_notional', 'hedge_ratio_gas_notional']])
```

### By year

```python
# Average hedge ratios by fiscal year (oil hedgers only)
yearly = (hr[hr['hedge_ratio_oil_notional'] > 0]
    .groupby('fiscal_year')['hedge_ratio_oil_notional']
    .agg(['mean', 'median', 'count'])
    .reset_index())
print(yearly)
```

### By instrument type

```python
# Breakdown of instrument types
type_counts = instruments_df.groupby(['instrument_type', 'commodity']).size().unstack(fill_value=0)
print(type_counts)
```

### By commodity

```python
# Oil instruments only, swaps
oil_swaps = instruments_df[
    (instruments_df['commodity'] == 'crude_oil') &
    (instruments_df['instrument_type'] == 'swap')
]
print(f"Oil swaps: {len(oil_swaps)}, mean fixed price: ${oil_swaps['fixed_price'].mean():.2f}/Bbl")
```

### High-confidence filings only

```python
high_conf = idx[idx['hedging_confidence'] == 'high']
hr_high = hr[hr['cik'].isin(high_conf['cik'])]
```

### Hedging trend over time

```python
# Fraction of companies hedging oil each year
trend = (hr.groupby('fiscal_year')
    .apply(lambda g: (g['hedge_ratio_oil_notional'] > 0).mean())
    .reset_index(name='fraction_hedging_oil'))
```

### Overhedged firms (ratio > 100%)

```python
overhedged = hr[hr['hedge_ratio_oil_notional'] > 1.0][
    ['cik', 'company', 'fiscal_year', 'hedge_ratio_oil_notional', 'warnings']
].sort_values('hedge_ratio_oil_notional', ascending=False)
# Note: many >100% cases are legitimate (production decline, trust structures, etc.)
# Check the 'warnings' column and DATA_QUALITY.md for context

# Exclude known non-E&P CIKs (see DATA_QUALITY.md section 6)
NON_EP_CIKS = {'1338613', '1166036'}  # always non-E&P
# Also consider filtering 821483 (PAR Pacific, post-2011) and 61398 (Tellurian, post-2016)
ep_only = hr[~hr['cik'].isin(NON_EP_CIKS)]
```

---

## 6. Using the Verification Log

```python
vlog = pd.read_csv("data/verification_log.csv",
                   names=['cik','company','filing_date','instrument_count',
                          'verdict','issue_count','issues_summary','verified_at'])

print(vlog['verdict'].value_counts())
# PASS     2766
# FIXED     638
# WARN       42
# FAIL        0

# All filings with remaining warnings
warns = vlog[vlog['verdict'] == 'WARN']
print(warns[['cik','company','filing_date','issues_summary']])
```

---

## 7. R Code Examples

```r
library(readr)
library(dplyr)
library(jsonlite)

# Load hedge ratios
hr <- read_csv("data/hedge_ratios.csv")

# Oil hedgers summary
hr %>%
  filter(!is.na(hedge_ratio_oil_notional), hedge_ratio_oil_notional > 0) %>%
  summarise(
    n = n(),
    mean_hr = mean(hedge_ratio_oil_notional),
    median_hr = median(hedge_ratio_oil_notional)
  )

# Trend by year
hr %>%
  group_by(fiscal_year) %>%
  summarise(
    frac_hedging = mean(hedge_ratio_oil_notional > 0, na.rm = TRUE),
    mean_hr = mean(hedge_ratio_oil_notional, na.rm = TRUE)
  )

# Load a single extraction JSON
rec <- fromJSON("data/extractions/1528129_2019-02-14.json")
instruments <- as.data.frame(rec$instruments$instruments)
print(instruments[, c("instrument_type","commodity","volume_total","fixed_price","period_start","period_end")])

# Load all extractions (slower — 3,446 files)
ext_files <- list.files("data/extractions", pattern = "\\.json$", full.names = TRUE)
all_instruments <- lapply(ext_files, function(f) {
  rec <- fromJSON(f)
  instrs <- rec$instruments$instruments
  if (length(instrs) == 0 || is.null(instrs)) return(NULL)
  instrs$cik <- rec$cik
  instrs$filing_date <- rec$filing_date
  instrs
}) |> bind_rows()
```

---

## 8. Notes on Data Interpretation

**Instrument periods vs. fiscal year.** Instruments in a 10-K typically cover future production. A filing dated 2019-03-12 for FY2018 will have instruments covering 2019 (and sometimes 2020+). The hedge ratio calculation allocates instrument volume to the appropriate fiscal year.

**Basis swaps.** `basis_swap` instruments are included in `extractions/` but excluded from `hedge_ratios.csv`. They hedge location/quality differentials, not outright price exposure. They appear in `instrument_deltas.csv` with `included=False`.

**Swaptions.** `swaption` instruments are also excluded from hedge ratios. They represent the right to enter into a swap in the future, not a current hedge.

**NGL hedge ratios.** `hedge_ratio_ngl_delta` is included in `hedge_ratios.csv` for 202 firm-years where NGL instruments are present and NGL production is reported. Note that NGL production uses the same-filing denominator (not FY+1 cross-lookup, which is used for oil and gas). 13 firm-years have NGL ratios >150% and are flagged in `warnings`. For 39 firm-years with NGL instruments but no reported NGL production, the ratio is null.

```python
# NGL hedgers
ngl_hedgers = hr[(hr['n_ngl_instruments'] > 0) & hr['hedge_ratio_ngl_delta'].notna()]
print(f"NGL hedge ratios computed: {len(ngl_hedgers)} firm-years")
print(f"Mean NGL HR (excl. flagged): {ngl_hedgers[ngl_hedgers['hedge_ratio_ngl_delta'] <= 1.5]['hedge_ratio_ngl_delta'].mean():.1%}")
```

**Three-way collars.** These have three price levels: `sub_floor_price` < `floor_price` < `ceiling_price`. The company receives: floor price (if market < floor), market price (if floor ≤ market ≤ ceiling), ceiling price (if market > ceiling), and pays back the excess if market < sub_floor. Delta is approximated at −0.65.

**Missing production.** 66 firm-years have hedging instruments but no computable oil or gas hedge ratio. Their oil/gas hedge ratios are null. The `instrument_deltas.csv` still contains their instrument-level details. Most are midstream non-producers, limited partnerships, or royalty trusts without standard E&P production reporting (see DATA_QUALITY.md §7).

**Confidence levels.** The `hedging_confidence` column reflects extraction reliability: "high" = clear quantitative tables found; "medium" = some ambiguity; "low" = limited disclosure or parsing issues.
