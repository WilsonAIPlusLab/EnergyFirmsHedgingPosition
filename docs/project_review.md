# Hedging Workspace — Project Quality Review

*Last updated: 2026-03-19*

## 1. Executive Summary

- Project extracts structured hedging instrument + production data from 3,443 SEC 10-K filings for oil & gas companies (SIC 1311)
- Two-stage pipeline: (1) deterministic SGML/HTML parsing with keyword classification, (2) AI-powered extraction using Claude Sonnet 4.5 (batch) and Claude Opus 4.6 (interactive)
- 10,638 total instruments extracted, 0 errors, 322 verified warnings (all confirmed as source data limitations)
- 454 unique companies (CIKs), filings spanning 2005-2025

## 2. Project Scorecard

| Dimension | Grade | Summary |
|-----------|-------|---------|
| Data Correctness | A | 0 errors, 322 verified warnings, 10,638 instruments, full provenance chain |
| Documentation | C+ | One excellent post-mortem, no README (until now), no setup guide |
| Reproducibility | D | No requirements.txt, no Python version pinned, no env docs, no git (until now) |
| Code Quality (core) | B+ | Core 8 scripts: good error handling, logging, docstrings, portable paths |
| Code Quality (one-offs) | D | 166 generator scripts + 96 root scripts: hardcoded paths, no tests |
| Version Control | F→B | Was: not a git repo. Now: initialized with .gitignore |
| Testing | D- | 5 informal test files, no test framework, no CI |
| Portability | D | 367 hardcoded C:/Users/zheng occurrences across 187 files; core scripts portable |
| Audit Trail | B | Metadata in 100% of files; model+timestamp present; gaps in batch_id (54%) |

## 3. Data Inventory

### 3.1 Extraction Files (extractions/)

- 3,443 JSON files, ~21 MB total
- Naming: {CIK}_{YYYY-MM-DD}.json
- Schema: cik, company, filing_date, accession_number, instruments{}, production{}, metadata{}
- 10,638 total instruments across 961 files with instruments
- 2,482 files have no instruments (uses_derivatives=false or empty)
- 2,528 files have production data

### 3.2 Model Distribution

- 1,867 files: claude-sonnet-4-5-20250929 (batch API, high-confidence group)
- ~1,536 files: claude-opus-4-6 (interactive extraction, failed/medium/low groups)
- 40 files: model field missing (older extractions)

### 3.3 Confidence Distribution

- High: 1,869 filings (Sonnet 4.5 batch)
- Medium: 774 filings (Opus 4.6 interactive)
- Low: 799 filings (Opus 4.6 interactive)
- None: 1,230 filings parsed but not extracted (no hedging indicators)

### 3.4 Indexes

- parsed_index.csv: 4,672 rows, 20 columns (all parsed filings)
- ai_extraction_index.csv: 3,443 rows, 12 columns (all extractions)
- Both consistent with extraction files (verified by verify_extractions.py)

### 3.5 Parsed Filings (parsed/)

- 4,672 filings across 640 company directories
- Each filing has: metadata.json, keyword_scan.json, sections/ (8 text files)
- Sections: full_text.txt, header.txt, item1.txt, item1a.txt, item7.txt, item7a.txt, item8_notes.txt, item8_oil_gas.txt

### 3.6 Supporting Files

- logs/: verification_report.txt, warning_manifest.json, reextract_manifest.json, merge manifests
- extractions_v1_backup/: 3,444 files, pre-item7-merge snapshot
- pipeline/: 8 core scripts + 166 generator scripts + audit/normalize/fix scripts

## 4. Verification Results

### 4.1 Current State (after all fixes)

```
Errors:   0
Warnings: 322
  - 94  no-volume    (source discloses % of production or notional only)
  - 180 no-price     (basis swaps without differentials, floating-price, fair-value-only)
  - 48  price-outlier (confirmed legitimate: roll swaps, basis differentials, accumulators)
  - 0   no-instruments (all resolved)
```

### 4.2 Verification Pipeline (6 checks)

1. JSON Integrity: 3,443/3,443 pass
2. Schema Consistency: 3,440/3,443 pass (3 have minor field gaps)
3. Index-to-JSON Consistency: 3,443/3,443 match
4. Logical Consistency: 0 contradictions (uses_derivatives vs instruments)
5. Cross-Index: All 3,443 matched to parsed_index
6. Coverage: 3,443 of 4,672 parsed (1,229 "none" confidence correctly skipped)

### 4.3 Fixes Applied During Quality Audit

- 31 files: standardized_measure_mm / pv10_mm divided by 1,000 (stored in thousands not millions)
- 6 files: gas production divided by 1,000 (Bcf stored as Mcf)
- 7 files: reserves divided by 1,000 or 1,000,000 (wrong scale)
- 52 files: 153 duplicate instruments removed
- 80 files: hedging_summary counts corrected
- 355 files: uses_derivatives flag rationalized (13→false, 342 kept true with note)
- 138 files: period dates normalized to YYYY-MM (from YYYY-Qn, YYYY-MM-DD, YYYY)
- 65 files: volume_unit normalized to canonical values
- 18 files: direction values normalized (fixed_for_float→short, null→unknown)
- 4 files: negative-price swaps reclassified as basis_swap

### 4.4 Warning Verification Against Source

- 235 files with warnings checked against original 10-K section text
- 456 warnings resolved (57% reduction from 794 to 338, then further to 322 after audit)
- Root causes: 85% info genuinely not in 10-K, 15% unusual but correct pricing, 0% extraction errors remaining

## 5. Metadata & Audit Trail

### 5.1 Fields Present

- model: 99% of files (40 missing, now tagged "unknown-legacy")
- extraction_timestamp: 99% (ISO 8601)
- batch_id: 54% (Sonnet batch API runs only)
- extraction_method: 6.7% (Opus interactive only)
- hedging_confidence: 100% (via parsed_index)
- item7_merge_timestamp: ~71% (merged files only)
- warning_fix_timestamp: ~7% (warning-verified files)

### 5.2 Known Gaps

- No prompt_version or prompt_hash — cannot determine which prompt produced which extraction
- No source_sections_used — cannot determine if item7a, item7, or both were used
- batch_id only present in Sonnet batch runs, not Opus interactive
- extraction_method only present in Opus interactive, not Sonnet batch

### 5.3 Provenance Chain

Full traceability for every filing:

1. Raw 10-K text (Google Drive) → recorded in parsed_index.csv source_filepath
2. Parsed sections (parsed/{CIK}_{Company}/{date}/sections/) → metadata.json, keyword_scan.json
3. Parsed index (indexes/parsed_index.csv) → 20-column classification
4. Extraction JSON (extractions/{CIK}_{date}.json) → instruments + production + metadata
5. Extraction index (indexes/ai_extraction_index.csv) → 12-column summary

## 6. Code Architecture

### 6.1 Core Pipeline (8 scripts, portable)

| Script | Lines | Purpose |
|--------|-------|---------|
| config.py | 319 | 238 regex keyword patterns for hedging detection |
| stage1_parser.py | 802 | SGML/HTML parsing, section extraction, confidence classification |
| stage2_claude_code.py | 744 | Claude Code CLI extraction (instruments + production) |
| stage2_ai_extract.py | 974 | Claude Batches API extraction variant |
| stage2_reextract.py | 155 | Manifest builder for re-extraction |
| verify_extractions.py | 584 | 6-check verification + auto-fix |
| queue_filings.py | 136 | Queue helper for processing order |
| write_extraction.py | ~200 | Helper for writing extraction JSONs |

### 6.2 Quality Scripts (created during audit)

| Script | Purpose |
|--------|---------|
| audit_extractions.py | JSON validity & schema audit |
| audit_data_ranges.py | Data range & anomaly detection |
| audit_data_quality.py | Cross-file & temporal consistency |
| normalize_period_dates.py | YYYY-Qn → YYYY-MM normalization |
| normalize_volume_units.py | Volume unit canonicalization |
| deduplicate_instruments.py | Exact-match instrument dedup |
| fix_unit_scale_errors.py | Unit/scale error correction |

### 6.3 One-Off Scripts (archived)

- 166 generator scripts (_gen_*.py): per-filing instrument extraction helpers
- ~96 root-level scripts: research, batch fixes, checks — created during iterative development
- ~20 merge scripts: item7 merge batch processors

## 7. Known Limitations

### 7.1 Data Limitations (inherent to source)

- 322 instruments with missing volume/price — SEC disclosure doesn't require per-contract detail
- Basis swaps often lack differential prices in 10-K filings
- Percentage-of-production hedging (e.g., "75% of estimated production") cannot be converted to absolute volumes
- Fair-value-only disclosures (ASC 820) have no per-unit price breakdown
- Floating-price marketing contracts have no fixed price by design

### 7.2 Pipeline Limitations (addressable)

- No formal test suite — verification pipeline serves as integration test but no unit tests
- Code duplication between stage2_claude_code.py and stage2_ai_extract.py
- Inconsistent timestamps (naive local vs UTC)
- No prompt versioning in metadata
- 2 files have incomplete instrument schemas (missing 7-10 fields each)

### 7.3 Portability Limitations

- 367 hardcoded absolute paths in one-off scripts (core scripts are portable)
- Source filings on Google Drive path (machine-specific)
- No containerization (Docker)

## 8. Processing History

### 8.1 Timeline

| Date | Event |
|------|-------|
| 2026-02-27 | Stage 1 complete: 4,672 filings parsed |
| 2026-02-27 | Stage 2 high-confidence batch: 1,869 extractions (Sonnet 4.5) |
| 2026-02-27 | Failed group (181): instruments patched (Opus 4.6) |
| 2026-02-28 | Medium + low groups: 1,573 extractions (Opus 4.6) |
| 2026-02-28 | Item7 gap discovered and fixed: +419 instruments merged |
| 2026-02-28 | Warning verification: 794→338 warnings (235 files checked against source) |
| 2026-02-28 | Quality audit: unit/scale errors, dedup, normalization — 338→322 warnings |
| 2026-02-28 | Project review and remediation |

### 8.2 Instrument Evolution

| Stage | Total Instruments |
|-------|-------------------|
| Initial high-confidence batch | 10,584 |
| After item7 merge | 11,003 (+419) |
| After warning verification (false positives removed) | 10,791 (-212) |
| After quality audit (dedup) | 10,638 (-153) |

## 9. Recommendations for Future Work

1. **Add unit tests** for core functions: parse_claude_response(), normalize_volume_units(), normalize_period_dates(), dedup logic
2. **Unify stage2 scripts** — extract shared functions (assemble_text, prompts) into a common module
3. **Add prompt versioning** — hash the extraction prompt and store in metadata
4. **Add source_sections_used** to metadata — record which sections were available and used
5. **Consider PostgreSQL** — 3,443 JSON files work but a database would enable faster queries
6. **Periodic re-verification** — run verify_extractions.py after any future modifications
