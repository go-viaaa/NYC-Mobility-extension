5. Disaster Recovery & Operational Runbook5.1 Failure Isolation & Gating LogicSelective Holdback: A failure on a non-required asset isolates only the affected table, allowing unaffected pipelines to proceed downstream.   Hard Stop Gate: If a check fails on a required table (green_taxi, green_taxi_clean, fact_taxi_trip, dim_taxi_zone), or total layer failures reach 5 (v_max_total_failures), the job writes a STOP gate record and invokes raise_error() to prevent corrupted data propagation.   5.2 Deterministic Rules & Recovery (TRUNCATE Rule)UTC Pinning: Every notebook explicitly sets SET TIME ZONE 'UTC' to ensure deterministic key calculation.   The TRUNCATE Safety Rule: Before re-running a MERGE task where join key derivation logic or session parameters have changed, execute TRUNCATE TABLE <target_table>. Re-running a MERGE with updated key logic on an un-truncated table causes duplicate row insertion (e.g., expanding row count from 44,208 to 88,416).   PlaintextRecovery Sequence for Key or Schema Modifications:

   Key / Schema Change Identified
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
    Validate Results (QC Gates)
5.3 Operational Runbooks🛠️ Runbook A: Remediating Pipeline FailuresLocate the failed job run in the Databricks UI and note the failing task.Query nyc_quality.dq_results to isolate failing checks:SQLSELECT * FROM nyc_quality.dq_results
WHERE layer = '<layer_name>'
  AND batch_month = '<batch_month>'
  AND status = 'FAIL';
Apply required code fixes or data cleanup.Execute a Repair Run in Databricks Jobs or re-trigger the workflow for the target year_month.Confirm that dq_run_log.overall_status evaluates to PASS or WARN.🛠️ Runbook B: Fixing CI/CD Deployment ErrorsReview the failure step log in GitHub Actions.For 401 Unauthorized errors, update expired DATABRICKS_HOST or DATABRICKS_TOKEN environment secrets.For bundle validation failures, execute databricks bundle validate locally to test YAML syntax before re-pushing.
