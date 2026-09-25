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
