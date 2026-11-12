# PPK Operations: proposal performance analytics

A fact base on proposal performance across PPK's 14 offices and 11 practices, from January 2025 to the extract date of 15 October 2026. It is input to FY27 planning.

> All data is fictitious and was prepared for this assessment.

## Repository contents

```
.
├── README.md                       this file
├── Answer1.ipynb                   data profiling, cleaning, public holidays (Task 1), business days, export
├── Clean files/                    clean, model-ready data (output of the notebook, input to Power BI)
│   ├── fact_proposals.csv          one row per proposal
│   ├── fact_status_events.csv      one row per status change
│   ├── dim_office.csv              office → country, region, holiday-data flag
│   └── public_holidays_2025_2026.csv   national holidays per office country (Nager.Date API)
├── PPK Dashboard.pbix              Power BI dashboard: Growth, Speed, Data Quality pages
└── PPK_Executive_Summary.pptx      2-slide executive summary: findings and recommendations
```

The raw extract (`proposals.csv`, `status_events.csv`, `offices.csv`, `data_dictionary.csv`) and the case instructions are not committed. They were supplied with the case.

---

## How to reproduce

1. **Environment:** Anaconda (Python 3) with `pandas`, `numpy`, `requests`, `matplotlib`, `seaborn` and `openpyxl`.
2. **Notebook:** in `Answer1.ipynb`, point the three `read_csv` paths at your copy of the raw extract. Then run all cells in order. The holiday step calls `https://date.nager.at`, so you need an internet connection. The last cells write the four files in `Clean files/`.
3. **Dashboard:** open `PPK Dashboard.pbix` in Power BI Desktop. The data is embedded, so the report works straight away. To refresh it from your own clean files, go to *Transform data → Data source settings*, point the three sources at `Clean files/`, and click Refresh.

---

## What the notebook does (`Answer1.ipynb`)

| Section | Purpose |
|---|---|
| Load | Reads all raw files **as text** (`dtype=str`), so leading zeros and mixed date formats stay visible instead of being converted silently |
| Profiling | Counts of office and practice spellings, date-format patterns, value distribution, duplicates, key mismatches, DD/MM vs MM/DD test, practice names by year |
| Clean proposals table | Applies the fixes in the data issues log below (duplicates, IDs, test records, dates, names, practice rename, value units) |
| Clean events table | Normalises IDs, drops test events, and builds one row per proposal with outcome, review date, decision date and calendar-day turnaround |
| Task 1: public holidays | Calls Nager.Date for each office country for 2025–2026 and keeps **national** holidays (`global = true`). Countries with no data are flagged, not treated as errors |
| Business days | `numpy.busday_count` from submission to decision per country, excluding weekends and national holidays |
| Export | Writes the star-schema files to `Clean files/` |

---

## Data model (Power BI)

A **star schema**: one row per proposal in the fact table, with geography held in a dimension.

| Table | Grain / role | Key columns |
|---|---|---|
| `fact_proposals` | One row per proposal (24,372 after cleaning) | `proposal_id`, `office` → `dim_office`, `submitted_date` → `dim_date`, `practice`, `proposal_value_usd`, `value_flag`, `outcome`, `review_date`, `decision_date`, `turnaround_calendar_days`, `turnaround_business_days`, `tat_band` (calculated) |
| `dim_office` | One row per office (14) | `office`, `country`, `country_code`, `subdivision_code`, `region`, `holidays_available` |
| `dim_date` | One row per day, 1 Jan 2025 to 15 Oct 2026 (DAX calendar table) | `Date`, `Year`, `Year Quarter`, `Month`, `Year Month`, `Is Jan-Sep`, `Is Partial Month` |
| `fact_status_events` | One row per status change | `proposal_id` → `fact_proposals`, `event_date`, `status` |
| `dq_issues` | The data issues log, shown on the Data Quality page | issue, rows affected, impact, treatment |

The DAX measures are grouped into display folders: Volume & Value, Outcomes, Speed, Growth, Data Quality.

---

## Metric definitions

| Metric | Definition |
|---|---|
| **Volume** | Number of proposals, by `submitted_date` |
| **Value** | Sum of `proposal_value_usd` (estimated, USD) |
| **Outcome** | The proposal's final status: Won, Lost or Withdrawn. **Open** means no decision by 15 Oct 2026 |
| **Win rate** | Won ÷ (Won + Lost). Withdrawn and Open proposals are excluded, because the client never chose |
| **Turnaround (TAT)** | **Business days** from submission to decision (Won, Lost or Withdrawn). Saturdays, Sundays and national public holidays for the office's country are excluded. Submission day counts and decision day doesn't, so Monday → Tuesday = 1. Medians are reported, because the distribution has a long tail |
| **% decided ≤ 20 BD** | Decided proposals with TAT ≤ 20 business days ÷ all decided proposals |
| **Growth (YoY)** | Jan–Sep 2026 vs Jan–Sep 2025, like for like. Dubai and Riyadh August is excluded in **both** years (see issue 13) |
| **Win rate Δ (pp)** | Win rate in the selected period minus the same period last year, in percentage points |

---

## Data issues log

| # | Issue | Rows affected (raw) | Impact if untreated | Treatment |
|---|---|---|---|---|
| 1 | Exact duplicate proposal rows | 300 | Volume and value counted twice | Dropped exact duplicates |
| 2 | `proposal_id` zero-padded in proposals (`0000995`) but not in status events (`995`) | All IDs | Events don't join to proposals | Stripped leading zeros on both sides |
| 3 | Test records: `ZZ…` IDs, clients "TEST CLIENT", "Test - do not use", "Dummy Co", with round values of 1M–5M | 50 proposals, 100 events | Inflated value and win counts | Removed, along with their events |
| 4 | Three date formats: `YYYY-MM-DD`, `DD/MM/YYYY`, `DD Mon YYYY` | 8,678 | Wrong or missing dates | Parsed each format explicitly. DD/MM confirmed because only the first part is ever > 12 |
| 5 | Office names with different case or trailing spaces (56 variants for 14 offices) | 753 | Offices split into many groups | Trimmed and matched to the office list without regard to case |
| 6 | Practice names with different case or trailing spaces | 717 | Practices split into many groups | Trimmed and mapped to the 11 standard names |
| 7 | "Customer Strategy & Marketing" (2025) appears as "Customer" from 2026 | 952 | Looks like one practice ended and a new one grew from zero | **Assumption:** a rename. Merged into Customer Strategy & Marketing |
| 8 | Values under 1,000 USD (e.g. 208.3), unlike the rest of the distribution (median about 244k) | 596 | Value understated | **Assumption:** entered in thousands. Multiplied by 1,000 and flagged `scaled_x1000` |
| 9 | One value of 1.5 billion USD | 1 | One proposal distorts every value total | **Assumption:** a units error. Divided by 1,000 and flagged `scaled_div1000` |
| 10 | Proposals with no decision yet (Open) | 1,152 | Biases win rate and TAT | Kept for volume; excluded from win rate and TAT |
| 11 | October 2026 is a partial month (extract taken 15 Oct) | 610 | The latest period looks like a drop | Growth compares Jan–Sep with Jan–Sep. `Is Partial Month` flag in `dim_date` |
| 12 | Nager.Date returns no data (HTTP 204) for India, UAE and Saudi Arabia | 3 offices | Business days include local holidays | Weekends only for those offices; flagged `holidays_available = FALSE` |
| 13 | Dubai and Riyadh have **zero** submissions in August 2026. Decisions on their older proposals continued that month, and volume resumed in September | 2 office-months | India & ME growth shows −1.8% instead of +6.4% | Treated as an extraction gap. That office-month is excluded in **both** years in the growth measures only. Totals, win rate and TAT keep all rows |

Raw → clean: 24,722 proposal rows, −300 duplicates, −50 test records, **24,372 clean proposals**.

---

## Assumptions

- **Values are in USD**, as the data dictionary states. No currency conversion is applied.
- **Units corrections:** values under 1,000 were entered in thousands, and the single value over 100M is a units error. Both are flagged in `value_flag`, so they can be audited or excluded.
- **Practice rename:** "Customer" is a rename of "Customer Strategy & Marketing", because the two names never appear in the same period.
- **Missing extract:** the Dubai/Riyadh August 2026 gap is a missing extract, not a real stop in business.
- **One decision per proposal:** each proposal has at most one decision (Won, Lost or Withdrawn). This was checked and holds in the data.
- **National holidays only:** regional holidays (`global = false`) are not excluded, as the brief specifies. Each office uses its country's calendar.
- **Recent cohorts are immature:** about 15% of Q3 2026 proposals are still open. Recent turnaround and win-rate figures will change as the slower ones close.

---

## Dashboard (`PPK Dashboard.pbix`)

Region, office, practice and period slicers sit on the analytical pages.

1. **Growth:** KPI cards for volume, value, win rate and YoY change; monthly trend against the prior year; growth by practice; region/office scorecard; practice scatter of volume growth against win-rate change. Defaults to Jan–Sep 2026 vs Jan–Sep 2025.
2. **Speed:** median TAT, % decided ≤ 20 BD, and open proposals; TAT by quarter and region; TAT by office; win rate by TAT band; TAT by practice.
3. **Data Quality:** raw → clean summary, data-quality KPIs, the issues log, a monthly submissions-by-office heatmap (it shows the August gap), and value corrections.

---

## Key findings (Jan–Sep 2026 vs Jan–Sep 2025)

- **Growth:**
  - Firm totals: volume **+11.8%** (10,762 proposals), value **+10.3%** ($3.66bn), win rate 40.1% → **41.6%**.
  - **APAC** grew fastest (+15.0%) and **Americas** wins most (~50%).
  - **EMEA** is the only region whose win rate fell (−1.2 pp).
  - **India & ME** has the lowest win rate (32.9%) but improved the most (+4.3 pp).
- **Practices:** Performance Improvement, TMT and Strategy lead volume growth (+16–17%). Energy & Natural Resources and M&A are almost flat in volume, but their win rates rose the most (+4–5 pp).
- **Speed:**
  - Median TAT has held at **17 business days** since 2025.
  - **Paris** is the outlier: a 23 BD median, with only 38% decided within 20 BD (about 65% elsewhere).
  - Most of the elapsed time comes after review, i.e. the client's decision.
  - Win rate is about 40% in every TAT band.

## Recommendations

1. **Fix Paris turnaround.** Review the Paris bid process and target ≤ 17 BD.
2. **Back the growth engines and qualify harder.** Keep investing in PI, TMT and Strategy. Add a bid/no-bid gate where win rates are slipping (PI, Financial Services, EMEA). Shift business development towards Energy and M&A.
3. **Fix data at the source.**
   - Picklists for office and practice, and one date format.
   - Validation on value units, and no test records in production.
   - A monthly extract completeness check, and holiday calendars for India, UAE and Saudi Arabia.
