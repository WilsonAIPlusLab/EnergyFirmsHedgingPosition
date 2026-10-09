# Manual audit of a random sample (2026-10-08, second audit)

**Method.** 38 of the 603 annual reports, drawn at random within status groups (seed 2026; `validation/manual_audit_sample.csv`): hedger 8, hedger without next-year figure 5, no open positions 4,
no hedging 6, no hedging inferred 4, not discussed 6, stopped 1, refinery 1, plus 3 reports of registrants that are new to the dataset. For each I read what the dataset says (status, summary, positions, quotes)
and searched the **whole** filing text (all documents, not only the evidence the agent saw) for fuel-hedging passages the evidence missed, and for gallons-consumed figures
(`code/pipeline/manual_audit_cards.py`). Reviewer: the assistant, not an independent person. A sample of 38 gives about 87% to 99% confidence for a 97% hit rate; it finds frequent problems, not rare ones.

## Result in one line
**Hedging status was right in 37 of 38 reports. The problems are in secondary fields and labels**, and the sample plus follow-up counts over the whole dataset quantify them below.

## Report-by-report verdict
| n | Registrant, FY | Status in dataset | Verdict |
|---|---|---|---|
| 0 | Gulfstream Intl 2008 | hedger | status right; **headline 20% is for 2 months (through Feb 2009), not next year** (I1) |
| 1 | JetBlue 2003 | hedger | correct (40% of 2004) |
| 2 | Alaska 2013 | hedger | correct |
| 3 | Southwest 2006 | hedger | correct (95% of 2007 ... 15% of 2012); gallons consumed not recorded (I3) |
| 4 | Southwest 2013 | hedger | correct (matches the answer key); **gallons consumed 1,818 million printed but not recorded** (I3) |
| 5 | Hawaiian 2017 | hedger | correct |
| 6 | Midwest Air 2004 | hedger | correct; headline 16.45% is an unweighted mean of four quarters (approximation) |
| 7 | AMR 2007 | hedger | correct; the 24% is "as of January 2008" (I6) |
| 8 | UPS 2002 | hedger, no next-year figure | weak: generic "we use options, swaps and futures" language, no outstanding positions quoted (I4) |
| 9 | UPS 2008 | hedger, no next-year figure | **wrong status**: net fair value of the contracts is $0 at year-end, so nothing was outstanding (I4) |
| 10 | Airborne 2002 | hedger | correct (heating-oil call option through March 2003; later contract is a subsequent event) |
| 11 | JetBlue 2012 | hedger, no figure | status right; no headline because collars (5%) and fixed forward prices (6%) are listed separately (I5) |
| 12 | Frontier FYE Mar-2004 | hedger, no figure | correct; covers only the next quarter |
| 13 | Sun Country 2021 | no open positions | correct |
| 14 | UPS 2020 | no open positions | correct |
| 15 | Continental 2001 | no open positions | correct |
| 16 | UPS 2016 | no open positions | correct |
| 17 | Air T FYE Mar-2004 | no hedging | correct |
| 18, 19 | Great Lakes 2012, 2004 | no hedging | correct ("does not hedge", no derivatives) |
| 20, 21 | United 2024, 2020 | no hedging after hedging | correct |
| 22 | FedEx FYE May-2014 | no hedging after hedging | correct (no jet fuel contracts, indexed surcharge) |
| 23 | Republic 2008 | no hedging inferred | acceptable; **gallons consumed 101,998,657 printed, not recorded** (I3) |
| 24, 25 | World Airways 2003, 1999 | no hedging inferred | **label wrong**: the report states it enters no futures or swap contracts, an explicit statement (I2) |
| 26 | ExpressJet 2009 | no hedging inferred | correct; consumption 5.5 million gallons is right but covers only fuel ExpressJet itself buys (I7) |
| 27 | Republic 2006 | not discussed | correct (no fuel-hedging statement) |
| 28, 37 | Baltia 2009, 2010 | not discussed | correct (development-stage company, Item 7A omitted) |
| 29, 32 | Air T 2017, 2019 | not discussed | correct (only interest-rate swaps; fuel reimbursed by FedEx) |
| 30 | Alpine Air 2009 | not discussed | defensible; the report mentions a "loss on heating oil investments", the agent could not tell whether they were hedges |
| 31 | SkyWest 2019 | not discussed | correct; partial gallons table (prorate flying) not captured (I3) |
| 33 | United 2016 | stopped this year | correct |
| 34 | Delta 2023 | refinery hedges only | correct |
| 35 | Evergreen 2005 | no hedging | correct |
| 36 | TWA 1999 | no open positions | correct |

**Recall of the evidence selection.** For 28 of the 38 reports the search of the full text found no fuel-hedging passage that the evidence bundle lacked. For the other 10 it returned only boilerplate ("cost of hedging has increased", risk-factor text), XBRL/JSON noise, or unrelated text (aircraft purchase options); I read these for the large hedgers and for Republic 2006 and Air T 2017, but did not read the single leftover hit for United 2020 and Delta 2023. No sampled status depends on a missed passage.

## Issue register (with size over the whole dataset)
| ID | Issue | Size | Effect | Suggested fix |
|---|---|---|---|---|
| I1 | **Partial-period figure used as the next-year headline.** When only one row exists for next year the build accepts it even if the period is a quarter or a few months (Delta FY2006 36% = March-2007 quarter; Northwest 2000/2001 Q1; Gulfstream 2 months; US Airways Q1/Q2...) | 16 of the 20 `single_row` headlines (the other 4 are annual) | `hedge_pct_next` is overstated as annual coverage for 16 reports | accept a single row only if its period is annual; otherwise leave empty and keep the value in min/max |
| I2 | **Explicit "no futures / swaps" statements labelled inferred** because the check looks only for "hedg" or "derivative" | 12 of 76 `no_hedging_inferred` (World Airways 1999-2003, World Air 2004-06, Evergreen 2004, Pinnacle 2009, Republic 2004, US Airways 1999) | explicit zeros undercounted | widen the pattern (futures, swap, forward, fixed-price) |
| I3 | **Fuel gallons consumed printed but not captured.** Operating-statistics tables are outside the cue windows | regex flags 118 of 282 reports without a value; spot check suggests about half are real (about 55) | consumption denominator sparse (226 of 603) | targeted read of the operating-statistics table, or a table regex |
| I4 | **UPS "hedger" without positions** (generic "we use options, swaps and futures" sentence; FY2008 fair value $0) | 9 UPS reports plus Mesa FY2008 | status over-states open hedging; UPS is a parcel carrier | treat "uses" language without any quoted position or non-zero fair value as `programme_no_open_positions` |
| I5 | **Several instruments for the same year** (collars + fixed forward prices; options + fixed-price contracts) give no headline | 25 reports | headline missing, min/max kept; overlap between instruments is not disclosed | keep blank, or sum with a flag when instruments are stated to be additive |
| I6 | **Percentages "as of" a January date** rather than the balance-sheet date | 29 of the 161 headline reports (Southwest 11, AMR 6, Hawaiian, Northwest, Delta, AirTran, Frontier, Atlas) | basis differs by a few weeks; Southwest FY2013: 43% at year-end vs 20% in January | add a `basis_date` column |
| I7 | **Consumption for pass-through carriers** covers only fuel the carrier itself buys | ExpressJet 2009 (5.5 vs 172.9 million gallons the year before) | not comparable across years | flag in a column or blank for pass-through carriers |

None of I1-I7 changes the finding that 37 of 38 sampled statuses are right; I1, I2 and I4 change published labels or the headline and should be fixed before the headline column is used for inference.

## Fixes applied after this audit (2026-10-08)
| ID | Status | Result |
|---|---|---|
| I1 | fixed | `hedge_pct_next` now comes only from an annual figure (or the mean of four quarters). 31 reports with only part-year figures lost the headline (min / max kept, basis `partial_period_only`); 4 gained one through a better annual-label rule ("FY" rows); headline count 161 to 149 |
| I2 | fixed | wider negation pattern (futures, swaps, forwards, fixed-price, "chosen not to"); 17 reports moved from `no_hedging_inferred` to explicit `no_hedging`, all 17 quotes read |
| I4 | fixed | 7 reports (UPS 1999, 2002, 2003, 2005, 2006, 2008 and Mesa 2008) with an "active" claim but no position and no or zero fair value are `no_open_positions`; the column `weak_active_claim` marks them; they no longer count as earlier hedging |
| I6 | fixed | new column `hedge_pct_next_asof`: 80 `year_end`, 15 `after_year_end` (AMR 6, Southwest 4, Hawaiian 3, Atlas 1, Delta 1), 54 `mixed_or_unspecified` (the quote carries no date, e.g. quarterly tables) |
| I3, I5, I7 | open | I3 needs a targeted read of the operating-statistics tables (about 55 reports); I5 leaves 6 reports without a headline; I7 is a documentation caveat |
Validation after the fixes: 28 of 28 checks pass (three new checks, L-6 to L-8, test I1, I4 and I6).
