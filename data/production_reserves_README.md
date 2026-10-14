# production_reserves.csv  (data/production_reserves.csv)

One row per extracted 10-K (9,077 filings, 1,231 firms, fiscal years 1992-2026). Built by `builders/build_production_reserves.py` from `extractions_all_merged/*.json` (same filings as `hedge_ratios.csv`). Values are copied as extracted; nothing is imputed or corrected. Column `source_tier` marks the filing's group (`ep_v2.0`, `pre2010A`, `update_2026`, `tierBCD`). Key: `cik` + `accession_number` (3 filings have no accession in the JSON). 367 cik-fiscal_year pairs have more than one filing (amendments, late filings).

| Block | Columns | Units |
|---|---|---|
| ids | cik, company, fiscal_year, fiscal_year_source, filing_date, accession_number, production_year, reserves_year_end | fiscal_year from the hedge-ratio panel (8,563 rows), else year of reserves date or production year (230), else missing (284, mostly filings with no data) |
| production | oil_production_bbls, gas_production_mcf, ngl_production_bbls, total_production_boe, oil_production_bpd, gas_production_mmcf_per_day, total_production_boepd | annual bbl / mcf / boe; per-day columns as reported |
| reserves | proved_oil_reserves_mbbls, proved_gas_reserves_mmcf, proved_ngl_reserves_mbbls, total_proved_reserves_mboe, proved_developed_reserves_mboe, proved_undeveloped_reserves_mboe | Mbbl / MMcf / Mboe as extracted |
| reserves (panel units) | proved_oil_reserves_bbls, proved_gas_reserves_mcf, total_proved_reserves_boe | same values x 1,000, identical to `hedge_ratios.csv` (100% equal) |
| economics | standardized_measure_mm, pv10_mm, oil_price_used, gas_price_used, lease_operating_expense_per_boe, production_tax_per_boe, gathering_transport_per_boe, total_production_cost_per_boe, dd_and_a_per_boe, finding_development_cost_per_boe | USD million; USD per bbl / mcf / boe |
| coverage | has_production, has_reserves, has_economics | production 4,551; reserves 3,936; economics 4,299 filings |
| checks | flags, extraction_method, verification_status, source_file | see below |

## Flags (information only, values are not changed)
nonstandard_schema 192 (different field names; core columns empty) | sm_zero_placeholder 30 (standardized measure 0 with no PV-10: treat as missing) | reserves_sum_vs_total 33 | pd_pu_vs_total 14 | oil_bbls_vs_bpd 18 | gas_mcf_vs_mmcfd 17 (annual and per-day figures disagree by more than 40%) | production_scale_outlier 2 (ConocoPhillips FY2000 gas 6.2e11 mcf and Hess FY2018 gas 2.2e11 mcf: x1000 unit errors).

## Known limits
- **Not the same production as `hedge_ratios.csv`.** That panel uses the NEXT year's production (fiscal_year+1, from another filing) as the hedge-ratio denominator; this table has each filing's own year. Only 23% of oil-production values match for that reason, not because of errors.
- Majors (Exxon, Chevron, ConocoPhillips) are in the sample: very large values are real unless flagged.
- Not hand-verified field by field: the 10-K reads were quote-verified for hedge instruments; production, reserves and economics fields carry the extraction's own `verification_status`.
- Pre-2010 and tier B-D filings have fewer reserve and economics fields (excerpt-based reads).
