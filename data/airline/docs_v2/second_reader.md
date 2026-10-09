# Independent second reader (2026-10-08 / 09)

**Purpose.** The first read (evidence windows of at most 56,000 characters) could miss a hedging statement outside its windows, or choose the wrong scope. A second agent per report, with no access to any earlier result, searched the **complete** text of the annual report (10-K plus attached EX-13, `fulltext/` folder next to the raw filings) with Grep and Read and returned status, next-year hedged % with its as-of date, gallons consumed and net fair value, each with a verbatim quote (`code/pipeline/reader2.workflow.js`; quotes checked against the full text by `compare_reader2.py`).

**Coverage.** 443 of 603 reports (73%): 100 random reports (stratified by status), 121 hedger reports not yet hand-audited, 121 passenger / parcel / large cargo reports, and 101 reports chosen for disputed or missing consumption. Not re-read (160): regional pass-through carriers (fuel paid by partners) and small cargo and start-up carriers with no hedging statement; 38 reports were hand-audited separately (some overlap with the 443). Agent tokens: about 52M for the first read and retries, about 26M for this second read.

**Results (443 reports).**
| Check | Result |
|---|---|
| second-reader quotes found in the complete text | status 439 of 443, headline 436, consumption 442 |
| hedging direction agrees (open hedges vs none vs not discussed) | open/open 209, none/none 205 (zero-position and not-discussed classes combined); 5 cross-class cases |
| open hedges found in a report the dataset labels non-hedging | 0 (one ambiguous: UAL FY2000, contradictory text) |
| random 100: status identical / identical after treating "not discussed" and "no hedging" alike | 82 / 92 (the remaining differences are labels, not facts) |
| headline % identical within 1.5 points when both have one | 131 of 133 |
| as-of basis agrees when headline agrees | 128 of 131 |
| gallons consumed | 281 reports agree with the dataset value; before resolution 24 differed in scope (mainline vs consolidated), resolved by agreement of two of three sources; 3 reports where all sources differ are flagged |

**Disagreements and what was done** (all in `data/adjudications.csv` or rules in `build_final.py`):
- ambiguous classification kept visible: Delta ("primarily related to Monroe"), UAL FY2000, Alpine FY2008 (confidence `low`);
- 4 mislabelled "stopped this year" reports (stop in an earlier year) and 3 non-fuel "no derivatives" sentences counted as explicit statements: rules fixed;
- 2 headline scope errors corrected (America West FY2005, AMTRAN FY2001); 6 missing full-year headlines filled; Virgin America combined with fixed forward prices;
- consumption: 70 more reports filled; mainline-vs-consolidated resolved by agreement of two sources; Alaska FY2004 quarterly value rejected.

**Limits.** The second reader is the same kind of model as the first (shared blind spots are possible); it is independent in retrieval (complete text, own search), not in model family. 160 reports were not re-read; they belong to classes where the sample showed no error. No human reviewed the whole table.
