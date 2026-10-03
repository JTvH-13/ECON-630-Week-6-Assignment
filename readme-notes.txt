README.md has four parts:

1. Question: the economic question and the expected result, the treated and control groups, the outcome and years, and a short list of the charts, tests and two robustness checks. It points to Plan.md for the full specification and to the end of the notebook for the interpretation.
2. Data: the ACS tables, SAHIE and the KFF expansion dates. It notes that everything is pulled from the Census API inside the notebook.
3. Files: a short table of the project files.
4. How to rerun:
  a. Get a key at api.census.gov/data/key_signup.html and activate it from the email.
  b. Run cp .env.example .env and replace placeholder_value with the key. .env is git-ignored and the key is never printed.
  c. Run python -m pip install -r requirements.txt, using Python 3.14 / Anaconda base.
  d. Choose Run All, or use the nbconvert command. Keep policy-dates.ipynb in the same folder, because the notebook loads it with %run.