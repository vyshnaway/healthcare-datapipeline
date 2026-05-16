# Mini-HealthLake: Healthcare Data Platform

&gt; End-to-end Azure data pipeline: dirty data generation → ADF ingestion → 
&gt; Databricks Delta Lake → dbt star schema → Power BI dashboard.

## What This Solves
- Power BI timeout → Pre-aggregated Gold tables (1K rows vs 500M raw)
- GDPR deletes → Delta Lake soft-delete + time-travel
- Data quality → 3-layer framework (quarantine → Silver flags → dbt tests)

## Repo Map
| Folder | Day | What It Contains |
|--------|-----|------------------|
| `data_generator/` | 1 | Python scripts that create intentional data quality flaws |
| `adf/` | 2 | Azure Data Factory pipeline screenshots & ARM export |
| `databricks/` | 3 | PySpark notebooks for Silver layer + Delta Lake demo |
| `dbt/` | 4–5 | Star schema models, tests, and auto-generated docs |
| `powerbi/` | 5 | Dashboard .pbix file + relationship screenshots |
| `streaming/` | Optional | Event Hubs producer + Stream Analytics SQL |
| `docs/` | 6–7 | Architecture decisions, interview script, setup notes |

## Quick Start
1. `pip install -r data_generator/requirements.txt`
2. `python data_generator/generate_patients.py`
3. [Link to full setup in docs/setup_notes.md]

## Tech Stack
Python | Azure Data Factory | Databricks | Delta Lake | dbt | Power BI