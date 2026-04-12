---
layout: post
draft: true
title: "Building a Team Activity Pipeline: From Trello Cards to AI-Powered Reports"
date: 2026-04-08
categories: [engineering, automation]
tags: [python, gitlab, trello, claude-api, pipelines, reporting]
description: "How I built a nine-script pipeline that pulls deployment data from Trello and GitLab, processes it through schema-validated stages, and generates AI-powered team activity reports."
---

# Building a Team Activity Pipeline: From Trello Cards to AI-Powered Reports

Twelve developers. Dozens of projects, from the main museum website to ticketing APIs to a Wildlife Photographer of the Year microsite. Deployments tracked on a Trello board. Code in GitLab, split across a self-hosted instance and gitlab.com. Reviews, commits, and merge requests scattered across repositories on both.

Every month, someone stitches all of that into a coherent picture of what happened and who contributed. It takes half a day. Nobody wants to do it.

The obvious fix is automation. But reporting pipelines become their own maintenance problem: a monolith script that breaks in opaque ways, a database-backed service that needs its own monitoring, a DAG runner that requires more configuration than the reporting logic itself. For something that runs twelve times a year, the automation itself becomes the thing you dread maintaining.

I wanted something different. A pipeline where every intermediate artifact is a file you can inspect, edit, and re-run from. Nine scripts. One command. No database.

---

## Nine Scripts, One Command

The pipeline splits into nine numbered scripts, 0 through 8.1. The numbering reflects execution order. Script 0 orchestrates. Scripts 1-3 collect data from Trello and two GitLab instances. Scripts 4-5.2 process and aggregate. Scripts 6-8.1 generate reports, including AI-powered narratives, per-developer activity summaries, and project-level HTML reports for review meetings.

{% include svg/deployment-analysis-pipeline/pipeline-overview.svg %}

A single command runs everything:

```bash
# Monthly report
python scripts/0.batch-report.py 2025-10

# Quarterly (Q1 = April-June, NHM fiscal year)
python scripts/0.batch-report.py 2025-Q1

# Full fiscal year (expands to 12 monthly runs)
python scripts/0.batch-report.py 2025

# Range of months
python scripts/0.batch-report.py 2025-06 2025-10

# With options
python scripts/0.batch-report.py 2025-10 --report-type hybrid --continue-on-error
```

The orchestrator uses a step registry:

```python
steps = [
    (1, "Fetch Trello deployments", self.step_1_fetch_trello),
    (2, "Analyze GitLab MRs", self.step_2_analyze_mrs),
    (3, "Generate deployed work summary", self.step_3_deployed_work_summary),
    (4, "Capture ongoing activity", self.step_4_ongoing_activity),
    (5, "Generate team activity report", self.step_5_team_activity_report),
    (5.1, "Generate time-based metrics", self.step_5_1_time_based_metrics),
    (5.2, "Generate project data files", self.step_5_2_project_data),
    (6, "Generate AI reports", self.step_6_ai_reports),
]
```

Each step is a tuple of (number, description, method). Adding a new step means adding one line. The orchestrator runs each as a subprocess, tracks execution time, and handles errors. You can skip steps (`--skip-trello`, `--skip-ai`), resume from a specific step (`--start-from 5`), or power through failures (`--continue-on-error`).

Subprocesses instead of module imports was a deliberate choice. They're slower (Python interpreter startup per step) and the coupling is looser (data passes through files, not function calls). That looseness is the point. A crash in Script 3 doesn't corrupt Script 2's state. Any script works standalone. You can re-run just Script 6 to regenerate the AI report without re-collecting data. And each step's output sits on disk, where you can open it, inspect it, hand-edit it if needed, and re-run downstream.

The same scripts that handle a single month scale to quarterly and yearly reporting without modification. Pass `2025-Q1` and the orchestrator expands it into its constituent months. Pass `2025` and it becomes twelve monthly runs (April 2025 through March 2026, aligned to the Natural History Museum's non-standard fiscal year). The `--continue-on-error` flag keeps the run going when one month's data has an API hiccup.

---

## Harvesting Data from Three APIs

Deployments live on a Trello board. When something ships to production, a card gets added to a "Done [Month] [Year]" list. Developers paste GitLab URLs into card descriptions, checklists, and comments, linking to the merge requests and issues that made up that deployment.

Script 1 fetches those cards and harvests the GitLab URLs using regex extraction. Three patterns cover the range of URLs developers paste:

```python
gitlab_patterns = [
    r'https?://gitlab\.com/[^\s\)]+',
    r'https?://[a-zA-Z0-9.-]+gitlab[a-zA-Z0-9.-]*\.[a-zA-Z]{2,}/[^\s\)]+',
    r'https?://[a-zA-Z0-9.-]+/[^\s\)]*/(?:merge_requests|issues|commits?)/[^\s\)]*'
]
```

The first catches gitlab.com links. The second handles self-hosted instances with "gitlab" in the hostname (like our `git-repo-2.nhm.ac.uk`). The third is a fallback for any URL containing `/merge_requests/` or `/issues/`. Cleanup logic strips trailing punctuation, because developers paste URLs at the end of sentences.

I deliberately avoided building a formal Trello-GitLab integration. Developers already paste URLs into cards. Harvesting them is zero-friction. The regex approach is fuzzy, but it has a 99%+ success rate because the URL formats are predictable.

Script 1 also extracts Trello custom fields (effort estimates, team allocation) that feed into later reporting. The output is a set of GitLab URLs and deployment metadata, saved as JSON and Markdown.

Script 2 takes those URLs and fetches details for each merge request from the GitLab API, hitting multiple endpoints per MR:

- `/merge_requests/{iid}` for basic info (title, author, dates, status)
- `/merge_requests/{iid}/diffs` for code changes (files, additions, deletions)
- `/merge_requests/{iid}/discussions` for review comments and threads
- `/merge_requests/{iid}/commits` for commit history
- `/merge_requests/{iid}/approvals` for approval status

The result is a JSON file per MR, stored in a centralized cache at `data/merge-requests/`. Each file contains code change metrics, reviewer participation, commit authors, approval status, and timing data. Every downstream script reads from this cache.

Code lives on both a self-hosted GitLab instance and gitlab.com. Script 2 handles this with multi-instance token routing. It inspects each URL, determines which GitLab instance it belongs to, and uses the appropriate API token. Two tokens, two base URLs, one unified pipeline.

Script 3 captures *ongoing* work: draft MRs, open MRs with recent activity, and newly created issues. This gives reports a "what's in progress" section alongside "what shipped," turning a deployment log into a team activity picture.

---

## Caching for a Pipeline That Runs Twelve Times a Year

A full monthly report hits the GitLab API several hundred times. A yearly report, several thousand. Without caching, that's slow and rude to the API.

{% include svg/deployment-analysis-pipeline/cache-tiers.svg %}

Every API response gets saved to disk at `docs/data/api/<hostname>/`, organized by GitLab instance so self-hosted and cloud responses don't collide. Cache filenames encode the resource type, project path, and resource ID:

```
docs/data/api/git-repo-2.nhm.ac.uk/
    mrs_digital-development_nhm-www-frontend_725.json
    mr-discussions_digital-development_nhm-www-frontend_725.json
    mr-changes_digital-development_nhm-www-frontend_725.json
```

Before any API call, the script checks for a cache file. If one exists, it loads it. If not, it fetches and saves. Merged MRs don't change, so their cache is permanent. A second tier of in-memory caching holds project member lists once per GitLab project per session, avoiding redundant calls when analyzing multiple MRs from the same project.

Two bugs taught me where this approach needs care.

The first: Script 3's cache needs to be date-range-aware. Ongoing work changes between months. Querying open MRs for October should return different results than November. The fix was to include the date range in the cache key:

```
# Before (shared across all months):
mrs_digital-development_activity.json

# After (month-specific):
mrs_digital-development_activity_2025-10_to_2025-10.json
```

The second: when data comes from cache, don't rewrite the analysis files. Early versions regenerated analysis files with a fresh `analysis_date` timestamp even when all the data was cached, making stale data look fresh. The pipeline now checks a `was_cached` flag and skips file generation for cached data, preserving the original analysis date.

I chose file-based caching over a database on purpose. Files are portable (copy the cache directory to another machine) and inspectable (open any cache file and read it). The entire pipeline works offline after the initial data fetch. For a reporting tool that runs once a month, a directory of JSON files requires zero infrastructure beyond the filesystem.

---

## Twelve Schemas, Zero Ambiguity

Loose coupling between scripts means data passes through files, not function calls. Without contract enforcement, a subtle change in Script 4's output corrupts Script 6's input. You find out three stages later when the AI report comes out garbled, with no obvious cause.

I put a JSON schema on every handoff point in the pipeline. There are schemas for Trello deployments, individual MR analyses, deployed work summaries, ongoing activity, team activity reports, and all three AI report types, each with variants for monthly versus quarterly/yearly periods. Twelve schemas total.

Writing them felt tedious. They've since caught bugs that would have been invisible: a field that was supposed to be an integer arriving as a string, an array that was supposed to be non-empty coming through as `[]`, subtle data shape changes after a refactor. When Script 4 produces a `deployed-work.json` that doesn't match the schema, you know immediately. Each script is independently testable: feed it valid input, check that the output validates.

Schemas are contracts between pipeline stages. They're also documentation that runs.

The processing phase itself (Scripts 4 through 5.2) transforms hundreds of individual MR files into aggregated metrics. Script 4 reads every cached MR JSON, filters to the requested time period, and produces `deployed-work.json`: total MRs, unique projects and authors, code change statistics, review metrics, and breakdowns by project and author. Script 5 combines that with ongoing activity from Script 3 into the master `team-activity-report.json`. Scripts 5.1 and 5.2 fan out into time-series data and project-specific historical data.

I also added MR size classification. Rather than counting MRs, the pipeline categorizes each one as "1-liner," "bug fix," "small feature," "new feature," "major work," or "refactoring." The classification uses a composite score:

```python
# Calculate complexity score
files_score = min(mr.files_changed / 5, 4)   # 0-4 scale
lines_score = min(total_lines / 500, 4)       # 0-4 scale
commits_score = min(mr.total_commits / 3, 4)  # 0-4 scale
```

Before scoring, it checks for special cases: MRs with "release" in the title are release rollups, single-file changes under 5 lines are 1-liners, and changes dominated by dependency files (package.json, requirements.txt) are package updates. MRs where deletions exceed additions get flagged as refactoring.

This adds texture. "The team shipped 47 MRs" is less useful than "the team shipped 8 new features, 15 bug fixes, 12 small improvements, and 12 package updates."

---

## Feeding Data to an LLM

Schema-validated JSON is useful on its own for dashboards, spreadsheets, and manual analysis. The pipeline produced useful reports for months before I added the AI layer. But structured data doesn't write itself into a narrative that a manager can skim in three minutes.

Script 6 takes the aggregated team data and uses the Claude API to generate activity reports.

### Compression

A year's worth of team activity data runs to thousands of lines of JSON, which means thousands of tokens. Tokens cost money, and large prompts push against context window limits.

{% include svg/deployment-analysis-pipeline/toon-compression.svg %}

The pipeline uses TOON (Token-Oriented Object Notation) to compress data before sending it to the LLM. Arrays of objects contain massive redundancy in JSON because every object repeats all its key names. TOON eliminates this with tabular encoding:

**JSON (repetitive keys):**
```json
[
  {"name": "alice", "mrs": 12, "reviews": 8},
  {"name": "bob", "mrs": 9, "reviews": 15},
  {"name": "charlie", "mrs": 6, "reviews": 3}
]
```

**TOON (tabular, keys declared once):**
```
[3]{name,mrs,reviews}:
  "alice",12,8
  "bob",9,15
  "charlie",6,3
```

The encoder handles this with a dedicated function:

```python
def _encode_object_array(arr: List[Dict[str, Any]], indent: int, depth: int) -> str:
    """Encode array of objects in tabular format."""
    if not arr:
        return "[]"

    keys = list(arr[0].keys())
    key_str = ",".join(keys)
    lines = [f"[{len(arr)}]{{{key_str}}}:"]

    current_indent = " " * (indent * (depth + 1))
    for obj in arr:
        values = [json.dumps(obj.get(k, None)) for k in keys]
        lines.append(f"{current_indent}{','.join(values)}")

    return "\n".join(lines)
```

The implementation is a custom Python encoder because the upstream TOON package's encoder wasn't ready when I needed it. The spec is simple enough that writing the encoder took less time than waiting. A `ToonHandler` wrapper falls back to regular JSON if anything goes wrong, so the pipeline never breaks because of format conversion. For a yearly report with 500+ MRs and a dozen team members, prompt size often drops from 15,000 tokens to under 8,000.

### Two-Phase Generation

Asking an LLM to produce both structured data and flowing narrative in a single prompt leads to JSON blocks embedded in prose, missing fields, and inconsistent formatting. Splitting the work into two phases avoids this and keeps the same principle as the rest of the pipeline: inspectable intermediate artifacts.

{% include svg/deployment-analysis-pipeline/two-phase-prompting.svg %}

Phase 1 generates structured JSON. The prompt includes the raw team data, a report template, and a JSON schema. Claude produces a structured analysis (metrics, highlights, trends, recommendations) that validates against the schema. If validation fails, you know exactly what went wrong.

Phase 2 generates a markdown narrative. It takes the Phase 1 JSON (not the raw data) and a report template, and produces a human-readable report. The narrative references the structured analysis without needing to re-derive it.

The Phase 1 JSON is inspectable. When the markdown report says "review participation increased 15%," you can trace that claim back to structured data. The JSON also feeds into other tools: Script 8 uses it for HTML report generation, and dashboards could consume it directly.

The pipeline supports three report flavours. Concise is executive-focused: key metrics and trends in 2-3 pages. Detailed covers every metric, project, and contributor across 8-10 pages. Hybrid sits between them: a short executive summary with detailed trend analysis. Each has its own prompt template and JSON schema.

Templates use YAML frontmatter for metadata and a placeholder system:

```yaml
---
version: 2.1.0
template_name: Hybrid Team Activity Report Generator
period_support:
  - monthly: "Single month with 3-month lookback"
  - quarterly: "3-month quarter with 3-quarter lookback"
  - yearly: "Full fiscal year with 3-year lookback"
schema: hybrid-activity-analysis-schema.json
placeholders:
  - PERIOD_NAME: "e.g., 'November 2025', 'Q4 2025', '2025'"
  - LOOKBACK_RANGE: "e.g., 'August-October 2025', 'Q1-Q3 2025'"
---
```

At generation time, Script 6 loads the template, substitutes placeholders, injects team context (member names, project list, conventions), and appends the TOON-compressed data. It always loads three previous periods alongside the current one. Monthly reports get the prior three months. Quarterly reports get the prior three quarters. This gives the LLM historical context for trend analysis without manual configuration.

---

## Where Real-World Messiness Lives

The hardest parts of this pipeline were institutional.

The Natural History Museum uses a non-standard fiscal year. Q1 is April-June, not January-March. This sounds minor until you're implementing "expand a year into its 12 months" and realize that `2025` means April 2025 through March 2026, not January through December.

{% include svg/deployment-analysis-pipeline/fiscal-year.svg %}

An early bug in the orchestrator expanded `2025` as just "April 2025," producing reports with a fraction of the expected data. Every date calculation, every quarter mapping, every "previous period" lookup needs to account for the fiscal calendar. Early on, each script had its own quarter-to-month mapping. Bugs were inevitable. Moving it all into shared utility functions (`DateParser`, `parse_month_year`) eliminated an entire class of errors. If your organisation uses a non-standard fiscal calendar, centralise that logic from day one.

The numbered-script convention creates its own friction. Execution order is obvious, files sort naturally, and new team members understand the flow immediately. But naming gets awkward when you insert steps. Scripts 5.1 and 5.2 are fine. Something between 5.1 and 5.2 would be 5.1.5, and that's where the convention starts straining.

The team data fans out into individual reports too. Script 7 generates per-developer activity reports across arbitrary date ranges, calculating trends, consistency scores, and accomplishments. Script 8 produces styled HTML reports for review meetings, with optional AI-powered executive summaries and year-over-year comparisons. Script 8.1 does the same for projects. These scripts all consume the same cached data and schema-validated files that the team reports use, which is one payoff of file-based intermediate artifacts: new output formats don't require re-collecting or re-processing data.

---

## What Holds It Together

The pipeline runs in under ten minutes for a monthly report. One command, three APIs, nine scripts, a report that used to take half a day.

The same design instinct runs through every decision: prefer inspectable artifacts over opaque state. Files instead of a database. Subprocesses instead of imports. Schema validation at every handoff, two-phase AI generation with an inspectable intermediate step. At every stage, you can stop the pipeline, look at what it produced, fix something by hand, and continue.

If I were starting over, I'd consider a DAG runner like Prefect or Dagster. I've built a minimal task runner with error handling, retry logic, and dependency tracking; that's what DAG runners provide out of the box, with better observability and parallelism. For something more complex than nine scripts, that trade-off would tip.

The specific tools matter less than the shape. Jira instead of Trello, GitHub instead of GitLab: the data collection scripts change, but the processing and reporting layers don't. The AI layer is optional. Scripts 1-5 generate clean, schema-validated JSON that works on its own. If you do add AI-generated narratives, the two-phase approach (structured data first, narrative second) saves you from debugging hallucinated statistics buried in flowing prose.

Nine scripts and a directory of files. Nothing more.
