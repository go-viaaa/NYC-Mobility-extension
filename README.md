# 🚕 NYC Mobility Extension

Production documentation for the [NYC-Mobility](https://www.google.com/search?q=https://github.com/JoanMaquinano/NYC-Mobility&utm_source=gemini) pipeline

---

## 📌 Project Overview

This project extends the existing **NYC Mobility** medallion data engineering pipeline on Databricks and Unity Catalog to make it more **reliable, repeatable, and collaboration-ready**—without unnecessarily redesigning the existing architecture.

Building on the integration of taxi trips, weather conditions, and taxi zone metadata, the extension introduces automated code testing, data-quality validation, and CI/CD checks while strengthening repository governance, configuration, deployment, and documentation practices.

> **As of:** 2026-09-25 · **Pipeline Branch:** `feature/staging-part-2`.

---

## ✨ Key Enhancements

* 🧪 **Automated Code Testing**: Integrated `pytest` unit testing to validate pipeline transformations[cite: 5].
* 🔍 **Data-Quality Validation**: Dual-layer verification pairing custom SQL quality checks with Great Expectations (GX) suites[cite: 5].
* 🚀 **CI/CD Automation**: Streamlined integration using GitHub Actions paired with Databricks Asset Bundles (DAB)[cite: 5].
* 🛡️ **Repository Governance**: Enforced branch protection, automated code formatting (`black`, `isort`, `sqlfluff`), PR checks, and peer reviewer routing.

---

## 📚 Documentation Index

| Module | Document | Core Focus |
| --- | --- | --- |
| **00** | [Delivery: Version, Deploy, Orchestrate](https://www.google.com/search?q=00-delivery.md&utm_source=gemini) | How code changes reach production and the execution order of the pipeline. |
| **01** | [Architecture](https://www.google.com/search?q=01-architecture.md&utm_source=gemini)[cite: 5] | Source systems, medallion layers, table schemas, and quality gate triggers. |
| **02** | [Monitoring](https://www.google.com/search?q=02-monitoring.md&utm_source=gemini) | Pipeline observability, check status resolution, and execution logging. |
| **03** | [Governance](https://www.google.com/search?q=03-governance.md&utm_source=gemini) | Naming conventions, ownership mappings, catalog permissions, and data lineage. |
| **04** | [Failure & Recovery Strategy](https://www.google.com/search?q=04-recovery-strategy.md&utm_source=gemini) | Handling task exceptions, table isolation, and safe, idempotent reruns. |
| **05** | [Extension Tools](https://www.google.com/search?q=05-extension-tool.md&utm_source=gemini) | Analysis of added tools, design choices, trade-offs, and operational results. |

---

## 👥 Audience Guide

* **Team Members & Contributors**: Use documents **[03 Governance](https://www.google.com/search?q=03-governance.md&utm_source=gemini)** and **[04 Failure & Recovery](https://www.google.com/search?q=04-recovery-strategy.md&utm_source=gemini)** for daily engineering standards, branching conventions, and operational runbooks.
* **Reviewers & Instructors**: Refer to documents **[00 Delivery](https://www.google.com/search?q=00-delivery.md&utm_source=gemini)** and **[05 Extension Tools](https://www.google.com/search?q=05-extension-tool.md&utm_source=gemini)** to audit architectural upgrades, CI/CD mechanics, and tool integration evaluations.
