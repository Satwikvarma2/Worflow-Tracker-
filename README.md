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
| **Orchestration Tenant** | Kicks off and tracks the distribution pipeline — triggers execution runbooks, syncs data, copies parquet files, and monitors completion |
| **Execution Tenant** | Runs the core end-to-end runbooks, Synapse pipelines, and stores pipeline telemetry in a SQL database |

### Systems & Components

```
┌──────────────────────────────────────────────────────────────────┐
│                    Orchestration Tenant                           │
│                                                                  │
│  ┌─────────────────────────────────────┐                         │
│  │  Automation Account (Runbook)       │                         │
│  │  - KickOff Execution Runbooks       │                         │
│  │  - Sync Metadata Tables             │                         │
│  │  - Copy Parquet Files               │                         │
│  │  - Track Execution Completion       │                         │
│  └─────────────────┬───────────────────┘                         │
│                     │ triggers                                   │
└─────────────────────┼────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Execution Tenant                             │
│                                                                  │
│  ┌─────────────────────────────────────┐                         │
│  │  Automation Account (Runbook)       │                         │
│  │  - End-to-End Publish Tasks         │                         │
│  │  - Upstream Data Restore Triggers   │                         │
│  │  - VM Management (Start/Stop)       │                         │
│  │  - Backup, Extract, Freeze          │                         │
│  └─────────────────────────────────────┘                         │
│                                                                  │
│  ┌─────────────────────────────────────┐                         │
│  │  Azure Synapse Pipelines            │                         │
│  │  - Publish & Distribute Domains     │                         │
│  │  - Publish Perspectives             │                         │
│  │  - Publish Inventory                │                         │
│  │  - GL/Geo/MR Pipelines             │                         │
│  └─────────────────────────────────────┘                         │
│                                                                  │
│  ┌─────────────────────────────────────┐                         │
│  │  SQL Server                         │                         │
│  │  - Pipeline telemetry archive       │                         │
│  │  - Runbook task execution logs      │                         │
│  └─────────────────────────────────────┘                         │
└──────────────────────────────────────────────────────────────────┘
```

---

## Scheduling & Hosting

- **Automated daily trigger** — The data collection runbooks execute at **9:00 AM every day**, pulling the latest pipeline and runbook logs into CSV files
- **Static website hosting** — The generated HTML dashboard is stored in an **Azure Storage Account** with the **static website** option enabled, making it accessible via a browser URL without any additional web server infrastructure

---

## Data Sources

The dashboard is powered by **4 CSV data files**, each pulled from a different log source:

| # | Data File | What It Contains | Tenant | Collection Method |
|---|-----------|-----------------|--------|-------------------|
| 1 | **Runbook Task Summary** | Execution tenant runbook task runs (task name, status, start/end times, avg runtime) | Execution | **SQL Pipeline** (queries telemetry archive) |
| 2 | **Publish Time Log** | Final publish timestamps + factory run mode (Daily YTD, FM Close, Restatement) | Execution | **Runbook** (KQL query against Log Analytics) |
| 3 | **Synapse Pipeline Runtime Summary** | Synapse pipeline runs (pipeline name, start/end, duration, status, attempts) | Execution | **Runbook** (KQL query against Log Analytics) |
| 4 | **Orchestration Pipeline Summary** | Orchestration pipeline runs (runbook group name, start/end times) | Orchestration | **Runbook** (KQL query against Log Analytics) |

### How Data Is Collected

```
                        Daily 9:00 AM Trigger
                               │
                               ▼
Runbook Task Logs ────► SQL Pipeline ────────► Runbook Task Summary CSV
                                                    │
Publish Times ────────► Execution Runbook ───► Publish Time CSV
                        (KQL Query)                 │
                                                    │
Synapse Pipeline Logs ► Execution Runbook ───► Synapse Runtime CSV
                        (KQL Query)                 │
                                                    │
Orchestration Logs ───► Orchestration Runbook ► Orchestration Summary CSV
                        (KQL Query)                 │
                                                    ▼
                                          PowerShell Script
                                                    │
                                                    ▼
                                          HTML Dashboard
                                                    │
                                                    ▼
                                     Azure Storage Account
                                      (Static Website)
```

---

## Key Concepts

### Workflow Instance ID

Every publish cycle is identified by a unique workflow ID. This ID links data across all four data files, allowing us to reconstruct the full timeline for a single publish run.

### Factory Run Mode

Each publish run operates in one of three modes:

| Run Mode | Description |
|----------|-------------|
| **Daily YTD** | Standard daily year-to-date publish |
| **FM Close** | Fiscal month close publish |
| **Restatement** | Ad-hoc restatement publish |

### Upstream Data Availability

"Upstream data available" is determined by the **later** of two key upstream restore/copy task completions in the execution tenant. This timestamp is critical for SLA analysis — if upstream data arrived late, downstream publish will also be late.

### SLA Tracking

Two key publish completion milestones are tracked:

| Metric | How It's Determined |
|--------|---------------------|
| **Primary Publish Time** | Completion of the transaction restore to pre-publish servers (falls back to overall runbook end) |
| **Final Publish Time** | From the publish time log (falls back to max Synapse pipeline end time) |

---

## Dashboard Output

The tool generates an **interactive HTML report** hosted on an Azure Storage Account static website. It includes:

1. **Trend Charts** — Publish times, upstream availability, and end-to-end durations over time
2. **Gantt Charts** — Per-workflow timeline showing all orchestration tasks, execution runbook tasks, and Synapse pipelines with their start/end times
3. **SLA Analysis** — Side-by-side comparison of upstream data availability vs. publish completion
4. **Executive Summary** — Per-workflow end-to-end duration, breakdown by phase (Orchestration / Runbook / Synapse), and % deviation from average

### Report Tabs

| Tab | Filter | Max Workflows Shown |
|-----|--------|---------------------|
| Daily YTD | Daily year-to-date runs | 30 |
| FM Close | Fiscal month close runs | 15 |
| Restatement | Restatement runs | 15 |

---

## Pipeline Flow (Simplified)

```
1. Orchestration KickOff
       │
       ▼
2. Sync Metadata (Shards → Factory → Mart → Perspectives)
       │
       ▼
3. Data Copy (Domains, Inventory, License Master, GL, MR, Geo)
       │
       ▼
4. Execution Runbooks:
   a. Start VMs
   b. Shard Redistribution & DB Reset
   c. Upstream Data Restore Triggers (upstream data arrives here)
   d. Prerequisite Extracts
   e. Publish to Perspectives & Domains
   f. Backup & Distribution
   g. Restore Transactions to Publish Servers (Primary SLA hit)
   h. Stop VMs
       │
       ▼
5. Synapse Pipelines:
   a. Inventory Pipeline
   b. Domains DB Pipeline
   c. Publish & Distribute Perspectives
   d. Publish & Distribute Domains
   e. Master Pipeline
   f. GL/Geo/MR Pipeline
       │
       ▼
6. Final Publish Complete (Final SLA hit)
       │
       ▼
7. Track Completion
```

---

## Technology Stack

| Category | Technologies |
|----------|-------------|
| Cloud | Azure Automation (Runbooks), Azure Synapse Analytics, Azure Log Analytics (KQL), Azure Storage (Static Website) |
| Data | SQL Server, CSV processing, Pipeline telemetry |
| Scripting | PowerShell |
| Visualization | Chart.js, HTML/CSS/JavaScript, Gantt charts |
| Concepts | SLA monitoring, Workflow orchestration, Root-cause analysis, ETL pipeline observability |

---

## SLA Root-Cause Analysis — How To Use

When an SLA is missed for a given day:

1. **Open the dashboard** (hosted on the static website) and navigate to the correct run mode tab
2. **Find the workflow** for that date using the Workflow ID
3. **Check "Upstream Data Available" time** — if it's later than expected, the delay is upstream
4. **Check the Gantt chart** — identify which specific task(s) ran longer than their average (red dashed markers show the average)
5. **Check the SLA chart** — compare Primary Publish and Final Publish times against the SLA target
6. **Report with evidence** — *"On [date], upstream data was not available until [time]. The normal availability is [time]. This caused a [X]-hour delay in the publish, resulting in the SLA miss."*
