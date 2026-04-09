---
layout: post
title: "Building a Team Activity Pipeline: From Trello Cards to AI-Powered Reports"
date: 2026-04-08
categories: [engineering, automation]
tags: [python, gitlab, trello, claude-api, pipelines, reporting]
description: "How I built a nine-script pipeline that pulls deployment data from Trello and GitLab, processes it through schema-validated stages, and generates AI-powered team activity reports."
---

# Building a Team Activity Pipeline: From Trello Cards to AI-Powered Reports

Our team of around twelve developers works across dozens of projects, from the main museum website to ticketing APIs to a Wildlife Photographer of the Year microsite. Deployments are tracked on a Trello board. Code lives in GitLab. Reviews, commits, and merge requests are scattered across repositories on both a self-hosted GitLab instance and gitlab.com. Stitching all of that into a coherent picture of what happened and who contributed takes hours of manual work each month. Nobody wants to do it.

So I built a pipeline to do it instead.

What started as a single script to pull deployment cards from Trello grew into a nine-script numbered pipeline that fetches data from three APIs (Trello, GitLab on-premises, GitLab cloud), processes it through schema-validated stages, and generates AI-powered narrative reports using the Claude API. It handles monthly, quarterly, and full fiscal year reporting, producing everything from raw JSON metrics to polished HTML reports with year-over-year comparisons.

---

## The Pipeline at 10,000 Feet

The pipeline is split into nine scripts, numbered 0 through 8.1. The numbering isn't arbitrary -- it reflects execution order. Script 0 is the orchestrator. Scripts 1-3 collect data. Scripts 4-5.2 process and aggregate it. Scripts 6-8.1 generate reports.

```
                         DATA COLLECTION
                    ========================

  Trello Board                              GitLab API
  (deployment cards)                        (on-prem + cloud)
        |                                        |
        v                                        |
  [Script 1]                                     |
  Fetch Trello deployments                       |
  Extract embedded GitLab URLs                   |
        |                                        |
        v                                        v
  [Script 2] -------- fetch MR details -------> GitLab
  Analyze each MR (code changes, reviews,        |
  commits, approvals)                            |
        |                                        |
        v                                        v
  [Script 3] -------- fetch open MRs ---------> GitLab
  Track ongoing work (drafts, open MRs,
  new issues)

                       DATA PROCESSING
                    ========================

  Individual MR files        Ongoing activity
  (data/merge-requests/)     (ongoing-activity.json)
        |                           |
        v                           |
  [Script 4]                        |
  Aggregate into                    |
  deployed-work.json                |
        |                           |
        v                           v
  [Script 5] <---- combines --------+
  team-activity-report.json (master data file)
        |
        +---> [Script 5.1] Time-series metrics
        |     (daily/weekly, per-user, per-project)
        |
        +---> [Script 5.2] Project historical data
              (team-level aggregates for yearly reviews)

                      REPORT GENERATION
                    ========================

  team-activity-report.json
        |
        v
  [Script 6] AI-powered reports
  Phase 1: structured JSON (schema-validated)
  Phase 2: markdown narrative
        |
        +---> [Script 7] Developer activity reports
        |     (trends, consistency scores, accomplishments)
        |
        +---> [Script 8] User HTML reports
        |     (Jinja2 templates, YoY comparison)
        |
        +---> [Script 8.1] Project HTML reports
              (releases, quarterly journey)
```

The orchestrator (Script 0) ties it all together with a single command:

```bash
python scripts/0.batch-report.py 2025-10
```

Internally, it uses a step registry pattern that makes the pipeline easy to extend:

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

Each step is a tuple of (number, description, method). Adding a new step means adding one line. The orchestrator iterates through these, running each as a subprocess, tracking execution time, and handling errors. You can skip steps (`--skip-trello`, `--skip-ai`), resume from a specific step (`--start-from 5`), or let it power through failures (`--continue-on-error`).

---

## Phase 1 -- Data Collection: Bridging Trello and GitLab

We track deployments on a Trello board. When something ships to production, a card gets added to a "Done [Month] [Year]" list. Developers paste GitLab URLs into card descriptions, checklists, and comments, linking to the merge requests and issues that made up that deployment.

Script 1 fetches those cards and harvests the GitLab URLs using regex extraction. Three patterns cover the range of URLs developers paste:

```python
gitlab_patterns = [
    r'https?://gitlab\.com/[^\s\)]+',
    r'https?://[a-zA-Z0-9.-]+gitlab[a-zA-Z0-9.-]*\.[a-zA-Z]{2,}/[^\s\)]+',
    r'https?://[a-zA-Z0-9.-]+/[^\s\)]*/(?:merge_requests|issues|commits?)/[^\s\)]*'
]
```

The first catches gitlab.com links. The second handles self-hosted instances with "gitlab" in the hostname (like our `git-repo-2.nhm.ac.uk`). The third is a fallback for any URL containing `/merge_requests/` or `/issues/`. After extraction, cleanup logic strips trailing punctuation -- developers often paste URLs at the end of sentences, leaving a period or comma attached.

I deliberately avoided building a formal Trello-GitLab integration. Developers already paste URLs naturally. Harvesting them is zero-friction. The regex approach is slightly fuzzy, but it has a 99%+ success rate because the URL formats are predictable.

Script 1 also extracts Trello custom fields (effort estimates, team allocation) which feed into later reporting. The output is a set of GitLab URLs and deployment metadata, saved as both JSON and Markdown.

Script 2 takes those URLs and fetches comprehensive details for each merge request from the GitLab API, hitting multiple endpoints per MR:

- `/merge_requests/{iid}` -- basic MR info (title, author, dates, status)
- `/merge_requests/{iid}/diffs` -- code changes (files, additions, deletions)
- `/merge_requests/{iid}/discussions` -- review comments and threads
- `/merge_requests/{iid}/commits` -- commit history
- `/merge_requests/{iid}/approvals` -- approval status

The result is a rich JSON file per MR, stored in a centralized cache at `data/merge-requests/`. Each file contains everything: code change metrics, reviewer participation, commit authors, approval status, and timing data. This is the raw material that every downstream script consumes.

One complication: we have code on both a self-hosted GitLab instance and gitlab.com. Script 2 handles this with multi-instance token routing -- it inspects each URL, determines which GitLab instance it belongs to, and uses the appropriate API token. Two tokens, two base URLs, one unified pipeline.

Script 3 captures *ongoing* work: draft MRs, open MRs with recent activity, and newly created issues. This gives the reports a "what's in progress" section alongside "what shipped", turning a deployment log into a team activity picture.

---

## The Caching Layer: Respecting Rate Limits and Your Time

A full monthly report might hit the GitLab API several hundred times. A yearly report, several thousand. Without caching, that's slow and rude to the API.

The pipeline uses two tiers of caching.

Every API response gets saved to disk at `docs/data/api/<hostname>/`, organized by GitLab instance so self-hosted and cloud responses don't collide. Cache filenames encode the resource type, project path, and resource ID:

```
docs/data/api/git-repo-2.nhm.ac.uk/
    mrs_digital-development_nhm-www-frontend_725.json
    mr-discussions_digital-development_nhm-www-frontend_725.json
    mr-changes_digital-development_nhm-www-frontend_725.json
```

Before making any API call, the script checks for a cache file. If one exists, it loads it. If not, it fetches from the API and saves. Merged MRs don't change, so their cache is permanent.

The second tier is in-memory: project member lists get fetched once per GitLab project per session, avoiding redundant calls when analyzing multiple MRs from the same project.

A bug taught me that Script 3's cache needs to be date-range-aware. Ongoing work changes between months, so querying open MRs for October should return different results than November. The fix was to include the date range in the cache key:

```
# Before (shared across all months):
mrs_digital-development_activity.json

# After (month-specific):
mrs_digital-development_activity_2025-10_to_2025-10.json
```

Another lesson: when data comes from cache, don't rewrite the analysis files. Early versions would regenerate analysis files with a fresh `analysis_date` timestamp even when all the data was cached, making stale data look fresh. Now the pipeline checks a `was_cached` flag and skips file generation for cached data, preserving the original analysis date.

I chose file-based caching over a database on purpose. Files are portable (copy the cache directory to another machine) and inspectable (you can `cat` any cache file). The entire pipeline works offline after the initial data fetch. For a reporting tool that runs once a month, this is more than adequate.

---

## Phase 2 -- Data Processing: Schema-Driven Aggregation

With raw data collected, the processing phase transforms hundreds of individual MR files into aggregated metrics.

Script 4 reads every MR JSON file from the centralized cache, filters to the requested time period, and aggregates into `deployed-work.json`. This single file contains team-wide metrics: total MRs, unique projects and authors, code change statistics, review metrics, processing time distributions, and breakdowns by project and author.

Script 5 takes that deployed work data, combines it with the ongoing activity from Script 3, and produces the master `team-activity-report.json`, the canonical data file for programmatic consumption.

Scripts 5.1 and 5.2 fan out from there. Script 5.1 generates time-series data (daily and weekly activity metrics) for visualization, with per-user and per-project breakdowns. Script 5.2 creates project-specific historical data for yearly reviews, using team-level aggregates rather than individual metrics to respect privacy in broader reporting.

### Twelve schemas

Every handoff point in the pipeline is governed by a JSON schema. There are schemas for Trello deployments, individual MR analyses, deployed work summaries, ongoing activity, team activity reports, and all three AI report types (concise, detailed, hybrid), each with variants for monthly versus quarterly/yearly periods.

Schema validation catches data corruption early. When Script 4 produces a `deployed-work.json` that doesn't match the schema, I know immediately, not three steps later when the AI report comes out garbled. Each script is independently testable: feed it valid input, check that the output validates.

### Classifying merge requests

One interesting heuristic in the processing phase is MR size categorization. Rather than just counting MRs, the pipeline classifies each one into categories like "1-liner", "bug fix", "small feature", "new feature", "major work", or "refactoring". The classification uses a composite score across multiple factors:

```python
# Calculate complexity score
files_score = min(mr.files_changed / 5, 4)   # 0-4 scale
lines_score = min(total_lines / 500, 4)       # 0-4 scale
commits_score = min(mr.total_commits / 3, 4)  # 0-4 scale
```

Before scoring, it checks for special cases: MRs with "release" in the title are release rollups, single-file changes under 5 lines are 1-liners, and changes dominated by dependency files (package.json, requirements.txt) are package updates. MRs where deletions exceed additions get flagged as refactoring.

This classification adds texture to reports. "The team shipped 47 MRs" is less useful than "the team shipped 8 new features, 15 bug fixes, 12 small improvements, and 12 package updates".

---

## Token Optimization: Feeding Data to LLMs Efficiently

A year's worth of team activity data can be thousands of lines of JSON, which means thousands of tokens. Tokens cost money, and large prompts push against context window limits.

The pipeline uses TOON (Token-Oriented Object Notation) to compress data before sending it to the LLM. TOON is a compact format that achieves 30-60% token reduction compared to JSON while maintaining lossless data representation.

Arrays of objects, which are everywhere in this data, contain massive redundancy in JSON because every object repeats all its key names. TOON eliminates this with tabular encoding:

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

The encoder handles this with a dedicated function for object arrays:

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

The implementation is a custom Python encoder because the upstream TOON package's encoder wasn't ready when I needed it. The spec is simple enough that writing the encoder took less time than waiting. A `ToonHandler` wrapper class falls back to regular JSON if anything goes wrong, so the pipeline never breaks because of format conversion.

For a yearly report with 500+ MRs and a dozen team members, the token savings are meaningful -- often reducing prompt size from 15,000 tokens to under 8,000. That's real money at scale and leaves more room in the context window for the LLM to reason.

---

## Phase 3 -- AI Report Generation: Two-Phase Prompting

Script 6 takes the aggregated team data and uses the Claude API to generate activity reports. It uses a two-phase approach that turned out to be far more robust than the single-prompt version I started with.

Phase 1 generates structured JSON. The prompt includes the raw team data, a report template, and a JSON schema. Claude produces a structured analysis (metrics, highlights, trends, recommendations) that validates against the schema. If validation fails, you know exactly what went wrong.

Phase 2 generates a markdown narrative. It takes the Phase 1 JSON (not the raw data) and a report template, and produces a human-readable report. The narrative references the structured analysis without needing to re-derive it.

```
Phase 1:
  team-activity-report.json
  + report template
  + JSON schema
  ──> Claude API ──> validated JSON analysis

Phase 2:
  Phase 1 JSON output
  + report template
  ──> Claude API ──> markdown narrative
```

Two phases because the Phase 1 JSON is inspectable. When the markdown report says "review participation increased 15%", you can trace that claim back to structured data. The JSON also feeds into other tools: Script 8 uses it for HTML report generation, and dashboards could consume it directly.

Splitting the phases also produces cleaner output. Asking an LLM to generate both structured data and flowing narrative in a single prompt leads to JSON blocks embedded in prose, missing fields, and inconsistent formatting. Separating the concerns avoids this.

### Three report types

The pipeline supports three report flavours. Concise is executive-focused: key metrics and trends in 2-3 pages. Detailed covers every metric, project, and contributor across 8-10 pages. Hybrid sits between them: a short executive summary with detailed trend analysis. Each has its own prompt template and JSON schema.

### The prompt template system

Templates use YAML frontmatter for metadata and a placeholder system for dynamic content:

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

At generation time, Script 6 loads the template, substitutes placeholders with actual values, injects team context (member names, project list, conventions), and appends the TOON-compressed data. The result is a self-contained prompt that gives the LLM everything it needs.

A key detail: the script always loads **three previous periods** alongside the current one. Monthly reports get the prior three months. Quarterly reports get the prior three quarters. This gives the LLM historical context for trend analysis without manual configuration.

---

## Individual Reports: From Team Data to Personal Narratives

The team-level data fans out into individual reports.

Script 7 generates developer activity reports across arbitrary date ranges. Give it a username and a start/end month, and it extracts that person's metrics from every monthly report in the range. It calculates trends (increasing, stable, or decreasing activity), a consistency score (0-100, based on variance between months), and identifies accomplishments using rule-based heuristics: high MR volume, large code contributions, strong review participation, multi-project breadth.

Script 8 produces HTML reports for individual team members using Jinja2 templates, styled and structured for review meetings, with charts of quarterly activity and MR sizing breakdowns. An optional AI layer adds executive summaries and personalised recommendations. The `--comparison-period` flag enables year-over-year analysis: "you shipped 23% more MRs this year, but your review engagement dropped".

Script 8.1 does the same for projects. Give it a project name and time period, and it generates a report covering merge request history, team composition, technology profile (file type distribution), review quality metrics, and processing efficiency. For yearly reports, it includes a "quarterly journey" narrative showing how the project evolved. An optional `--fetch-releases` flag pulls release and tag data from GitLab to correlate deployments with formal releases.

### The fiscal year problem

One complication that permeates the entire pipeline: the Natural History Museum uses a non-standard fiscal year. Q1 is April-June, not January-March. This means every date calculation, every quarter mapping, every "previous period" lookup needs to account for it. The quarterly boundaries are:

| Quarter | Months |
|---------|--------|
| Q1 | April - June |
| Q2 | July - September |
| Q3 | October - December |
| Q4 | January - March |

This sounds minor until you're implementing "expand a year into its 12 months" and realize that `2025` means April 2025 through March 2026, not January through December. An early bug in the orchestrator expanded `2025` as just "April 2025", producing reports with a fraction of the expected data. The fiscal year logic now lives in shared utility functions, but the lesson is clear: if your organisation uses a non-standard fiscal calendar, centralise that logic from day one.

---

## The Orchestrator: Making Nine Scripts Feel Like One Command

Script 0 is the glue. It turns a nine-step pipeline into a single command:

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

Each step runs as a subprocess via `subprocess.run()`. I chose this over importing scripts as modules. Subprocesses are slower (Python interpreter startup per step) and coupling is looser (data passes through files, not function calls). But a crash in Script 3 doesn't corrupt Script 2's state. Any script works standalone, so you can re-run just Script 6 to regenerate the AI report without re-collecting data. And each step's output is a file on disk that you can inspect, hand-edit, and re-run downstream from.

The `--continue-on-error` flag is essential for production use. When generating yearly reports (12 months of data), occasional API failures or missing Trello cards shouldn't abort the entire run. The orchestrator logs the failure, continues to the next step, and reports a summary at the end.

Period expansion is where the orchestrator earns its keep. Passing `2025` doesn't run the pipeline once for the year -- it expands into 12 monthly runs (April 2025 through March 2026), each processing independently. This means the same scripts that handle a single month seamlessly scale to quarterly and yearly reporting without modification.

---

## Lessons Learned

The numbered-script convention works well. Execution order is obvious, files sort naturally in a directory listing, and new team members understand the flow immediately. The downside: naming gets awkward when you insert steps between existing ones. Scripts 5.1 and 5.2 are fine, but if I needed something between 5.1 and 5.2, I'd be reaching for 5.1.5.

File-based caching is underrated for a pipeline that runs monthly. No database to manage, no cache invalidation logic beyond "delete the file". The entire cache is a directory you can zip, move, or inspect.

Writing 12 JSON schemas felt tedious at the time. They've since caught bugs that would have been invisible, like subtle data shape changes that silently corrupted downstream reports. Schemas are contracts between pipeline stages, and they're documentation that runs.

The two-phase AI generation (structured JSON, then narrative) produces better results from both phases. The LLM doesn't have to simultaneously organise data and write prose. The intermediate JSON also gives you an audit trail for every claim in the narrative.

The NHM fiscal year (Q1 = April) touched every script. Early on, each script had its own quarter-to-month mapping. Bugs were inevitable. Moving it all into shared utility functions (`DateParser`, `parse_month_year`) eliminated an entire class of errors.

If I were starting over, I'd consider a DAG runner like Prefect or Dagster instead of numbered scripts. I've essentially built a minimal task runner with error handling, retry logic, and dependency tracking, which is what DAG runners do out of the box, with better observability and parallelism.

---

## Portability

The specific tools matter less than the shape. If your team uses Jira instead of Trello, the data collection scripts change but the processing and reporting layers don't. GitHub instead of GitLab, same thing: different API, same pipeline structure.

The AI layer is optional. The pipeline produced useful reports for months before I added Script 6. Scripts 1-5 generate clean, schema-validated JSON that works on its own for dashboards, spreadsheets, or manual analysis. If you do add AI-generated narratives, the two-phase approach (structured data first, narrative second) saves you from debugging hallucinated statistics buried in flowing prose.

The full pipeline runs in under ten minutes for a monthly report. One command, three APIs, nine scripts, and a report that used to take half a day to assemble by hand.
