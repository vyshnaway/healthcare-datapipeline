# Azure Data Engineering Problem: Healthcare Analytics Platform

## Problem Statement

You've just joined **HealthTech Solutions**, a mid-size healthcare analytics company processing patient data across 50+ hospital networks in Europe. The company provides real-time clinical dashboards and predictive analytics to improve patient outcomes.

**Current State (Problematic)**:
- Patient records scattered across 12 different hospital systems (various databases: SQL Server, PostgreSQL, Access databases)
- Data analysts manually exporting CSVs each morning (2-3 hour delays)
- Power BI dashboards failing on Fridays (data refresh timeout after 4 hours of processing)
- No audit trail for HIPAA/GDPR compliance (critical for healthcare)
- Data quality issues: duplicate patient records, missing vital signs, inconsistent formats
- ML team frustrated: Features they need don't exist in accessible format, must wrangle raw data themselves

**Business Drivers**:
1. **Clinical Impact**: Doctors need real-time alerts for patient deterioration (must be <10 min from data entry)
2. **Compliance**: GDPR right-to-be-forgotten + HIPAA audit logs required
3. **Analytics**: Executive dashboards for hospital performance, patient outcomes, cost analysis
4. **ML**: Data science team training predictive models for patient readmission & sepsis detection

---

## System Requirements

### Data Volumes
- **Patient Records**: 5M active patients, 50M+ annual encounters
- **Vital Signs**: 500M data points/month (continuous monitoring)
- **Lab Results**: 100M results/month
- **Medications**: 200M prescription records
- **Hospital Operations**: 10M+ events/month (admissions, discharges, transfers)

### Service Level Agreements (SLAs)
| Consumer | Latency SLA | Freshness | Availability |
|----------|------------|-----------|--------------|
| Clinical Dashboards | <10 min | Real-time | 99.95% (healthcare critical) |
| BI/Executive Reports | 1-2 hours | Daily | 99.9% |
| ML Training Data | Daily | Overnight refresh | 99% |
| Audit/Compliance Log | Real-time capture | Immutable, 7-year | 99.99% |

### Compliance Requirements
- **GDPR**: Data deletion within 30 days, audit trail for access
- **HIPAA**: Access controls (role-based), encryption at rest/in-transit, audit logging
- **HITRUST**: Periodic compliance audits
- **Data Residency**: All data must stay in EU (Azure Europe regions only)

---

## Architecture Decisions to Make

### Decision 1: Data Ingestion Strategy

You have heterogeneous source systems:
1. **Hospital A**: SQL Server with change data capture (CDC) capability
2. **Hospital B**: PostgreSQL with WAL (write-ahead logs)
3. **Hospital C-K**: Legacy systems with no CDC (via nightly REST API dumps)
4. **Clinical Devices**: Real-time vital signs streaming via MQTT/Kafka

**Your Questions**:
- How do you ingest all this without losing data or creating bottlenecks?
- Should you use Azure Data Factory Copy Activity (fully managed, simple) or Azure Event Hubs + Stream Analytics (more complex, real-time)?
- How do you handle late-arriving data? (A patient's vital sign recorded 2 hours late due to network delay)
- Where do you store failed/malformed records? (Dead-letter queue strategy)

**Hidden Constraints**:
- Hospital systems can't handle frequent polling (max 1 request/min)
- Some systems go offline during backups (unpredictable downtime)
- Legacy systems with no CDC require full table scans (expensive, slow)
- Compliance requires capturing the original "source of truth" before any transformation

**Options Considered**:
- **Option A**: Azure Data Factory (Simplicity + Managed) — Pro: No ops overhead, built-in monitoring, HIPAA-compliant. Con: <10min latency hard to achieve.
- **Option B**: Event Hubs + Stream Analytics (Real-time + Flexibility) — Pro: Sub-second latency possible. Con: Higher ops complexity.
- **Option C**: Hybrid (Production Reality) — Pro: Best of both worlds. Con: More complex, requires careful data consistency management.

**Decision**: **Option C (Hybrid)**
- Vital signs (streaming) -> Event Hubs -> Stream Analytics -> Cosmos DB (real-time cache)
- Hospital records (batch) -> ADF Copy Activity -> ADLS (Bronze layer)
- Failed records -> ADF error handling -> separate Blob container (quarantine)
- Late arrivals -> Stream Analytics temporal join window (allow 2-hour grace period)

---

### Decision 2: Medallion Architecture Implementation

You need to design your Azure Data Lake Storage (ADLS) Gen2 with the Bronze-Silver-Gold medallion pattern.

**Bronze Layer** (Raw Data):
- Immutable, append-only event log
- Keeps original column names, data types, file paths
- Enables time-travel (point-in-time recovery)
- Example: `/bronze/hospital_a/patients/2025-05-15/`

**Silver Layer** (Cleaned, Deduplicated):
- Remove duplicates (multiple inserts of same patient record)
- Handle nulls consistently
- Standardize formats (patient IDs, date formats)
- Add data quality flags
- Example: `/silver/patients/version=2025-05-15/`

**Gold Layer** (Business-Ready):
- Aggregates, dimensions, facts (star schema ready for Power BI)
- Pre-computed metrics (patient risk scores, hospital KPIs)
- Optimized for BI queries
- Example: `/gold/patient_encounters/year=2025/month=05/`

**Technology Choices for Each Layer**:

| Layer | Technology | Format | Ownership |
|-------|-----------|--------|-----------|
| Bronze | ADF Copy -> ADLS | Parquet (compressed) | Data Engineering |
| Silver | Databricks Jobs + Spark SQL | Delta Lake (ACID) | Data Engineering |
| Gold | dbt on Databricks | Delta Lake + SQL Views | Analytics Engineering |

**Key Decision**: Delta Lake vs. Parquet?

**Delta Lake** (Recommended for Healthcare):
- ACID transactions (no partial writes)
- Schema enforcement (enforce data types)
- Time travel (query historical data, GDPR compliance)
- Update/Delete support (critical for GDPR right-to-be-forgotten)
- Automatic optimization (Z-order for query performance)

**Parquet** (Simpler, no delete):
- More tools support it
- Cheaper (no transaction overhead)
- No GDPR deletion support (huge problem for healthcare!)

**Decision**: Use **Delta Lake** for all layers. GDPR deletions are non-negotiable in healthcare.

---

### Decision 3: Real-Time Processing Layer (Vital Signs)

**Requirement**: Clinical alerts must appear <10 min after data entry.

**Example Alert**:
- Patient's O2 saturation drops below 90%
- Heart rate spikes > 120 bpm
- Temperature > 39 C
- System sends SMS to nurse immediately

**Options Considered**:
- **Option A**: Stream Analytics (Azure-Native, Easy) — Pro: Fully managed, HIPAA-compliant. Con: Limited complex stateful logic. Latency: 1-5 minutes.
- **Option B**: Azure Databricks Structured Streaming (Powerful, Complex) — Pro: Unlimited flexibility, can chain transformations. Con: Higher ops burden. Latency: <30 seconds.
- **Option C**: Azure Functions + Cosmos DB (Lightweight) — Pro: Lowest cost, easiest to debug. Con: No windowing/grouping capability. Latency: 10-30 seconds.

**Decision**: **Stream Analytics + Cosmos DB (Hybrid)**
1. Stream Analytics processes vital signs from Event Hubs
2. Uses SQL to aggregate: "IF AVG(heart_rate last 5 min) > 120, alert"
3. Reference table: Cosmos DB with patient thresholds (customizable by doctor)
4. Output: Service Bus -> Azure Logic App -> SMS/Email

**Why this over Databricks?**
- Clinical alerts are rule-based, not ML-based (Stream Analytics is better fit)
- Operational simplicity (less Spark expertise needed)
- Cost (Stream Analytics cheaper for this scale)
- **Future enhancement**: Add Databricks ML model for sepsis prediction (model scores patient risk, Stream Analytics triggers alerts)

---

### Decision 4: Data Quality Framework

**Problem**: You have 5M patients, duplicate records are common:
- Patient registered at Hospital A under "John Smith"
- Same patient registers at Hospital B as "Jon Smyth" (typo + abbreviation)
- System can't link records -> analytics show inflated patient count

**Quality Dimensions**:
1. **Completeness**: Is required data present? (null checks)
2. **Accuracy**: Is data correct? (does lab value make medical sense?)
3. **Consistency**: Is data uniform? (patient_id format consistent across systems?)
4. **Timeliness**: Is data up-to-date? (vital signs should be <5 min old)
5. **Uniqueness**: No duplicates? (one patient = one record)

**Implementation Strategy**:

**Layer 1: Validation at Ingestion (ADF)**
- Schema validation (expected columns, data types)
- Null checks on critical fields (patient_id, encounter_id)
- Range checks (temperature must be 35-42 C)
- Redirect failures -> quarantine blob + alert

**Layer 2: Deduplication (Silver Layer with Databricks)**
- Implement fuzzy matching for patient names (Levenshtein distance)
- Deterministic ID matching (national ID + date of birth)
- Mark duplicates with confidence score
- Keep audit trail of what was merged (GDPR requirement)

**Layer 3: dbt Data Quality Tests (Gold Layer)**
- **Uniqueness**: `SELECT COUNT(*) FROM gold.patients GROUP BY patient_id HAVING COUNT(*) > 1` -> should be 0
- **Referential Integrity**: Every encounter references valid patient_id
- **Business Logic**: Alert if patient age > 150 years (data error)

**Observability** (detect data quality issues in production):
- Datadog/Azure Monitor dashboard showing:
  - % nulls in critical fields (trend over time)
  - Duplicate patient records (daily count)
  - Data freshness (time since last update)
  - Failed transformation jobs
- Alert if % nulls exceeds threshold (e.g., 5%)

**Decision**: Implement **3-layer approach** (validation -> deduplication -> testing). Use **dbt** for Silver->Gold tests (leverages SQL expertise in healthcare teams).

---

### Decision 5: GDPR Compliance Architecture

**Requirement**: "Patient requests deletion. You have 30 days to delete all their data."

**The Problem**:
- You have immutable Delta Lake tables (designed for audit)
- Patient data is denormalized across multiple tables (patients, encounters, vitals, labs)
- Historical aggregates already computed (can't be "un-aggregated")
- ML models trained on this patient (model contains patient patterns)

**Architecture Solution**:

**Approach**: "Soft Delete + Recomputation"

1. GDPR Request Received
   - Log deletion request to immutable audit table
   - Add patient_id to "deleted_users" reference table

2. Data Deletion (Day 1-7)
   - Silver Layer: Filter out deleted users in next refresh job (recreate tables without deleted patient data)
   - Gold Layer: Recompute all aggregates excluding deleted users (deterministic: same input -> same output)
   - Audit Log: Record "Patient X deleted on [date], tables [list] updated"

3. Verification (Day 8-30)
   - Compliance team verifies deletion
   - Archive copy created (for 7-year legal hold)
   - Send customer confirmation email

4. ML Models (Special Case)
   - Add patient to exclusion list for future training
   - Current models: Can't retroactively delete, but note in model lineage
   - Document: "Model trained on data including deleted patient X"

**Technology Choices**:

**Patient Master Data**:
```
Table: dim_patients (Gold Layer)
- patient_id (primary key)
- ... all patient attributes ...
- is_deleted (flag: Y/N)
- deleted_date
- deleted_reason (GDPR, patient request, fraud, etc.)

On deletion request:
UPDATE dim_patients SET is_deleted = 'Y', deleted_date = TODAY()
WHERE patient_id = @target_patient

Next dbt run: Filter out WHERE is_deleted = 'Y'
Result: Clean tables without deleted patient
```

**Immutable Audit Log**:
```
Table: audit.deletion_requests (Append-only, Delta Lake)
- deletion_id (unique)
- patient_id
- request_date
- deletion_date (when actually deleted)
- requested_by (email)
- reason (GDPR / patient request / other)
- affected_tables (JSON: [patients, encounters, vitals, ...])
- verification_date (when compliance confirmed)

This table is NEVER updated/deleted, only appended.
Used for GDPR audit responses.
```

**Data Lineage Tracking** (Which datasets contain which patients?):
- Use Azure Purview or custom metadata DB
- Document: "Dataset XYZ contains patient IDs from source ABC"
- On deletion: Automatically identify all affected datasets
- Trigger: "Recompute datasets [list]"

**Decision**: Use **soft delete + recomputation approach** with:
- Delta Lake `is_deleted` flags (standard healthcare pattern)
- Immutable audit log (Purview for lineage)
- dbt macros to automate recomputation
- Logic Apps to orchestrate deletion workflow

---

### Decision 6: Power BI Semantic Layer & dbt

**Problem**: Power BI was timing out on Fridays (4-hour refresh, 99M+ rows).

**Root Cause**:
- Raw table with 100M+ vital signs records
- Power BI doing calculation in-memory (slow)
- No aggregation strategy
- No column indexing

**Solution**: Build a proper **semantic layer** with dbt + optimized Power BI data model.

**Layer Structure** (dbt models):
```
dbt models/
- staging/
  - stg_patients.sql          (raw -> cleaned patient table)
  - stg_encounters.sql
  - stg_vital_signs.sql

- marts/
  - core/
    - dim_patients.sql        (dimension: patient attributes)
    - dim_hospital.sql        (dimension: hospital info)
    - dim_date.sql            (dimension: dates)
    - fct_encounters.sql      (fact: patient encounters)

  - analytics/
    - patient_daily_stats.sql (pre-computed vitals by patient/day)
    - hospital_metrics.sql    (KPIs: avg length of stay, readmission rate)
    - patient_risk_scores.sql (pre-computed ML scores)
```

**Key dbt Features for Healthcare**:

1. **Staging Models** (stg_*): Clean, rename, add quality flags, hash for change detection
2. **Dimension Models** (dim_*): Patient attributes, SCD handling, calculated fields (age)
3. **Fact Models** (fct_*): Encounters with length_of_say, readmission flags, deduplication
4. **Aggregate Tables**: Patient daily stats (1M rows vs 500M raw vitals) for Power BI performance

**Power BI Data Model** (Connected to Gold Layer):
- Star Schema relationships: fct_encounters -> dim_patients, dim_hospital, dim_date
- DAX Measures: TotalEncounters, AvgLOS, Readmission Rate, Patient Risk Score

**Performance Optimization**:
- Aggregated tables: Patient daily stats (1M rows vs 500M raw vitals)
- Column indexing in Delta Lake: Z-ORDER by (patient_id, vital_sign_date)
- Partitioning: Gold tables partitioned by year/month
- Incremental refresh: dbt only processes new/changed data

**Decision**: Use **dbt + Delta Lake star schema** with:
- dbt models organized: staging -> core -> analytics
- Incremental materialization for large facts
- Pre-computed aggregates for Power BI
- Z-ORDER indexing for query performance
- dbt tests for data quality validation

---

### Decision 7: DevOps & CI/CD for Data Pipelines

**Current Situation**: Manual, ad-hoc processes
- ADF pipelines deployed by clicking UI (no version control)
- dbt runs triggered manually every morning
- No way to rollback if something breaks
- Multiple people editing same pipeline (conflicts)

**Required**: Professional, repeatable deployment process.

**Technology Stack**:
```
Azure DevOps (Git) <- Source of Truth
- Repo: /adf/ (ADF pipelines in JSON)
- Repo: /dbt/ (dbt models, tests, macros)
- Repo: /databricks/ (Spark jobs, notebooks)
- Repo: /docs/ (data dictionary, runbooks)

-> GitHub Actions / Azure Pipelines (CI/CD Orchestration)
- Test Layer: dbt test, SQL validation, ADF validation
- Lint Layer: YAML lint, SQL format (sqlfluff)
- Deploy Layer: Deploy ADF (ARM templates), Deploy dbt (dbt Cloud or Databricks)

-> Production Environment
- PROD ADF pipeline
- PROD dbt jobs (Databricks)
- PROD dashboards (Power BI)
```

**Git Workflow** (for data team):
1. Feature Branch -> Local Testing -> Pull Request -> CI checks -> Merge to Main -> Production Deployment -> Rollback if needed

**dbt Cloud Specific**:
- Jobs: daily-refresh (2 AM), weekly-full-refresh (Sunday 3 AM)
- Environments: dev (dev_healthcare), prod (prod_healthcare)
- Notifications: Slack webhook on failure/success

**Azure Pipelines YAML** (ADF deployment):
- ValidateOnly stage -> dbt Test stage -> DeployProd stage (Incremental deployment)

**Decision**: Use **Azure DevOps + dbt Cloud** with:
- Git for version control (ADF, dbt, Spark)
- Pull requests required for code review
- Automated testing (dbt test, SQL lint)
- CI/CD pipelines for deployment (GitHub Actions or Azure Pipelines)
- dbt Cloud for scheduling/monitoring dbt jobs
- Slack notifications for failures

---

## Evaluation Checklist (What Excellent Candidates Address)

### Technical Architecture
- [ ] Identified need for hybrid approach (batch + streaming)
- [ ] Chose Delta Lake for healthcare (GDPR-critical)
- [ ] Understood medallion architecture purpose (governance layers)
- [ ] Recognized Power BI timeout as design issue (needs aggregates, not raw data)
- [ ] Designed GDPR deletion workflow (soft delete + recomputation)

### Data Engineering Fundamentals
- [ ] Star schema design (fct_encounters + dim_patients)
- [ ] Slowly Changing Dimensions handling (patient attributes changing over time)
- [ ] Data quality framework (3-layer: validation -> deduplication -> testing)
- [ ] Partition strategy (by date, patient_id for query performance)
- [ ] Indexing/compression choices (Z-order, Parquet compression levels)

### Azure-Specific
- [ ] ADF vs. Event Hubs + Stream Analytics (knows tradeoffs)
- [ ] Databricks for complex transformations (Spark, Delta Lake)
- [ ] dbt on Databricks (not just dbt on Snowflake)
- [ ] Cosmos DB for real-time cache (vital signs reference table)
- [ ] Azure Purview for data lineage (GDPR requirement)

### Governance & Compliance
- [ ] GDPR deletion at architecture level (not afterthought)
- [ ] HIPAA considerations (audit logs, encryption, role-based access)
- [ ] Data lineage tracking (Purview or custom metadata)
- [ ] Immutable audit log (why important, how implemented)

### DevOps & Production Readiness
- [ ] Git version control for pipelines (ADF templates, dbt models)
- [ ] CI/CD pipeline (tests before deployment)
- [ ] Monitoring/alerting (data quality metrics)
- [ ] Rollback strategy (how to undo bad deployment)
- [ ] dbt tests & validation (not just "it runs")

### Communication
- [ ] Explains why each tool chosen (vs. just listing tools)
- [ ] Acknowledges tradeoffs (complexity vs. performance vs. cost)
- [ ] Realistic about team skills (doesn't assume Spark experts exist)
- [ ] Phased rollout (doesn't rewrite everything at once)
- [ ] Knows unknowns (what questions to ask stakeholders)

---

## Red Flags (What to Watch For)

### Technical Naivete
- "We'll use Kafka for everything" (Kafka is great, but overkill for batch patient data)
- "Just use one big SQL Server database" (doesn't scale to 500M vitals/month)
- "Power BI can query raw 100M-row table directly" (will timeout, as they're experiencing)
- "We'll add security later" (HIPAA/GDPR can't be bolted on)

### Azure-Specific Mistakes
- Confuses ADF with Databricks (different purposes)
- Doesn't mention Delta Lake for healthcare (Parquet doesn't support deletion)
- Ignores Azure Purview for compliance (manual lineage tracking is nightmare)
- Doesn't address Azure costs (Event Hubs/Databricks are expensive if not optimized)

### Compliance/Governance Gaps
- No mention of GDPR/HIPAA (critical in healthcare context)
- "We'll implement audit logging later" (compliance failures are serious)
- Data deletion strategy missing (how do you handle right-to-be-forgotten?)
- No data lineage (can't audit compliance)

### Production Reality
- Over-engineers for 2-person team (picks tools requiring 5 ops specialists)
- "Team will learn Spark in 2 weeks" (unrealistic)
- No monitoring/alerting plan (how do you know if something breaks?)
- Single points of failure (no redundancy)

---

## Success Metrics (How to Know if Solution is Working)

- Power BI dashboard query latency < 5 seconds (was 60+ seconds)
- Vital sign alerts appear < 10 minutes after data entry (was never before)
- Patient duplicate rate < 0.1% (was 5-10%)
- GDPR deletion requests completed within 7 days (was impossible)
- dbt test coverage > 80% of data models (prevents regressions)
- 99.95% pipeline availability (clinical requirements)
- Data freshness <1 hour for all Gold tables (BI requirement)
- Zero manual intervention needed (fully automated)

---

## Conclusion

This problem mirrors **real healthcare data challenges**:
- Streaming vital signs meet batch patient records
- Compliance (HIPAA/GDPR) is architectural requirement, not afterthought
- Data quality directly impacts patient safety
- Performance matters (doctors won't wait for dashboards)
- Team skill & operational maturity are critical

Excellent candidates will show they can **think systemically**, **justify choices**, and **understand production reality**. They'll demonstrate Azure expertise while explaining architectural decisions that work at scale.
