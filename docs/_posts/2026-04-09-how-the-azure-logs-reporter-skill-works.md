---
published: false
layout: post
title: "How the Azure Logs Reporter Skill Works"
date: 2026-04-09
categories: [azure, claude-code, observability]
tags: [azure-monitor, kql, log-analytics, claude-code-skills, aks]
description: "How the azure-logs-reporter skill for Claude Code works: an automated Azure Monitor log analyst that queries, classifies, and reports on your infrastructure health."
---

# How the Azure Logs Reporter Skill Works

If you've ever spent twenty minutes clicking through Azure Portal blades to figure out why your AKS pods are throwing 502s, this skill was built for you.

The **azure-logs-reporter** is a Claude Code skill that connects to Azure Monitor via the Azure MCP server, runs a battery of KQL queries against your Log Analytics workspace, and produces a structured, severity-classified report with clickable Azure Portal links so you can verify every finding yourself.

## What It Does

In plain terms: you say "check the Azure logs" and the skill autonomously:

1. Discovers your Log Analytics workspace and what data it contains
2. Runs parallel KQL queries across tables like `ContainerLogV2`, `KubeEvents`, `AzureDiagnostics`, and Application Insights
3. Classifies findings by severity (Critical / High / Medium / Low)
4. Generates a markdown report with an executive summary, detailed findings, metrics, and actionable recommendations
5. Includes Azure Portal links for every query so you can drill into the data directly

The default analysis window is the last 24 hours, but it handles anything from "last hour" to "last 30 days" based on your request.

## Skill Structure

The skill lives in `.github/skills/azure-logs-reporter/` and is composed of:

![Skill structure: SKILL.md consults five reference documents and invokes a supporting encode-azure-url.py script](/assets/images/azure-logs-reporter/skill-structure.svg)

A supporting Python script (`encode-azure-url.py`) at the repository root handles the gzip + base64 encoding required to generate shareable Azure Portal query links.

## The Four-Phase Workflow

![Four-phase workflow: Discovery, Data Collection, Analysis, and Report Generation, with Azure MCP Server mediating phases 1-3](/assets/images/azure-logs-reporter/four-phase-workflow.svg)

The skill follows a four-phase approach. Each phase builds on the previous one, and the skill runs without asking the user questions.

### Phase 1: Discovery

Before running any analysis queries, the skill needs to know what data exists. It does this by:

1. Listing workspaces via the Azure MCP server's `monitor_workspace_list` command, which returns the workspace GUID (`customerId`)
2. Querying the Usage table to discover which log tables have data and their relative volumes

```kql
Usage
| where TimeGenerated > ago(24h)
| summarize TotalMB=sum(Quantity) by DataType
| order by TotalMB desc
```

The skill avoids using `table_type_list` because it requires extra permissions that many users don't have. The Usage table approach works with standard Log Analytics Reader access.

A key lesson baked into the skill: if resource group enumeration fails due to permissions, it falls back to using the workspace GUID directly. Log queries often work even when resource group-level access is denied.

### Phase 2: Data Collection

With the data landscape mapped, the skill fires off parallel KQL queries against the discovered tables. For a typical AKS workspace, this includes:

- HTTP 5xx errors from ingress logs in `ContainerLogV2`
- HTTP 404 counts for client-side routing issues
- stderr output by pod to catch application-level errors
- Kubernetes events summarised by reason (Unhealthy, BackOff, FailedScheduling, etc.)
- Total HTTP traffic volume for context

All queries go through the same MCP command pattern:

```json
{
  "command": "monitor_workspace_log_query",
  "parameters": {
    "subscription": "<subscription-id>",
    "resource-group": "<rg>",
    "workspace": "<workspace-guid>",
    "table": "<table-name>",
    "query": "<kql-query>",
    "hours": 24,
    "limit": 1000
  }
}
```

Running these in parallel rather than sequentially matters. A full analysis might involve 5-10 queries, and waiting for each one serially adds up fast.

### Phase 3: Analysis

With raw data collected, the skill runs deeper analytical queries depending on the type of report requested. For a general health report, this includes:

- Error summary by severity across multiple tables
- Exception frequency analysis (from `AppExceptions`)
- Failed request patterns (from `AppRequests`)
- Dependency failure rates (from `AppDependencies`)
- Activity log anomalies (from `AzureActivity`)

For targeted investigations (say, a specific pod or namespace) it filters all queries by the resource identifier and includes timeline analysis to show when issues started.

### Phase 4: Report Generation

The final output is a structured markdown report:

```markdown
# Azure Logs Report
**Period:** 2026-04-08 00:00 to 2026-04-09 00:00
**Workspace:** az-aks-logs-prod

## Executive Summary
Brief overview of findings...

## Critical Issues (Severity: High)
Issues requiring immediate attention...

## Warnings (Severity: Medium)
Issues to monitor or investigate...

## Metrics Summary
| Metric          | Value | Trend |
|-----------------|-------|-------|
| Total Errors    | 142   | +23%  |
| Failed Requests | 38    | -5%   |

## Recommendations
Prioritised action items...
```

Every finding includes an Azure Portal link so you can click through and see the raw data. Users should never have to take the report on trust alone.

## The KQL Query Library

The `references/kql-queries.md` file contains a categorised library of reusable query templates covering six domains:

| Category | What It Covers |
|----------|---------------|
| Error Analysis | Severity summaries, top exceptions, error trends, stack traces |
| Application Insights | Failed requests, slow requests, dependency failures, request volume |
| Container/AKS | Container errors, HTTP status codes, Kubernetes events, pod logs |
| Azure Activity | Failed operations, resource changes, authorisation failures |
| Performance | Response time percentiles, degradation detection, system counters |
| Security | Failed sign-ins, suspicious activity, access patterns |

Each template uses a `{hours}` placeholder that gets substituted based on the user's requested time period. The queries are designed to work reliably through the MCP server, which means they avoid complex regex patterns and the `extract()` function (which has known issues when passed through MCP).

## Azure Portal Link Generation

Azure's Log Analytics blade accepts KQL queries via URL, but the query must be:

1. gzip compressed
2. base64 encoded
3. URL-safe character escaped (`+` becomes `%2B`, `/` becomes `%2F`, `=` becomes `%3D`)

The `encode-azure-url.py` script handles this:

```python
def encode_kql_for_azure_portal(query: str) -> str:
    compressed = gzip.compress(query.encode("utf-8"))
    b64 = base64.b64encode(compressed).decode("ascii")
    return (
        b64.replace("+", "%2B")
        .replace("/", "%2F")
        .replace("=", "%3D")
    )
```

The resulting URL embeds the tenant ID, resource ID, and encoded query into a portal deep-link that opens Log Analytics with the query pre-loaded and ready to run.

## Severity Classification

The skill classifies findings into four severity levels based on clear criteria:

![Severity classification: Critical (red), High (orange), Medium (amber), Low (slate) with criteria for each level](/assets/images/azure-logs-reporter/severity-classification.svg)

Each finding is also assessed for impact across four dimensions: frequency, scope (how many users/services affected), recoverability (does the system auto-recover?), and business impact (customer-facing vs internal).

## Lessons Learned from Real Sessions

The skill includes a `session-learnings.md` reference document that captures practical lessons from actual use.

Not all stderr is errors. Many applications (ArgoCD, some Java services) write INFO-level JSON logs to stderr. The skill samples stderr logs before classifying them as errors rather than assuming all stderr output indicates problems.

Modern table names matter. AKS has moved from `ContainerLog` to `ContainerLogV2`, with different column names (`LogMessage` instead of `LogEntry`, `LogSource` instead of `LogEntrySource`). The skill checks which tables exist via the Usage table before querying.

Simple string matching beats regex through MCP. The Azure MCP server has issues with complex regex patterns. Instead of `extract(@'HTTP/1\.[01]" (\d{3})', 1, LogMessage)`, the skill uses `where LogMessage contains '" 502 '`. Simpler, more reliable, easier to debug.

Query the full requested time range upfront. If a user asks for 7 days of data, query all 168 hours. Don't query a smaller window and discover retention limits only when asked for more. The skill reports if data coverage doesn't span the full requested period.

## How the Pieces Fit Together

The skill is designed as a knowledge-augmented workflow rather than a simple prompt. Claude Code reads the `SKILL.md` file to understand the phases and approach, then uses the reference documents as needed:

- `kql-queries.md` provides the actual queries to run
- `table-schemas.md` helps interpret results and construct filters
- `azure-portal-links.md` and the Python script enable link generation
- `default-workspace.md` provides pre-configured connection details
- `session-learnings.md` prevents repeating known pitfalls

This separation means the skill can be extended. Add a new query template to `kql-queries.md` and it's available on the next run. Update `session-learnings.md` with a new gotcha and future runs avoid the same trap.

## Getting Started

To use the skill, you need:

1. The Azure MCP server configured and accessible in your Claude Code environment
2. Log Analytics Reader access to your target workspace
3. The skill files in `.github/skills/azure-logs-reporter/`

Then ask Claude Code to review your Azure logs. The skill handles workspace discovery, query execution, analysis, and report generation on its own.
