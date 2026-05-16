# Mini-HealthLake: 1-Week Build Blueprint

> A condensed, portfolio-grade implementation of the HealthTech Solutions data platform. Designed for a fresher with Python + Power BI confidence, fast learning ability, and limited memory retention.

---

## Philosophy

**Scope discipline**: One clean end-to-end flow beats five broken ones.  
**Memory-safe**: Every file is commented like explaining to future self.  
**Interview-ready**: Each deliverable maps directly to a talking point in a 1-year data engineering role interview.

---

## Tech Stack

| Layer | Tool | Cost | Why Selected |
|-------|------|------|--------------|
| Data Generation | Python (pandas, Faker) | Free | Simulates real hospital dirty data without needing 12 source systems |
| Cloud Storage | Azure Blob / ADLS Gen2 | Free tier ($200 credit) | Required for ADF ingestion demo; mirrors Bronze layer |
| Orchestration | Azure Data Factory | Free tier | Managed, HIPAA-aligned, visual pipeline design |
| Processing | Databricks Community Edition | Free forever | Delta Lake + Spark without Azure compute costs |
| Transformation | dbt Core (local) | Free | SQL-based, testable, versioned; industry standard |
| Visualization | Power BI Desktop | Free | Star schema consumption layer; fixes the "Friday timeout" problem |
| Streaming (Optional) | Azure Event Hubs Basic | ~$11/mo or free tier | Simulated IoT vital signs for real-time alert demo |

---

## Day-by-Day Plan

---

### Day 1: Generate Dirty Data & Land It

**Goal**: Create realistic, intentionally flawed healthcare data and land it in Azure Blob Storage.

**Time**: 4-5 hours

#### Option A: Static CSV Download (Fastest)
- Download pre-generated Synthea CSVs from Kaggle
- Upload manually to Azure Blob `bronze/synthea/YYYY-MM-DD/`

**Why rejected**: No data quality flaws built in. You can't demo quarantine logic convincingly.

#### Option B: Python Synthetic Generator (Selected)
Write `generate_patients.py` that creates:
- `patients.csv` with duplicates ("John Smith" vs "Jon Smyth")
- `encounters.csv` with null `discharge_date`s and negative `cost` values
- `vital_signs.csv` with impossible vitals (150 C temperature, negative heart rate)

**Why selected**:
- You control the dirtiness = you can narrate every quarantine decision
- Shows Python proficiency
- Reusable: change volume from 1K to 1M rows with one variable
- Maps directly to the problem's "5-10% duplicate rate" reality

#### Option C: Mock REST API + Requests Ingestion
Build a FastAPI server locally that serves JSON patient records. Hit it with Python `requests` and save to CSV.

**Why rejected**: Adds Flask/FastAPI learning curve. The problem states "legacy REST API dumps" happen at night; simulating the API server is overkill for a 1-week build.

**Deliverable**:
- `data_generator/generate_patients.py`
- `data_generator/generate_encounters.py`
- `data_generator/generate_vitals.py`
- Files uploaded to Azure Blob `bronze/synthea/2026-05-15/`
- `docs/data_dictionary.md` explaining which columns are dirty and why

**Memory hack**: Comment every line. Save a `COMMANDS_DAY1.md` with exact Azure Portal click paths.

---

### Day 2: Ingest with Azure Data Factory

**Goal**: Build a managed pipeline that moves Bronze data with failure handling.

**Time**: 5-6 hours

#### Option A: Python Script with `azure-storage-blob` SDK
Write a Python cron job that copies files from Blob to ADLS.

**Why rejected**: No visual pipeline diagram for interviews. No built-in monitoring. No quarantine routing without custom code. Doesn't demonstrate "Azure Data Engineering" skills.

#### Option B: Azure Data Factory Copy Activity (Selected)
Build one ADF pipeline:
- Source: HTTP connector (public CSV URL) or Blob connector
- Sink: ADLS Gen2 `bronze/synthea/YYYY-MM-DD/`
- Error handling: Redirect bad rows (null `patient_id`, schema mismatch) to `quarantine/`
- Trigger: Schedule trigger (manual run for demo)

**Why selected**:
- Visual pipeline diagram is interview gold (screenshot it)
- Built-in monitoring, logging, HIPAA compliance certifications
- Quarantine routing is a checkbox, not 50 lines of Python
- Directly maps to "Hospital C-K: nightly REST API dumps" ingestion pattern

#### Option C: Azure Logic Apps + Functions
Trigger Azure Functions per file arrival, run custom validation logic.

**Why rejected**: Serverless is cool, but ADF Copy Activity already does this. Functions add debugging complexity. Save Functions for the streaming alert path (Day 7 optional).

**Deliverable**:
- Working ADF pipeline (screenshot the canvas)
- `quarantine/` folder with sample bad rows
- `docs/adf_setup.md` with connection string patterns

---

### Day 3: Silver Layer with Databricks

**Goal**: Clean, deduplicate, validate, and write to Delta Lake.

**Time**: 6 hours

#### Option A: Spark SQL Only (Simpler Syntax)
Write `.sql` notebooks in Databricks. Pure SQL with `CREATE TABLE ... USING DELTA`.

**Why rejected**: You explicitly asked for PySpark experience. Also, Delta Lake `DELETE`/`UPDATE`/`DESCRIBE HISTORY` is more intuitive in Python API for a beginner.

#### Option B: PySpark DataFrame API (Selected)
Write `silver_transformations.py` notebook:
- Read Bronze CSVs with `spark.read.csv()`
- `.filter()`, `.withColumn()`, `.when()` for quality flags
- `md5(concat_ws())` for change detection hashes
- Write Delta with `.write.format("delta").save()`
- Demonstrate GDPR delete with `DeltaTable.update()` and `DESCRIBE HISTORY`

**Why selected**:
- You are confident in Python; this leverages that strength
- PySpark is a distinct resume skill from "SQL"
- Delta Lake API (`DeltaTable.forPath`) is cleaner in Python than SQL for updates
- Data profiling (null heatmaps, groupBy counts) integrates naturally with PySpark

#### Option C: Databricks SQL + Delta Live Tables (DLT)
Use DLT declarative pipelines with `@dlt.table` decorators.

**Why rejected**: DLT is a paid Databricks feature. Community Edition doesn't support it. Also, DLT abstracts too much; you want to show you understand the mechanics.

**Deliverable**:
- `databricks/silver_transformations.py` (PySpark notebook)
- 3 Delta tables: `silver_patients`, `silver_encounters`, `silver_vital_signs`
- GDPR delete proof: `DESCRIBE HISTORY` screenshot
- `docs/delta_lake_decision.md` explaining "Why Delta over Parquet"

---

### Day 4: dbt Core — Star Schema + Tests

**Goal**: Build a versioned, tested semantic layer.

**Time**: 6 hours

#### Option A: dbt on Snowflake
Use Snowflake free trial as the warehouse.

**Why rejected**: The problem is Azure-native. Snowflake is a competitor. Also, Databricks has a free community tier; Snowflake trial expires in 30 days.

#### Option B: dbt on Databricks (Selected)
Install `dbt-databricks` adapter. Build models:
- `stg_patients.sql`, `stg_encounters.sql`
- `dim_patients.sql` (dimension: patient attributes, calculated age)
- `fct_encounters.sql` (fact: encounters with length_of_stay)

Add `schema.yml` with tests:
- `not_null: patient_id`
- `unique: encounter_id`
- `accepted_values: gender [M, F, U]`
- Custom test: `discharge_date >= admission_date`

**Why selected**:
- dbt is the industry standard for analytics engineering
- Databricks adapter lets you stay in the Azure ecosystem
- Tests are declarative YAML; easy to remember and demo
- `dbt docs generate` produces a lineage graph (interview visual asset)

#### Option C: Pure Spark SQL Procedures
Write stored procedures in Databricks for transformations.

**Why rejected**: No version control integration. No built-in testing framework. No documentation generation. dbt solves all three.

**Deliverable**:
- `dbt/models/staging/` and `dbt/models/marts/core/`
- `dbt/tests/` with custom SQL tests
- `dbt/docs/` auto-generated
- `dbt test` passing in terminal (screenshot for repo)

---

### Day 5: Gold Aggregates + Power BI Dashboard

**Goal**: Pre-compute aggregates and build a consumption layer that never queries 500M rows.

**Time**: 5-6 hours

#### Option A: Power BI DirectQuery on Silver Tables
Connect Power BI directly to `silver_vital_signs` (500M rows).

**Why rejected**: This is exactly the "Friday timeout" problem you're solving. DirectQuery on massive fact tables is the anti-pattern.

#### Option B: Pre-Aggregated Gold Tables + Power BI Import (Selected)
Add dbt analytics models:
- `patient_daily_stats.sql` — `GROUP BY patient_id, date` (AVG heart rate, MIN/MAX, count)
- `hospital_kpis.sql` — avg length of stay, readmission rate

Export Gold data to Power BI:
- Build star schema relationships (`dim_patients` -> `fct_encounters`)
- Write DAX measure: `AvgLOS = AVERAGEX(fct_encounters, DATEDIFF(...))`
- KPI cards: Total Patients, Total Encounters, Avg Cost
- One drill-through page by patient

**Why selected**:
- Solves the stated business problem: "Power BI timeout after 4 hours"
- 1M aggregated rows vs 500M raw vitals = instant query performance
- Demonstrates you understand "semantic layer" architecture
- `.pbix` file is a tangible portfolio asset

#### Option C: Tableau instead of Power BI
Build the dashboard in Tableau Public.

**Why rejected**: The problem explicitly mentions Power BI timeouts. Also, Azure-native roles expect Power BI. Tableau is a different vendor stack.

**Deliverable**:
- `dbt/models/marts/analytics/` (2 aggregate models)
- `powerbi/healthlake_dashboard.pbix`
- Screenshots of dashboard + relationship view
- `docs/powerbi_performance.md`: "How pre-aggregation fixed the timeout"

---

### Day 6: Python Profiling + GitHub Repo + Documentation

**Goal**: Polish the repo into a 5-minute readable portfolio.

**Time**: 4-5 hours

#### Option A: Minimal README Only
Write a short README and push.

**Why rejected**: For memory retention, you need extensive docs. Also, hiring managers skim; structured docs win.

#### Option B: Full Documentation Suite (Selected)
Structure:
```
/data_generator/     — Python scripts + requirements.txt
/adf/                — ARM export + screenshots
/databricks/         — .py notebooks
/dbt/                — Full dbt project
/powerbi/            — .pbix + screenshots
/streaming/          — Optional producer script
/docs/
  ├── architecture.md        — ASCII/Draw.io diagram
  ├── decisions.md           — 5 key justifications
  ├── devops_roadmap.md      — Reference CI/CD + rollback strategy
  ├── data_quality_report.md — PySpark profiling output
  └── setup_notes.md         — Your memory cheat sheet
README.md
```

Write `decisions.md` with 5 key justifications:
1. Why Delta Lake over Parquet? (GDPR deletes)
2. Why dbt over ADF Mapping? (Version control + tests)
3. Why pre-aggregation? (Power BI performance)
4. Why quarantine pattern? (Data quality without pipeline death)
5. Why soft-delete? (Compliance audit trail)

**Why selected**:
- `decisions.md` is your interview script. You read it before every call.
- `setup_notes.md` is your memory prosthetic.
- `architecture.md` gives the interviewer a visual anchor.

**Deliverable**:
- Clean GitHub repo with 6+ markdown docs
- `requirements.txt` for Python dependencies
- `.gitignore` for dbt `target/`, Power BI temp files

---

### Day 7: Demo Video + Production Roadmap

**Goal**: Record a 5-minute walkthrough and show you know what's missing.

**Time**: 3-4 hours

#### Option A: Live Screenshare with No Script
Wing it. Show your screen and talk.

**Why rejected**: You will forget talking points. One bad demo ruins the repo.

#### Option B: Loom Video with Storyboard (Selected)
Script the 5 minutes:
1. **0:00-0:45** — Python generator creating dirty data (show 150 C temperature)
2. **0:45-1:30** — ADF pipeline trigger + quarantine folder
3. **1:30-2:15** — Databricks: `DESCRIBE HISTORY` after a DELETE
4. **2:15-3:00** — `dbt test` passing in terminal
5. **3:00-3:45** — Power BI dashboard with star schema
6. **3:45-4:30** — dbt docs lineage graph
7. **4:30-5:00** — "Production Roadmap" slide

**Why selected**:
- Storyboard ensures you hit every talking point
- 5 minutes is the attention span of a hiring manager
- Ending with "what's next" shows scope discipline

#### Option C: No Video, Just Repo
Let the code speak for itself.

**Why rejected**: Most hiring managers won't run your code. A video guarantees they see the value.

**Deliverable**:
- 5-minute Loom/YouTube unlisted link
- README updated with video link
- "Production Roadmap" in README:
  - Phase 2: Event Hubs for real-time vitals (Stream Analytics for <10min alerts)
  - Phase 3: Azure Purview for data lineage
  - Phase 4: CI/CD with GitHub Actions + dbt Cloud
  - Phase 5: ML feature store for sepsis prediction

---

## Optional Extensions (If Time Permits)

### Extension A: Streaming Simulation (+3 hours)
Add to Day 1 or Day 7:
- `streaming/vitals_producer.py` — Python script sending JSON to Azure Event Hubs every 2 seconds
- `streaming/stream_analytics_query.sql` — 5-minute tumbling window alert logic
- Run producer in terminal during demo; show Stream Analytics job in Azure Portal

**Why add it**: Makes your architecture diagram complete (batch + streaming). One terminal window running a script is enough to say "this is simulated IoT."

### Extension B: DevOps Reference (+1 hour)
Add to Day 6:
- `.github/workflows/ci.yml` — conceptual pipeline (not wired to secrets)
- `Makefile` with `make test`, `make run`, `make docs`
- `docs/devops_roadmap.md` — environment promotion strategy

**Why add it**: Answers the DevOps question without spending a week on it.

### Extension C: Data Profiling Notebook (+1 hour)
Add to Day 3 or Day 6:
- `databricks/data_profiling.py` — PySpark null analysis, quality score JSON
- Visual: bar chart of invalid records by source

**Why add it**: Shows you think about data quality before transformation.

---

## Decision Summary Table

| Decision | Options Considered | Selected | Why |
|----------|-----------------|----------|-----|
| Data Generation | Static CSV / Python generator / Mock API | **Python generator** | Controlled dirtiness, reusable, shows Python skill |
| Ingestion | Python SDK / ADF / Logic Apps | **ADF** | Visual pipeline, managed, HIPAA-aligned, quarantine built-in |
| Processing Language | Spark SQL / PySpark / DLT | **PySpark** | Your confidence + Delta API + profiling integration |
| Lake Format | Parquet / Delta Lake | **Delta Lake** | GDPR deletes, ACID, time-travel — non-negotiable in healthcare |
| Transformation Tool | Spark procedures / dbt / ADF Mapping | **dbt** | Version control, tests, docs, SQL-native |
| BI Tool | DirectQuery / Pre-agg + Import / Tableau | **Pre-agg + Power BI Import** | Solves the 4-hour timeout problem directly |
| Streaming | Skip / Full Kafka / Event Hubs simulation | **Event Hubs simulation (optional)** | Completes architecture diagram, 30-line script |
| DevOps | Skip / Full CI/CD / Reference docs | **Reference docs + Makefile** | Answers question without week-long setup |

---

## Interview Translation Guide

When they ask... | You say... | You show...
----------------|-----------|------------
"How do you handle data quality?" | "3-layer framework: quarantine at ingestion, fuzzy deduplication in Silver, dbt tests in Gold." | `docs/data_quality_report.md` + dbt `schema.yml`
"How do you do GDPR deletes?" | "Soft delete in Delta Lake with `is_deleted` flag, then recompute Gold aggregates deterministically." | `DESCRIBE HISTORY` screenshot
"Why not just use Parquet?" | "Parquet doesn't support updates/deletes. GDPR requires deletion support. Delta Lake gives ACID + time-travel." | `docs/decisions.md`
"How do you prevent Power BI timeouts?" | "Pre-aggregate in dbt. Power BI queries 1K rows, not 500M raw vitals." | `.pbix` dashboard + relationship view
"What about real-time?" | "Batch for records, streaming for vitals. I simulated the streaming path with Event Hubs." | `streaming/` folder + demo video segment
"Where's your CI/CD?" | "I focused Week 1 on pipeline + modeling. Here's my production roadmap and reference GitHub Actions YAML." | `docs/devops_roadmap.md`

---

## Memory Retention Checklist

Because your memory won't persist long, keep these files updated daily:

- [ ] `COMMANDS_DAY1.md` — Azure Portal paths, connection strings
- [ ] `COMMANDS_DAY3.md` — Databricks cluster URLs, Delta table paths
- [ ] `COMMANDS_DAY4.md` — dbt commands that worked (`dbt run --select dim_patients`)
- [ ] `docs/troubleshooting.md` — Every error you fixed and how
- [ ] `docs/decisions.md` — Your interview script (read before every call)

**Rule**: If you spend more than 10 minutes fixing something, document it immediately.

---

## Final Deliverables

| Asset | Location | Purpose |
|-------|----------|---------|
| GitHub Repo | `github.com/you/healthlake-mini` | Portfolio centerpiece |
| Demo Video | Loom/YouTube (unlisted) | 5-minute walkthrough |
| README | Repo root | Hiring manager's first impression |
| Architecture Diagram | `docs/architecture.md` | Visual talking point |
| Decisions Log | `docs/decisions.md` | Justification practice |
| Power BI File | `powerbi/healthlake_dashboard.pbix` | Tangible BI skill proof |
| dbt Docs | `dbt/target/index.html` | Lineage graph screenshot |

---

> **Start Day 1 now. Do not over-engineer. One working flow beats five broken ones.**
