# ETL Pipeline Demo Guide

## Overview

The pipeline processes CSV files automatically through: **Upload → S3 Raw → SQS → ETL Lambda → S3 Processed (Parquet)**

Four test cases cover all real-world scenarios.

| Link | Description |
|------|-------------|
| [ETL Upload Portal](https://d38xe3s2e16qjl.cloudfront.net/) | Upload interface and pipeline monitor |
| [Dashboard](https://etl-dashboard-app-sdhtogaprtskffwsamfn77.streamlit.app/) | Query processed data via Athena |
| [Raw CSV Repo](https://github.com/chianhluvC/CSV-RAW) | Sample CSV files for all 4 cases |

---

## Setup

1. Open the [ETL Upload Portal](https://d14vqntmuruhab.cloudfront.net/)
2. Download the CSV repo: [github.com/chianhluvC/CSV-RAW](https://github.com/chianhluvC/CSV-RAW)
   - Click **Code → Download ZIP** → extract
   - Four folders inside: `case_1/`, `case_2/`, `case_3/`, `case_4/`
3. **Reset the pipeline once** before starting:
   - Left sidebar → **Advanced** → **Full Pipeline Reset**
   - Type `delete` → click **Delete everything**

> After this, run Case 1 → 2 → 3 → 4 in order. No reset needed between cases.

---

## Case 1 — Normal Pipeline (SUCCESS & FAILED)

> **Schema version required: v1** — Before uploading, go to the **Schema** tab → Version History → confirm **v1 is ACTIVE**. If not, click **Activate** on v1 first.

**Goal:** Verify the pipeline handles both valid files and files with data errors.

**Files:** all 14 files in `case_1/`

```
online_retail-1.csv, online_retail-3.csv through online_retail-8.csv,
online_retail-15.csv through online_retail-21.csv
```

### Steps

1. Go to the **Home** tab
2. Click **CSV Files** → select all 14 files from `case_1/`
3. Click **Upload**
4. Watch the pipeline animation: `Upload → S3 Raw → SQS → ETL λ → Done`
5. Wait for completion (~20–30 seconds)

### Expected Results

| Status | Meaning |
|--------|---------|
| **SUCCESS** (green) | Valid file — transformed and saved as Parquet |
| **FAILED** (red) | Data error — invalid columns, null values, negative price, etc. |

- **Recent Jobs** table shows the status of each file
- Go to **File Inspector → All Files** to see the detailed error message for any FAILED file

### Verify Processed Data

Open the [Dashboard](https://etl-dashboard-app-sdhtogaprtskffwsamfn77.streamlit.app/) — data from SUCCESS files is queryable in Athena, partitioned by `year_month`.

---

## Case 2 — Dead Letter Queue (System Error)

> **Schema version required: v1** — Before uploading, go to the **Schema** tab → Version History → confirm **v1 is ACTIVE**. If not, click **Activate** on v1 first.

**Goal:** Demonstrate how the pipeline handles Lambda crashes — the failed message lands in the DLQ instead of being lost.

**Files:** all 7 files in `case_2/`

```
online_retail-9.csv through online_retail-14.csv,
online_retail-2_syserr_.csv     ← special file that triggers a crash
```

> `online_retail-2_syserr_.csv` contains `_syserr_` in its filename. The ETL Lambda detects this and intentionally raises an exception to simulate a real system failure.

### Steps

1. Go to the **Home** tab → select all 7 files from `case_2/`
2. Click **Upload**
3. Wait for the pipeline to finish

### Expected Results

| Status | File | Meaning |
|--------|------|---------|
| **SUCCESS** / **FAILED** | Regular files | Processed normally |
| **TIMEOUT** (amber) | `online_retail-2_syserr_.csv` | Lambda crashed after marking PROCESSING — job is stuck |

### Check the DLQ

1. Go to the **File Inspector** tab
2. Scroll down to **Dead Letter Queue**
3. Click **Check DLQ**
4. The failed message appears with the file key, first received time, and message ID

> The `_syserr_` file causes Lambda to crash after `mark_processing()` but before returning a success response. SQS receives no acknowledgement → retries once → moves the message to the DLQ.

### FAILED vs TIMEOUT

| | FAILED (red) | TIMEOUT (amber) |
|--|--------------|-----------------|
| **Cause** | Data validation error | Lambda crash / throttle |
| **Error file in S3** | Yes | No |
| **Appears in DLQ** | No | Yes |

---

## Case 3 — Schema Evolution

> **Schema version required: v2** — This case upgrades from v1 to v2. Before starting, confirm **v1 is ACTIVE** (Schema tab → Version History). You will upload `schema_v2.json` as part of the steps below.

**Goal:** Adapt to a new CSV format from a data partner without redeploying the Lambda.

**Files:** `case_3/` folder

```
schema_v2.json         ← new mapping config
online_retail-22.csv   ← CSV using the new column names
```

> **Context:** The data partner renames `UnitPrice` to `UnitCost` and `Country` to `Nation` in their CSV export. The pipeline must handle the new format with a config change only — no code deployment.

### Step 1 — View the active schema (v1)

1. Go to the **Schema** tab
2. Click **Reload** → see the current mapping: `UnitPrice → unit_price`, `Country → country`

### Step 2 — Upload the new schema (v2)

1. Under **Upload New Version**, click **Load from file**
2. Select `schema_v2.json` from `case_3/`
3. The validation feedback should show **"Valid — ready to upload as version 2"**
4. Click **Upload Schema**
5. A success toast confirms: *"Schema updated to version 2"*

### Step 3 — Confirm the schema changed

1. Click **Reload** in the Active Schema section
2. The mapping now shows: `UnitCost → unit_price`, `Nation → country`
3. Click **Load** in Version History → both v1 and v2 are listed, v2 is **ACTIVE**

### Step 4 — Upload the CSV with the new format

1. Go to the **Home** tab
2. Select `online_retail-22.csv` from `case_3/`
3. Click **Upload** → wait for the pipeline to finish
4. Result: **SUCCESS** — Lambda correctly reads `UnitCost` and `Nation`, the output Parquet still has `unit_price` and `country` matching the Glue catalog

### Step 5 — Roll back to v1 (optional)

1. **Schema** tab → **Version History** → click **Activate** on v1
2. The pipeline immediately switches back to the original schema for all subsequent uploads

---

## Case 4 — Schema Evolution: Add New Column

> **Schema version required: v4** — This case upgrades from v2 to v4. Before starting, confirm **v2 is ACTIVE** (Schema tab → Version History). You will upload `schema_v4.json` as part of the steps below.

**Goal:** Extend the pipeline to handle a CSV that adds a brand-new column not present in any previous schema version — no code deployment required.

**Files:** `case_4/` folder

```
schema_v4.json          ← new mapping config that adds the extra column
online_retail-23.csv    ← CSV containing the new column
```

> **Context:** The data partner adds a new field `Region` to their export. The pipeline must map and preserve this column in the output Parquet by uploading an updated schema config only.

### Step 1 — Confirm the current schema (v2 active from Case 3)

1. Go to the **Schema** tab → click **Reload**
2. Verify the active mapping does **not** include `Region`

### Step 2 — Upload the new schema (v4)

1. Under **Upload New Version**, click **Load from file**
2. Select `schema_v4.json` from `case_4/`
3. Validation feedback should show **"Valid — ready to upload as version 4"**
4. Click **Upload Schema**
5. Success toast confirms: *"Schema updated to version 4"*

### Step 3 — Confirm the schema changed

1. Click **Reload** in the Active Schema section
2. The mapping now includes the new column entry for `Region`
3. Click **Load** in Version History → v4 is listed as **ACTIVE**

### Step 4 — Upload the CSV with the new column

1. Go to the **Home** tab
2. Select `online_retail-23.csv` from `case_4/`
3. Click **Upload** → wait for the pipeline to finish
4. Result: **SUCCESS** — Lambda reads the new `Region` column and the output Parquet contains it alongside all existing columns

### Step 5 — Roll back (optional)

1. **Schema** tab → **Version History** → **Activate** any earlier version to revert

---

## Schema Config Versions

The `schema_config/` directory in this repo tracks every schema version used across all cases.

```
schema_config/
├── v1/   ← Original mapping  (used by Case 1 & Case 2)
├── v2/   ← Renamed columns   (used by Case 3 — UnitCost, Nation)
├── v3/   ← Optional variant  (needs adjustment to match raw column names before use)
└── v4/   ← New column added  (used by Case 4 — adds Region)
```

| Version | Used in | Change |
|---------|---------|--------|
| **v1** | Case 1, Case 2 | Baseline mapping — `UnitPrice`, `Country` |
| **v2** | Case 3 | Renamed columns — `UnitCost → unit_price`, `Nation → country` |
| **v3** | *(optional)* | Alternative variant — column names must be updated to match actual raw CSV headers before uploading |
| **v4** | Case 4 | New column added — `Region` mapped to output field |

---

## Summary

| | Case 1 | Case 2 | Case 3 | Case 4 |
|--|--------|--------|--------|--------|
| **Folder** | `case_1/` | `case_2/` | `case_3/` | `case_4/` |
| **Files** | 14 CSV | 7 CSV | 1 CSV + 1 JSON | 1 CSV + 1 JSON |
| **Scenario** | Normal pipeline | Lambda crash → DLQ | Renamed columns | New column added |
| **Statuses** | SUCCESS + FAILED | TIMEOUT + DLQ | SUCCESS | SUCCESS |
| **Highlight** | Data validation | Dead Letter Queue | Schema evolution | Schema evolution |
| **Schema version** | v1 | v1 | v2 | v4 |
