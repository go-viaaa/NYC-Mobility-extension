## Disaster Recovery & Operational Standards

This section defines the operational standards for timezone management, deterministic key generation, safe recovery, `MERGE` operations, and automated data-quality validation.

### Task Failure Behavior & Isolation

- **Selective Table Holdback** — A `FAIL` on a non-required table holds back only the affected table, allowing unaffected pipelines to proceed.
- **Hard Stop Gating** — If a check on `v_required_tables` fails, or if total failures exceed `v_max_total_failures` (`5`), the layer writes a `STOP` gate record and triggers `raise_error()`. This halts downstream execution before contaminated data can reach the Gold layer.

### Idempotency & Safe Rerun Strategy

- **Session Timezone Pinning** — All jobs execute `SET TIME ZONE 'UTC'` to ensure deterministic key generation, including `unix_timestamp` and date-based keys, regardless of cluster location.

- **Safe Re-execution (`TRUNCATE` Rule)**:
  - Before rerunning a `MERGE` task where key-generation logic or session parameters have changed, execute a `TRUNCATE` on the target table.
  - **Reason:** Rerunning `WHEN NOT MATCHED THEN INSERT` with modified key expressions without truncating can create duplicate fact records, such as increasing row counts from `44,208` to `88,416`.

- **Atomic Batch Scoping** — Ingestion and cleaning tasks are scoped using `batch_month` and `source_file`, allowing individual monthly batches to be rerun independently and idempotently.

### Timezone Management

All notebooks must explicitly use UTC to ensure consistent timestamp handling across environments.

```sql
SET TIME ZONE 'UTC';
```

### Deterministic Key Generation

Primary and surrogate keys must be generated using **timezone-independent logic**.

Avoid deriving keys directly from session-dependent expressions such as:

```sql
CAST(ts AS STRING)
DATE(ts)
```

Different session timezones can produce different values from the same timestamp, which may result in **inconsistent keys or duplicate records**.

> **Standard:** Key-generation logic must produce the same key regardless of the execution environment or session timezone.

###  Recovery & MERGE Safety

Changes to schema definitions or key-generation logic require a controlled recovery process.

#### Schema or Key Changes

When key-generation logic or the target schema changes, follow this sequence:

```text
TRUNCATE Target Table
        ↓
Apply Updated Logic
        ↓
Run MERGE
        ↓
Validate Results
```

This ensures that records created using the previous logic do not remain in the target table.

#### Preventing Duplicate Records

Running a `MERGE` after changing key logic without resetting the target table can result in duplicate records:

```text
Existing Records
       +
Updated Key Logic
       ↓
     MERGE
       ↓
Potential Duplicate Records
```

> **Recovery Rule:** Truncate the affected target table before rerunning a `MERGE` whenever the key-generation strategy has changed.

### Data Quality Automation

Data-quality validation is automated using **Great Expectations (GX)** and supporting validation scripts.

The validation workflow is:

```text
07_gx_checks.py
       ↓
GX Expectations
       ↓
Layer Validation
       ↓
PASS / WARN / FAIL
       ↓
Quality Gate
       ↓
Continue / Block Pipeline
```

GX provides a standardized validation framework across the **Bronze, Silver, and Gold layers**, replacing the previous layer-specific QC implementation.

#### Quality Status

| Status | Description | Pipeline Action |
|---|---|---|
| `PASS` | All required expectations are satisfied. | Continue |
| `WARN` | Issues are within the configured tolerance. | Continue with warning |
| `FAIL` | Quality thresholds are breached. | Block when configured as a blocking check |

###  Operational Principles

The pipeline follows these core operational standards:

- **Consistency** — Use UTC across all notebook sessions.
- **Determinism** — Generate keys independently of session timezone.
- **Safe Recovery** — Reset affected targets when key-generation logic changes.
- **Controlled MERGE** — Validate target state before and after `MERGE` operations.
- **Automated Quality** — Use GX to standardize data-quality validation.
- **Quality Gates** — Prevent critical data-quality failures from progressing through the pipeline.

> **Goal:** Maintain a reliable and recoverable pipeline by combining deterministic processing, controlled recovery procedures, and automated data-quality gates.
