# Plan: ACA Medicaid Expansion and Uninsurance Convergence Across States

## 0. Agent Instructions

Read this section first. The rest of the plan is the spec; this section says how to carry it out.

- **Deliverable:** `week-6-assignment.ipynb`, built following the outline in §6. It must run top to bottom from a fresh kernel with no errors:
  `jupyter nbconvert --to notebook --execute --inplace week-6-assignment.ipynb`
- **Environment / kernel:** use the **`base` / Python 3.14** kernel (anaconda, `/opt/anaconda3/bin/python`). This matches the kernelspec already in `week-6-assignment.ipynb` and `policy-dates.ipynb`, and the interpreter in `.vscode/settings.json`. Install dependencies into that environment with `/opt/anaconda3/bin/python -m pip install -r requirements.txt`. If the `base` kernel is not registered, register it with `/opt/anaconda3/bin/python -m ipykernel install --user --name python3 --display-name "base"`. Do not switch the notebook to the `.venv` (Python 3.9) interpreter.
- **Policy dates:** the single source of truth is the `EXPANSION_YEAR` dictionary in `policy-dates.ipynb`. Load it into the main notebook with `%run policy-dates.ipynb`. Do not retype it, change it, or fill in dates from memory.
- **Rules:**
  - Never print, log, or commit `CENSUS_API_KEY`. Load it with `python-dotenv` from `.env`.
  - Do not change the economic design (outcome, groups, sample years, specifications) without asking the user first.
  - If an `assert` fails, stop and report what failed and the observed values. Do not loosen the check or work around it.
- **Out of scope unless asked:** new data sources, event-study or synthetic-control models, extra covariates.

## 1a. Economic Question

Did ACA Medicaid expansion lower state uninsurance rates? And did it speed up convergence between high-uninsurance and low-uninsurance states? High-uninsurance states tend to have lower incomes than states with low uninsurance rates.

### 1b. Expectation

After 2014, uninsurance should fall in every state, because ACA marketplaces and subsidies started everywhere. But it should fall more in states that expanded Medicaid. Among these expansion states, the ones that started with high uninsurance should see the largest percentage-point drops. So uninsurance rates should converge faster among expansion states than among states that did not expand (non-expansion states).

## 2. Data Sources

All data are pulled inside the notebook with `requests` from the U.S. Census Bureau Data API. The key is stored in `.env` as `CENSUS_API_KEY` and loaded with `python-dotenv`.

**Caching:** save each raw API response to `data/raw/acs_{year}.csv` or `data/raw/sahie_{year}.csv`. Before calling the API, check whether that file already exists; if it does, read it instead. This makes reruns fast and deterministic. Add `data/raw/` to `.gitignore`.

### 2a. American Community Survey (ACS) 1-year detailed tables

- Endpoint: `https://api.census.gov/data/{year}/acs/acs1`
- Years: 2010–2019 and 2021–2022 (12 years). The Census Bureau did not release standard 2020 ACS 1-year estimates, so 2020 is not in the API.
- Geography: `for=state:*` (50 states + DC). Puerto Rico (FIPS 72) is not returned by this endpoint, but filter it out if it appears.
- Variables: include `NAME` in the `get=` list, plus the following from table B27010 (Types of Health Insurance Coverage by Age):
  - `B27010_001E`: total civilian noninstitutionalized population
  - `B27010_002E`, `B27010_018E`, `B27010_034E`: population in the three under-65 age groups
  - `B27010_017E`, `B27010_033E`, `B27010_050E`: no health insurance coverage in those three groups
  - `B27010_066E`: no health insurance coverage, 65 and over (used only for the all-ages rate)
  - Note: the youngest age band is labeled "Under 18" in older years and "Under 19" in newer years. The line numbers are the same, so the totals are unaffected.
- Covariate: `B19013_001E`, median household income (the 2013 value is used as the baseline).

### 2b. Small Area Health Insurance Estimates (SAHIE), robustness check

- Endpoint: `https://api.census.gov/data/timeseries/healthins/sahie`
- Variables: `NAME`, `PCTUI_PT` (percent uninsured), with `IPRCAT=3` (income at or below 138% of poverty, the group Medicaid expansion targets), `AGECAT=1` (ages 18–64), and `for=state:*`. Make one call per year for 2010–2022, using `time={year}`.
- Do not assume every year is available. Print which years the API actually returned before using the data.

### 2c. Policy coding

- Expansion years come from the `EXPANSION_YEAR` dictionary in `policy-dates.ipynb` (source: KFF, "Status of State Medicaid Expansion Decisions"). Keys are state postal codes; values are the calendar year expansion took effect, or `None` for states that have not expanded.
- The Census API identifies states by FIPS code, not postal code. Build an explicit FIPS → postal-code mapping (51 entries) in the notebook and use it to merge. Assert that every state in the ACS data matches a key in `EXPANSION_YEAR`, and the reverse.
- Known caveats, to mention in the interpretation: WI covers adults up to 100% FPL without having expanded; DE, DC, MA, NY, and VT already had broad coverage before 2014.

## 3. Cleaning

Each step ends with checks that must pass before moving on.

1. Loop over the years, call the API (or read the cache), and stack the results into one state-year DataFrame. Convert all numeric columns from strings to numbers.
   - Assert: `len(acs) == 51 * 12`; `(state, year)` pairs are unique; 2020 does not appear.
2. Keep the 50 states + DC.
   - Assert: 51 states in every year; no missing values in the B27010 or B19013 columns.
3. Compute the outcome: `uninsured_rate_under65 = (B27010_017E + B27010_033E + B27010_050E) / (B27010_002E + B27010_018E + B27010_034E) * 100`. Also compute the all-ages rate for reference.
   - Assert: every rate is between 0 and 40.
4. Map each state to its expansion year from `EXPANSION_YEAR` and define the groups:
   - **Treated:** `EXPANSION_YEAR == 2014`. This is 27 states including DC, and includes MI and NH, which expanded partway through 2014.
   - **Control:** `EXPANSION_YEAR is None` or `>= 2023`. These states had not expanded by the end of 2022: 10 that never expanded, plus NC and SD, which expanded in 2023. That makes 12.
   - **Late expanders:** `2015 <= EXPANSION_YEAR <= 2021`. That makes 12. They are dropped from the main DiD because they become treated partway through the window. They are used only in a robustness check.
   - Assert: group counts are 27 / 12 / 12 and sum to 51.
5. Create `post = year >= 2014` and `treated_post = treated * post`.
   - Assert: the main DiD sample (treated + control) has `39 * 12 = 468` rows.
6. Merge the 2013 median household income and the 2013 uninsurance rate onto every state as baseline variables.
   - Assert: no missing baseline values.

## 4. Charts and Tests

**Weighting:** all means and regressions are **unweighted**, so each state counts equally.

### 4a. Charts: each with a title, labeled axes with units, and a source note

1. **Trends:** mean under-65 uninsurance rate by year, treated vs. control, with a dashed vertical line at 2014. This is the visual check for parallel pre-trends. Leave a gap at 2020; do not connect the line across the missing year.
2. **Dispersion (sigma convergence):** standard deviation of state uninsurance rates by year, for each group.
3. **Catch-up scatter (beta convergence):** change in uninsurance from 2013 to 2019 (y) against the 2013 uninsurance rate (x), colored by group, with a fitted line for each group.

### 4b. Tests

1. **Two-way fixed effects difference-in-differences.**
   `smf.ols("uninsured_rate_under65 ~ treated_post + C(state) + C(year)", data=did).fit(cov_type="cluster", cov_kwds={"groups": did["state"]})`
   State and year fixed effects absorb the separate `treated` and `post` terms. Note in the write-up that with 39 clusters, clustered standard errors are acceptable but somewhat imprecise.
2. **Convergence regression**, one row per state (treated + control):
   `smf.ols("change_2013_2019 ~ baseline_rate_2013 * treated", data=conv).fit(cov_type="HC1")`
3. **Simple check:** a Welch two-sample t-test (`scipy.stats.ttest_ind(..., equal_var=False)`) of the 2013–2016 change in uninsurance, treated vs. control.
4. **Robustness:**
   - (a) Rerun test 1 using the SAHIE ≤138% poverty, ages 18–64 rate as the outcome.
   - (b) Rerun test 1 on all 51 states, with `treated_post = (year >= EXPANSION_YEAR)` for each state that expanded by 2021. Control states (`None` or ≥ 2023) stay at 0.

Collect the results of tests 1–4 into one summary table with columns: test, coefficient of interest, SE, p-value, N.

## 5. What would support or contradict the expectation

- **Supports:** the `treated_post` coefficient is negative and significant (p < 0.05); pre-2014 trends in chart 1 look parallel; the slope on `baseline_rate_2013` is more negative for treated states (a negative interaction); and dispersion in chart 2 falls faster for the treated group.
- **Contradicts:** `treated_post` is zero or positive; or the groups were already diverging before 2014, which would undermine the DiD design; or there is no difference in catch-up between the groups.

## 6. Notebook Outline (`week-6-assignment.ipynb`)

Build the cells in this order, with a short markdown header before each section:

1. Title and author (already present)
2. Setup: imports, `load_dotenv()`, read `CENSUS_API_KEY` (do not print it), create `data/raw/`
3. Policy table: `%run policy-dates.ipynb`, then the FIPS → postal mapping and its asserts
4. ACS pull (cached), §2a
5. SAHIE pull (cached), §2b
6. Cleaning and asserts, §3
7. Charts 1–3, §4a
8. Tests 1–4 and the summary table, §4b
9. Interpretation (markdown): go through each §5 criterion, say whether it is met, and note the caveats from §2c

## 7. Definition of Done

- [ ] The notebook runs cleanly from a fresh `base` / Python 3.14 kernel using the `nbconvert` command in §0.
- [ ] Every `assert` in §3 passes.
- [ ] All 3 charts have a title, axis labels with units, and a source note.
- [ ] The summary table reports coefficient, SE, p-value, and N for each test.
- [ ] The interpretation states whether each §5 criterion is met or not met.
- [ ] The API key appears nowhere in the notebook outputs or the committed files.