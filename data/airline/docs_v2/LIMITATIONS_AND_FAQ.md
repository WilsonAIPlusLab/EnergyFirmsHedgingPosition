# Limitations, warnings and frequently asked questions

*Data work by The AI Plus Research Lab at the University of Northern Iowa (attribution required; CC BY-NC 4.0, see ../LICENSE.txt and ../ATTRIBUTION.md).*

## Read this before using the numbers

1. **This is coverage, not prices.** Airlines disclose how much of next year's fuel is hedged, rarely at what price. There are no strikes, no premiums, no hedge returns.
2. **The headline `hedge_pct_next` exists for 155 of 603 reports.** The rest print only a quarter or part-year, several instruments that cannot be added, dollar amounts, or nothing at all. Do not read an empty headline as zero hedging; look at `status`.
3. **"No evidence" is not "no hedging".** `not_discussed` (80 reports) and `no_hedging_inferred` (69) mean the report says nothing about fuel hedging or only implies there is none (typically regional carriers whose partners pay for fuel). `no_hedging`, `no_hedging_after_hedging`, `no_open_positions` rest on an explicit statement.
4. **Dates inside a report differ.** A percentage "as of December 31" and one "as of January 17" can differ a lot (Southwest FY2013: 43% versus about 20%). Check `hedge_pct_next_asof` (112 year-end, 22 after year-end, 21 undated).
5. **Scope differs.** "Mainline" gallons exclude regional affiliates; company-wide gallons include them. The dataset uses the larger figure when two sources agree (see `fuel_gallons_candidates`). For regional carriers, "gallons consumed" covers only fuel the carrier buys itself.
6. **Delta is special.** From 2012 Delta owns the Monroe oil refinery; its derivative contracts are described as "primarily related to Monroe". FY2018-FY2025 are `refinery_hedges_only`. Delta FY2012-FY2017 mix airline and refinery positions: do not use their ratios as consumer-fuel hedging (FY2013 and FY2014 exceed 100%).
7. **FedEx and UPS are parcel carriers**, regional carriers (SkyWest, Mesa, Republic, ExpressJet, Pinnacle, Great Lakes, Air Wisconsin...) pass fuel to partners, start-ups and charter airlines rarely hedge. Filter on `carrier_type` for a study of airline hedging as a corporate decision (for example `passenger_scheduled`, 324 reports).
8. **Non-December fiscal years.** "Next year" means the next fiscal year. Check `fiscal_year_end` for Mesa, Frontier, Air T, FedEx, Alpine, SkyWest (before 2002).
9. **Reports before 2000 and amendments (10-K/A) were not read.** The earliest fiscal year is 1999 (filed January 2000 or later); some 1999-2001 reports exist only on form 10-K405, which was included.
10. **Two reads by the same kind of model.** Both readers are Claude models; they are independent in how they find evidence, not in model family. A human did not review every report. The independent second reading covers 443 of 603 reports (73%); the remaining 160 are regional pass-through and small cargo or start-up carriers where no error was found in the sample.
11. **Averaging quarters (7 reports) is an approximation.** The unweighted mean of four quarterly percentages is not the same as the share of the year's fuel hedged.
12. **One adjudication per case is documented, not hidden.** Nineteen reports have manual corrections (`data/adjudications.csv`); the column `adjudication` repeats them.

## FAQ

**Which table do I use?** `data/hedge_ratios.csv` (one row per company-year). Use `filings.csv` if you want every report, including the 2 firm-years that have two (SkyWest FY2001, Mesa/Republic 2025).

**How do I get only airlines that hedge?** `status in ['hedger','hedger_no_next_year_figure']` (open hedges) or `status == 'hedger'` (with a next-year figure).

**How do I measure "stopped hedging"?** Use `status == 'no_hedging_after_hedging'` (explicit statement after hedging was reported before) and `stopped_this_year`. Version 1's `stopped` label stuck to every later year: 129 reports carried it, although 9 of them actually showed active hedging and most others were simply non-hedging years.

**Can I trust a single value?** Every value has a quote (`positions.csv`, `fuel_gallons_table_quote`, or the report text behind `program_status` in `data/reread_results/final_raw.json`) and can be looked up in the filing by accession number. Spot-check anything that drives a conclusion.

**Why are some CIKs missing, such as American Airlines Inc., United Air Lines Inc., US Airways Inc.?** They file one combined report with their parent; it is stored once under the parent (American Airlines Group, United Airlines Holdings, US Airways Group).

**Why are some obvious airlines missing?** Companies that file no 10-K (foreign carriers file 20-F or 40-F; private airlines file nothing), and registrants whose reports end before 2000. See `universe_register.csv`.

**Is the dataset ready for regressions?** Descriptively, yes. For inference, mind the selection: only 155 reports have a headline, mostly large U.S. passenger carriers; the years 2000-2008 are far denser than 2015-2025 because few airlines hedge any more.

**What changed from the first version?** See `VERSION_HISTORY.md`.

**How do I cite it?** See the end of `README.md`.
