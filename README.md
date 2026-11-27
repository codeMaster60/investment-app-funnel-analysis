# Investly: where do people quit before their first investment?

10,000 people signed up for an investment app.

Only 9% of them ever put money into a mutual fund.

I wanted to know where the other 91% went. So I built the app's data from scratch and went looking in SQL.

Quick heads up before you read further. Investly isn't real. The data is synthetic. I wrote `generate_data.py` to create it, and every rule I baked in is written down in `data/DATA_GENERATION.md`. So read this as a look at how I analyse a product, and read every finding as a correlation.

![Investly dashboard](dashboard/screenshots/dashboard.png)

Tools: PostgreSQL, Excel for the dashboard. Covers funnel, cohort and segment analysis, a PRD and an experiment design.

---

## The question

Where do users drop off between signup and their first investment?

Which users behave differently?

What should the product team test next?

## What I was working with

Four tables.

- `users`: 10,000 people who signed up between January and June 2026. Age group, city tier, acquisition channel, platform, risk profile.
- `events`: 125,149 in-app actions. Onboarding, browsing funds, investing, checking the portfolio.
- `investments`: 2,729 transactions, monthly SIP auto-debits included.
- `campaigns`: 30 rows of monthly marketing spend per channel.

I also messed up the data on purpose. Duplicates, events from users who don't exist, timestamps from before signup, negative amounts. Real data is never clean, so I didn't want mine to be clean either.

The journey I tracked:

```
Signup → KYC started → KYC completed → Bank linked → MF discovery
       → Fund viewed → Investment started → Investment completed → Repeat
```

---

## What I found

**1. Most people never got there.**

9.0% of signups invested. 35.3% activated in their first 7 days. (02, 08)

**2. The fund page is where people leave.**

Fund viewed to investment started converts at 35.5%. That's the weakest step in the whole journey. 2,378 people opened a fund, looked at it, and left. (03)

**3. Some people quit at checkout.**

31% of users who started an investment never finished it. These people had already made up their mind, which makes this the most expensive drop-off in the app. (03)

**4. Referral wins on cost.**

Referral converts best at 12.2% and costs ₹849 per investor. YouTube costs ₹3,031 per investor, about 3.6x more. Instagram has the worst conversion at 7.2% but is still the 2nd cheapest per investor. If I'd only looked at conversion, I would have ranked Instagram last. (04)

**5. Tier 3 cities and 45+ users fall behind at KYC.**

Tier 3 users complete KYC at 58.7% against 67.4% in Tier 1. Users aged 45+ complete it at 55.8% against about 66% for everyone else. Both groups invest less too, and their gap opens at KYC, before they've seen a single fund. (04)

**6. Week 1 to week 2 is the cliff.**

About 21% of users are still active a month after signing up. The biggest drop happens between week 1 and week 2. (05)

**7. The SIP calculator is interesting.**

Among people who viewed a fund, the ones who used the SIP calculator invested at 2.3x the rate of the ones who didn't. I can't say the calculator caused that. People who were already planning to invest might just be the ones who open a calculator. That's why I designed recommendation 3 as an A/B test. (06)

**8. SIP "repeat rate" flatters itself.**

SIP starters show a 91% repeat rate. Only 39% of them invested again on their own. The rest is the auto-debit doing its job. (07)

**9. People like looking at index funds. They don't buy them.**

Index funds are the 2nd most viewed category and convert the worst, 9.7% against 14.8% for equity. (07)

The numbers in brackets point to the SQL file in `sql/` behind each finding.

---

## What I'd tell the product team

1. Nudge people stuck in onboarding during their first week. It reaches the most users and it's the cheapest thing to test.
2. Fix checkout. Start logging why payments fail, then add retries and reminders.
3. Show an SIP projection right on the fund page. Run it as an A/B test, because the calculator result is only a correlation.
4. Talk to Tier 3 and 45+ users about KYC. Then test vernacular instructions and assisted KYC.
5. Do some research on index funds before building anything for them.
6. Track cost per investor instead of cost per signup, and move spend toward referral.
7. Measure SIP health with persistence. Repeat rate makes it look better than it is.

The full write-up has the hypotheses, success metrics, guardrails, RICE scores and the experiment design with sample size: [`insights/product_recommendations.md`](insights/product_recommendations.md)

The PRD for the SIP projection feature: [`insights/prd_sip_projection.md`](insights/prd_sip_projection.md)

---

## A mistake I almost shipped

My first retention query said retention goes up over time.

That can't happen.

I had used 1-day, 7-day and 30-day windows. A 30-day window has way more room to catch activity than a 1-day one, so of course the numbers climbed. I switched to equal 7-day windows and the curve finally made sense. The notes are in `05`.

A few other calls I made along the way:

- `COUNT(*)` would inflate every stage of the funnel, so I count distinct users everywhere.
- Search and the calculator are optional features, so I didn't treat them as funnel steps.
- When comparing feature users with everyone else, I only looked at people who had viewed a fund. Otherwise I'd be comparing people at different points in the journey.
- Auto-debits are kept separate when I talk about repeat investing.

## Cleaning the data first

I ran 8 checks before touching any analysis. They caught:

- 600 duplicate events
- 25 events from users who don't exist
- 30 events logged before the user had even signed up
- 6 investments of zero or less
- 4 duplicate transactions
- 8 users with no age

The raw tables stay untouched. Everything runs on cleaned copies.

## SQL I used

JOINs and LEFT JOINs (including referential integrity checks), GROUP BY with HAVING, CASE WHEN, COUNT(DISTINCT), COALESCE, chained CTEs, DATE_TRUNC, INTERVAL, UNION ALL, subqueries, ROW_NUMBER() and LAG(), and PERCENTILE_CONT for medians.

---

## What's in the repo

```
investment-app-funnel-analysis/
├── README.md
├── generate_data.py                 # builds the synthetic dataset (seed 42)
├── data/
│   ├── users.csv  events.csv  investments.csv  campaigns.csv
│   └── DATA_GENERATION.md           # every rule built into the data
├── sql/
│   ├── 00_setup.sql                 # create tables, load CSVs
│   ├── 01_data_quality.sql          # 8 data quality checks
│   ├── 01b_clean_data.sql           # clean analysis tables
│   ├── 02_user_activation.sql       # onboarding funnel, 7-day activation
│   ├── 03_investment_funnel.sql     # mutual fund funnel
│   ├── 04_user_segmentation.sql     # channels, cost per investor, segments
│   ├── 05_retention_cohort.sql      # monthly cohort retention
│   ├── 06_product_usage.sql         # calculator, search, watchlist vs investing
│   ├── 07_investment_analysis.sql   # repeat investing, gaps between investments, fund categories
│   └── 08_final_insights.sql        # headline KPIs
├── dashboard/
│   ├── investly_dashboard.xlsx
│   └── screenshots/dashboard.png
└── insights/
    ├── product_recommendations.md
    └── prd_sip_projection.md
```

## Running it yourself

1. Install PostgreSQL. On a Mac, Postgres.app is the easiest.
2. If you want to regenerate the data, run `pip install numpy pandas` and then `python generate_data.py`. This step is optional.
3. From the project folder:
   ```
   createdb investly
   psql -d investly -f sql/00_setup.sql
   psql -d investly
   ```
4. Inside psql, run the files in order, for example `\i sql/01_data_quality.sql`.

## What this can't tell you

- The data is synthetic, so every pattern comes from rules I wrote.
- Every behaviour comparison is a correlation. Proving cause needs a controlled experiment.
- I didn't run an A/B test. The one in `insights/` is only a design, and I'm not claiming any results.
- The channel analysis counts investors. It doesn't look at how much they invested or their lifetime value.
- Week 1 retention includes onboarding events, which makes it look higher than the weeks after it.

---

If you've worked on a fintech funnel and something here looks off to you, I'd really like to hear it.
