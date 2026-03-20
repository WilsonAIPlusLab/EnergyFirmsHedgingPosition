# Item 7 Merge: Experience, Lessons, and Technical Details

## 1. Problem Discovery

### The Gap
The `assemble_instrument_text()` function in both pipeline files (`stage2_claude_code.py` and `stage2_ai_extract.py`) only read two section files for instrument extraction:
- `item7a.txt` — Item 7A: Quantitative and Qualitative Disclosures About Market Risk
- `item8_notes.txt` — Item 8 Financial Statement Notes

It completely **skipped `item7.txt`** (Item 7: Management's Discussion and Analysis), even though MD&A frequently contains hedging summary tables, especially for smaller companies that put all derivative information in MD&A rather than a separate Item 7A section.

### Evidence of the Gap
- **Range Resources (CIK 315852)**: `item7a.txt` = 0 bytes, `item7.txt` = 103 KB with collar instrument data. Extraction showed 0 instruments.
- **473 files** had empty `item7a.txt` but substantive `item7.txt` content — these companies likely had their hedging data entirely in MD&A.
- The asymmetry was notable: `assemble_production_text()` already read `item7.txt` for production/reserve data, but `assemble_instrument_text()` did not.

### Manifest Statistics (Pre-Merge)
| Metric | Count |
|--------|-------|
| Total extraction files | 3,443 |
| Files with item7a.txt (>100 bytes) | 2,571 |
| Files with item7.txt (>100 bytes) | 2,917 |
| Files with item8_notes.txt (>100 bytes) | 3,070 |
| Empty item7a but has item7 | 473 |
| Total instruments (pre-merge) | 10,584 |

---

## 2. Design Decision: Incremental Merge vs. Full Re-extraction

### Option A: Full Re-extraction (Rejected)
Re-run AI extraction from scratch on all 3,443 files using expanded section text. This would:
- Discard 10,584 existing instruments (some manually verified and patched)
- Risk introducing new errors in previously-correct extractions
- Waste compute on files where item7 adds nothing new

### Option B: Incremental Merge (Chosen)
Read only `item7.txt` for each filing, extract instruments found there, and merge them into existing extraction JSONs. Benefits:
- Preserves all existing work (high-confidence extractions, manual patches)
- Only adds genuinely new data
- Much faster — single-pass read of one section per file
- Risk-free for files where item7 has no hedging data

### Deduplication Rules
When merging item7 instruments into existing files, two instruments are considered duplicates if ALL match:
1. Same `instrument_type` (e.g., both "swap")
2. Same `commodity` (e.g., both "crude_oil")
3. Same or overlapping `period_start` / `period_end`
4. Similar volume (within ~15%, or both null)

On duplicate match:
- **Keep existing instrument** as the base
- **Fill missing fields only**: if existing has `fixed_price=null` but item7 version has a price, add it
- Do NOT overwrite non-null fields

On no match:
- **Append** the new instrument

---

## 3. Pipeline Code Changes

### `assemble_instrument_text()` — Updated in Both Pipeline Files
Before (lines 161-187 in `stage2_claude_code.py`):
```python
# Only read item7a.txt and item8_notes.txt
MAX_CONTEXT_CHARS = 80000
```

After:
```python
MAX_CONTEXT_CHARS = 120000  # Increased from 80K to accommodate item7

def assemble_instrument_text(sections_dir):
    parts = []
    total_chars = 0

    # Primary: Item 7A (up to 50K chars)
    item7a = _read_section(sections_dir, "item7a.txt")
    if item7a.strip():
        chunk = item7a[:50000]
        parts.append("=== ITEM 7A: MARKET RISK DISCLOSURES ===\n" + chunk)
        total_chars += len(chunk)

    # Secondary: Item 7 / MD&A (up to 40K chars)
    item7 = _read_section(sections_dir, "item7.txt")
    if item7.strip():
        remaining = MAX_CONTEXT_CHARS - total_chars
        if remaining > 5000:
            budget = min(remaining, 40000)
            chunk = item7[:budget]
            parts.append("=== ITEM 7: MANAGEMENT'S DISCUSSION AND ANALYSIS ===\n" + chunk)
            total_chars += len(chunk)

    # Tertiary: Item 8 Notes (remaining budget)
    notes = _read_section(sections_dir, "item8_notes.txt")
    if notes.strip():
        remaining = MAX_CONTEXT_CHARS - total_chars
        if remaining > 5000:
            chunk = notes[:remaining]
            parts.append("=== FINANCIAL STATEMENT NOTES (HEDGING-RELATED) ===\n" + chunk)
            total_chars += len(chunk)

    # Fallback: full_text.txt if no sections found
    if not parts:
        full_text = _read_section(sections_dir, "full_text.txt")
        if full_text.strip():
            chunk = full_text[:60000]
            parts.append("=== FULL FILING TEXT ===\n" + chunk)
            total_chars += len(chunk)

    return "\n\n".join(parts), total_chars
```

### Context Budget Allocation
| Section | Budget | Priority | Rationale |
|---------|--------|----------|-----------|
| item7a.txt | Up to 50K chars | Primary | Most focused hedging disclosure |
| item7.txt | Up to 40K chars | Secondary | Supplementary hedging tables in MD&A |
| item8_notes.txt | Remaining (up to 30K) | Tertiary | Fair values, derivative note details |
| full_text.txt | 60K chars | Fallback | Only if no sections found |
| **Total** | **120K chars max** | | Increased from 80K |

### New File Created: `pipeline/stage2_reextract.py`
Manifest builder that scans all extraction JSONs, looks up source `sections_dir` from `parsed_index.csv`, and reports:
- Section file sizes (item7a, item7, notes)
- Current instrument counts per file
- Batch assignments for parallel processing

---

## 4. Execution: Two-Phase Merge

### Phase 1 (P1): Empty item7a Files — 473 files
These are the highest-priority files: companies with no item7a.txt content where item7.txt was the only available hedging source. Most had 0 instruments before the merge.

**5 Sonnet 4.6 agents**, ~100 files each:

| Batch | Files | New Instruments | Notable Companies |
|-------|-------|-----------------|-------------------|
| B0 | 100 | 37 | Carrizo Oil & Gas, US Energy Corp, Encana |
| B1 | 100 | 178 | Rockies Region, Allied Resources, Hallwood Energy |
| B2 | 100 | 1 | NiMin Energy |
| B3 | 100 | 2 | Penn Virginia Resource Partners |
| B4 | 73 | 21 | Carbon Natural Gas |
| **Total P1** | **473** | **239** | |

### Phase 2 (P2): Files with Both item7a and item7 — 2,444 files
Lower marginal benefit since these already had item7a-based instruments. Item7 may contain additional instruments not in item7a, or may just repeat the same data.

**7 Sonnet 4.6 agents**, ~350 files each:

| Batch | Files | New Instruments | Notable Companies |
|-------|-------|-----------------|-------------------|
| B0 | 350 | 6 | Various |
| B1 | 350 | 38 | Eclipse Resources, Devon Energy |
| B2 | 350 | 0 | (item7 repeated item7a content) |
| B3 | 350 | 36 | Gastar Exploration, Bill Barrett |
| B4 | 350 | 0 | |
| B5 | 350 | 0 | |
| B6 | 344 | 12 | GeoMet, BreitBurn Energy |
| **Total P2** | **2,444** | **~92** | |

### Merge Results Summary
| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Total instruments | 10,584 | 11,003 | +419 |
| Files processed | — | 2,917 | |
| Files with metadata stamp | — | 2,454 | |
| Errors (verify) | — | 0 | |
| Warnings (verify) | — | 794 | |

---

## 5. False Positive Patterns Discovered

When extracting instruments from `item7.txt` (MD&A), agents encountered many false positives because MD&A contains narrative financial discussion alongside hedging data. Key patterns identified:

### 5.1 Interest Rate Swaps
MD&A often discusses interest rate risk management alongside commodity hedging. Item7a.txt typically separates these, but item7.txt mixes them.
- **Solution**: Prompt already says "IGNORE interest rate derivatives. Only commodity instruments." Agents correctly filtered these.

### 5.2 Forward-Looking Statements
Phrases like "forward curve," "forward-looking estimates," "forward production" are NOT forward contracts.
- **Pattern**: `forward` in narrative context ≠ `instrument_type: "forward"`

### 5.3 Dollar Amounts from Financial Narrative
MD&A discusses revenues, expenses, capital budgets — all with dollar amounts that can look like commodity prices:
- "$13.4 million mark-to-market loss" → NOT a $13.4/Bbl price
- "$4.50 per Mcf average realized price" → NOT a swap fixed_price (this is historical sales price)
- "$25 million borrowing base" → NOT a hedge volume or fair value

### 5.4 SEC Ceiling Test / Impairment Prices
Standardized measure calculations use benchmark prices (e.g., "$94.19/Bbl WTI") — these are SEC-mandated pricing assumptions, not hedge prices.

### 5.5 Property/Acreage Swaps
Some companies describe property exchanges using the word "swap" — "the company swapped 40,000 acres in the Permian Basin." This is NOT a commodity swap.

### 5.6 Average Realized Prices
"The company's average realized oil price was $82.50/Bbl" — this is historical revenue data, not a hedge instrument.

### 5.7 Drilling Terminology
"pre-collar hole" refers to well drilling, not a collar derivative.

### Mitigation
The Sonnet 4.6 agents generally handled these well through multi-pass verification:
1. First pass: Extract candidate instruments
2. Second pass: Cross-reference against existing instruments for dedup
3. Third pass: Validate that extracted prices/volumes are reasonable for hedging (not financial narrative)

---

## 6. Post-Merge Issues and Fixes

### Issue 1: Non-Standard hedging_summary Format (73 files)
Some merge agents created `hedging_summary` using a non-standard schema:
```json
{"total_instruments": 5, "by_commodity": {...}, "by_type": {...}}
```
Instead of the standard 5-key schema:
```json
{
  "total_oil_instruments": 3,
  "total_gas_instruments": 2,
  "total_ngl_instruments": 0,
  "uses_derivatives": true,
  "stated_hedging_policy": "..."
}
```
**Fix**: Bulk Python script that rebuilt the summary from the instruments list, counting oil/gas/NGL instruments and preserving the `stated_hedging_policy`.

### Issue 2: uses_derivatives Contradictions (54 files)
After merge, some files had `uses_derivatives: false` in the summary but instruments were now present (added by merge).
**Fix**: `verify_extractions.py --fix` automatically sets `uses_derivatives=true` when instruments exist and recalculates counts.

### Issue 3: Warning Increase (402 → 794)
Warnings increased because instruments from item7 narrative text often lack structured price/volume data:
- **363 no-volume warnings**: Item7 prose says "the company has oil swaps" without specifying volumes
- **312 no-price warnings**: Instruments mentioned without specific prices
- **87 price outlier warnings**: Edge cases with unusual pricing
- **32 no-instruments warnings**: Files where `uses_derivatives=true` but 0 instruments extracted

These are informational warnings, not errors. The increase is expected — item7 narrative is less structured than item7a tables.

---

## 7. Targeted Warning Verification (Post-Merge)

After the item7 merge produced 794 warnings, a second pass verified **all 235 unique warning files** against the original 10-K source sections. Sonnet 4.6 agents read item7a.txt, item7.txt, and item8_notes.txt for each file and made one of three decisions per instrument: fix the data, remove a false positive, or confirm the data is truly absent from the source.

### 7.1 Warning Categories and Actions Taken

#### Price Outliers — High (>$300/Bbl): 15 files, 29 warning lines
**Root cause: Extraction errors.** The AI misidentified non-commodity values as hedge prices.

Patterns found and fixed:
- **Year-as-price**: CIK 1407805 (Rockies Region) had `ceiling_price=$2009`, `$2011`, `$2017`, `$2018` — calendar years placed in price fields
- **Notional-as-price**: `fixed_price=$750000`, `$1400000`, `$3861083` — total dollar notional amounts stored as per-unit price
- **Financial derivatives misidentified as commodity hedges**: Eos Petro ($2.5M warrant fair values), Circle Star Energy ($3.86M impairment charges), Petron Energy ($8K management fees)

**Result**: 13 of 15 files had ALL instruments removed (were entirely fake). 1 file reconstructed with real instruments from source derivative table. 2 files had 2 hallucinated instruments each removed.

#### Price Outliers — Low (<$5/Bbl): 29 files, 56 warning lines
**Root cause: Extraction is correct.** These are legitimate low-value differential instruments.

Confirmed categories:
- **Roll swaps / CMA differentials** ($0.05–$1.08/Bbl): Pioneer, Crescent Energy, Devon, PDC Energy, Continental Resources — hedges timing risk between physical sales and NYMEX trade month settlement
- **Basis swap differentials** ($0.22–$0.90/Bbl): Civitas, Chaparral, Chesapeake, Linn Energy — location basis between e.g., Midland and Cushing
- **Range bonus accumulators** ($3.00–$4.94/Bbl): Vanguard Natural Resources — bonus payment per barrel within a price range, not the actual commodity price
- **NGL prices in $/Gallon**: Chesapeake ($0.28 ethane, $0.73 propane), TXO Energy ($0.23–$0.27 ethane) — price_unit corrected from $/Bbl to $/Gallon

**Result**: 25 files confirmed correct. 4 files fixed (price_unit corrections).

#### No-Instruments: 32 files
**Root cause: Extraction errors.** The AI set `uses_derivatives=true` but found no instruments because none existed.

Source verification found:
- 6 companies explicitly state "we do not use derivatives" (Petro River Oil, Earthstone Energy, US Energy Corp)
- 8 companies have only financial derivatives (convertible note embedded derivatives, stock warrants) — not commodity hedges
- 6 companies (Allied Resources, all 6 filings) had zero mention of hedging anywhere
- 6 companies (United American Petroleum, ForgeHouse) reference NYMEX/futures markets in discussion but have no actual positions
- 3 companies (Tiger Oil & Energy) have only note payable derivative liabilities
- 2 companies (Rockies Region 2006 LP) explicitly state derivatives were liquidated years ago
- 1 company (Tengasco) genuinely had derivatives but Note 13 details were not in the parsed sections

**Result**: 31 files set to `uses_derivatives=false`. 1 file kept `true` (Tengasco — derivatives confirmed but details in unparsed note).

#### No-Volume + No-Price (both missing): 79 files
**Root cause: Mix of extraction errors and source limitations.**

- ~26 files had ALL instruments removed as false positives (financial derivatives, expired contracts, hallucinated entries from non-hedging text)
- ~8 files had data partially or fully recovered from source sections
- ~45 files confirmed truly missing — source discloses only fair values or aggregate notional amounts

Notable false positive patterns discovered:
- **Stock warrants/options**: Synergy Resources (7 instruments were stock warrants Series A/B/C/D)
- **Mining companies**: Casablanca Mining (copper/gold), Daulton Capital (Yukon gold claims)
- **Non-oil/gas companies**: Restaurant Concepts of America, Bakken Resources (speculative shell)
- **Expired instruments**: Recovery Energy (all liquidated Nov 2012), Lilis Energy (2014 swap expired), VAALCO (puts expired Dec 2017)
- **Hallucinated from boilerplate**: Laredo Oil (CFTC regulatory text parsed as hedging activity)

#### No-Volume Only: 19 files
- 4 files had volumes recovered from source (BreitBurn 2.2M Bbls, Denbury 2,000 Bbl/d, California Resources 2.29M MMBtu)
- 12 files had false positives removed entirely
- 3 files confirmed truly missing (percentage-of-production disclosures with no absolute volumes)

#### No-Price Only: 61 files
- 11 instruments across 4 files had prices recovered (Centennial basis swaps, Cabot basis swaps, Chesapeake basis swap, Callon basis swap, Bill Barrett basis swap, EPL Oil three-way collar)
- ~3 instruments removed as false positives (expired, settled)
- ~47 files confirmed truly missing — especially basis swaps and marketing derivatives

### 7.2 Warning Verification Results

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| **Errors** | 0 | 0 | — |
| **Total Warnings** | **794** | **338** | **-456 (-57%)** |
| no-instruments | 32 | 0 | -32 (all resolved) |
| no-volume | 363 | 101 | -262 |
| no-price | 312 | 188 | -124 |
| price-outlier | 87 | 49 | -38 |
| Total instruments | 11,003 | 10,791 | -212 (false positives removed) |

### 7.3 Root Cause Analysis of Remaining 338 Warnings

All remaining warnings were verified against source 10-K text. The data is genuinely absent from the filings:

| Root Cause | Warnings | % | Explanation |
|---|---|---|---|
| **Info not in the 10-K** | 289 | 85% | Source discloses only fair values, percentages of production, or aggregate notionals |
| **Extraction correct, looks unusual** | 49 | 15% | Legitimate basis/roll swap differentials < $5/Bbl |
| **Extraction error** | 0 | 0% | All extraction errors were fixed |

**Why the 10-K doesn't have the data:**

1. **Basis swaps without differentials** (~100 no-price): Companies like Range Resources, Encana, Southwestern Energy report "basis swaps covering 313 MMBtu" but never disclose the actual differential price. The SEC doesn't require per-contract price disclosure.

2. **Floating-price marketing derivatives** (~40 no-price): Pioneer Natural Resources' marketing contracts buy at Midland WTI and sell at Brent-correlated WASP — there IS no fixed price by design.

3. **Fair-value-only disclosures** (~48 no-price): Occidental, Murphy Oil, APA Corp show only aggregate fair value tables (ASC 820 Level 2/3 hierarchy) without per-contract prices.

4. **Percentage-of-production volumes** (~60 no-volume): Viking Energy hedges "75% of estimated production" without stating absolute volumes. Ivanhoe Energy hedges "approximately 50% of estimated production."

5. **Aggregate notional-only disclosures** (~40 no-volume): TransCoastal Corp discloses $3,111K notional contract values — not physical volumes or per-unit prices.

---

## 8. Comprehensive Quality Audit & Fixes

After warning verification, a full quality audit was performed on all 3,443 extraction files using three parallel Sonnet 4.6 agents covering: (1) JSON validity & schema consistency, (2) data range & anomaly detection, (3) cross-file & temporal consistency.

### 8.1 Audit Findings

#### HIGH Priority — Unit/Scale Errors

| Issue | Files | Root Cause |
|-------|-------|------------|
| `standardized_measure_mm` / `pv10_mm` stored in thousands not millions | 31 | AI extracted raw numbers from filings with "(in thousands)" headers without dividing |
| Gas production (Mcf) inflated ~1,000x | 6 | Bcf values stored as Mcf (Anadarko, Occidental, Marathon Oil) |
| Reserves inflated ~1,000x | 7 | Bbls stored as MBbls or similar (Range Resources, Ring Energy, Abraxas, Chesapeake, MV Oil Trust, Recovery Energy, Victory Energy) |
| Stale extraction index | 42 | Index not rebuilt after item7 merge |
| Duplicate instruments within files | 52 | Same instrument extracted from multiple tables in the source filing |

#### MEDIUM Priority — Schema Normalization

| Issue | Instruments/Files | Root Cause |
|-------|-------------------|------------|
| `uses_derivatives=true` but instruments=[] | 355 files | AI couldn't extract specific instruments despite hedging policy text |
| `hedging_summary` count mismatches | 80 files | Counts not updated after instrument additions/removals |
| Period dates not YYYY-MM format | 2,734 instruments in 138 files | YYYY-Qn (Marathon Oil etc.), YYYY-MM-DD, YYYY formats |
| `volume_unit` case variants | 311 instruments in 65 files | `Gal`/`Gallons`, `MMBtu`/`MMbtu`, `Bbl`, `MBbls`, `GJ`, etc. |
| `direction` non-standard values | 89 instruments in 8 files | `fixed_for_float` (27) and `null` (62) |
| Negative-price swaps misclassified | 6 instruments in 4 files | Roll/basis swaps with negative `fixed_price` labeled as `swap` not `basis_swap` |

#### Informational (No Action Needed)

| Issue | Count | Notes |
|-------|-------|-------|
| Company name changes across filings | 50 CIKs | All legitimate M&A/rebrands (Oasis→Chord, Cabot→Coterra, Swift→SilverBow) |
| Filing date gaps >18 months | 44 CIKs | Dormant/shell companies that went dark then re-emerged |
| Close-together filings (<8 months) | 553 pairs | Original + amendment (10-K/A) pairs |
| Missing `model` in metadata | 40 files | Older extractions predating model-logging convention |
| Stray `batch_info.json` in extractions/ | 1 file | Metadata file, not an extraction |

### 8.2 Fixes Applied

All HIGH and MEDIUM issues were fixed by 9 parallel Sonnet 4.6 agents:

| Fix | Agent | Files | Details |
|-----|-------|-------|---------|
| `standardized_measure_mm` / `pv10_mm` ÷ 1,000 | 1 | 31 | Cimarex, Bill Barrett, Oasis, Contango, Talos, etc. |
| Gas production ÷ 1,000 | 2 | 6 | Anadarko (809B→809M), Occidental (338-649B→338-649M), Marathon |
| Reserves ÷ 1,000 (or ÷ 1,000,000) | 2 | 7 | Range Resources, Ring Energy, Abraxas, Chesapeake, MV Oil, Recovery, Victory |
| Deduplicate instruments | 3 | 52 | 153 instruments removed (exact match on 9 key fields) |
| Fix `hedging_summary` counts | 4 | 80 | Recounted oil/gas/NGL from actual arrays |
| `uses_derivatives` flag fix | 5 | 355 | 13 flipped to false (no hedging keywords); 342 kept true + added "[Note: instruments not extractable from filing]" |
| Period dates → YYYY-MM | 6 | 138 | 2,730 fields: Q1→01/03, Q2→04/06, Q3→07/09, Q4→10/12; YYYY-MM-DD truncated; YYYY expanded |
| `volume_unit` normalization | 7 | 65 | 311 instruments: Gal→gallons, MMBtu→Mmbtu, Bbl→Bbls, MBbls→Bbls, GJ→other, etc. |
| `direction` normalization | 8 | 18 | 27 fixed_for_float→short, 62 null→unknown |
| Negative swaps → `basis_swap` | 8 | 4 | 6 instruments reclassified (Pioneer, Cimarex, Rosetta, Legacy) |
| Remove `batch_info.json` | 4 | 1 | Deleted non-extraction file |
| Rebuild extraction index | — | 1 | All instrument counts synced |

### 8.3 Post-Audit State

```
Errors:   0
Warnings: 322
  - 94  no-volume    (source discloses % of production or notional only)
  - 180 no-price     (basis swaps without differentials, floating-price, fair-value-only)
  - 48  price-outlier (confirmed legitimate: roll swaps, basis differentials, accumulators)
  - 0   no-instruments
Total instruments: 10,638 (after dedup of 153)
```

### 8.4 Normalization Mappings Applied

**volume_unit canonical values**: `Bbls`, `Mmbtu`, `Mcf`, `Bcf`, `Boe`, `gallons`, `other`

| Non-canonical | → Canonical | Count |
|---------------|-------------|-------|
| Gal | gallons | 88 |
| MMBtu | Mmbtu | 44 |
| Bbl | Bbls | 39 |
| Bbtu | Mmbtu | 32 |
| Mmcf | Mcf | 24 |
| Gallons | gallons | 23 |
| MMbtu | Mmbtu | 17 |
| MBbls | Bbls | 9 |
| GJ | other | 9 |
| MMcf | Mcf | 8 |
| Others (TBtu, Mtons/month, etc.) | other | 18 |

**direction canonical values**: `short`, `long`, `collar`, `unknown`
- `fixed_for_float` → `short` (27 instruments)
- `null` → `unknown` (62 instruments)

**period date canonical format**: `YYYY-MM`
- `YYYY-Qn` → start=YYYY-{first month}, end=YYYY-{last month}
- `YYYY-MM-DD` → truncated to YYYY-MM
- `YYYY` → start=YYYY-01, end=YYYY-12

---

## 9. Verification Pipeline

### `verify_extractions.py` — 6 Check Categories
| Check | Description | Auto-Fix? |
|-------|-------------|-----------|
| 1. JSON Integrity | Can each file be parsed? Required keys present? | No |
| 2. Schema Consistency | hedging_summary has standard 5 keys? Instruments have all 15 fields? | Yes (rebuilds summary) |
| 3. Index-to-JSON | Every index row has a matching JSON file? | Yes (rebuilds index) |
| 4. Logical Consistency | uses_derivatives vs instrument count? Price outliers? Date issues? | Yes (fixes contradictions) |
| 5. Cross-Index | Parsed index vs extraction index alignment? | No |
| 6. Coverage | All parsed filings have extractions? | No |

### Final State After All Fixes
```
Errors:   0
Warnings: 322  (all verified as source limitations, not extraction errors)
  - 94  no-volume    (source discloses % of production or notional amounts only)
  - 180 no-price     (basis swaps without differentials, floating-price contracts, fair-value-only)
  - 48  price-outlier (confirmed legitimate: roll swaps, basis differentials, bonus accumulators)
  - 0   no-instruments (all resolved)
Total instruments: 10,638  (after dedup of 153 duplicates)
```

---

## 10. Key Lessons Learned

### 10.0 Verify Warnings Against Source Before Accepting Them
794 warnings looked like "the AI didn't extract well enough." After source verification, 456 were resolved: ~212 were false positives (financial derivatives, expired contracts, hallucinated instruments), ~20 had recoverable data, and ~31 were mislabeled flags. Only 338 were genuinely missing from source. **Never assume warnings reflect extraction quality — go back to the source.**

### 10.1 Section Coverage Matters
Always verify that your text assembly function covers ALL relevant sections. The item7 gap went unnoticed through the entire high-confidence batch (1,869 filings) because many companies DO put hedging in item7a. The gap only became visible when analyzing warning patterns.

**Rule**: Cross-reference section assembly logic for ALL extraction types. If `assemble_production_text()` reads a section, ask whether `assemble_instrument_text()` should too.

### 10.2 Incremental > Destructive
When you discover a gap in previously-completed work, prefer incremental fixes over full redo:
- Preserves manually-verified data
- Lower risk of introducing new errors
- Faster execution
- Easy to audit (metadata stamps show what was merged)

### 10.3 Priority Ordering by Impact
Split work into priority tiers based on expected marginal value:
- **P1 (empty item7a)**: 473 files, yielded 239 instruments (0.51 per file)
- **P2 (has item7a)**: 2,444 files, yielded ~92 instruments (0.04 per file)

P1 was 12x more productive per file. Always process highest-impact cohort first.

### 10.4 MD&A is Noisy for Instrument Extraction
Item 7 (MD&A) is a narrative section mixing hedging with operational discussion, revenue analysis, capital budgets, and forward-looking statements. This creates many false positive candidates. Effective extraction requires:
- Strong negative rules (ignore interest rate, ignore property swaps, ignore realized prices)
- Cross-reference with existing instruments for dedup
- Price/volume sanity checks against commodity norms

### 10.5 Schema Enforcement is Critical
When launching many parallel agents, each may invent slightly different output formats. Always validate against a canonical schema and fix deviations programmatically. The 73 non-standard hedging_summary files would have caused downstream issues if not caught.

### 10.6 Backup Before Bulk Operations
The `extractions_v1_backup/` directory proved essential as a safety net. Always create a full backup before any bulk modification, even when the operation is "additive only."

### 10.7 Context Budget Planning
With 3 sections to assemble, budget allocation matters:
- Item7a (primary): 50K chars — most focused, highest signal-to-noise
- Item7 (secondary): 40K chars — broader, more noise, but uniquely valuable for companies without item7a
- Item8 notes (tertiary): remaining budget — detailed fair values and footnotes
- Total: 120K chars max (up from 80K)

The priority ordering ensures that if context is limited, the highest-value section gets full budget.

### 10.8 Run Comprehensive Quality Audits After Bulk Operations
After fixing warnings, a full audit of all 3,443 files revealed unit/scale errors (31 files with standardized_measure in thousands not millions), duplicate instruments (52 files), and schema inconsistencies (volume_unit variants, non-standard directions, period date formats). These were invisible to the warning-focused verification pipeline. **Always run a broad audit — not just targeted checks — after large-scale modifications.**

### 10.9 AI Extraction Unit Confusion is Systematic
When 10-K tables say "(in thousands)" in headers, the AI often extracts the raw number without dividing. This affected standardized_measure (31 files), gas production (6 files), and reserves (7 files). The pattern is predictable: large companies with "(in thousands)" table headers. **Build explicit unit-conversion validation into the pipeline, not just post-hoc audits.**

---

## 11. File Reference

### Pipeline Scripts
| File | Purpose |
|------|---------|
| `pipeline/stage2_claude_code.py` | Main extraction pipeline (Claude Code CLI variant) |
| `pipeline/stage2_ai_extract.py` | Batches API extraction variant |
| `pipeline/stage2_reextract.py` | Manifest builder for re-extraction/merge |
| `pipeline/verify_extractions.py` | 6-check verification + auto-fix |
| `pipeline/fix_warnings.py` | Warning inspection utility |
| `pipeline/audit_extractions.py` | JSON validity & schema audit |
| `pipeline/audit_data_ranges.py` | Data range & anomaly detection |
| `pipeline/audit_data_quality.py` | Cross-file & temporal consistency |
| `pipeline/normalize_period_dates.py` | Period date format normalization |
| `pipeline/normalize_volume_units.py` | Volume unit normalization |
| `pipeline/deduplicate_instruments.py` | Instrument deduplication |
| `logs/warning_manifest.json` | 235 warning files with sections_dir and warning details |

### Data Files
| File | Contents |
|------|----------|
| `logs/reextract_manifest.json` | Full manifest: 3,443 files with section sizes |
| `logs/merge_p1_manifest.json` | P1 cohort: 473 empty-item7a files |
| `logs/merge_p2_manifest.json` | P2 cohort: 2,444 files with both sections |
| `extractions_v1_backup/` | Pre-merge backup of all extraction files |
| `indexes/ai_extraction_index.csv` | 3,443-row extraction index |
| `indexes/parsed_index.csv` | 4,672-row parsed filing index |

### Extraction JSON Schema
```
extractions/{CIK}_{filing_date}.json
├── cik, company, filing_date, accession_number
├── instruments
│   ├── instruments[] (15 fields each)
│   └── hedging_summary (5 keys)
├── production
│   ├── production (8 keys)
│   ├── reserves (7 keys)
│   └── economics (10 keys)
└── metadata
    ├── model, extraction_timestamp
    ├── hedging_confidence (high/medium/low)
    ├── item7_merge_timestamp (if merged)
    └── warning_fix_timestamp (if warning-verified)
```

---

## 12. Timeline

| Date | Event |
|------|-------|
| 2026-02-27 | Stage 1 + Stage 2 (high-confidence) complete: 1,869 extractions |
| 2026-02-27 | Failed group (181 filings) patched: instruments added via Opus 4.6 |
| 2026-02-28 | All 3,443 extractions complete (medium + low groups) |
| 2026-02-28 | Item7 gap discovered during warning analysis |
| 2026-02-28 | Pipeline code updated: item7.txt added to assemble_instrument_text() |
| 2026-02-28 | Manifest built: 2,917 files with item7 content identified |
| 2026-02-28 | P1 merge complete: 473 files, +239 instruments |
| 2026-02-28 | P2 merge complete: 2,444 files, +~92 instruments |
| 2026-02-28 | Post-merge fixes: schema normalization, contradiction resolution |
| 2026-02-28 | Post-merge state: 11,003 instruments, 0 errors, 794 warnings |
| 2026-02-28 | Warning verification: 235 files checked against source 10-K sections |
| 2026-02-28 | B1: 15 high-price outlier files fixed (year-as-price, notional-as-price, fake instruments) |
| 2026-02-28 | B2: 32 no-instruments files resolved (31→false, 1 kept true) + 29 low-price confirmed |
| 2026-02-28 | B3: 79 no-vol+no-price files verified (false positives removed, data recovered, rest confirmed) |
| 2026-02-28 | B4: 80 single-warning files verified (volumes/prices recovered where available) |
| 2026-02-28 | Post-verification state: 10,791 instruments, 0 errors, 338 warnings |
| 2026-02-28 | Comprehensive quality audit: 3 Sonnet 4.6 agents audited all 3,443 files |
| 2026-02-28 | HIGH fixes: unit/scale errors (31+6+7 files), dedup (52 files), index rebuild |
| 2026-02-28 | MEDIUM fixes: uses_derivatives flags (355), dates (138), units (65), directions (18), reclassifications (4), summary counts (80) |
| 2026-02-28 | **Final state: 10,638 instruments, 0 errors, 322 warnings (all verified source limitations)** |
