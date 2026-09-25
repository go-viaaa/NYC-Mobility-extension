## Disaster Recovery & Operational Standards

This section defines operational practices for **timezone consistency, deterministic key generation, safe recovery, MERGE operations, and automated data-quality validation**.

### Timezone Management

#### Session Enforcement

All notebooks must explicitly enforce UTC to ensure consistent timestamp handling across environments:

```sql
SET TIME ZONE 'UTC';


**Deterministic Key Generation**
Primary and surrogate keys must not be derived from session-dependent timestamp representations, such as:

CAST(ts AS STRING)
DATE(ts)

Reason: Different session timezones can produce different key values from the same timestamp, potentially resulting in inconsistent keys and duplicate records.

Standard: Use timezone-independent and deterministic logic when generating primary or surrogate keys.

------------
**Recovery & MERGE Safety**

**Schema or Key Strategy Changes**

When the schema or key-generation logic changes, the affected target table must be reset before rerunning the pipeline.

**Required sequence:**
1. TRUNCATE affected target table
            ↓
2. Apply updated schema/key logic
            ↓
3. Re-run MERGE operation

This prevents records generated using the previous schema or key logic from remaining in the target table.

**Key Duplication Prevention**

Changing key-generation logic without truncating the existing target table can result in duplicate records:

Existing Records
       +
Updated Key Logic
       ↓
     MERGE
       ↓
Duplicate Fact Records

**Recovery Rule: ** Always truncate the affected target table before rerunning a MERGE when key-generation logic has changed.

------
**Data Quality Automation**

Data-quality validation is automated using Great Expectations (GX) and supporting validation scripts.

The validation process follows:
07_gx_checks.py
       ↓
GX Expectations
       ↓
Layer Quality Validation
       ↓
PASS / WARN / FAIL
       ↓
Pipeline Quality Gate

GX standardizes data-quality validation across the **Bronze, Silver, and Gold layers**, replacing the previous layer-specific QC implementation.

**Quality Gate Behavior**
**Status**	| **Meaning**
| PASS |	Data meets the defined quality expectations. |
| WARN |	Data has issues within the configured tolerance.|
| FAIL |	Data breaches defined quality thresholds and may block pipeline execution.|

Operational Goal: Ensure that data-quality issues are detected consistently before unreliable data progresses to the next pipeline layer.
