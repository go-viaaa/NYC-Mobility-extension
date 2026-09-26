# Governance and Process Improvements

## Overview

To improve security, maintainability, collaboration, and operational reliability, multiple governance controls and standards were introduced throughout the project lifecycle.

The following sections compare the project's previous state with the current implementation.

| Governance Domain | Historical State ("Before") | Governed State ("After") | Operational & Platform Impact |
|---|---|---|---|
| **Repository Governance** | Direct merges allowed; PR reviews defined but unenforced. | Enforced branch protection, mandatory PR approvals, and active rulesets. | Higher code stability, eliminated unreviewed commits, and strict auditability. |
| **Access & Security** | Broad workspace/job access; excessive administrative rights. | Least-privilege role boundaries; administrative rights pruned. | Reduced blast radius, lower security risk, and clear access accountability. |
| **Engineering Standards** | Mixed naming (`camelCase`, `PascalCase`, `snake_case`); informal layouts. | Standardized `snake_case` across catalogs, schemas, tables, and SQL assets. | Lower developer cognitive load, faster onboarding, and consistent maintainability. |
| **Change Management** | Lack of approval tracking; loose control over pipeline modifications. | Formal PR review workflows with automated approval logging. | Complete auditability of pipeline updates and enhanced collaboration. |
| **Deployment** | Manual deployments with environment-specific configuration risks. | Infrastructure-as-code deployment via Databricks Asset Bundles (DABs). | Reproducible, risk-reduced deployments across environments. |
| **Data Governance** | Custom SQL checks lacking central management or reusable suites. | Dual-layer validation (SQL + Great Expectations) integrated into Medallion gates. | Trusted data assets, automated blocking gates, and standardized rule suites. |
| **Documentation** | Distributed tribal knowledge with limited setup coverage. | Standardized READMEs, architecture guides, and runbooks. | Reduced onboarding friction and long-term platform sustainability. |

---

# 1. Repository Governance

## Before

- Main branch could be modified without sufficient safeguards.
- Direct merges to protected branches were possible.
- Pull request review requirements were defined but not enforced.
- Governance controls relied largely on team discipline.
- Code ownership and approval accountability were limited.

## After

- Branch protection rules implemented.
- Pull request approvals required before merging.
- Repository rulesets activated and enforced.
- Merge process standardized through PR workflow.
- Review and approval process became mandatory for repository changes.

### Impact

- Improved code quality and repository stability.
- Reduced risk of unreviewed or accidental changes.
- Increased accountability and auditability of repository activity.

Directory Ownership & Review Routing (`CODEOWNERS`)

Code reviews are automatically routed to area owners via `.github/CODEOWNERS`[cite: 10]. A secondary reviewer is dynamically assigned in rotation via `.github/workflows/auto-reviewer.yml` to eliminate self-approvals[cite: 10]:

| Domain / Folder | Code Path | Designated Owner |
|---|---|---|
| **Green Taxi Pipeline** | `src/green_taxi/` | `@jess-christine` |
| **Taxi Zones Lookup** | `src/taxi_zones/`| `@go-viaaa` |
| **Weather Integration** | `src/weather/` | `@catweyine` |
| **Data Quality Framework** | `tests/` | `@jg0901` |
| **CI/CD, DABs & Documentation** | `*` (Repository Root) | `@JoanMaquinano|

---

# 2. Access & Security Governance

## Before

- Team members had broader access than required.
- Access included:
  - Databricks jobs
  - Workspace resources
  - Project files
  - Administrative permissions beyond assigned responsibilities
- Least-privilege principles were not consistently applied.

## After

- Permissions reviewed and restricted based on role and responsibility.
- Unnecessary administrative access removed.
- Workspace and job access limited to required users.
- Access control aligned more closely with least-privilege principles.

### Impact

- Reduced security risk.
- Lower chance of accidental modifications.
- Improved governance and access accountability.

Recommended Unity Catalog RBAC Matrix

Access to Unity Catalog objects under `nyc_mobility` is segmented by principal responsibility:

| Role Principal | Target User / Service | `nyc_bronze` | `nyc_silver` | `nyc_gold` | `nyc_quality` |
|---|---|---|---|---|---|
| **Pipeline Service Principal** | CI/CD Automated Deployments | `READ / WRITE`| `READ / WRITE` | `READ / WRITE` | `READ / WRITE` |
| **Data Engineers** | Platform Developers | `READ` | `READ` | `READ` | `READ` |
| **Analysts & Dashboards** | BI Consumers / Reporting | — | — | `READ` | `READ` |
---

# 3. Standards Governance

## Before

- Naming conventions were inconsistent.
- Mixed use of camelCase, PascalCase, and snake_case.
- Repository organization depended on individual developer preferences.
- Standards were informal and inconsistently applied.

## After

- Standardized `snake_case` naming convention adopted across:
  - Catalogs
  - Schemas
  - Tables
  - Columns
  - Files
  - SQL objects
- Improved consistency across project assets.
- Shared development standards established.

### Impact

- Easier onboarding for contributors.
- Improved readability and maintainability.
- Reduced confusion during development and review.

Asset Standards

| Object Type | Naming Convention | Operational Example |
|---|---|---|
| **Catalog** | Project-level single catalog | `nyc_mobility` |
| **Schema** | `nyc_<medallion_layer>` | `nyc_bronze`, `nyc_silver`, `nyc_gold`, `nyc_quality` |
| **Bronze Tables** | `<source_name>` | `green_taxi`, `taxi_zones`, `weather` |
| **Silver Tables** | `<source>_clean` / Views: `vw_<source>_valid` | `green_taxi_clean`, `vw_green_taxi_valid` |
| **Gold Tables** | `dim_<entity>` / `fact_<process>` | `dim_date`, `dim_taxi_zone`, `fact_taxi_trip` |
| **Quality Views** | `vw_` prefix | `vw_dq_by_month`, `vw_latest_dq_run`|
| **Columns** | `snake_case` (Keys end in `_key` or `_id`) | `trip_key`, `pickup_location_id` |
| **Source Code** | `src/<source>/<layer>/` | `src/weather/silver/weather_clean.sql`10] |
| **QC Notebooks** | Prefix numbered by execution order | `tests/02_bronze_all_qc.sql |
| **Git Branches** | `feature/<topic>` or `fix/<topic>` | `feature/dashboard-qc`, `fix/github-actions |

---

# 4. Change Management Governance

## Before

- Development process lacked consistently enforced approval controls.
- Changes could move through the workflow with limited governance oversight.
- Review history was not guaranteed for every change.

## After

- Formal PR review workflow adopted.
- Approval history captured through repository governance controls.
- Change management process standardized across contributors.

### Impact

- Better traceability of changes.
- Stronger accountability.
- Improved collaboration and knowledge sharing.

---

# 5. Deployment Governance

## Before

- More deployment decisions depended on manual processes.
- Environment-specific configurations increased deployment risk.
- Changes were harder to reproduce consistently.

## After

- Databricks Asset Bundles (DAB) used for deployment-as-code.
- Configuration-driven deployments reduced hardcoded dependencies.
- Deployments became more repeatable across environments.
- Environment configuration standardized.

### Impact

- Improved deployment consistency.
- Reduced operational risk.
- Better reproducibility and maintainability.

---

# 6. Data Governance

## Before

- Data quality controls primarily relied on custom SQL checks.
- Validation rules were less centralized and less reusable.
- Governance controls varied by implementation.

## After

- Great Expectations validation framework implemented.
- Reusable validation suites created.
- Data quality gates integrated into the Medallion pipeline.
- Validation became standardized and easier to maintain.

### Impact

- Improved trust in data outputs.
- More consistent data quality enforcement.
- Stronger governance over critical datasets.

Data governance relies on dual-layer validation and cataloged quality metrics.

```text
Source Landing ──► Bronze QC Gate ──► Silver QC Gate ──► Gold QC Gate ──► Analytics Warehouse
                    (28 Blocking)     (34 Blocking)     (21 Blocking)     (2 Blocking At-Rest)

Data Lineage Enforcement
Mandatory Lineage Attributes:

Bronze: source_file, ingestion_time (or source_file_month, ingestion_timestamp).

Silver: Inherits Bronze lineage and appends silver_at.

Gold: Appends created_at and carries forward qc_error_descriptions.

Quality: Every check record stores run_id, run_ts, layer, table_name, and batch_month.

Lineage Blocking Gate: source_file_recorded acts as a blocking check in Bronze—untraceable records halt the load immediately[cite: 10].

6.2 Data Quality Rule Management Lifecycle
Rule Definition: QC rules are authored directly in layer notebooks (tests/0*_qc.sql).

Catalog Synchronization: Executing tests/06_dq_rule.sql populates nyc_quality.dq_rules with rule logic, thresholds, and blocking flags.

Drift Detection: Automated drift queries compare dq_rules definitions against executed checks in dq_results to detect unauthorized modifications.
---

# 7. Documentation Governance

## Before

- Project knowledge was distributed across implementation details and team discussions.
- Documentation coverage was limited.
- Operational knowledge depended heavily on individual contributors.

## After

- README and project documentation expanded.
- Architecture and deployment documentation improved.
- Setup and operational procedures documented.
- Version tracking and project history better maintained.

### Impact

- Improved maintainability.
- Reduced onboarding effort.
- Better long-term project sustainability.

Data Classification: Public NYC open data and Open-Meteo weather datasets contain zero Personal Identifiable Information (PII)].

Delta History Retention: Delta table history is retained for 7 days by default, supporting point-in-time recovery (RESTORE).

Operational Documentation: Platform architecture (01-architecture.md), monitoring ladders (02-monitoring.md), and disaster recovery strategies (04-recovery-strategy.md) are versioned alongside pipeline source code.

---

# Summary of Governance Improvements

✅ Branch protection enforced

✅ Mandatory PR approvals implemented

✅ Repository rulesets activated

✅ Least-privilege access controls applied

✅ Standardized snake_case naming conventions

✅ Deployment governance through DAB

✅ Data quality governance through Great Expectations

✅ Formalized change management process

✅ Improved documentation governance

✅ Enhanced auditability and collaboration controls

---

# Overall Impact

The project evolved from a largely trust-based development environment into a governed, secure, and scalable data platform.

Key outcomes include:

- Stronger repository governance
- Improved security and access control
- Better change management practices
- More reliable deployment processes
- Standardized development practices
- Enhanced data quality governance
- Improved maintainability and collaboration

**Result:** A more secure, auditable, maintainable, and collaboration-ready platform aligned with software engineering, DevOps, and data governance best practices.
