# DSC Project: Did the 2020 lockdown clear India's air?

CSE558 Data Science, midterm project (Monsoon 2026). Group **Day-ta**.

We took the hourly air quality data that CPCB publishes for Indian monitoring stations and asked a simple question: did PM2.5 really drop during the COVID-19 lockdown (25 March to 30 June 2020), and by how much? Along the way we tested four more questions about the same data (Delhi vs the rest of the north, differences between regions, weekday vs weekend NO2, and days above the national limit). The main point of the assignment was statistical inference, so most of the work is in choosing the right test for each question and checking that the answer doesn't depend on one assumption.

The full write-up is in the report PDF. This README just explains what is in the repo and how to run it.

## What is in this repo

```
DSC_Project/
├── CSE558_Midterm_Analysis.ipynb   # the whole analysis, run top to bottom
├── README.md
├── results.json                    # written by the notebook (all numbers used in the report)
├── figs/                           # figures saved by the notebook (jpg)
└── report/
    ├── CSE558_Midterm_Report.pdf   # final report
    ├── main.tex                    # LaTeX source (Overleaf)
    └── figs/                       # same figures, used by main.tex
```

If your copy of the repo is laid out a bit differently, the only thing that matters is that the notebook and `figs/` stay together.

## Data

Kaggle, [Air Quality Data in India (2015 to 2020)](https://www.kaggle.com/datasets/rohanrao/air-quality-data-in-india), which republishes CPCB data. We use two files from it:

- `station_hour.csv`: one row per station and hour, with PM2.5, PM10, NO, NO2, NOx, NH3, CO, SO2, O3, benzene, AQI and the AQI category
- `stations.csv`: city and state of every station

The raw file has 2,589,083 rows and 16 columns. After cleaning and encoding we have 2,352,157 rows and 69 columns, so 2,352,157 × 69 = 162,298,833, well above the 3 million required by the assignment. The data covers 110 stations in 26 cities from January 2015 to 1 July 2020.

You do not need to download anything by hand. The notebook downloads the dataset itself with `kagglehub`.

## How to run it

1. Open `CSE558_Midterm_Analysis.ipynb` in Jupyter or Google Colab.
2. Run all cells from the top. The first cells install `statsmodels` and `kagglehub` if they are missing.
3. When it finishes you will have `results.json` and the figures in `figs/`.

It takes a few minutes, mostly because of the permutation tests. Random seeds are fixed (558), so the permutation and bootstrap numbers are the same on every run.

Libraries used: numpy, pandas, scipy, statsmodels, matplotlib, kagglehub.

## What the notebook does

The notebook follows the structure of the report.

1. **Dataset and size check.** Loads both files, joins the city and state, builds `Region` (North, South, East, West, grouped from the state) and `Season` (from the month), and does the rows × columns check.
2. **Data preparation.** Each step is its own small cell with a printout:
   - duplicates (none found)
   - rows where every pollutant and the AQI are missing (236,926 dropped)
   - readings stuck at the sensor ceiling (set to missing)
   - the heavy right tail (capped at the 99.9th percentile, rows kept because the high hours are real smog episodes)
   - AQI above 500 (capped at 500)
   - short gaps of 1 to 3 hours (interpolated and flagged with `*_imputed`)
   - one-hot encoding of city, region and season, plus new columns such as `DelhiNCR` and `LockdownWin`

   All the tests use only observed values (`PM2.5_obs`, `NO2_obs`), never the interpolated ones.
3. **Before and after.** A table and a figure comparing the data before and after cleaning.
4. **EDA and problem statements.** The basic questions (dirtiest city, worst season and hour, PM2.5 vs PM10), a few plots with a one line takeaway each, and the five problem statements.
5. **Hypothesis tests.** One section per test, each with the hypotheses written before the result, the test, other tests we considered, the statistic, p-value, confidence interval, and whether the effect is big enough to matter.
6. **Summary.** Holm correction over the five tests and the conclusion.

One thing we did on purpose: hourly values of one station are almost copies of each other (lag-1 autocorrelation is 0.78), so using every hour as an independent observation would make every p-value meaningless. The tests therefore use one number per station (or one matched station-day for Test 5).

## The five tests

All at α = 0.05. The "matters" column is the smallest difference we decided, before running anything, would be worth caring about.

| # | Question | Test | Matters if |
|---|----------|------|-----------|
| 1 | Was PM2.5 lower in the lockdown window than in the same dates of 2019? | paired t-test on station means | 10 µg/m³ |
| 2 | Is winter PM2.5 higher at Delhi-NCR stations than at other northern cities? | Welch two-sample t-test | 15 µg/m³ |
| 3 | Do the four regions differ in 2019? | Welch ANOVA with Games-Howell | 10 µg/m³ between two regions |
| 4 | Is NO2 lower on weekends than on weekdays? | paired t-test, plus an equivalence test (TOST) | 5 µg/m³ |
| 5 | Did the share of days above 60 µg/m³ fall during the lockdown? | exact McNemar test | 10 percentage points |

For every test we also ran the obvious alternatives (Welch, Wilcoxon, sign test, pooled t, Kruskal-Wallis, two-proportion z and so on), a permutation test where it made sense, and a sensitivity check, and wrote down why we kept the test we kept.

## Main results

| # | Result |
|---|--------|
| 1 | PM2.5 fell by **24.2 µg/m³ (38.1%)**, from 63.0 to 38.8, over 71 stations. 95% CI: −27.9 to −20.5, p ≈ 8.6e-21. Way above the 10 µg/m³ threshold. 16 of 17 cities improved (Mumbai went up). |
| 2 | Delhi-NCR is about **72.3 µg/m³ higher** than other northern cities in winter (CI 36.5 to 108.1, p ≈ 6.1e-4). The north is bad everywhere, but Delhi-NCR is much worse. |
| 3 | Regions differ a lot (Welch F(3, 22.4) = 104.5, ω² = 0.73). North ≈ 102.5, East ≈ 65.4, South ≈ 39.4, West ≈ 37.8 µg/m³. South and West cannot be told apart. |
| 4 | Weekend NO2 is lower by 1.60 µg/m³ (4.6%). It is clearly real (p ≈ 1.6e-17) but the whole interval sits inside ±5, so it does not matter in practice. |
| 5 | Days above 60 µg/m³ fell from 46% to 16% (5,438 matched station-days, p < 1e-300). |

After Holm correction all five decisions stay the same.

## Things to keep in mind

- This is association, not proof of cause. 2020 also differs from 2019 in weather and monsoon timing, and we only control for the season by using the same calendar window.
- Stations in the same city share air, so they are not fully independent. The city-level check in Test 1 is the stricter version and it agrees.
- Test 2 compares 38 Delhi-NCR stations with only 9 others, so it has low power for small gaps. The gap we found is far above what it can detect.
- Our four-region grouping of states is a simplification.

## Team

| Name | Roll no. | What they did |
|------|----------|---------------|
| Shubham | 2023516 | EDA, hypothesis testing code and report |
| Aryan Rishi | 2023146 | Data cleaning and preparation |
| Divyank | 2023206 | Hypothesis test suggestions and problem statements |
