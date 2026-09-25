## Disaster Recovery & Operational Standards

This section defines operational practices for **timezone consistency, deterministic key generation, safe recovery, MERGE operations, and automated data-quality validation**.

### Timezone Management

#### Session Enforcement

All notebooks must explicitly enforce UTC to ensure consistent timestamp handling across environments:

```sql
SET TIME ZONE 'UTC';
