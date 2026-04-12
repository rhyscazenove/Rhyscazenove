---
draft: true
layout: post
title: "How the Jira Catchup Reporter Works"
date: 2026-04-09
categories: [engineering, automation]
tags: [python, jira, flask, caching, reporting, websockets]
description: "A Python tool that pulls data from Jira's API v3, applies multi-tier caching, and generates executive-ready HTML reports with real-time progress updates."
---

# How the Jira Catchup Reporter Works

The DIGIDEV Jira project tracks work across multiple product teams at the Natural History Museum. Every week, stakeholders need the same picture: which issues moved, who got assigned, what went red. Assembling this from Jira's interface takes hours. The filters are slow, the board views don't aggregate across teams, and exporting to a spreadsheet loses the changelog data that tells you *when* things changed.

I built a tool to automate it. The Jira Catchup Reporter pulls data from Jira's REST API v3, caches it with age-based TTLs, runs it through 11 analysis functions, and generates styled HTML reports. It works as both a CLI tool and a Flask web app, deployed to Fly.io.

{% include svg/jira-catchup-reporter/data-pipeline.svg %}

---

## Forcing API v3 on a v2 Library

Jira's REST API has two versions in play. The `jira` Python library defaults to v2. I needed v3 for the Atlassian Document Format (rich text fields) and better custom field support on Cloud instances. Staying on v2 would have meant parsing raw wiki markup and working around gaps in custom field responses.

I forced v3 on the library:

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

This works. The library fetches v3 data. But the library's object model was built for v2 responses. It creates `Issue` objects with nested attribute access (`issue.fields.status.name`, `issue.fields.assignee.displayName`), and v3 JSON doesn't map onto those objects. Every downstream function in the codebase expects the v2-style attribute chain.

Forcing v3 gave me the data I needed, but broke the interface every consumer relied on.

---

## The IssueWrapper Pattern

Every function in the codebase expected `issue.fields.status.name`. The v3 JSON had the data, but not in that shape. I had two options: rewrite every consumer to work with raw v3 JSON, or create wrapper classes that present v3 JSON through the v2 interface.

Rewriting every consumer would have been cleaner long-term but would have touched dozens of functions across the data processor, report generators, and attention item logic. I chose wrappers:

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

                status_data = data.get('status', {})
                self.status = type('Status', (), {
                    'name': status_data.get('name', 'Unknown')
                })()

                issuetype_data = data.get('issuetype', {})
                self.issuetype = type('IssueType', (), {
                    'name': issuetype_data.get('name', 'Unknown')
                })()

                self._fields_data = data

        return FieldsWrapper(fields_data)
```

The `type('Status', (), {'name': ...})()` idiom creates an anonymous class with the attributes I needed, then instantiates it. This gives `issue.fields.status.name` without defining a full class hierarchy. A similar `ChangelogWrapper` handles changelog entries, wrapping each history item into objects with `.field`, `.fromString`, and `.toString` attributes.

If I were starting over, I'd skip the `jira` library and work with the REST API v3 JSON throughout. The library adds complexity without enough benefit when you're on v3, and the wrapper creates a maintenance surface where every Jira field structure change means updating both the wrapper and the consumers. But at the time, the wrapper was the fastest path to a working tool without rewriting the whole codebase.

The wrapper handled the structural mismatch. Custom fields were a separate problem.

---

## Jira's Custom Field Chaos

Custom fields in Jira return data in different formats depending on the field type, Jira version, and API version. The same field might come back as a `CustomFieldOption` object with a `.value` attribute, a JSON string, a Python dict literal string, a plain string, a list of any of the above, or `None`.

I could assume one format and let the tool break when Jira returns something else. Or I could parse defensively, handling every format I'd seen in production:

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
        return [devs.value]
    elif isinstance(devs, str):
        return [d.strip() for d in devs.split(',')]
```

The RAG status field gets the same treatment. It might arrive as `"3 GREEN Green"`, `{"value": "GREEN"}`, or a `CustomFieldOption` object. The parser normalises all of these to `RED`, `AMBER`, `GREEN`, or `UNKNOWN`.

To avoid running this expensive parsing 11 times (once per analysis function), the `DataProcessor` pre-processes all issues in a single pass. Extracted values are cached on each issue dictionary under `_processed_*` keys. The downstream grouping functions read from those cached values.

The data processor itself runs 11 analysis functions against the pre-processed issues: summary metrics, grouping by team and status, status change analysis (reconstructing what status each issue had at the start of the reporting period by walking the changelog backwards), day-by-day timelines, and attention items. The attention items surface five categories: RED RAG status, blocked issues, unassigned issues, stale issues (no updates in 7+ days), and high comment activity (5+ comments, which often indicates contention).

{% include svg/jira-catchup-reporter/attention-items.svg %}

The Natural History Museum's financial year starts in April (standard UK public sector), so quarter mapping uses `Q1 = [4, 5, 6]` through `Q4 = [1, 2, 3]`. This feeds the CLI's `--preset lastquarter` option and the web UI's date range presets.

This pre-processing and analysis pipeline produces a single dictionary that all report generators consume. But generating that dictionary hits the Jira API hundreds of times for a quarterly report, which created a caching problem.

---

## Caching Data That Ages at Different Rates

Generating a quarterly report hits the Jira API hundreds of times across paginated searches, changelog expansions, and comment fetches. Re-running the same report with a different template, or iterating on the output format during development, shouldn't re-fetch everything.

I could use a single cache TTL. But Jira data doesn't age uniformly. An issue updated today might change again in the next hour. An issue that last changed two weeks ago won't. A single TTL either caches too aggressively (serving stale data for recent issues) or too conservatively (re-fetching frozen historical data).

{% include svg/jira-catchup-reporter/cache-tiers.svg %}

I chose age-based tiering:

| Data Age | Cache TTL | Rationale |
|----------|-----------|-----------|
| Today | Skipped | Still actively changing |
| < 2 days | 1 hour | Might get late updates |
| 2-7 days | 1 day | Unlikely to change much |
| 7+ days | 30 days | Historical, frozen |
| Custom fields | 7 days | Schema-level, rarely changes |
| Comments | 3 hours | Special handling for recent activity |

The core logic:

```python
if days_ago >= 7:
    return timedelta(days=30)   # Cache for 30 days
elif days_ago >= 2:
    return timedelta(days=1)    # Cache for 1 day
else:
    return timedelta(hours=1)   # Cache for 1 hour
```

Each API call's parameters are hashed with MD5 to produce a cache key. The response is serialized with pickle and stored in `cache/api/`. Before making a call, the client checks whether a valid cache file exists and whether its TTL has expired. If the JQL query targets today's date range, caching is bypassed because the data is guaranteed to be incomplete.

The cache duration is also configurable via environment variable (`JIRA_CACHE_DURATION`) with human-readable units: `"30m"`, `"2h"`, `"1d"`, `"1w"`. This overrides the automatic tiering when you want explicit control.

Pickle files work for a single-instance deployment on Fly.io. If the tool needed horizontal scaling, Redis would be the next step; the cache key structure (MD5 of request parameters) would transfer directly.

With caching in place, iterating on the output side became fast. That mattered, because the tool needed to serve different audiences.

---

## Three Contexts, One Pipeline

{% include svg/jira-catchup-reporter/three-entry-points.svg %}

Developers want CLI output for automation and cron jobs. Project managers want a web UI with dropdowns and progress bars. Production needs authentication. Building three separate tools would mean three copies of the Jira client, data processor, and report generation logic.

I shared the pipeline and varied only the interface layer.

The CLI (`main.py`) uses Click for argument parsing and Rich for terminal output. It supports presets (`--preset lastmonth`), explicit date ranges, team and RAG filters, and multiple output formats. Rich provides progress spinners during API calls and formatted summary tables after generation.

The web UI (`web_app.py`) is a Flask app with SocketIO for real-time progress. Report generation runs in a background thread. Progress updates go to a SocketIO room keyed by session ID, so multiple users can generate reports without cross-talk. The completed HTML report comes back through the WebSocket (with a 100MB buffer configured for large reports).

Production (`app.py` + Dockerfile + `fly.toml`) deploys to Fly.io in the London region. Authentication is toggled by environment, checking for the `FLY_APP_NAME` variable:

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

The decorator is defined at module level based on the environment check, so there's no runtime overhead per request. Same codebase, no login friction during development, Flask-Login in production.

The tool produces five output formats: Markdown (for pasting into Confluence or Slack), JSON, CSV via Pandas, and two HTML templates. The "Professional" template focuses on clean typography. The "Priority" template puts RAG status and attention items front and centre for executives. Both HTML generators produce self-contained files with inline CSS and no external dependencies. You can email the file or open it offline and it renders identically.

If I were building the HTML reports again, I'd use Jinja2 templates with a shared base. The two generators diverged from a common starting point, and the duplication shows.

---

The Jira Catchup Reporter started as a script to save time on Monday morning standups and grew into a tool that project managers use weekly. Most of the engineering was in the gaps between systems: bridging API versions, normalising Jira's inconsistent data formats, and tuning cache invalidation for data that ages at different rates.
