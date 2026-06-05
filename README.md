# MS Fabric Pipeline Execution Monitoring and Logging

### A Self-Installing Pipeline Monitoring Framework for Microsoft Fabric
**Built by Jagjeet Makhija · Microsoft Fabric + Power BI**

---

## What Is This?
The **Pipeline Operations Tracker** is a lightweight, self-installing monitoring framework for Microsoft Fabric. Upload one Python file to your notebook, run two lines of code, and you instantly get:

- A **Lakehouse** with three Delta tables
- A **Power BI Direct Lake semantic model** with relationships and 8 DAX measures
- A single method call to log any pipeline run — success or failure

No configuration files. No manual table creation. No separate Power BI setup. Everything is created automatically on first run.

---

## The Three Files
| File | Role | When to Use |
|---|---|---|
| `jagjeet_logging_utils.py` | **Core engine** — the only file required in production | Every notebook that needs tracking |
| `jagjeet_test_data_creation.py` | Sample data generator — populates the framework with realistic dummy data | Once, during setup/testing |
| `jagjeet_usage_examples.py` | 8 copy-paste usage patterns | Reference when building your own notebooks |

---

## What Gets Created
When you run `JagjeetPipelineTracker("MyProject")` for the first time, the following Fabric artifacts are created **automatically**:

```
YOUR FABRIC WORKSPACE
│
├── 📦 LH_MyProject_Jagjeet_PipelineOps        ← Lakehouse
│       ├── 📋 pipeline_activity_log             ← Fact table (empty, fills as you log)
│       ├── 📅 date_dimension                    ← 4 years of dates pre-filled
│       └── ⏰ time_dimension                    ← 1,440 minute slots pre-filled
│
└── 📊 SM_MyProject_Jagjeet_PipelineOps         ← Power BI Semantic Model
        ├── 🔗 pipeline_activity_log → date_dimension   (Many-to-One)
        ├── 🔗 pipeline_activity_log → time_dimension   (Many-to-One)
        └── 📐 8 DAX Measures
                ├── Pipeline Overview:      Total Pipeline Runs, Total Rows Processed
                │                           Active Tables, Active Notebooks
                ├── Performance:            Average Run Duration
                ├── Quality & Reliability:  Failed Runs, Success Rate
                └── Time Intelligence:      Runs Today
```

> **Smart behaviour:** If any of these already exist, they are preserved. Existing data is never overwritten unless you pass `force_recreate=True`.

> **⚠️ Date range note:** The `date_dimension` table covers 4 years from the date of first setup. If your project runs longer than that, Power BI relationships to dates beyond that range will break. Run `tracker = JagjeetPipelineTracker("MyProject", force_recreate=True)` to rebuild with a fresh 4-year window — **this will delete all existing log data**.

---

## Prerequisites

> **⚠️ You must have Contributor or Admin role** in your Microsoft Fabric workspace. A Viewer role does not have permission to create Lakehouses or Power BI semantic models — setup will fail.

Before running anything, make sure you have:
- [ ] A **Microsoft Fabric workspace** with Spark compute enabled
- [ ] **Contributor or Admin role** in that workspace
- [ ] A **Fabric notebook** open and attached to a Spark session
- [ ] The file `jagjeet_logging_utils.py` uploaded to your notebook's **Resources** folder

> **Important:** This framework must run **inside a Fabric notebook**. It will not work locally (VS Code, Jupyter on your machine, etc.) because it requires `notebookutils` and Fabric's native authentication — both of which are only available inside the Fabric environment.

> **Note:** `semantic-link-labs` (`sempy_labs`) is installed automatically if missing — you do not need to pip install anything manually.

---
---

# EXECUTION GUIDE

---

## PHASE 1 — First-Time Setup

### Step 1 · Upload the core file
Upload `jagjeet_logging_utils.py` to your Fabric notebook's **Resources** folder.
```
Notebook → Resources (left panel) → Upload → jagjeet_logging_utils.py
```
This is the **only file required for production**. The other two files are optional tools.

---

### Step 2 · Initialise the tracker
In a new notebook cell, run:
```python
from builtin.jagjeet_logging_utils import JagjeetPipelineTracker
tracker = JagjeetPipelineTracker("MyProject")
```

**What happens when this runs:**
| Step | What the Framework Does | Code Reference |
|---|---|---|
| 1 | Detects your Fabric workspace ID | `_setup()` → line 93 |
| 2 | Finds or creates the lakehouse `LH_MyProject_Jagjeet_PipelineOps` | `_setup()` → lines 98–128 |
| 3 | Creates `pipeline_activity_log` fact table | `_create_activity_log_table()` → line 187 |
| 4 | Creates `date_dimension` (4 years of dates) | `_create_date_dimension_table()` → line 206 |
| 5 | Creates `time_dimension` (1,440 minute slots) | `_create_time_dimension_table()` → line 246 |
| 6 | Polls until all 3 tables are readable | `_verify_all_tables_ready()` → line 152 |
| 7 | Authenticates using Fabric native token | `_authenticate_for_tom()` → line 176 |
| 8 | Builds the Direct Lake Power BI semantic model | `_build_power_bi_model()` → line 279 |
| 9 | Adds 2 relationships and 8 DAX measures | `_add_model_relationships()` → line 305 |

**Expected output:**
```
============================================================
JAGJEET PIPELINE OPERATIONS TRACKER v3.4.3
============================================================
  • Project:        MyProject
  • Lakehouse:      LH_MyProject_Jagjeet_PipelineOps
  • Semantic Model: SM_MyProject_Jagjeet_PipelineOps
Workspace: YourWorkspaceName
Tables:
    pipeline_activity_log: created
    date_dimension: created (4 years)
    time_dimension: created (1,440 slots)

    ✓ pipeline_activity_log: 0 rows
    ✓ date_dimension: 1,461 rows
    ✓ time_dimension: 1,440 rows

  Power BI model created: SM_MyProject_Jagjeet_PipelineOps
  8 DAX measures applied
  Model refreshed

============================================================
Ready. Use: tracker.track_pipeline_run(...)
============================================================
```

> **Second run onwards:** If the lakehouse and tables already exist, the framework detects them and preserves all data. Setup takes only a few seconds on subsequent runs.

> **Slow Spark startup:** On the very first run, Spark may take 1–3 minutes to initialise before the framework can create tables. If you see a timeout during table verification, simply re-run the cell — the framework will detect what already exists and continue.

---

### Step 3 · Verify setup
Run a health check to confirm everything is working:

```python
tracker.show_health_check()
```

**Expected output:**
```
============================================================
JAGJEET PIPELINE TRACKER — HEALTH CHECK
============================================================
  Project:   MyProject
  Lakehouse: LH_MyProject_Jagjeet_PipelineOps
  Model:     SM_MyProject_Jagjeet_PipelineOps
  Workspace: YourWorkspaceName

  Tables:
    pipeline_activity_log : 0 records
    date_dimension        : 1,461 dates (up to 2027-XX-XX)
    time_dimension        : 1,440 slots

  Power BI Model: EXISTS
============================================================
```

---
---

## PHASE 2 — Populate with Test Data (Optional)
> Skip this phase if you want to start logging real pipeline runs immediately. Use this phase to verify your Power BI reports before real data exists.

### Step 4 · Upload the test data file
Upload `jagjeet_test_data_creation.py` to your notebook's **Resources** folder.

---

### Step 5 · Run a quick smoke test (5 entries)
```python
from builtin.jagjeet_test_data_creation import quick_test
tracker = quick_test("MyProject")
```

**What it creates:**
```
[OK] INSERT   | customers | Δ+1,000 rows | 5.2s
[OK] UPDATE   | orders    | Δ+100 rows   | 8.7s
[OK] VALIDATE | products  | Δ0 rows      | 3.1s
[OK] MERGE    | sales     | Δ+500 rows   | 12.4s
[OK] DELETE   | inventory | Δ-50 rows    | 2.8s

Quick test complete
```

---

### Step 6 · Run full test environment (optional — 80+ entries)
```python
from builtin.jagjeet_test_data_creation import setup_complete_test_environment
tracker = setup_complete_test_environment("MyProject")
```

> **Note:** If you run `setup_complete_test_environment` more than once, it will append additional test entries each time rather than overwriting. To start fresh, use `JagjeetPipelineTracker("MyProject", force_recreate=True)` first — **this deletes all existing data**.

**What it creates:**
| Function | What It Generates | Line |
|---|---|---|
| `create_test_data(tracker, 30)` | 30 random runs — 90% success / 10% failure, spread over 30 days | 10 |
| `create_etl_pipeline_scenarios(tracker)` | Daily ETL (Extract → Transform → Load → Validate) + quality failure + slow aggregation + incremental merge | 76 |
| `create_failure_scenarios(tracker)` | API 429, schema mismatch, OOM, PK violation, DB timeout | 109 |
| `create_performance_benchmarks(tracker)` | SELECT / JOIN / AGGREGATE / SORT / MERGE across 1K → 10M row datasets | 126 |

---
---

## PHASE 3 — Log Real Pipeline Runs

### Step 7 · Log a successful run
```python
tracker.track_pipeline_run(
    notebook_name    = "DailyETL",           # which notebook ran
    table_name       = "sales_fact",         # which table was affected
    operation_type   = "INSERT",             # INSERT / UPDATE / DELETE / MERGE / VALIDATE etc.
    rows_before      = 1000000,              # row count before
    rows_after       = 1005000,              # row count after (rows_changed auto-calculated)
    duration_seconds = 67.2,                 # how long it took
    run_message      = "Daily load — 5,000 new rows added"
)
```

**Output:**
```
  [OK] INSERT | sales_fact | Δ+5,000 rows | 67.2s | 2025-09-15
```

---

### Step 8 · Log a failed run
```python
tracker.track_pipeline_run(
    notebook_name    = "DataValidation",
    table_name       = "orders",
    operation_type   = "VALIDATE",
    rows_before      = 100000,
    rows_after       = 98500,
    duration_seconds = 25.4,
    error_details    = "1,500 records failed — missing customer_id",   # triggers FAILED status
    run_message      = "Validation flagged data quality issues"
)
```

**Output:**
```
  [FAILED] VALIDATE | orders | Δ-1,500 rows | 25.4s | 2025-09-15
```
> When `error_details` is populated, the entry is marked **FAILED**. The Power BI measures **Failed Runs** and **Success Rate** pick this up automatically.

---

### Step 9 · Auto-time your functions with the decorator
```python
from builtin.jagjeet_logging_utils import JagjeetPipelineTracker, measure_execution_time

@measure_execution_time
def load_sales_data():
    # your actual ETL logic here
    df = spark.sql("SELECT * FROM source.sales")
    df.write.format("delta").mode("append").saveAsTable("sales_fact")
    return {"rows_loaded": df.count()}

# Call it — get back (result, duration_seconds, error)
result, duration, error = load_sales_data()

# Log — duration captured automatically even if function failed
tracker.track_pipeline_run(
    notebook_name    = "SalesETL",
    table_name       = "sales_fact",
    operation_type   = "LOAD",
    rows_after       = result["rows_loaded"] if not error else 0,
    duration_seconds = duration,                 # auto-captured
    error_details    = error                     # None if successful
)
```

---

### Step 10 · Backfill historical data
```python
from datetime import datetime, timedelta

for i in range(7):
    past_date = datetime.now() - timedelta(days=i)
    tracker.track_pipeline_run(
        notebook_name    = "DailyETL",
        table_name       = "sales_fact",
        operation_type   = "REFRESH",
        rows_before      = 100000,
        rows_after       = 100000 + (i * 1000),
        duration_seconds = 15.0 + (i * 2),
        run_message      = f"Daily refresh — {past_date.strftime('%Y-%m-%d')}",
        custom_timestamp = past_date          # overrides the log timestamp
    )
```

> Use `custom_timestamp` when deploying to an existing project that already has a history of pipeline runs you want to represent in Power BI.

---

### Step 11 · Track a multi-stage ETL pipeline
```python
from datetime import datetime, timedelta
pipeline_start = datetime.now()

# Stage 1 — Extract
tracker.track_pipeline_run(
    notebook_name="ETL_Pipeline", table_name="source_system",
    operation_type="EXTRACT", rows_before=0, rows_after=50000,
    duration_seconds=45.2, run_message="Extracted from source",
    custom_timestamp=pipeline_start
)

# Stage 2 — Transform (runs 30 min later)
tracker.track_pipeline_run(
    notebook_name="ETL_Pipeline", table_name="staging_area",
    operation_type="TRANSFORM", rows_before=50000, rows_after=49500,
    duration_seconds=120.7, run_message="Business rules applied",
    custom_timestamp=pipeline_start + timedelta(minutes=30)
)

# Stage 3 — Load (runs 45 min after transform)
tracker.track_pipeline_run(
    notebook_name="ETL_Pipeline", table_name="data_warehouse",
    operation_type="LOAD", rows_before=1000000, rows_after=1049500,
    duration_seconds=85.3, run_message="Loaded to warehouse",
    custom_timestamp=pipeline_start + timedelta(minutes=75)
)
```

---

### Step 12 · Track a batch with mixed outcomes
```python
batch_start = datetime.now()
batch_id    = f"BATCH_{batch_start.strftime('%Y%m%d_%H%M%S')}"

tables = [
    {"name": "customers",    "records": 50000,  "success": True,  "duration": 30.5},
    {"name": "orders",       "records": 200000, "success": True,  "duration": 45.2},
    {"name": "products",     "records": 10000,  "success": False, "duration": 5.1},
    {"name": "transactions", "records": 500000, "success": True,  "duration": 120.3}
]

for i, t in enumerate(tables):
    tracker.track_pipeline_run(
        notebook_name="BatchPipeline", table_name=t["name"],
        operation_type="BATCH_LOAD",
        rows_before=0, rows_after=t["records"] if t["success"] else 0,
        duration_seconds=t["duration"],
        run_message=f"[{batch_id}] {t['name']} — {'ok' if t['success'] else 'failed'}",
        error_details=None if t["success"] else f"Schema validation failed for {t['name']}",
        custom_timestamp=batch_start + timedelta(minutes=i * 15)
    )

# Batch summary entry
successful = sum(1 for t in tables if t["success"])
tracker.track_pipeline_run(
    notebook_name="BatchPipeline", table_name="batch_control_log",
    operation_type="BATCH_COMPLETE",
    duration_seconds=sum(t["duration"] for t in tables),
    run_message=f"[{batch_id}] {successful}/{len(tables)} tables succeeded",
    custom_timestamp=batch_start + timedelta(hours=2)
)
```

---
---

## PHASE 4 — Monitor and Maintain

### Step 13 · View recent runs
```python
tracker.show_recent_runs(10)         # show last 10 runs
```

---

### Step 14 · Get summary statistics
```python
tracker.show_summary_stats()
```

**Output:**
```
Pipeline Activity Summary:
  Total Runs:         1,247
  Unique Notebooks:   8
  Unique Tables:      15
  Unique Operations:  7
```

---

### Step 15 · Query specific logs
```python
# Filter by table
df = tracker.fetch_activity_logs(table_name="sales_fact", limit=50)
df.show()

# Filter by operation type
df = tracker.fetch_activity_logs(operation_type="VALIDATE")
df.show()

# All failures
df = tracker.fetch_activity_logs(limit=200)
failures = df.filter(df.error_details.isNotNull())
failures.show()
```

---

### Step 16 · Refresh the Power BI model
Run this after adding new tables, changing schemas, or updating DAX measure definitions:

```python
tracker.refresh_power_bi_model()
```

---

### Step 17 · Build the Power BI model manually (if setup was skipped)
If the model was not created during initial setup (because tables weren't ready in time), run:

```python
tracker.build_power_bi_model(max_wait_minutes=5)
```

---

### Step 18 · Clean up old records
Run this on a schedule to keep the activity log lean. The recommended approach is to call this from a **scheduled Fabric notebook** (set up via the notebook's scheduling feature in the Fabric workspace) or from inside a recurring Data Pipeline activity.

```python
tracker.purge_old_records(days_to_keep=90)    # delete anything older than 90 days
```

**To schedule this automatically:**
```
Fabric Workspace → Open the notebook → Schedule → Add schedule → Set daily/weekly frequency
```

---
---

## PHASE 5 — Power BI Reports

After setup, your semantic model **SM_MyProject_Jagjeet_PipelineOps** is ready in your Fabric workspace. To build reports on top of it:

1. Go to your **Fabric workspace**
2. Find **SM_MyProject_Jagjeet_PipelineOps** (type: Semantic Model)
3. Click **"Create report"** to open Power BI report builder
4. The 8 pre-built DAX measures will appear in the Fields pane — drag them onto your canvas

The following measures are pre-built and ready to use:

| Measure | Use In Report | DAX |
|---|---|---|
| `Total Pipeline Runs` | KPI card | `COUNTROWS(pipeline_activity_log)` |
| `Total Rows Processed` | KPI card | `SUM(rows_changed)` |
| `Active Tables` | KPI card | `DISTINCTCOUNT(table_name)` |
| `Active Notebooks` | KPI card | `DISTINCTCOUNT(notebook_name)` |
| `Average Run Duration` | Bar chart by notebook | `AVERAGE(duration_seconds)` |
| `Failed Runs` | Alert card | `CALCULATE(...NOT ISBLANK error_details)` |
| `Success Rate` | Gauge | `DIVIDE(successful rows, total rows)` |
| `Runs Today` | Live KPI | `CALCULATE(...run_date = TODAY)` |

**Suggested report pages:**
```
Page 1 — Overview
  ├── Total Pipeline Runs  (KPI)
  ├── Success Rate          (Gauge)
  ├── Failed Runs           (KPI with alert)
  └── Runs Today            (KPI)

Page 2 — Performance
  ├── Average Run Duration by Notebook  (Bar chart)
  ├── Total Rows Processed over Time    (Line chart)
  └── Duration trend by operation_type  (Column chart)

Page 3 — Failures
  ├── Table: all runs where error_details is not blank
  └── Failure count by notebook         (Bar chart)

Page 4 — Activity Log
  └── Full table: all columns from pipeline_activity_log
      filtered by date_dimension and time_dimension
```

---
---

## Complete Method Reference
All methods below are in `jagjeet_logging_utils.py`.

### `JagjeetPipelineTracker(project_name, force_recreate, workspace_name)`
**Line 58–77** — Class constructor. Triggers full setup sequence.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `project_name` | str | required | Names the lakehouse and semantic model |
| `force_recreate` | bool | `False` | If `True`, drops and rebuilds everything — **all data is permanently deleted** |
| `workspace_name` | str | `None` | Auto-detected if not provided |

> **⚠️ WARNING — `force_recreate=True`:** This parameter **permanently deletes** your Lakehouse, all Delta tables, and the Power BI semantic model before rebuilding from scratch. All logged pipeline history will be lost. Only use this intentionally — never pass it by accident.

---

### `track_pipeline_run(...)`
**Line 369** — Core logging method. Always appends, never overwrites.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `notebook_name` | str | required | Which notebook ran |
| `table_name` | str | required | Which table was affected |
| `operation_type` | str | required | INSERT / UPDATE / DELETE / MERGE / VALIDATE / EXTRACT / TRANSFORM / LOAD / etc. |
| `rows_before` | int | `0` | Row count before operation |
| `rows_after` | int | `0` | Row count after — `rows_changed` auto-calculated |
| `duration_seconds` | float | `0.0` | Execution time. Use `@measure_execution_time` to auto-capture |
| `run_message` | str | `None` | Free-text description |
| `error_details` | str | `None` | Populate if failed — triggers FAILED status in Power BI |
| `run_by` | str | `None` | Auto-detected from `getpass` if not provided |
| `custom_timestamp` | datetime | `None` | Override run timestamp for historical backfill |

---

### Monitoring Methods
| Method | Line | Description |
|---|---|---|
| `show_recent_runs(limit=10)` | 401 | Print last N runs |
| `show_summary_stats()` | 405 | Print totals: runs, notebooks, tables, operations |
| `fetch_activity_logs(table_name, operation_type, limit)` | 394 | Return filtered Spark DataFrame |
| `show_health_check()` | 470 | Full status: table counts, model existence, available methods |
| `purge_old_records(days_to_keep=90)` | 423 | Delta delete + vacuum for records older than N days |
| `refresh_power_bi_model()` | 441 | Re-apply relationships + measures + trigger refresh |
| `build_power_bi_model(max_wait_minutes=5)` | 456 | Wait for tables, then build model |

---

### Utility Functions
| Function | Line | Description |
|---|---|---|
| `measure_execution_time(func)` | 500 | Decorator — returns `(result, duration_seconds, error)` |
| `resolve_current_user()` | 494 | Auto-detect current Fabric/system user |

---
---

## Execution Order Summary
```
FIRST TIME
  1.  Upload jagjeet_logging_utils.py → Resources folder
  2.  Run: tracker = JagjeetPipelineTracker("MyProject")
  3.  Run: tracker.show_health_check()

POPULATE TEST DATA (optional)
  4.  Upload jagjeet_test_data_creation.py → Resources folder
  5.  Run: quick_test("MyProject")                         ← smoke test (5 entries)
  6.  Run: setup_complete_test_environment("MyProject")    ← full test data (80+ entries)

PRODUCTION LOGGING (in every pipeline notebook)
  7.  from builtin.jagjeet_logging_utils import JagjeetPipelineTracker
  8.  tracker = JagjeetPipelineTracker("MyProject")
  9.  tracker.track_pipeline_run(...)  ← after every operation

MONITOR
  10. tracker.show_recent_runs(10)
  11. tracker.show_summary_stats()
  12. tracker.show_health_check()

MAINTAIN
  13. tracker.refresh_power_bi_model()          ← run after schema changes
  14. tracker.purge_old_records(90)             ← run on a schedule (set up scheduled notebook)
```

---
---

## What This Framework Does NOT Do
| What | Why |
|---|---|
| Auto-capture ADF pipeline runs | Requires explicit `track_pipeline_run()` call from inside a Fabric notebook activity |
| Monitor Spark job metrics | Not connected to Spark UI or execution plans |
| Create Power BI reports or dashboards | Only the semantic model is created — you build reports on top |
| Run automatically on a schedule | Logging only happens when you call the method |
| Connect to external monitoring tools | Self-contained, no third-party integrations |
| Run outside Microsoft Fabric | Requires `notebookutils` and Fabric native auth — local environments not supported |

---

## Troubleshooting
| Problem | Likely Cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'builtin'` | File not in Resources folder | Upload `jagjeet_logging_utils.py` to notebook Resources |
| `Power BI model not created` | Tables weren't ready in time during setup | Run `tracker.build_power_bi_model()` |
| `TOM auth failed` | `notebookutils` not available | Must run inside a Fabric notebook, not locally |
| `Model not found. Run build_power_bi_model() first` | Model was skipped or deleted | Run `tracker.build_power_bi_model()` |
| `Refresh failed` | Fabric dataset refresh quota | Retry after a few minutes or refresh manually in Power BI |
| Lakehouse not created / permission error | Insufficient workspace role | Ensure you have Contributor or Admin role in the workspace |
| Setup times out on first run | Spark session cold start | Re-run the initialisation cell — framework will resume from where it stopped |
| Duplicate test data after re-running setup | `setup_complete_test_environment` appends each time | Use `force_recreate=True` to reset, or filter duplicates in Power BI |

---

## Built By
**Jagjeet Makhija**
`Microsoft Fabric` · `PySpark` · `Delta Lake` · `Power BI Direct Lake` · `TOM (Tabular Object Model)`
