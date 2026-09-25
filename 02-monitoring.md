# 🔍 02 · Data Quality & Runtime Monitoring

[![Owner](https://img.shields.io/badge/owner-jg0901-blue?style=flat-square)](https://github.com/jg0901)
[![As Of](https://img.shields.io/badge/as%20of-2026--09--25-brightgreen?style=flat-square)](02-monitoring.md)
[![Scope](https://img.shields.io/badge/DQ%20Scope-291%20Checks%20%2F%20102%20Blocking-orange?style=flat-square)](02-monitoring.md)

This document details the observability pillars, data-quality matrix, status resolution ladder, and threshold configurations governing the NYC Mobility Data Pipeline.

---

## 1. Observability & Monitoring Pillars

Runtime monitoring tracks four core operational pillars across pipeline execution, failure handling, data freshness, and quality metrics:

| Pillar | What is Monitored | Implementation / Mechanism | Frequency |
|---|---|---|---|
| **Execution** | Task status, duration, and overall pipeline progress | Databricks job history (`NYC_Mobility`, `NYC Mobility - GX`), GitHub Actions logs, `nyc_quality.dq_run_log`, and `vw_latest_dq_run`. | Every run |
| **Failures** | Task exceptions, rule breaches, and blocking quality gate stops | Synthetic gate records in `nyc_quality.dq_results` (`check_category = 'gate'`) triggering `raise_error` on `STOP` / `FAIL`. | Every run |
| **Freshness** | Batch arrival times, land timestamps, and max boundary updates | `tests/09_freshness_check.sql` (`bronze_taxi_fresh`, `bronze_weather_fresh`, `silver_taxi_fresh`, `silver_weather_fresh`, `gold_fact_fresh`) and Gold `v_as_of` labels. | On demand / Daily schedule |
| **Data Quality** | 291 rules across 5 layers + Great Expectations health trends | NYC Mobility Data Quality Dashboard over `nyc_quality.dq_results` and `vw_dq_by_month`. | Every run |

### 1.1 Alerting Configuration

Automated failure notifications are configured directly within the job workflow definition (`resources/jobs/nyc_mobility_job.yml`):

```yaml
email_notifications:
  on_failure:
    - <team-alias@example.com>## Data Quality & Runtime Monitoring

### Monitoring Pillars

| **Pillar** | **What is Monitored** | **Implementation / Mechanism** |
|---|---|---|
| **Execution** | Task status, duration, and pipeline progress | `dq_run_log` and `vw_latest_dq_run` tables. |
| **Failures** | Task exceptions, rule breaches, and blocking gates | `dq_results` synthetic gate records with `raise_error` on `STOP`. |
| **Freshness** | Batch arrival times and maximum timestamp boundaries | Preload month auto-detection and Gold `v_as_of` labels. |
| **Data Quality** | 291 rules across 5 layers + Great Expectations | 102 blocking SQL checks + Great Expectations suite. |

Runtime data quality is governed by **291 checks across five layers**, including **102 blocking checks** that can halt pipeline execution when critical quality thresholds are breached.

### Five-Layer Data Quality Matrix

| Layer | Total Checks | Blocking Checks | Governance Focus | Gate Indicator |
|---|---:|---:|---|---|
| **Preload** | 101 | 17 | Evaluates source file layout, cast safety, and domain rules before landing. | `source_cleared_for_bronze` |
| **Bronze** | 42 | 28 | Confirms source-to-target row fidelity and `no_nulls_added_*` checks. | `batch_cleared_for_silver` |
| **Silver** | 87 | 34 | Validates classification promises (`dq_status`), quarantines, and formulas. | `batch_cleared_for_gold` |
| **Gold** | 49 | 21 | Verifies star schema construction, key derivations, and calendar integrity. | `warehouse_cleared` |
| **At-Rest** | 12 | 2 | Assesses standing referential integrity across fact/dimension foreign keys. | `referential_integrity_holds` |
| **Total** | **291** | **102** | | |

### Status Resolution Ladder

Every data-quality check is evaluated using a defined five-branch condition order:

```sql
CASE
    WHEN total_rows = 0
         AND check_name <> 'table_not_empty'
        THEN 'SKIP'

    WHEN failed_rows = 0
        THEN 'PASS'

    WHEN failed_pct <= warn_pct
        THEN 'PASS'

    WHEN failed_rows <= min_failed_rows
        THEN 'WARN'

    WHEN failed_pct <= threshold_pct
        THEN 'WARN'

    ELSE 'FAIL'
END

Status Definitions
SKIP — Applied when the batch is empty and the check is not table_not_empty.
PASS — Assigned when no failures exist or the failure rate is within the defined warning tolerance.
WARN — Assigned when failures remain within the configured row-count or percentage tolerance.
FAIL — Triggered when failure limits are breached. Blocking checks can prevent pipeline execution.

Threshold Configurations
Configuration	Threshold	Application
STRICT	0%	Scalar checks, primary keys, and non-null key constraints such as location_id_unique, location_id_not_null, and date_not_null.
TOL	10%	Source-data checks where minor data imperfections are considered tolerable.
CAST_WARN	5%	Warning threshold for string-to-datatype conversions.
CAST_FAIL	10%	Failure threshold for string-to-datatype conversions, including the Bronze no_nulls_added_* checks.
MIN_ROWS	5 rows	Minimum row threshold used to prevent isolated anomalies from unnecessarily stopping large batches.

📌 Key Rule Exemption: The MIN_ROWS floor is explicitly disabled (min_failed_rows = 0) for scalar checks, primary key checks, and key not-null guards (location_id_unique, one_row_per_hour, location_id_not_null, date_not_null).

5. Known Advisories (Expected Behavior)
The following anomalies represent expected domain behavior and do not indicate pipeline bugs:

UTC / US-Eastern Boundary Offsets: Offset hours at month boundaries appear in hour_within_batch_month and hour_within_covered_months checks due to timezone alignment.

End-of-Month Trips Without Weather: Approximately 0.4% of trips occurring at month boundaries may lack matching weather observations (trips_without_weather).
