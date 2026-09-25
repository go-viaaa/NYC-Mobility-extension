```markdown
# 🚀 00 · Delivery: Versioning, Deployment & Orchestration

[![Owner](https://img.shields.io/badge/owner-JoanMaquinano%20%28CI%2FCD%29-blue?style=flat-square)](https://github.com/JoanMaquinano)
[![As Of](https://img.shields.io/badge/as%20of-2026--09--25-brightgreen?style=flat-square)](00-delivery.md)
[![Target](https://img.shields.io/badge/orchestrator-Databricks%20Asset%20Bundles-orange?style=flat-square&logo=databricks)](https://databricks.com

This document details how code changes move from development into production, the automated CI/CD workflows enforcing code standards, and the orchestration DAG governing the `NYC_Mobility` Databricks job.

---

## 1. How a Change Reaches Production

Every repository update follows a strict, automated 7-step deployment lifecycle[cite: 6]:

```text
Change  ──►  Test  ──►  Commit  ──►  Push  ──►  Deploy  ──►  Run  ──►  Verify

```

### 1.1 Deployment Lifecycle Steps

| Step | Action | Execution Context / Path |
| --- | --- | --- |
| **Change** | Work on an isolated feature branch (`feature/...`, `fix/...`).

 | Git

 |
| **Test** | Opening a PR triggers automated testing: formatting, linting, `pytest`, and bundle validation.

 | `.github/workflows/ci.yaml` <br>

<br> `pr-checks.yml`<br> |
| **Commit** | Auto Format commits style fixes back to the PR. Auto Reviewer assigns a teammate. `CODEOWNERS` assigns directory owners.

 | `auto-format.yml` <br>

<br> `auto-reviewer.yml` <br>

<br> `.github/CODEOWNERS`<br> |
| **Push** | The PR is merged into `feature/staging-part-2` after peer review approval.

 | GitHub

 |
| **Deploy** | Bundle validation and deployment: `databricks bundle validate` $\rightarrow$ `databricks bundle deploy -t dev`.

 | `.github/workflows/deploy.yaml` <br>

<br> `databricks.yml`<br> |
| **Run** | Executes the pipeline job: `databricks bundle run -t dev NYC_Mobility`.

 | `resources/jobs/nyc_mobility_job.yml`<br> |
| **Verify** | Quality tasks write results to `nyc_quality.dq_results` / `dq_run_log`, rendering results in the Data Quality dashboard.

 | `tests/0*_qc.sql` <br>

<br> `dashboards/`<br> |

---

### 1.2 GitHub Actions Workflows

Automated workflows handle continuous integration, code formatting, reviewer assignment, and deployment:

| Workflow | Trigger | Execution Steps |
| --- | --- | --- |
| **CI** | PR or push targeting `feature/staging-part-2`<br> | Runs `black`, `isort`, `flake8`, `sqlfluff`, `pytest`, and `databricks bundle validate`.

 |
| **Deploy Databricks Bundle** | Push targeting `feature/staging-part-2`<br> | Validates $\rightarrow$ deploys $\rightarrow$ runs `NYC_Mobility` (uses `DATABRICKS_HOST` and `DATABRICKS_TOKEN` secrets).

 |
| **PR Checks** | PR targeting `main` or `staging`<br> | Project structure validation + `pytest` suite execution.

 |
| **Auto Format & Commit** | PR opened or updated

 | Applies `sqlfluff fix`, `black`, `isort`, `nbstripout`, and commits changes directly back to the PR branch.

 |
| **Auto Assign Reviewer** | PR opened

 | Rotates reviewers: `jess-christine` $\rightarrow$ `catweyine` $\rightarrow$ `jg0901` $\rightarrow$ `go-viaaa` $\rightarrow$ `JoanMaquinano` $\rightarrow$ `jess-christine`.

 |

> ⚠️ **Deployment Status & Known Blockers (As of 2026-09-25):**
> 
> * ✅ **PR Checks** & **Auto Assign Reviewer** workflows pass successfully.
> 
> 
> * ❌ **CI Workflow Failure**: Fails at the `sqlfluff` step due to missing `param_style` in the placeholder templater config (blocking downstream `pytest` and bundle validation).
> 
> 
> * ❌ **Deploy Run #1 Failure**: Failed with `401 Unauthorized`—requires the `DATABRICKS_TOKEN` secret to be configured in the GitHub `staging` environment.
> 
> 
> 
> 

---

### 1.3 Databricks Asset Bundle Targets (`databricks.yml`)

The repository supports distinct execution environments via Databricks Asset Bundles:

| Target | Mode | Operational Purpose |
| --- | --- | --- |
| `dev` *(Default)* | Development mode (resources prefixed per user)

 | Staging integration runs triggered from CI workflows.

 |
| `prod` | Production mode (root path: `~/.bundle/nyc_mobility/prod`)

 | Production pipeline deployment and schedule execution.

 |

---

## 2. Pipeline Orchestration (`NYC_Mobility` Job)

The pipeline is orchestrated as a single Databricks job named **`NYC_Mobility`**, defined in YAML using explicit `depends_on` task dependencies:

```text
                     Preload_Checks
                           │
                           ▼
   ┌───────────────────────┼───────────────────────┐
   │                       │                       │
Green_Taxi_Bronze   Taxi_Zones_Bronze    Weather_Bronze
   │                       │                       │
   └───────────────────────┼───────────────────────┘
                           │
                           ▼
                       Bronze_QC            ◄── Gate: batch_cleared_for_silver
                           │
                           ▼
   ┌───────────────────────┼───────────────────────┐
   │                       │                       │
Green_Taxi_Silver   Taxi_Zone_Silver     Weather_Silver
   │                       │                       │
   └───────────────────────┼───────────────────────┘
                           │
                           ▼
                       Silver_QC            ◄── Gate: batch_cleared_for_gold
                           │
                           ▼
   ┌───────────────┬───────┴───────┬───────────────┐
   │               │               │               │
Dim_Date    Dim_Taxi_Zone    Dim_Weather       Fact_Trip
   │               │               │               │
   └───────────────┴───────┬───────┴───────────────┘
                           │
                           ▼
                        Gold_QC             ◄── Gate: warehouse_cleared

```

### 2.1 Key Orchestration Mechanics

* **Job Parameter (`year_month`)**: Accepts a batch month parameter (e.g., `2026-03`). If left blank, `Preload_Checks` automatically detects and processes the oldest landed month not yet present in Bronze.


* **Quality Gate Short-Circuiting**: If any Quality Control task (`Bronze_QC`, `Silver_QC`, `Gold_QC`) raises an exception, all downstream tasks are automatically skipped, preventing invalid data from reaching subsequent layers.


* **Parallel Great Expectations Job**: A parallel, manually-triggered job named `NYC Mobility - GX` (`resources/jobs/nyc_mobility_gx_job.yml`) executes the same pipeline layout using Great Expectations tasks (`GX`, `GX2`, `GX3`) in place of standard SQL QC tasks. Automated CI (`deploy.yaml`) deploys and runs `NYC_Mobility` exclusively.


* **Standalone / Scheduled QC Tasks**: `tests/05_at_rest_integrity_qc.sql` and `tests/09_freshness_check.sql` run on demand or as separate scheduled tasks to monitor warehouse drift and data freshness.



```

```
