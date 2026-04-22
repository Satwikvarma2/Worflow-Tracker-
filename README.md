# Workflow Monitoring & SLA Analytics Dashboard

## Project Overview

This project is a **workflow automation and monitoring dashboard** built to track the end-to-end publish lifecycle across multiple systems.

The primary goal is to provide visibility into **when publishes complete** and **when upstream data becomes available**, enabling the team to quickly diagnose SLA misses with evidence — e.g.,
*"Upstream data was not available until X time, which is why the SLA was missed."*

---

## Problem Statement

We have a large number of pipelines, workflows, and systems across distributed environments. When an SLA is missed, there is no single view to answer:

* **When did the upstream data become available?**
* **When did the publish actually complete?**
* **Which step in the pipeline caused the delay?**

This dashboard consolidates all of that information into one place, making SLA root-cause analysis straightforward.

---

## Architecture

### Layers

| Layer                   | Purpose                                      |
| ----------------------- | -------------------------------------------- |
| **Orchestration Layer** | Triggers workflows and coordinates execution |
| **Execution Layer**     | Runs pipelines, jobs, and processing tasks   |
| **Monitoring Layer**    | Collects logs and telemetry for analysis     |

---

## Systems & Components

```
┌──────────────────────────────────────────────┐
│           Orchestration Layer                │
│  - Workflow Trigger                         │
│  - Task Coordination                        │
│  - Execution Tracking                       │
└─────────────────────┬────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────┐
│             Execution Layer                  │
│  - Data Pipelines                           │
│  - Batch Processing Jobs                    │
│  - Data Transformation Tasks                │
│  - Data Distribution                        │
└─────────────────────┬────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────┐
│             Monitoring Layer                │
│  - Pipeline Logs                            │
│  - Execution Metrics                        │
│  - Publish Timestamps                       │
│  - Telemetry Storage                        │
└──────────────────────────────────────────────┘
```

---

## Data Sources

The dashboard is powered by **4 structured datasets**, each pulled from a different log source:

| # | Data File                         | What It Contains                                                              |
| - | --------------------------------- | ----------------------------------------------------------------------------- |
| 1 | **Workflow_Run_Summary.csv**      | Workflow task execution details (task name, status, start/end times, runtime) |
| 2 | **Publish_Timestamps.csv**        | Final publish timestamps and run modes                                        |
| 3 | **Pipeline_Runtime_Summary.csv**  | Pipeline execution details (pipeline name, start/end, duration, status)       |
| 4 | **Orchestration_Run_Summary.csv** | Orchestration workflow execution logs                                         |

---

## How Data Is Collected

```
Workflow Logs ──► Extraction Process ──► Workflow_Run_Summary.csv
                          │
Publish Events ──► Extraction Process ──► Publish_Timestamps.csv
                          │
Pipeline Runs ──► Extraction Process ──► Pipeline_Runtime_Summary.csv
                          │
Orchestration Logs ──► Extraction Process ──► Orchestration_Run_Summary.csv
                          │
                          ▼
                    Dashboard / HTML Report
```

---

## Key Concepts

### WorkflowInstanceId

Every execution cycle is identified by a unique **WorkflowInstanceId**.
This ID links data across all datasets, allowing reconstruction of the full workflow timeline.

---

### Run Mode

| Run Mode           | Description                  |
| ------------------ | ---------------------------- |
| **Daily Run**      | Standard scheduled execution |
| **Periodic Close** | End-of-cycle processing      |
| **Ad-hoc Run**     | On-demand execution          |

---

### Upstream Data Availability

"Upstream data available" is determined by the **completion of critical dependency tasks** in the workflow.

This timestamp is crucial for SLA analysis — if upstream data arrives late, downstream processing is delayed.

---

### SLA Tracking

Two key completion metrics are tracked:

| Metric                       | Description                             |
| ---------------------------- | --------------------------------------- |
| **Primary Publish Time**     | Final workflow completion time          |
| **Pipeline Completion Time** | Completion time of underlying pipelines |

---

## Dashboard Output

The system generates an **HTML report** with:

1. **Trend Charts** — Execution times and durations over time
2. **Gantt Charts** — End-to-end workflow timelines
3. **SLA Analysis** — Upstream vs publish completion comparison
4. **Executive Summary** — Duration breakdown and performance insights

---

## Report Tabs

| Tab            | Filter              | Max Workflows |
| -------------- | ------------------- | ------------- |
| Daily Run      | Standard executions | 30            |
| Periodic Close | End-of-cycle runs   | 15            |
| Ad-hoc Run     | On-demand runs      | 15            |

---

## Pipeline Flow (Simplified)

1. Workflow Trigger Initiated
2. Data Synchronization Steps
3. Data Copy & Preparation
4. Execution Layer Processing:

   * Resource initialization
   * Data extraction
   * Transformation
   * Publishing
   * Backup and distribution
5. Pipeline Execution
6. Final Publish Completion
7. Workflow Completion Tracking

---

## Scripts

| Script                   | Purpose                                                 |
| ------------------------ | ------------------------------------------------------- |
| generate_report.ps1      | Main script to process data and generate HTML dashboard |
| visualization_script.ps1 | Generates trend charts and comparisons                  |

---

## SLA Root-Cause Analysis — How To Use

When an SLA is missed:

1. Open the dashboard
2. Select the appropriate run mode
3. Identify the workflow using WorkflowInstanceId
4. Check **Upstream Data Availability Time**
5. Analyze the Gantt chart to find delays
6. Compare actual vs expected SLA
7. Report findings with evidence

Example:

> "On [date], upstream data was available at [time], later than usual.
> This caused a delay of [X hours], leading to an SLA miss."

---

## Key Benefits

* Centralized monitoring across systems
* Faster SLA root-cause identification
* Clear visibility into workflow execution
* Data-driven decision making
* Reduced debugging time

---

## Technologies Used

* Data Pipelines / Workflow Orchestration
* Log Analytics / Monitoring Systems
* Data Processing Scripts
* HTML Dashboard Generation
* Structured Data Processing (CSV-based)
