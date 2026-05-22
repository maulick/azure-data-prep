# Azure Data Engineering Prep

Hands-on Azure data engineering exercises building toward interview-readiness for an Azure DevOps role with data platform responsibilities (ADF, Synapse, Databricks, SQL DB, Key Vault, networking, CI/CD).

## Sessions

### [Session 1 — Azure Data Factory](session-1-adf/)

End-to-end ADF pipeline: CSV ingestion from ADLS Gen2 → Mapping Data Flow with derived column + aggregate → Parquet output. Full Git integration, version-controlled JSON for all resources.

**Stack:** Azure Data Factory, ADLS Gen2, Mapping Data Flow, Spark, Parquet, GitHub integration

**See:** [Session 1 detailed notes](session-1-adf/notes.md)

### Session 2 — Synapse Analytics (planned)
Serverless SQL pool queries over Parquet, Spark notebooks, comparison of SQL vs Spark for analytics workloads.

### Session 3 — Key Vault + ML Workspace + Managed Identity (planned)
Secrets management end-to-end: managed identity → RBAC → Key Vault Secrets User → notebook secret retrieval.

## Approach

Each session is a self-contained exercise designed to produce concrete artifacts (working pipelines, screenshots, notes) that can be discussed in interviews. Built in the KodeKloud Azure Data Playground (3-hour sandboxed sessions), with work persisted to this repo via Git integration.

## Background

Building from a DevOps foundation (Azure DevOps, YAML pipelines, Bicep, Terraform) toward Azure data platform competency.
