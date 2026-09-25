## Data Quality & Runtime Monitoring

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

Disaster Recovery & Operational Standards
 Timezone Management

Session Enforcement: Every notebook must explicitly execute:

SET TIME ZONE 'UTC';
Deterministic Keys: Primary and surrogate keys must not be derived from session-dependent string representations such as CAST(ts AS STRING) or DATE(ts).
Reason: Session timezone differences can change key values and potentially create duplicate records.

Recovery & MERGE Safety
Schema or Key Strategy Changes

When the schema or key-generation logic changes:

TRUNCATE the affected target table.
Apply the updated key or schema logic.
Re-run the MERGE operation.

This prevents records generated using the previous key logic from remaining in the target table.

Key Duplication Prevention

Changing key-generation logic without truncating the target table can cause:

Existing Records
      +
New Key Logic
      ↓
MERGE
      ↓
Duplicate Fact Records

Therefore, target tables should be truncated before rerunning a MERGE when key-generation logic has changed.

Data Quality Automation

Data-quality validation is automated through Great Expectations (GX) and supporting validation scripts.

Example:

07_gx_checks.py
      ↓
GX Expectations
      ↓
Layer Quality Validation
      ↓
PASS / WARN / FAIL
      ↓
Pipeline Gate

The GX implementation standardizes quality validation across the pipeline layers and replaces the previous layer-specific QC implementation.
