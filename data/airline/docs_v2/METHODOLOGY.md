# Methodology: how the data was composed, extracted and validated

*Data work by The AI Plus Research Lab at the University of Northern Iowa (attribution required; CC BY-NC 4.0, see ../LICENSE.txt and ../ATTRIBUTION.md).*

This document tells the whole story, in the order things happened. A shorter overview is in `README.md`; every column is explained in `CODEBOOK.md`.

## 1. The question and the source
Airlines buy jet fuel and often protect themselves against price rises with derivatives (purchased call options, collars, swaps, fixed-price purchase agreements) written on crude oil, heating oil or jet fuel itself. U.S. listed companies describe these positions in their **annual report on Form 10-K**, filed with the SEC and free on EDGAR (sec.gov). The disclosure is usually an aggregate: "we have hedged 43% of our expected 2014 fuel", or a table of gallons by year. Contract prices are almost never given, so **this dataset records coverage, not prices**.

The dataset answers: *for each airline and each year, did the company hedge fuel, how much of the following year's fuel was hedged, and how big is the company's fuel use?* Every value comes with the sentence that supports it.

## 2. Which companies (the universe)
1. EDGAR's own company lists for the airline industry codes SIC 4512 (scheduled air transport), 4513 (air courier) and 4522 (non-scheduled air transport): 219 registrants, plus UPS (code 4210) and the registrants of the first version of this dataset.
2. For each, the full list of annual reports since 2000 from the SEC's `data.sec.gov` interface. A name search (airlines, airways, aviation, cargo, jet...) found no further airline missing.
3. A human decision per registrant (`data/universe_register.csv`). **In scope:** passenger and cargo airlines, including regional carriers and parcel carriers (FedEx, UPS). **Out:** helicopter and air-ambulance operators, private-charter and fractional-ownership companies, subsidiaries that merely duplicate a parent's report, shells, and registrants whose annual reports all pre-date 2000. Reports filed before January 2000 were not read.
4. Result: 53 registrants in scope; 51 have reports of their own (America West Airlines Inc. and AirTran Airways Inc. are co-registrants whose reports are carried under their parents). 616 annual reports (original Form 10-K, 10-K405 or 10-KT; amendments were not read), of which 603 are airline reports and 13 belong to a drug developer that held one registrant number (excluded).

Independent completeness check: every 10-K in a separate list of SEC quarterly index files (2010-2025) for an included company is in the dataset.

## 3. Getting the text
All 616 reports were downloaded as complete SEC submission files (the main report plus attached exhibits, including the annual-report exhibit EX-13, where many pre-2010 airlines put their market-risk and financial-statement disclosures). The SEC rate limit (5 requests per second) and an identifying User-Agent were respected. No downloads failed. The raw files (5.7 GB) are not part of this folder; they can be re-downloaded with `code/pipeline/download_filings.py`.

## 4. Reading the reports (first read)
* **Evidence bundle.** For each report a program (no AI) converts HTML to text keeping table rows on one line, finds every passage about fuel, hedging, derivatives, gallons, collars, swaps, pass-through, refineries, and keeps the best-scoring windows up to 56,000 characters. The bundle contains nothing from any earlier extraction (a "blind" read). Bundles are in `evidence/evidence_bundles.zip`.
* **One isolated agent per report.** A language-model agent (Claude) reads only its own bundle and returns, for each field, a value **and a verbatim quote**. The prompt forbids inference, arithmetic and outside knowledge, allows "not stated" as the correct answer, excludes interest-rate and currency derivatives (often disclosed in the same note), and defines "position" as what is outstanding at the fiscal year-end for future periods (not hedges entered after year-end and not past gains). Agents run in batches of about 100 with at most three batches at once, because heavy concurrency earlier in the project caused agents to confuse documents.
* **Mechanical verification.** Code checks that every quote appears in the evidence (comparing letters and digits only), that every number appears in its quote (allowing thousands/millions scaling and the sign of fair values), and that the agent confirmed the company identity. Anything unverified is never written to the tables. 37 reports failed a check and were read again.
* A 32-report pilot was run three times while the method was fixed (it found a ranking defect that hid the gallons table of Southwest FY2013 and a quote-matching defect); only then was the full run made.

## 5. Building the tables (deterministic rules)
`code/pipeline/build_final.py` turns the verified readings into the tables with fixed rules (no AI):
* **Fiscal year:** from the year-end date the report prints; otherwise from the SEC header and the company's fiscal-year-end code.
* **Headline `hedge_pct_next`:** only from a clean annual figure for the next fiscal year, or the average of Q1-Q4 figures (each quarter once). A quarter alone, a few months, or conflicting figures give no headline (min/max kept).
* **Status:** derived from the reading plus history (a company that hedged before and now says it does not is `no_hedging_after_hedging`; a stop in an earlier year is not "stopped this year"; a claim to "use options" with no position and no fair value is `no_open_positions`).
* **Explicit versus inferred "no hedging":** explicit only if the sentence concerns fuel, hedging, futures or swaps.
* **Gallons consumed:** from three sources (agent text read, a table parser, the independent second reader); two sources must agree, otherwise the largest is shown and flagged (mainline-only versus company-wide figures are the usual cause).
* **Duplicates:** one report per accession; a combined parent-and-subsidiary report appears once.

## 6. Checking the work
**Layer 1: version 1 versus version 2.** The first version of the dataset (May 2026) had no systematic verification. Every report it held was re-read from source and compared (`corrections_vs_original.csv`). The differences are the substance of `VERSION_HISTORY.md`.

**Layer 2: hand audit of a random sample** (38 reports, drawn within status groups; `validation/manual_audit_sample.md`). For each, the full filing text was searched for passages the evidence missed. Status was right in 37 of 38; seven weaknesses were found and four fixed immediately.

**Layer 3: independent second reader** (443 reports = 73%; `validation/second_reader.md`). A second agent per report searched the complete filing text with its own queries and reported status, headline, as-of date and gallons with quotes. Hedging direction agreed in 438 of 443; no report labelled non-hedging was found to have open hedges. Disagreements were resolved by reading the sources and are logged in `data/adjudications.csv` (19 reports).

**Layer 4: automated tests** (`code/validate_airline.py`, 28 checks): completeness against EDGAR lists, unique keys, every quote present in the evidence, value ranges, status reproduces from its inputs, no part-year headline, answer keys (Southwest FY2013: 764/1,156/977/933 million gallons and 43%; Delta FY2018: the 219-million-gallon Monroe position).

**What validation cannot do:** the same family of language model did both reads; a shared blind spot is possible. A human has not reviewed the whole table. 160 reports (small cargo and start-up carriers and regional carriers with no hedging statement) were not read a second time because the samples showed no errors in those classes.

## 7. Cost and time (for planning)
About 75,000 agent tokens per report for the first read (603 reports), a pilot of three rounds, 37 retries, and about 26 million tokens for the second reader on 443 reports. Roughly 78 million tokens in all, run in one day, plus a few hours of human-guided fixes.

## 8. Reproducing
`code/pipeline/` holds every script in the order used (see `README.md` section "Re-running"). The agent steps need a Claude Code environment with the Workflow tool; everything else is plain Python (pandas, requests, beautifulsoup4, lxml).
