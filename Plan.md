# Plan: ACA Medicaid Expansion and Uninsurance Convergence Across States

## 1a. Economic Question

Did the ACA Medicaid Expansion policies lower state uninsurance rates, and did it increase the speed of convergence between high-ununisurance states and low-uninsurance states? High-uninsurance states tend to be lower-income compared to states with low rates of uninsurance.

### 1b. Expectation

After 2014 Medicaid Expansion, uninsurance will fall in all states since ACA marketplaces and subsidies started in all states. However, uninsurance rates will fall more in states that Expanded Medicaid. Among the states that expanded, those that had initially high rates of uninsurance should see the largest percentage-point drops. Among these 'expansion states,' convergence of uninsurance rates will thus occur faster than in the states that did not expand Medicaid 'non-expansion states.'

## 2. Data Sources

All data are pulled inside the notebook with `requests` from the U.S. Census Bureau Data API. The key is stored in `.env` as `CENSUS_API_KEY` and loaded with `python-dotenv`.

### 2a. American Community Survey (ACS) 1-year detailed tables

- Endpoint: `https://api.census.gov/data/{year}/acs/acs1`
- Years: 2010-2019 and 2021-2022. The Census Bureau did not release standard 2020 ACS 1-year estimates, so 2020 is not in the API.
- Geography: `for=state:*` (50 states + DC; Puerto Rico, FIPS 72, is not returned by this endpoint but is filtered out if present).
- Variables (table B27010, Types of Health Insurance Coverage by Age):
  - `B27010_001E` total civilian noninstitutionalized population
  - `B27010_002E`, `B27010_018E`, `B27010_034E` population in the three under-65 age groups
  - `B27010_017E`, `B27010_033E`, `B27010_050E` no health insurance coverage in those three groups
  - `B27010_066E` no health insurance coverage, 65 and over (used only for the all-ages rate)
  - Note: the youngest age band is labeled “Under 18” in older years and “Under 19” in newer years. The line numbers are the same, so the totals are unaffected.
- Covariate: `B19013_001E` median household income (2013 value used as the baseline).

### 2b. Small Area Health Insurance Estimates (SAHIE), robustness check

- Endpoint: `https://api.census.gov/data/timeseries/healthins/sahie`
- Variables: `PCTUI_PT` (percent uninsured), with `IPRCAT=3` (income at or below 138% of poverty, the Medicaid expansion target group) and `AGECAT=1` (ages 18-64), `for=state:*`, one call per year 2010-2022 using `time={year}`.

### 2c. Policy coding

- Expansion effective dates come from KFF, “Status of State Medicaid Expansion Decisions” (kff.org). They are hard-coded as a Python dictionary with a source comment, not downloaded as a file.

## 3. Cleaning

1. Loop over years, call the API, and stack the results into one state-year DataFrame. Convert all numeric columns from strings to numbers.
2. Keep the 50 states + DC (51 rows per year). Assert there are no missing values.
3. Compute the outcome: `uninsured_rate_under65 = (B27010_017E + B27010_033E + B27010_050E) / (B27010_002E + B27010_018E + B27010_034E) * 100`. Also compute the all-ages rate for reference.
4. Assign each state an expansion year from the KFF dictionary and define groups:
    - **Treated:** expanded during 2014 (27 states incl. DC, includes MI and NH which expanded mid-2014).
    - **Control:** had not expanded by the end of 2022 (12 states: the 10 states that never expanded plus NC and SD, which expanded in 2023).
    - **Late expanders (2015-2021):** dropped from the main DiD (they are “treated” in the middle of the window). Used only in a robustness check.
5. Create `post = year >= 2014` and `treated_post = treated * post`.
6. Merge the 2013 median household income and the 2013 uninsurance rate onto every state as baseline variables.

## 4. Charts and Tests

### 4a. Charts – each with a title, labeled axes with units, and a source note

1. Trends: mean under-65 uninsurance rate by year, treated vs. control, with a dashed vertical line at 2014. This is the parallel pre-trends check.
2. Dispersion (sigma convergence): standard deviation of state uninsurance rates by year, for each group.
3. Catch-up scatter (beta convergence): change in uninsurance 2013-2019 (y) vs. 2013 uninsurance rate (x), colored by group, with a fitted line for each group.

### 4b. Tests

1. **Two-way fixed effects difference-in-differences**:
`uninsured_rate ~ treated_post + C(state) + C(year)`, standard errors clustered by state. (State and year fixed effects absorb the separate `treated` and `post` terms.)
2. **Convergence regression**, one row per state:
`change_2013_2019 ~ baseline_rate_2013 * treated`, robust standard errors.
3. **Simple check: two-sample t-test** of the 2013-2016 change in uninsurance, treated vs. control.
4. **Robustness**: rerun test 1 with the SAHIE <=138% poverty rate, and with late expanders coded as treated from their own start year.

## 5. What would support or contradict the expectation

- **Supports:** the `treated_post` coefficient is negative and significant (p < 0.05); pre-2014 trends in chart 1 look parallel; the slope on `baseline_rate_2013` is more negative for treated states (negative interaction), and the dispersion in chart 2 falls faster for the treated group.
- **Contradicts:** `treated_post` is zero or positive, or the groups were already diverging before 2014 (which would undermine the DiD design), or there is no difference in catch-up between the groups.