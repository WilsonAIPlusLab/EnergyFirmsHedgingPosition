# Oil & Gas Hedging Extraction Pipeline -- Summary

*Last updated: 2026-03-19*

## Overview

Two-stage pipeline to extract commodity hedging data from 4,761 SEC 10-K filings (SIC 1311: Crude Petroleum & Natural Gas).

- **Stage 1**: Deterministic parsing (regex, Python stdlib only) -- section extraction, 238 keyword patterns, hedging confidence classification
- **Stage 2**: AI extraction -- structured instrument and production data, followed by item7 merge, warning verification, and quality audit
  - High-confidence batch: Claude Sonnet 4.5 (Batches API, 1,867 files)
  - Failed/medium/low reruns: Claude Opus 4.6 (interactive, ~1,536 files)
  - Item7 merge, warning verification, quality audit: Claude Sonnet 4.6 agents

See also: `README.md`, `docs/project_review.md`, `docs/item7_merge_summary.md`

Source data: `g:/My Drive/AI+Econ/Oil hedging/edgar_sic1311/10K/` (read-only, 4,761 `.txt` files)
Workspace: `C:/Users/zheng/hedging_workspace/`

---

## Pipeline Files

**Core pipeline scripts (16 files):**

| File | Lines | Description |
|------|------:|-------------|
| `pipeline/config.py` | ~319 | 238 keyword regex patterns in 11 categories |
| `pipeline/stage1_parser.py` | ~802 | SGML/HTML parsing, section extraction, keyword scan, hedging classification |
| `pipeline/stage2_claude_code.py` | ~744 | Claude Code CLI extraction (instruments + production) |
| `pipeline/stage2_ai_extract.py` | ~974 | Claude Batches API extraction (batch + sequential), `--rerun-failed` support |
| `pipeline/stage2_reextract.py` | ~155 | Manifest builder for re-extraction/merge |
| `pipeline/queue_filings.py` | ~136 | Queue helper for processing order |
| `pipeline/write_extraction.py` | ~200 | Helper for writing extraction JSONs |
| `pipeline/verify_extractions.py` | ~584 | 6-check verification + auto-fix |
| `pipeline/fix_warnings.py` | — | Warning inspection utility |
| `pipeline/audit_extractions.py` | — | JSON validity and schema audit |
| `pipeline/audit_data_ranges.py` | — | Data range and anomaly detection |
| `pipeline/audit_data_quality.py` | — | Cross-file and temporal consistency |
| `pipeline/normalize_period_dates.py` | — | YYYY-Qn → YYYY-MM normalization |
| `pipeline/normalize_volume_units.py` | — | Volume unit canonicalization |
| `pipeline/deduplicate_instruments.py` | — | Exact-match instrument deduplication |
| `pipeline/fix_unit_scale_errors.py` | — | Unit/scale error correction |

One-off scripts (generator scripts, merge scripts, research scripts) are in `archive/`.

---

## What Was Done

### Stage 1: Full Parse (Complete, 0 errors)

All 4,761 filings processed through the deterministic parser:
- SGML/HTML stripping with XBRL tag handling and PEM wrapper removal
- Section extraction (Item 1, 1A, 7, 7A, 8 Notes, Oil & Gas Supplement)
- 238 keyword regex scan across 11 categories
- Hedging confidence classification (high/medium/low/none)

**Hedging Confidence Distribution:**

| Confidence | Count | % | Description |
|-----------|------:|--:|-------------|
| high | 1,869 | 40.0% | 5+ instrument keyword types AND position data |
| medium | 774 | 16.6% | 2+ instrument keyword types |
| low | 799 | 17.1% | Some hedging keywords |
| none | 1,230 | 26.3% | No hedging content detected |

**Section Extraction:**

| Section | Found | % |
|---------|------:|--:|
| Item 7A (Market Risk) | 2,981 | 63.8% |
| Item 8 Notes (hedging-related) | 3,254 | 69.6% |
| Oil & Gas Supplement | 1,358 | 29.1% |

Output: `indexes/parsed_index.csv` (4,672 rows)

### Stage 2: AI Extraction (Complete -- All Groups)

All 3,443 filings across high/medium/low confidence groups have been extracted.

**Model by group:**
- High (1,869 filings): `claude-sonnet-4-5-20250929` via Batches API
- Failed reruns (181), medium (774), low (799): Claude Opus 4.6 (interactive)

**High-confidence initial batch (Sonnet 4.5):**

Processed 1,869 filings via Batches API (8 chunks of 500 requests).

| Metric | Count | % of 1,869 |
|--------|------:|----------:|
| Has actual instruments extracted | 997 | 53.3% |
| Confirmed hedger, empty instrument list | 353 | 18.9% |
| Confirmed non-hedger (correctly empty) | 338 | 18.1% |
| Instrument extraction failed (null, later patched) | 181 | 9.7% |
| Has production data | 1,866 | 99.8% |

**Final extraction state (all groups, post all fixes):**

| Metric | Count |
|--------|------:|
| Total extraction files | 3,443 |
| Files with instruments | 961 |
| Files with no instruments (non-hedger or not extractable) | 2,482 |
| Files with production data | 2,528 |
| **Total instruments** | **10,638** |

**Instrument type breakdown (10,638 instruments):**

| Type | Notes |
|------|-------|
| swap | ~51% |
| collar | ~21% |
| basis_swap | ~9% |
| put_option | ~6% |
| three_way_collar | ~5% |
| call_option, fixed_price_contract, swaption, other | remainder |

**Commodity Split (approximate):** ~49% crude oil, ~47% natural gas, ~4% NGL

### Stage 2 Post-Processing (Complete)

Three additional passes were applied after initial extraction. See `docs/item7_merge_summary.md` for full details.

#### Item7 Merge

Discovered that `assemble_instrument_text()` had been skipping `item7.txt` (MD&A), which frequently contains hedging tables for smaller companies that don't file a separate Item 7A. Fix: added `item7.txt` as a secondary context source (up to 40K chars; context budget raised from 80K to 120K). Two-phase incremental merge across all 2,917 files with item7 content:

- P1 (473 files with empty item7a): +239 instruments
- P2 (2,444 files with both sections): +~92 instruments
- Net after merge: 11,003 instruments

#### Warning Verification

Post-merge warnings rose to 794. All 235 unique warning files were checked against original 10-K source sections by Sonnet 4.6 agents. Results:

| Action | Count |
|--------|-------|
| False positives removed (financial derivatives, expired, hallucinated) | ~212 instruments |
| Data recovered from source | ~20 instruments |
| Flags corrected (uses_derivatives, schema) | ~31 files |
| Warnings confirmed as source-data limitations | 338 |

Post-verification: 10,791 instruments, 338 warnings.

#### Quality Audit

Full audit of all 3,443 files by three parallel Sonnet 4.6 agents (JSON/schema validity, data ranges, cross-file consistency):

| Fix category | Files affected |
|---|---|
| standardized_measure_mm / pv10_mm ÷ 1,000 (stored in thousands) | 31 |
| Gas production ÷ 1,000 (Bcf stored as Mcf) | 6 |
| Reserves rescaled | 7 |
| Duplicate instruments removed (153 total) | 52 |
| hedging_summary counts corrected | 80 |
| uses_derivatives flags rationalized | 355 |
| Period dates normalized to YYYY-MM | 138 |
| volume_unit canonicalized | 65 |
| direction values normalized | 18 |
| Negative-price swaps reclassified as basis_swap | 4 |
| Extraction index rebuilt | 1 |

**Final state:** 10,638 instruments, 0 errors, 322 warnings (all verified as source data limitations).

---

## What We Learned

### 1. `max_tokens=4096` truncates large derivative portfolios

181 instrument extractions (9.7%) failed with JSON parse errors -- all caused by the response being cut off at 4096 tokens. Companies with 15-30+ instruments per filing (Chesapeake Energy, Oasis Petroleum, Laredo Petroleum, Whiting Petroleum, Civitas Resources, Panhandle Oil & Gas) consistently exceeded the limit. **Fixed**: `max_tokens` raised to 8192; all 181 re-extracted via Opus 4.6.

### 2. Zero prompt cache hits -- wasted money on repeated prompt text

The first batch sent the full extraction prompt (~400-500 tokens) inside the user message for every request. With 3,718 requests, the identical instruction prefix was paid for at full price each time. The Batches API supports prompt caching via the `system` parameter with `cache_control`, but only if the cached content exceeds 1,024 tokens (Sonnet minimum).

### 3. Two API calls per filing doubled input token costs

Each filing's text was sent twice -- once for instrument extraction, once for production extraction. Most sections (item8_notes, item7) overlap between the two queries. Combining into a single call halves the filing text transmitted.

### 4. Production extraction is near-perfect; instruments are the bottleneck

1,866/1,869 filings (99.8%) returned valid production data. The extraction challenge is on the instrument side -- complex tabular data in varying formats across 15+ years of filing styles.

### 5. 353 filings have confirmed hedging but no extractable instruments

These are companies that describe hedging in prose ("we use derivative instruments to manage commodity price risk") but don't provide a tabular breakdown. The AI correctly identifies `uses_derivatives: true` but returns an empty instruments list. This is a data limitation, not an extraction failure.

### 6. Item 7 (MD&A) is a critical section that was initially skipped

`assemble_instrument_text()` only read item7a.txt and item8_notes.txt. 473 files had an empty item7a.txt but substantive hedging tables in item7.txt -- these companies put all derivative data in MD&A. The gap went unnoticed through the entire high-confidence batch because many companies DO use item7a. **Fixed**: item7.txt added as a secondary context source; incremental merge applied to all 2,917 files.

### 7. Warnings require source verification before acceptance

794 warnings initially looked like extraction failures. After source verification, 456 were resolved: ~212 were false positives (financial derivatives misidentified as commodity hedges, expired contracts, hallucinated instruments), ~20 had recoverable data. Only 338 were genuine source-data limitations. Never assume warning count reflects extraction quality.

### 8. AI unit confusion is systematic for "(in thousands)" table headers

When 10-K tables use "(in thousands)" headers, the AI frequently extracts the raw number without dividing. Affected standardized_measure (31 files), gas production (6 files), reserves (7 files). Build explicit unit-conversion validation into the pipeline, not just post-hoc audits.

---

## Pipeline Optimizations Applied

The following optimizations were implemented before processing the medium/low groups:

### Single-call extraction (instrument + production combined)

`stage2_claude_code.py` uses a single API call per filing requesting both instruments and production data in one JSON response, rather than two separate calls. This halves the filing text transmitted.

### Context budget expansion

`assemble_instrument_text()` raised from 80K to 120K character budget after item7 was added as a third source section:
- item7a.txt: up to 50K chars (primary)
- item7.txt: up to 40K chars (secondary)
- item8_notes.txt: remaining budget (tertiary)

---

## Data Quality Summary

**Current state (final, all fixes applied):**
- 3,443 extraction files across 454 unique companies (CIKs), filings spanning 2005-2025
- 961 filings with instrument-level hedge data; 10,638 individual instruments
- 2,528 filings with production data
- 0 errors, 322 warnings (all verified as source data limitations, not extraction errors)

**Warning breakdown (322 total, all confirmed against source 10-K text):**
- 94 no-volume: source discloses only percentage-of-production or aggregate notional amounts
- 180 no-price: basis swaps without differentials, floating-price marketing contracts, fair-value-only disclosures
- 48 price-outlier: confirmed legitimate (roll swaps, basis differentials, bonus accumulators)
- 0 no-instruments: all resolved

**Normalization applied (no further action needed downstream):**
- Volume units canonicalized: `Bbls`, `Mmbtu`, `Mcf`, `Bcf`, `Boe`, `gallons`, `other`
- Period dates in `YYYY-MM` format throughout
- `direction` values: `short`, `long`, `collar`, `unknown`
- hedging_summary counts match actual instrument arrays

**Known limitations:**
- 322 instruments with missing volume or price -- SEC disclosure does not require per-contract detail
- 2 files have incomplete instrument schemas (missing 7-10 fields each)
- No formal test suite -- `verify_extractions.py` serves as integration test
- No prompt versioning in metadata (cannot determine which prompt version produced which extraction)

---

## Directory Structure

```
hedging_workspace/
├── README.md
├── pipeline/                  # 16 core scripts (see Pipeline Files above)
│   ├── config.py
│   ├── stage1_parser.py
│   ├── stage2_claude_code.py
│   ├── stage2_ai_extract.py
│   ├── stage2_reextract.py
│   ├── queue_filings.py
│   ├── write_extraction.py
│   ├── verify_extractions.py
│   ├── fix_warnings.py
│   ├── audit_extractions.py
│   ├── audit_data_ranges.py
│   ├── audit_data_quality.py
│   ├── normalize_period_dates.py
│   ├── normalize_volume_units.py
│   ├── deduplicate_instruments.py
│   └── fix_unit_scale_errors.py
├── archive/                   # One-off scripts (generators, merge scripts, research)
│   └── generators/            # 166 _gen_*.py filing-specific helpers
├── parsed/                    # Stage 1 output (4,672 filing dirs)
│   └── {CIK}_{Company}/{date}/
│       ├── metadata.json
│       ├── keyword_scan.json
│       └── sections/
│           ├── item1.txt
│           ├── item1a.txt
│           ├── item7.txt
│           ├── item7a.txt
│           ├── item8_notes.txt
│           ├── item8_oil_gas.txt
│           ├── header.txt
│           └── full_text.txt
├── extractions/               # Stage 2 output (3,443 JSONs)
│   └── {CIK}_{date}.json     # Instruments + production per filing
├── extractions_v1_backup/     # Pre-item7-merge snapshot (3,444 files)
├── indexes/
│   ├── parsed_index.csv       # 4,672 rows (Stage 1)
│   └── ai_extraction_index.csv # 3,443 rows (Stage 2)
├── docs/
│   ├── project_review.md      # Quality scorecard and code architecture
│   ├── item7_merge_summary.md # Item7 merge, warning verification, quality audit
│   ├── extraction-patterns.md
│   ├── production-extraction.md
│   └── processing-workflow.md
└── logs/
    ├── stage1.log
    ├── stage2.log
    ├── stage2_batch.log
    ├── verification_report.txt
    ├── warning_manifest.json  # 235 warning files with source section details
    ├── reextract_manifest.json # 3,443 files with section sizes
    ├── merge_p1_manifest.json # 473 empty-item7a files
    └── merge_p2_manifest.json # 2,444 files with both sections
```

---

## Output JSON Schema

Each extraction JSON (`extractions/{CIK}_{date}.json`):

```json
{
  "cik": "1006655",
  "company": "EVOLUTION PETROLEUM CORP",
  "filing_date": "2015-09-11",
  "accession_number": "0001006655-15-000038",
  "instruments": {
    "instruments": [
      {
        "instrument_type": "collar",
        "commodity": "crude_oil",
        "direction": "collar",
        "volume_per_day": 1100,
        "volume_total": null,
        "volume_unit": "Bbls",
        "fixed_price": null,
        "floor_price": 55.0,
        "ceiling_price": 64.05,
        "price_unit": "$/Bbl",
        "period_start": "2015-07",
        "period_end": "2015-12",
        "counterparty": "Cargill, Incorporated",
        "fair_value_mm": -0.109974,
        "notes": "Costless collar..."
      }
    ],
    "hedging_summary": {
      "total_oil_instruments": 1,
      "total_gas_instruments": 0,
      "total_ngl_instruments": 0,
      "uses_derivatives": true,
      "stated_hedging_policy": "..."
    }
  },
  "production": {
    "production": {
      "oil_production_bbls": 448187,
      "gas_production_mcf": null,
      "total_production_boe": 453401,
      "production_year": 2015
    },
    "reserves": {
      "total_proved_reserves_mboe": 12400,
      "proved_developed_reserves_mboe": 7316,
      "proved_undeveloped_reserves_mboe": 5084
    },
    "economics": {
      "standardized_measure_mm": 159.2,
      "pv10_mm": 219,
      "oil_price_used": 71.88,
      "total_production_cost_per_boe": 19.0
    }
  },
  "metadata": {
    "model": "claude-sonnet-4-5-20250929",
    "extraction_timestamp": "2026-02-21T05:51:14.281687",
    "hedging_confidence": "high",
    "item7_merge_timestamp": "2026-02-28T...",   // present if item7 merge was applied
    "warning_fix_timestamp": "2026-02-28T..."    // present if warning-verified
  }
}
```
