# Project rules

- Analysis lives in analysis.ipynb. Follow Plan.md.
- Load CENSUS_API_KEY from .env with python-dotenv. Never print, read aloud,
or hard-code the key, and never open the .env file.
- Pull all data from the Census API inside the notebook. No downloaded CSVs.
- Put repeated steps in functions with clear names. No unused imports or code.
- Every chart: title, axis labels with units, and a source note.
- Add a Markdown cell before each step explaining what it does and why.
- Add clean, concise comments to code
- Don't run git commands; I commit myself.
