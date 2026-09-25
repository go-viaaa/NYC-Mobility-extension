##  Data Quality & Runtime Monitoring

Runtime data quality is governed by **291 checks across five layers**, including **102 blocking checks** that halt pipeline execution when critical quality limits are breached.

### Five-Layer Data Quality Matrix

| **Layer** | **Total Checks** | **Blocking Checks** | **Governance Focus** | **Gate Indicator** |
|---|---:|---:|---|---|
| **Preload** | 101 | 17 | Evaluates source file layout, cast safety, and domain rules before landing. | `source_cleared_for_bronze` |
| **Bronze** | 42 | 28 | Confirms source-to-target row fidelity and `no_nulls_added_*` checks. | `batch_cleared_for_silver` |
| **Silver** | 87 | 34 | Validates classification promises (`dq_status`), quarantines, and formulas. | `batch_cleared_for_gold` |
| **Gold** | 49 | 21 | Verifies star schema construction, key derivations, and calendar integrity. | `warehouse_cleared` |
| **At-Rest** | 12 | 2 | Assesses standing referential integrity across fact/dimension foreign keys. | `referential_integrity_holds` |
| **Total** | **291** | **102** | | |

###  Status Resolution Ladder

Every check resolves through a strict five-branch condition order:

```sql
CASE
  WHEN total_rows = 0 AND check_name <> 'table_not_empty' THEN 'SKIP'
  WHEN failed_rows = 0 THEN 'PASS'
  WHEN failed_pct <= warn_pct THEN 'PASS'
  WHEN failed_rows <= min_failed_rows THEN 'WARN'
  WHEN failed_pct <= threshold_pct THEN 'WARN'
  ELSE 'FAIL'
END
```

- **`SKIP`** — Evaluated when the batch is empty and the check is not `table_not_empty`.
- **`PASS`** — Assigned when no failures exist or the failure rate remains within `warn_pct`.
- **`WARN`** — Assigned when failure counts or percentages remain within the defined tolerance thresholds (`min_failed_rows` or `threshold_pct`).
- **`FAIL`** — Triggered when configured failure limits are breached; blocks execution when the `(table, check)` pair is listed on the layer's blocking list.

###  Threshold Configurations

| **Configuration** | **Threshold** | **Application** |
|---|---:|---|
| **`STRICT`** | 0% | Applied to scalar checks, primary keys, and non-null key constraints such as `location_id_unique`, `location_id_not_null`, and `date_not_null`. |
| **`TOL`** | 10% | Applied to imperfect source-data categories where minor data quality issues are tolerable. |
| **`CAST_WARN`** | 5% | Warning threshold for string-to-datatype conversions. |
| **`CAST_FAIL`** | 10% | Failure threshold for string-to-datatype conversions, including the `no_nulls_added_*` family in Bronze. |
| **`MIN_ROWS`** | 5 rows | Minimum row threshold used to prevent single-row anomalies from unnecessarily stopping large batches. |

###  Great Expectations (GX) Infrastructure & Operational State

#### Output Location & Catalog Structure

The **Great Expectations (GX)** suite writes to the `nyc-mobility` catalog (hyphenated) under the `nyc_quality` schema. This is distinct from the standard SQL QC implementation, which targets the `nyc_mobility` catalog (underscored).

| **Table / View** | **Rows** | **Description** |
|---|---:|---|
| `dq_results` | 16 | Individual check results. |
| `dq_run_log` | 0 | Run summaries; currently unpopulated while `save_run_log` is pending completion. |
| `dq_rules` | 0 | Threshold overrides; none currently configured. |
| `vw_latest_dq_results` | 16 | View of the most recent check results. |
| `vw_latest_dq_run` | 0 | View of the most recent run log. |
