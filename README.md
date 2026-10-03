# ACA Medicaid Expansion and Uninsurance Convergence Across States

ECON 630, Week 6 Assignment. Julia van Helmond.

## Question

Did the ACA's Medicaid expansion lower state uninsurance rates? And did it help states with high uninsurance catch up to
states with low uninsurance?

From 2014, ACA marketplaces and subsidies were available in every state, so uninsurance should fall everywhere. The
expectation is that it fell **more** in states that expanded Medicaid. Among those states, the ones that started with
high uninsurance should see the largest drops, so rates should converge faster among expansion states than among states
that did not expand.

The analysis compares 27 states (including DC) that expanded in 2014 with 12 states that had not expanded by the end of 2022.
The outcome is the under-65 uninsurance rate from 2010–2019 and 2021–2022. There is no standard 2020 ACS 1-year release.
The notebook covers:

- **Charts:** mean uninsurance trends, dispersion across states (sigma convergence), and catch-up from 2013 to 2019 (beta convergence).
- **Tests:** a two-way fixed effects difference-in-differences, a convergence regression, and a Welch t-test.
- **Robustness checks:**
  - **SAHIE outcome:** the main DiD rerun with the Census SAHIE uninsurance rate for adults 18–64 at or below 138% of poverty.
  - **Staggered timing:** the main DiD rerun on all 51 states, with each expansion state treated from its own expansion year.

See `Plan.md` for the full specification and the last section of the notebook for the interpretation.

## Data

All data are pulled from the U.S. Census Bureau Data API inside the notebook. No CSVs are downloaded by hand.

- **ACS 1-year estimates:** table B27010 (health insurance by age) and B19013 (median household income), for all states.
- **SAHIE** (Small Area Health Insurance Estimates): percent uninsured for adults 18–64 at or below 138% of poverty.
- **Medicaid expansion dates:** the `EXPANSION_YEAR` dictionary in `policy-dates.ipynb` (source: KFF, "Status of State
  Medicaid Expansion Decisions").

## Files

| File | Contents |
| --- | --- |
| `analysis.ipynb` | The analysis: data pulls, cleaning, charts, tests, interpretation |
| `policy-dates.ipynb` | Medicaid expansion year for each state; loaded by `analysis.ipynb` |
| `Plan.md` | Analysis plan and specification |
| `requirements.txt` | Python packages needed |
| `.env.example` | Template for the `.env` file that holds your Census API key |

## How to rerun the notebook

1. **Get a Census API key.** Request one at <https://api.census.gov/data/key_signup.html>. The key arrives by email,
   and you must activate it with the link in that email.

2. **Create your `.env` file.** From the project folder, copy the template:

   ```bash
   cp .env.example .env
   ```

   Open `.env` and replace `placeholder_value` with your key:

   ```dotenv
   CENSUS_API_KEY=your_key_here
   ```

   `.env` is listed in `.gitignore`, so your key is never committed. The notebook reads it with `python-dotenv` and never prints it.

3. **Install the packages.** The notebook was built with Python 3.14 (Anaconda `base`):

   ```bash
   python -m pip install -r requirements.txt
   ```

4. **Run the notebook.** Open `analysis.ipynb` in Jupyter or VS Code and choose **Run All**. To run it from the command line instead:

   ```bash
   jupyter nbconvert --to notebook --execute --inplace analysis.ipynb
   ```

   Keep `policy-dates.ipynb` in the same folder. The notebook loads it with `%run`.

### Notes

- **Caching:** the first run calls the Census API and saves each response to `data/raw/` (for example `acs_2013.csv`).
  Later runs read those files instead, so they are faster and give the same results. To pull fresh data, delete `data/raw/`.
  The folder is in `.gitignore`.
- **Checks:** the notebook contains `assert` checks (row counts, group sizes, rate ranges, no missing values). If one fails,
  the run stops at that cell and shows the values it observed.
