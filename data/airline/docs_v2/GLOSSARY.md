# Glossary

**10-K** annual report that U.S. listed companies file with the SEC. **10-K405** the same report on a form variant used until 2002. **10-KT** a report for a transition period after a change of fiscal year. **10-K/A** an amendment (not read here).
**Accession number** the SEC's unique number for a filing, for example `0000092380-14-000010`.
**CIK** Central Index Key: the SEC's number for a registrant (company). Keep it as text.
**EDGAR** the SEC's free filing database (sec.gov). **SIC code** industry code (4512 scheduled air transport, 4513 air courier, 4522 non-scheduled air transport).
**EX-13** an exhibit that contains the company's annual report to shareholders; before about 2010 many airlines placed their risk disclosures there instead of in the 10-K body.

**Hedge / hedging** using financial contracts to fix or cap the price of future fuel purchases.
**Call option (purchased call)** the right to buy at a fixed price: a cap on fuel cost. **Collar** a purchased call plus a sold put: a cap and a floor. **Three-way / four-way collar** collars with an extra sold option. **Call spread** a bought call plus a sold higher call. **Swap** an agreement to exchange a floating price for a fixed one. **Fixed-price (forward) purchase agreement (FFP)** a supplier contract at a set price; counted as a hedge here.
**Proxy hedge** airlines hedge with contracts on crude oil, heating oil or Brent because jet fuel contracts are thinly traded; the commodity is recorded in `positions.csv`.
**Capacity purchase agreement (CPA)** a major airline pays a regional airline a fee per flight and buys the fuel; the regional carrier has no fuel price exposure (`regional_pass_through`).
**Pass-through** a company's fuel cost is passed to a customer or partner.
**Mainline** the large aircraft operated by the airline itself, excluding regional affiliates.
**Monroe** Monroe Energy, an oil refinery in Trainer, Pennsylvania, owned by Delta Air Lines since 2012.
**Year-end** the balance-sheet date of the fiscal year. A hedge "as of year-end" is measured on that date; "after year-end" is measured on a later date mentioned in the report.
**Coverage year** the year in which hedged fuel will be bought; the report is filed in the following year.
**Headline** the single summary number `hedge_pct_next`: hedged share of the next fiscal year's expected fuel use.
**Evidence bundle** the passages of a report (up to 56,000 characters) given to the first reader.
**Verbatim quote** text copied exactly from the filing, kept with every value so it can be checked.
**Adjudication** a manual correction after the two readings disagreed; all are logged.
**Blind read** a reading done without seeing any earlier result.
