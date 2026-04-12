---
published: false
layout: post
title: "How the Jira Catchup Reporter Works"
date: 2026-04-09
categories: [engineering, automation]
tags: [python, jira, flask, caching, reporting, websockets]
description: "A Python tool that pulls data from Jira's API v3, applies multi-tier caching, and generates executive-ready HTML reports with real-time progress updates."
---

# How the Jira Catchup Reporter Works

The DIGIDEV Jira project tracks work across multiple product teams at the Natural History Museum. Stakeholders regularly need to know what changed: which issues moved, who got assigned, what went red. Assembling that picture manually from Jira's interface takes hours.

The Jira Catchup Reporter is a Python tool that automates this. It pulls data from Jira's REST API v3, caches it intelligently, runs it through a processing pipeline, and generates styled HTML reports. It works as both a CLI tool (for developers and cron jobs) and a web app (for project managers), deployed to Fly.io.

## The Data Pipeline

The architecture is a four-stage pipeline:

![Data pipeline: Jira API v3 to JiraClient to Cache Layer to DataProcessor to Report Generators](/assets/images/jira-catchup-reporter/data-pipeline.svg)

Everything starts with the Jira client, which forces the `jira` Python library to use API v3 instead of its default v2:

```python
self.jira = JIRA(
    server=self.base_url,
    basic_auth=(self.email, self.api_token),
    options={
        'rest_path': 'api',
        'rest_api_version': '3',
        'agile_rest_path': 'agile',
        'agile_rest_api_version': '1.0'
    }
)
```

API v3 gives us the Atlassian Document Format for rich text fields, better custom field support on Cloud instances, and a more consistent response structure. The problem is that the `jira` library's object model doesn't understand v3 responses.

## The IssueWrapper Pattern

The `jira` Python library creates `Issue` objects with nested attribute access: `issue.fields.status.name`, `issue.fields.assignee.displayName`, and so on. When you force API v3, the library still fetches data, but the raw JSON doesn't map cleanly onto those objects. Every downstream function in the codebase expects the v2-style attribute chain.

Rather than rewrite every consumer, I created wrapper classes that present v3 JSON through the same interface:

```python
class IssueWrapper:
    def __init__(self, data):
        self.key = data['key']
        self.id = data['id']
        self.raw = data
        self.fields = self._create_fields_wrapper(data['fields'])
        self.changelog = self._create_changelog_wrapper(
            data.get('changelog', {})
        )

    def _create_fields_wrapper(self, fields_data):
        class FieldsWrapper:
            def __init__(self, data):
                self.raw = data
                self.summary = data.get('summary', '')
                self.created = data.get('created', '')
                self.updated = data.get('updated', '')

                # Create objects with .name attribute to match v2 interface
                status_data = data.get('status', {})
                self.status = type('Status', (), {
                    'name': status_data.get('name', 'Unknown')
                })()

                issuetype_data = data.get('issuetype', {})
                self.issuetype = type('IssueType', (), {
                    'name': issuetype_data.get('name', 'Unknown')
                })()

                # Store raw data for custom field access
                self._fields_data = data

        return FieldsWrapper(fields_data)
```

The `type('Status', (), {'name': ...})()` idiom creates an anonymous class inline with the attributes we need, then immediately instantiates it. This gives us `issue.fields.status.name` without defining a full class hierarchy. It keeps the wrapper self-contained and avoids a parallel set of data classes.

A similar `ChangelogWrapper` handles the changelog entries, wrapping each history item and its constituent field changes into objects with `.field`, `.fromString`, and `.toString` attributes.

## Multi-Tier Caching

![Cache tiers by data age: today is skipped, under 2 days gets 1 hour TTL, 2-7 days gets 1 day, 7+ days gets 30 days](/assets/images/jira-catchup-reporter/cache-tiers.svg)

Generating a quarterly report can hit the Jira API hundreds of times across paginated searches, changelog expansions, and comment fetches. Re-running the same report with a different template, or iterating on the output format during development, shouldn't re-fetch everything.

The caching layer uses different TTLs based on data age:

| Data Age | Cache TTL | Rationale |
|----------|-----------|-----------|
| Today | Skipped | Still actively changing |
| < 2 days | 1 hour | Might get late updates |
| 2-7 days | 1 day | Unlikely to change much |
| 7+ days | 30 days | Historical, effectively frozen |
| Custom fields | 7 days | Schema-level, rarely changes |
| Comments | 3 hours | Special handling for recent activity |

The core decision logic:

```python
if days_ago >= 7:
    return timedelta(days=30)   # Cache for 30 days
elif days_ago >= 2:
    return timedelta(days=1)    # Cache for 1 day
else:
    return timedelta(hours=1)   # Cache for 1 hour
```

Each API call's parameters are hashed with MD5 to produce a cache key. The response is serialized with pickle and stored in `cache/api/`. Before making a call, the client checks whether a valid cache file exists and whether its TTL has expired. If the JQL query targets today's date range, caching is bypassed entirely because the data is guaranteed to be incomplete.

The cache duration is also configurable via environment variable (`JIRA_CACHE_DURATION`) with human-readable units: `"30m"`, `"2h"`, `"1d"`, `"1w"`. This overrides the automatic tiering for cases where you want explicit control.

## Defensive Custom Field Parsing

If you've built a Jira integration, you know this pain. Custom fields return data in wildly inconsistent formats depending on the field type, Jira version, and API version. The same field might come back as a `CustomFieldOption` object with a `.value` attribute, a JSON string, a Python dict literal string, a plain string, a list of any of the above, or `None`.

The DigiDev-Devs field extraction handles all five paths:

```python
def _extract_digidev_devs(self, issue):
    devs = issue.get('digidev_devs', [])

    if isinstance(devs, list):
        dev_names = []
        for dev in devs:
            if hasattr(dev, 'value'):
                # CustomFieldOption object
                dev_names.append(dev.value)
            elif isinstance(dev, str):
                if dev.startswith('{') and dev.endswith('}'):
                    try:
                        dev_obj = json.loads(dev.replace("'", '"'))
                    except json.JSONDecodeError:
                        dev_obj = ast.literal_eval(dev)
                    if isinstance(dev_obj, dict) and 'value' in dev_obj:
                        dev_names.append(dev_obj['value'])
                    else:
                        dev_names.append(str(dev))
                else:
                    dev_names.append(str(dev))
            else:
                dev_names.append(str(dev))
        return dev_names
    elif hasattr(devs, 'value'):
        return [devs.value]           # Single object
    elif isinstance(devs, str):
        return [d.strip() for d in devs.split(',')]  # Comma-separated
```

The RAG status field gets similar treatment. It might arrive as `"3 GREEN Green"`, `{"value": "GREEN"}`, or a `CustomFieldOption` object. The parser normalises all of these to `RED`, `AMBER`, `GREEN`, or `UNKNOWN`.

To avoid running this expensive parsing 11 times (once per analysis function), the `DataProcessor` pre-processes all issues in a single pass. Extracted values are cached on each issue dictionary under `_processed_*` keys. The downstream grouping functions read from those cached values directly.

## NHM Corporate Quarters

The Natural History Museum uses a financial year starting in April. Standard for UK public sector, but not what most software assumes:

```python
QUARTERS = {
    'Q1': [4, 5, 6],    # Apr-Jun
    'Q2': [7, 8, 9],    # Jul-Sep
    'Q3': [10, 11, 12], # Oct-Dec
    'Q4': [1, 2, 3]     # Jan-Mar
}
```

This feeds into the `--preset lastquarter` CLI option and the web UI's date range presets. The date calculation logic maps the current month to the correct quarter boundaries, so running `--preset lastquarter` in May 2026 gives you Q4 (January-March 2026), not Q1 (January-March in calendar quarters).

## The Data Processor

The `DataProcessor` is the analytics engine. Its `process_issues()` method runs 11 analysis functions and returns a single dictionary that all report generators consume:

- Summary metrics: total issues, type distribution, status breakdown
- Grouping by team, status, developer, and RAG status
- Status change analysis, which reconstructs what status each issue had at the start of the reporting period by walking the changelog backwards, enabling statements like "3 issues moved from In Progress to Done"
- Day-by-day timeline of status changes, comments, and assignments
- Change frequency: which fields changed most often
- Developer assignment tracking: net assignment changes (assigned minus unassigned)
- Attention items: automated flags for issues needing immediate attention
- Quarter distribution mapped to NHM corporate quarters

The attention items surface five categories: RED RAG status, blocked issues, unassigned issues, stale issues (no updates in 7+ days), and high comment activity (5+ comments, often indicating contention). These appear at the top of reports so managers see the problems first.

![Five attention item categories: RED RAG, Blocked, Unassigned, Stale, and High Activity](/assets/images/jira-catchup-reporter/attention-items.svg)

## Three Entry Points, One Pipeline

![Three entry points (CLI, Web UI, Production) all converging into the same shared JiraClient, DataProcessor, ReportGenerator pipeline](/assets/images/jira-catchup-reporter/three-entry-points.svg)

The same JiraClient, DataProcessor, and ReportGenerator pipeline serves three contexts.

### CLI

`main.py` uses Click for argument parsing and Rich for terminal output. It supports presets (`--preset lastmonth`), explicit date ranges, team and RAG filters, and multiple output formats. Rich gives us progress spinners during API calls and formatted summary tables after generation. Good for automation and cron.

### Web UI

`web_app.py` is a Flask app with SocketIO for real-time progress. Report generation runs in a background thread. Progress updates are emitted to a SocketIO room keyed by session ID, so multiple users can generate reports simultaneously without seeing each other's progress. The completed HTML report is sent back through the WebSocket (with a 100MB buffer configured for large reports).

### Production

`app.py` + `Dockerfile` + `fly.toml` deploys to Fly.io in the London region. Authentication is toggled by environment, checking for the `FLY_APP_NAME` variable to determine if it's running in production:

```python
if IS_PRODUCTION:
    def production_auth_required(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if IS_PRODUCTION and not current_user.is_authenticated:
                return redirect(url_for('login'))
            return f(*args, **kwargs)
        return decorated_function
else:
    def production_auth_required(f):
        return f  # No-op in development
```

This means the same codebase runs without authentication locally (no login friction during development) and with Flask-Login in production. The decorator is defined at module level based on the environment check, so there's no runtime overhead per request.

## Report Generation

The tool produces five output formats: Markdown (structured text with tables and Jira links, for pasting into Confluence or Slack), JSON, CSV via Pandas, and two HTML templates. The "Professional" HTML template focuses on clean typography and readability. The "Priority" template puts RAG status and attention items front and centre for executives.

Both HTML generators produce self-contained files with inline CSS and no external dependencies. You can email the file or open it offline and it renders identically. The web UI lets users choose between templates via a dropdown, and the generated report displays in an iframe within the dashboard.

## What I Would Do Differently

The IssueWrapper pattern works, but it creates a maintenance surface. Every time Jira changes a field structure, both the wrapper and the consumers might need updating. If starting again, I'd skip the `jira` library entirely and work directly with the REST API v3 JSON throughout. The library adds complexity without enough benefit when you're already on v3.

The two HTML report generators share significant structure but were built separately as requirements diverged. A Jinja2 template approach with shared base templates would reduce duplication while still allowing different layouts.

Caching with pickle files works well for a single-instance deployment on Fly.io. If the tool needed horizontal scaling, Redis would be the next step. The cache key structure (MD5 of request parameters) would transfer directly.

---

The Jira Catchup Reporter started as a script to save time on Monday morning standups and grew into a tool that project managers use weekly. Most of the engineering work was in the gaps between systems: bridging API versions, normalising Jira's inconsistent data formats, and tuning cache invalidation for data that ages at different rates. Be defensive about Jira's field formats and generous with your caching. The API is slow enough that even a short cache makes a real difference to the development loop.
