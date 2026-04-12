---
draft: true
layout: post
title: "How the Azure Logs Reporter Skill Works"
date: 2026-04-09
categories: [azure, claude-code, observability]
tags: [azure-monitor, kql, log-analytics, claude-code-skills, aks]
description: "How the azure-logs-reporter skill for Claude Code works: an automated Azure Monitor log analyst that queries, classifies, and reports on your infrastructure health."
---

# How the Azure Logs Reporter Skill Works

ContainerLogV2, KubeEvents, AzureDiagnostics, Application Insights. Each in a different Azure Portal blade, each needing its own KQL query, each with its own schema. For a team running a dozen services on AKS, checking logs daily is a tax nobody budgets for. Twenty minutes clicking through portal blades to figure out why pods are throwing 502s, repeated every time something looks wrong.

I built a Claude Code skill to automate it. The azure-logs-reporter connects to Azure Monitor via the Azure MCP server, runs parallel KQL queries against your Log Analytics workspace, classifies findings by severity, and produces a markdown report with clickable Azure Portal links. You say "check the Azure logs" and it does the rest.

{% include svg/azure-logs-reporter/four-phase-workflow.svg %}

The skill follows four phases (discovery, data collection, analysis, report generation) and runs without asking the user questions. The default window is the last 24 hours, but it handles anything from "last hour" to "last 30 days."

The implementation required working around restrictive permissions, evolving table schemas, and an MCP server that chokes on certain KQL patterns.

---

## Finding Out What's in the Workspace

Before running any analysis queries, the skill needs to know what log tables exist and have data. The Azure MCP server provides a `table_type_list` command for this purpose. I tried it. It requires permissions that most users don't have. In locked-down enterprise environments (which describes most production Azure tenancies), the call fails.

I could require elevated permissions, limiting who can use the skill. Or I could find another way to discover tables.

The Usage table solved it. Every Log Analytics workspace has one, and it's readable with standard Log Analytics Reader access:

```kql
Usage
| where TimeGenerated > ago(24h)
| summarize TotalMB=sum(Quantity) by DataType
| order by TotalMB desc
```

This returns every table that received data in the last 24 hours, along with ingestion volume. The volume data turned out to be a bonus; it helps prioritise which tables to query first, since a table ingesting 500MB is more likely to contain interesting findings than one ingesting 2MB.

This decision shaped how the skill handles permissions everywhere else. If resource group enumeration fails (which it does in most restricted environments), the skill falls back to the workspace GUID. Log queries work even when resource group-level access is denied. The skill adapts rather than failing, because the first query proved that minimal permissions can get you useful data.

---

## Writing Queries That Survive MCP

The Azure MCP server passes KQL queries through to Azure Monitor, but it mangles some of them. Complex regex patterns in `extract()` break. Patterns like `extract(@'HTTP/1\.[01]" (\d{3})', 1, LogMessage)` fail or return no results.

I discovered this after an early version of the skill produced reports claiming zero HTTP errors on clusters that were throwing 502s. The queries looked correct. The results were empty. The MCP server's regex handling was stripping or mangling the patterns somewhere in transit.

I could try to debug the MCP server's regex handling, but that's not my code to fix, and a fix would break with the next server update. The alternative: rewrite the queries without regex.

I went with simple string matching. Instead of parsing status codes with regex, the skill uses `where LogMessage contains '" 502 "'`. Less precise, but it works every time through MCP. All queries use the same MCP command pattern:

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

Running queries in parallel rather than sequentially matters here. A full analysis involves 5-10 queries, and waiting for each one serially adds up.

Another query-level constraint surfaced in the same period: table schema evolution. AKS moved from `ContainerLog` to `ContainerLogV2`, changing column names in the process (`LogMessage` instead of `LogEntry`, `LogSource` instead of `LogEntrySource`). The Usage table discovery from the previous step handles this; the skill checks which tables exist before constructing queries, so it uses the right column names for the workspace's actual schema.

These constraints shaped the entire KQL query library. Every template in `references/kql-queries.md` avoids complex regex and uses `{hours}` placeholders for time ranges. The library covers six domains:

| Category | What It Covers |
|----------|---------------|
| Error Analysis | Severity summaries, top exceptions, error trends, stack traces |
| Application Insights | Failed requests, slow requests, dependency failures, request volume |
| Container/AKS | Container errors, HTTP status codes, Kubernetes events, pod logs |
| Azure Activity | Failed operations, resource changes, authorisation failures |
| Performance | Response time percentiles, degradation detection, system counters |
| Security | Failed sign-ins, suspicious activity, access patterns |

---

## Not All Stderr Is Errors

The first report I ran against a production cluster flagged 12,000 "errors" in 24 hours. The cluster was healthy. The on-call engineer confirmed nothing was wrong.

Early versions of the skill counted all stderr output as errors. This seemed reasonable; stderr is the error stream. But ArgoCD, some Java services, and several other tools write INFO-level JSON logs to stderr. Applications write structured logs to stderr and reserve stdout for unstructured output; some logging frameworks default to stderr regardless of severity.

I had two options: flag all stderr as errors and let users filter the noise themselves, or add a sampling step that classifies stderr content before counting it.

Flagging everything would have been simpler, but a report that cries wolf 12,000 times is a report nobody trusts. I added the sampling step. The skill now runs a sampling query against stderr output and checks whether the lines contain INFO or DEBUG level indicators in their JSON structure. Lines that do get excluded from error counts.

This added a query to Phase 3 (analysis), but it's the difference between a report that gets ignored and one that gets acted on. The lesson went into `session-learnings.md`, which the skill reads on every run to avoid repeating known pitfalls.

---

## Making Every Finding Verifiable

A report that says "142 errors in the last 24 hours" is useful only if you can check it. Log analysis that produces findings without evidence trains users to distrust it.

I could include the raw KQL query text and ask users to copy it, open Azure Portal, navigate to the right workspace, paste the query, and run it. That works, but the friction means nobody does it.

The alternative: generate clickable deep links that open Log Analytics with the query pre-loaded and ready to run. Azure's Log Analytics blade accepts KQL queries via URL, but the encoding is non-obvious. The query must be gzip compressed, base64 encoded, then URL-escaped.

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

The resulting URL embeds the tenant ID, resource ID, and encoded query into a portal deep-link. Click it and Log Analytics opens with the query ready to run. No copy-paste, no navigation.

Every finding in the report includes one of these links alongside a severity classification:

{% include svg/azure-logs-reporter/severity-classification.svg %}

The skill scores each finding on four dimensions: frequency, scope (how many users or services affected), recoverability (does the system auto-recover?), and business impact (customer-facing vs internal). The severity label gives you triage priority. The link gives you evidence.

The report itself follows a consistent structure:

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

Users should never have to take the report on trust alone.

---

## Keeping the Skill Extensible

Every session against a new workspace reveals something: a table I hadn't encountered, or a query that fails for a specific AKS configuration. The skill needed to absorb these lessons without requiring changes to the core workflow.

I could keep everything in one large `SKILL.md` file. Simpler to ship, easier to read in one place. But updating a query template means editing the same file that defines the workflow, and a formatting mistake in a KQL snippet could break the skill's ability to parse its own instructions.

I split the skill into separate reference files instead.

{% include svg/azure-logs-reporter/skill-structure.svg %}

Claude Code reads `SKILL.md` to understand the phases and approach, then pulls from reference documents as needed:

- `kql-queries.md` provides the query templates
- `table-schemas.md` helps interpret results and construct filters
- `azure-portal-links.md` and `encode-azure-url.py` enable link generation
- `default-workspace.md` provides pre-configured connection details
- `session-learnings.md` captures known pitfalls from real sessions

Add a query template to `kql-queries.md` and it's available on the next run. Update `session-learnings.md` with a new gotcha and future runs avoid the same trap. The skill improves without touching the core workflow.

---

## Query the Full Time Range

An early version queried the last 4 hours regardless of what the user asked for, then expanded the window if needed. When someone asked for 7 days, the skill discovered that data coverage didn't span the full range only after multiple rounds of queries.

Now the skill queries the full requested time range upfront. If someone asks for a week, it queries all 168 hours. If data coverage doesn't span the full period, the report states the gap.

---

## Getting Started

The skill needs three things: the Azure MCP server configured in your Claude Code environment, Log Analytics Reader access to your target workspace, and the skill files in `.github/skills/azure-logs-reporter/`. Then ask Claude Code to review your Azure logs. The skill handles workspace discovery, query execution, analysis, and report generation on its own.
