# 🚨 04 · Disaster Recovery & Operational Standards

[![Owner](https://img.shields.io/badge/owner-JoanMaquinano%20%2F%20Team-blue?style=flat-square)](https://github.com/JoanMaquinano)
[![As Of](https://img.shields.io/badge/as%20of-2026--09--25-brightgreen?style=flat-square)](04-recovery-strategy.md)
[![Strategy](https://img.shields.io/badge/recovery-Idempotent%20%2F%20Deterministic-orange?style=flat-square)](04-recovery-strategy.md)

This document defines the operational standards for **task failure isolation, timezone management, deterministic key generation, idempotent reruns, `MERGE` safety, and automated data-quality validation** across the NYC Mobility Data Pipeline.

---

## 📌 Table of Contents

- [1. Task Failure Isolation & Gate Architecture](#1-task-failure-isolation--gate-architecture)
- [2. Timezone Governance & Deterministic Key Generation](#2-timezone-governance--deterministic-key-generation)
- [3. Idempotency & Safe Recovery (`MERGE` Rules)](#3-idempotency--safe-recovery-merge-rules)
- [4. Automated Data Quality (Great Expectations & SQL QC)](#4-automated-data-quality-great-expectations--sql-qc)
- [5. Operational Runbooks](#5-operational-runbooks)
- [6. Operational Principles Summary](#6-operational-principles-summary)

---

## 1. Task Failure Isolation & Gate Architecture

When a check evaluates to `FAIL`, the pipeline distinguishes between required core assets and secondary tables to isolate failures without unnecessarily halting unaffected processing.

```text
Check Result (Status Ladder)
         ↓
FAIL on a blocking check pair?
         │
    ┌────┴────┐
    │         │
   NO        YES
    │         │
    ▼         ▼
 Proceed   Table Verdict = STOP
                  │
                  ├──► Total FAILs on table >= 5
                  │    (v_max_total_failures)
                  │           │
                  │           ▼
                  │       STOP Table
                  │
                  └──► Is table in v_required_tables?
                              │
                    ┌─────────┴─────────┐
                    │                   │
                   YES                  NO
                    │                   │
                    ▼                   ▼
             Write STOP Gate       Hold Back Table
             Record                │
                    │              └──► Unaffected Pipelines Proceed
                    ▼
             raise_error()
                    │
                    ▼
             Halt Downstream Job
```

### 1.1 Isolation Mechanisms

- **Selective Table Holdback** — A `FAIL` on a non-required table holds back only the affected table, allowing unaffected pipelines to proceed.
- **Hard Stop Gating** — If a check fails on a table listed in `v_required_tables`, or if total failing checks on a table exceed `v_max_total_failures` (`5`), the layer writes a `STOP` gate record to `nyc_quality.dq_results` with `check_category = 'gate'` and triggers `raise_error()`. This halts downstream execution before contaminated data can reach the Gold layer.

### 1.2 Required Tables by Layer

| **Layer** | **Required Tables (`v_required_tables`)** | **Operational Failure Action** |
|---|---|---|
| **Bronze** | `green_taxi` | Triggers `raise_error()` and halts Silver ingestion tasks. |
| **Silver** | `green_taxi_clean` | Triggers `raise_error()` and halts Gold transformation tasks. |
| **Gold** | `fact_taxi_trip`, `dim_taxi_zone` | Triggers `raise_error()` and halts downstream warehouse exports. |

---

## 2. Timezone Governance & Deterministic Key Generation

To guarantee reproducibility across execution environments and clusters, timezone dependencies are strictly controlled.

### 2.1 Session Timezone Pinning

All notebooks must explicitly execute UTC session pinning as their first SQL command to ensure consistent timestamp handling across environments:

```sql
SET TIME ZONE 'UTC';
```

This ensures timestamp evaluations, including `unix_timestamp` and date-based keys, yield consistent output regardless of cluster location.

### 2.2 Deterministic Key Logic

Primary and surrogate keys must be generated using timezone-independent logic. Avoid session-dependent transformations such as:

```sql
-- ❌ AVOID: Produces inconsistent keys across timezones
CAST(ts AS STRING)
DATE(ts)
date_format(ts, 'yyyy-MM-dd HH:mm:ss')
```

Different session timezones can produce different string or date values from the same underlying timestamp, which can result in inconsistent keys or duplicate records.

> ⚠️ **Known Technical Gap & Planned Fix:** `trip_key` currently uses `CAST(timestamp AS STRING)`, which is reproducible because every notebook pins UTC. The planned migration is to use timezone-independent epoch-second derivation.

```sql
-- ✅ RECOMMENDED: Timezone-independent epoch seconds derivation
COALESCE(
    CAST(unix_timestamp(lpep_pickup_datetime) AS STRING),
    '<NULL>'
)
```

Epoch seconds provide a timezone-independent representation, while `COALESCE` prevents NULL values from creating ambiguous key components.

---

## 3. Idempotency & Safe Recovery (`MERGE` Rules)

Pipeline tasks are designed to be rerun safely without corrupting historical data or introducing duplicate records.

### 3.1 Idempotency Foundation

- **Key-Based `MERGE` Ingestion** — Reloading an existing batch updates matching rows instead of appending duplicates.
- **Atomic Batch Scoping** — Processing tasks are scoped using `batch_month` and `source_file`, allowing individual monthly batches to be rerun independently and idempotently.
- **Quality Metric Clear-Down** — Rerunning a batch replaces previous quality results in `nyc_quality.dq_results` for the `(layer, table, batch_month)` tuple instead of appending secondary verdicts.

### 3.2 Schema or Key Logic Changes (`TRUNCATE` Rule)

When changing key-generation logic or target schema definitions, running a `MERGE` without resetting the target table can create duplicate records, such as increasing row counts from **44,208 to 88,416**.

```text
Target Table Recovery Sequence

   Schema / Key Change Identified
                 │
                 ▼
       TRUNCATE Target Table
                 │
                 ▼
       Apply Updated Logic
                 │
                 ▼
            Run MERGE
                 │
                 ▼
       Validate Results
      (Reconciliation Checks)
```

> **Recovery Rule:** Truncate the affected target table before rerunning a `MERGE` whenever the key-generation strategy or session parameters have changed.

### 3.3 Recovery Strategy Matrix

| **Scenario** | **Recovery Rule** | **Action Sequence** |
|---|---|---|
| **Key Logic / Schema Update** | **Truncate and Rebuild** | 1. `TRUNCATE TABLE <target_table>;`<br>2. Re-run transformation and quality tasks. |
| **Corrupted Ingestion Run** | **Point-in-Time Restore** | 1. `DESCRIBE HISTORY <target_table>;`<br>2. `RESTORE TABLE <target_table> TO VERSION AS OF <version>;`<br>3. Re-execute batch load. |

---

## 4. Automated Data Quality (Great Expectations & SQL QC)

Data-quality validation is automated using **Great Expectations (GX)** and supporting SQL validation scripts.

```text
Great Expectations Validation Workflow

   07_gx_checks.py
          │
          ▼
   GX Expectations
          │
          ▼
   Layer Validation
          │
          ▼
    PASS / WARN / FAIL
          │
          ▼
     Quality Gate
          │
          ▼
  Continue / Block Pipeline
```

### 4.1 Quality Status & Pipeline Actions

| **Status** | **Description** | **Pipeline Action** |
|---|---|---|
| **`PASS`** | All required expectations and checks are satisfied. | Continue execution downstream. |
| **`WARN`** | Issues are within configured tolerance limits. | Continue execution with warning logs. |
| **`FAIL`** | Quality thresholds are breached. | Block the pipeline when configured as a blocking check. |

### 4.2 Great Expectations Parallel Configuration

The GX job (`NYC Mobility - GX`) operates in parallel with SQL QC checks and writes results directly to `nyc_quality`. Enforcement is controlled through layer parameters in `resources/jobs/nyc_mobility_gx_job.yml`.

| **Layer Parameter** | **Default Value** | **Operational Behavior** |
|---|---:|---|
| `gx_bronze_enforce` | `false` | Report-only: results are recorded and the job continues. |
| `gx_silver_enforce` | `false` | Report-only: results are recorded and the job continues. |
| `gx_gold_enforce` | `true` | Fail-fast: raises an exception on breach and stops the job. |

---

## 5. Operational Runbooks

### 🛠️ Runbook A: Pipeline Task Failure

1. **Identify Failure** — Open the failed run in the Databricks Jobs UI and review the error logs.

2. **Inspect Quality Logs** — Query `nyc_quality.dq_results` for the failing layer and batch month:

   ```sql
   SELECT *
   FROM nyc_quality.dq_results
   WHERE layer = '<failed_layer>'
     AND batch_month = '<batch_month>'
     AND status = 'FAIL';
   ```

3. **Remediate Cause** — Correct source-data issues, fix code transformations, or adjust rule thresholds after review.

4. **Re-execute Pipeline** — Use **Repair run** in the Databricks Jobs UI or rerun the workflow using the target `year_month` parameter.

5. **Verify Resolution** — Confirm `dq_run_log.overall_status` is `PASS` or `WARN` and that reconciliation checks such as `rows_reconcile_with_silver`, `revenue_preserved`, and `one_row_per_trip_key` pass.

### 🛠️ Runbook B: CI/CD Deployment Failure

1. **Identify Failure** — Open the failed workflow run under the GitHub Actions tab.

2. **Diagnose Error**:
   - **`401 Unauthorized`** — Verify `DATABRICKS_HOST` and `DATABRICKS_TOKEN` environment secrets in the `staging` environment.
   - **`Bundle Validation Failed`** — Run `databricks bundle validate` locally to identify syntax issues in `resources/jobs/*.yml`.

3. **Re-trigger Deployment** — Push the fix to the branch or execute `workflow_dispatch` manually in GitHub Actions.

---

## 6. Operational Principles Summary

| **Principle** | **Core Operational Standard** |
|---|---|
| **Consistency** | Use UTC explicitly across all notebook sessions with `SET TIME ZONE 'UTC'`. |
| **Determinism** | Generate keys independently of session-dependent timezone expressions. |
| **Safe Recovery** | Reset or truncate affected target tables when key-generation logic changes. |
| **Controlled MERGE** | Validate target-table state and row counts before and after `MERGE` operations. |
| **Automated Quality** | Use Great Expectations and SQL check suites to standardize data-quality validation. |
| **Quality Gates** | Prevent critical data-quality failures from progressing through to the Gold layer. |

> **Operational Goal:** Maintain a reliable, deterministic, recoverable, and quality-controlled NYC Mobility data pipeline through controlled failure isolation, safe reruns, automated validation, and enforced quality gates.
