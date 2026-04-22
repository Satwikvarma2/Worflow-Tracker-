# Revenue Reporting Workflow Automation & SLA Dashboard

## Project Overview

This project is a **workflow automation and monitoring dashboard** built to track the end-to-end publish lifecycle across multiple tenants and systems. The primary goal is to provide visibility into **when publishes complete** and **when upstream data becomes available**, enabling the team to quickly diagnose SLA misses with evidence — e.g., *"Upstream data was not available until X time, which is why the SLA was missed."*

---

## Problem Statement

We have a large number of pipelines, runbooks, and systems across different tenants. When an SLA is missed, there is no single view to answer:

- **When did the upstream data become available?**
- **When did the publish actually complete?**
- **Which step in the pipeline caused the delay?**

This dashboard consolidates all of that information into one place, making SLA root-cause analysis straightforward.

---

## Architecture

### Tenants

| Tenant | Purpose |
|--------|---------|
| **PME** (Publishing & Monitoring Engine) | Orchestration tenant — kicks off and tracks the distribution pipeline (CORP runbooks, data copy, parquet publishing) |
| **CORP** (Corporate) | Execution tenant — runs the core E2E runbooks, Synapse pipelines, and stores pipeline telemetry in SQL |

### Systems & Components

```
┌──────────────────────────────────────────────────────────────────┐
│                             Tenant -1                            │
│                                                                  │
│  ┌─────────────────────────────────────┐                         │
│  │  PME Automation Account (Runbook)   │                         │
│  │  - KickOff CORP Runbooks            │                         │
│  │  - Sync Shards / Factory / Mart     │                         │
│  │  - Copy Parquets                    │                         │
│  │                                     │                         │
│  │  - Track CORP Runbook Completion    │                         │
│  └─────────────────┬───────────────────┘                         │
│                     │ triggers                                   │
└─────────────────────┼────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────────┐
│                         CORP Tenant                              │
│                                                                  │
│  ┌─────────────────────────────────────┐                         │
│  │  CORP Automation Account (Runbook)  │                         │
│  │  - E2E Publish Pipeline tasks       │                         │
│  │  - FDL Restore Triggers             │                         │
│  │  - VM Start/Stop                    │                         │
│  │  - Backup, Extract, Freeze          │                         │
│  └─────────────────────────────────────┘                         │
│                                                                  │
│  ┌─────────────────────────────────────┐                         │
│  │  Synapse Pipelines (FDL)            │                         │
│  │  - PublishAndDistributeDelta        │                         │
│  │  - PublishDomainDelta               │                         │
│  │  - PublishPerspective               │                         │
│  │  - DistributePerspective            │                         │
│  │  - PublishDomainsDB                 │                         │
│  │  - GLGeoMR                          │                         │
│  │  - Inventory                        │                         │
│  └─────────────────────────────────────┘                         │
│                                                                  │
│  ┌─────────────────────────────────────┐                         │
│  │  SQL Server (MSSDomainPRD)          │                         │
│  │  [Platform].[Telemetry]             │                         │
│  │      .[PipelineDetail_Archive]      │                         │
│  │  - Stores CORP runbook task logs    │                         │
│  └─────────────────────────────────────┘                         │
└──────────────────────────────────────────────────────────────────┘
```

---

## Data Sources

The dashboard is powered by **4 CSV data files**, each pulled from a different log source:

| # | Data File | What It Contains | Tenant | Collection Method | Source System |
|---|-----------|-----------------|--------|-------------------|---------------|
| 1 | **CORP_Runbooks_Run_Summary.csv** | CORP runbook task runs (task name, status, start/end times, avg runtime) | CORP | **SQL Pipeline** | SQL: `MSSDomainPRD [Platform].[Telemetry].[PipelineDetail_Archive]` |
| 2 | **FDL_PublishTime.csv** | Final FDL publish timestamps + factory run mode (Daily YTD, FM Close, Restatement) | CORP | **Runbook** (KQL query) | Log Analytics: `MSSOA-LA-PROD` |
| 3 | **FDL_Pipeline_Runtime_Summary.csv** | Synapse pipeline runs (pipeline name, start/end, duration, status, attempts) | CORP | **Runbook** (KQL query) | Log Analytics: `MSSOA-LA-PROD` |
| 4 | **PME_Pipeline_Run_Summary.csv** | PME orchestration pipeline runs (runbook group name, start/end times) | PME | **Runbook** (KQL query) | Log Analytics: `la-finpub-revdist-prod` |

### How Data Is Collected

```
CORP Runbook Logs ──► SQL Pipeline ──► CORP_Runbooks_Run_Summary.csv
                                          │
FDL Publish Times ──► CORP Runbook ──► FDL_PublishTime.csv
                      (KQL Query)        │
                                          │
FDL Pipeline Runs ──► CORP Runbook ──► FDL_Pipeline_Runtime_Summary.csv
                      (KQL Query)        │
                                          │
PME Pipeline Runs ──► PME Runbook  ──► PME_Pipeline_Run_Summary.csv
                      (KQL Query)        │
                                          ▼
                                    Dashboard / HTML Report
```

---

## Key Concepts

### WorkflowInstanceId

Every publish cycle is identified by a unique `WorkflowInstanceId` (e.g., `20418`). This ID links data across all four CSV files, allowing us to reconstruct the full timeline for a single publish run.

### Factory Run Mode

Each publish run operates in one of three modes (from `FDL_PublishTime.csv`):

| Run Mode | Description |
|----------|-------------|
| **Daily YTD** | Standard daily year-to-date publish |
| **FM Close** | Fiscal month close publish |
| **Restatement** | Ad-hoc restatement publish |

### Upstream Data Availability

"Upstream data available" is determined by the **later** of two CORP runbook task completions:

- `MSSales-FDL-Restore-Trigger`
- `MSSales-FDL-LicenseMaster-Copy-Trigger`

This timestamp is critical for SLA analysis — if upstream data arrived late, downstream publish will also be late.

### SLA Tracking

Two key publish completion times are tracked:

| Metric | How It's Determined |
|--------|---------------------|
| **MSRA Publish Time** | Completion of `MSSales-E2E-Restore-Transactions-On-PrePublish-Servers` (falls back to overall runbook end) |
| **FDL Publish Time** | From `FDL_PublishTime.csv` (falls back to max Synapse pipeline end time) |

---

## Dashboard Output

The tool generates an **HTML report** (`Revenue_Reporting_Publishing_Report.html`) with:

1. **Trend Charts** — Publish times, upstream availability, and E2E durations over time
2. **Gantt Charts** — Per-workflow timeline showing all PME tasks, CORP runbook tasks, and Synapse pipelines with their start/end times
3. **SLA Analysis** — Side-by-side comparison of upstream data availability vs. publish completion
4. **Executive Summary** — Per-workflow E2E duration, breakdown by phase (PME / Runbook / Synapse), and % deviation from average

### Report Tabs

| Tab | Filter | Max Workflows Shown |
|-----|--------|---------------------|
| Daily YTD | `FactoryRunMode = "Daily YTD"` | 30 |
| FM Close | `FactoryRunMode = "FM Close"` | 15 |
| Restatement | `FactoryRunMode = "Restatement"` | 15 |

---

## Pipeline Flow (Simplified)

```
1. PME KickOff
       │
       ▼
2. PME Sync (Shards → Factory → Mart → Perspectives)
       │
       ▼
3. PME Data Copy (Big Domains, Inventory, License Master, GL, MR, Geo)
       │
       ▼
4. CORP Runbooks Execute:
   a. Start VMs
   b. Shard Redistribution & DB Reset
   c. FDL Restore Triggers (upstream data arrives here)
   d. Prerequisite Extracts
   e. Publish to Perspectives & Domains
   f. Backup & Distribution
   g. Restore Transactions to Pre/Post Publish Servers (MSRA SLA hit)
   h. Stop VMs
       │
       ▼
5. Synapse Pipelines Execute:
   a. Inventory Pipeline
   b. DomainsDB Pipeline
   c. PublishPerspective & DistributePerspective
   d. PublishDomainDelta & DomainDelta
   e. PublishAndDistributeDelta (Master)
   f. GLGeoMR Pipeline
       │
       ▼
6. FDL Publish Complete (FDL SLA hit)
       │
       ▼
7. PME TrackCompletion
```

---

## Scripts

| Script | Purpose |
|--------|---------|
| `generate_report_v3.ps1` | Main report generator — reads all 4 CSVs, processes data, generates the HTML dashboard |
| `new_script.ps1` | Supplementary chart — plots Fabric Publish Time vs. FDL Publish Time over time |

---

## SLA Root-Cause Analysis — How To Use

When an SLA is missed for a given day:

1. **Open the dashboard** and navigate to the correct run mode tab (Daily YTD / FM Close / Restatement)
2. **Find the workflow** for that date using the Workflow ID
3. **Check "Upstream Data Available" time** — if it's later than expected, the delay is upstream (not our fault)
4. **Check the Gantt chart** — identify which specific task(s) ran longer than their average (red dashed markers show the average)
5. **Check the SLA chart** — compare MSRA Publish and FDL Publish times against the SLA target
6. **Report with evidence** — "On [date], upstream data was not available until [time]. The normal availability is [time]. This caused a [X]-hour delay in the publish, resulting in the SLA miss."
