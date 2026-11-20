Paste each part into PowerShell, in order. Each prints a 'wrote ...' line per file.

**Part 1 of 19**

~~~~powershell
# Part 1 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W '.gitignore' @'
__pycache__/
data/
out/
events.ndjson
local/.env
dist/
'@
W 'README.md' @'
# BU AI observability layer (Hiscox RE)

Spend next to what it bought. Every AI session, CI run and QA result is tied
to one work item, so we can say "we spent X, and here is what it delivered".

Rebuilt Nov 2026 from the September Agentic Ops repo, the value view and the
October implementation brief, aligned to the PoV plan. Two changes from
before: everything joins on **our own work-item ID** (Jira keys are aliases),
and **Copilot** is captured alongside Claude Code.

```
Claude Code ─(OTel + SessionStart hook)─┐
Copilot ─────(managed OTel)─────────────┤
Own agents ──(OTel SDK)─────────────────┼─► BU OTel Collector ─► New Relic (system of record)
APIM v2 ─────(token metrics, Event Hub)─┘    (Container App)        │
Jira ────────(adapter: mints wi_ IDs)──────────────────────────────►├─► Power BI (value view)
GitHub ──────(CI, PR, QA gate events)──────────────────────────────►└─► Group LiteLLM (later: one exporter)
Langfuse: personal workbench only (local overlay)
```

## Decisions carried forward

- APIM is the only permitted route to model endpoints.
- New Relic is the operational system of record; Power BI is the leadership view.
- Langfuse stays a personal learning and eval workbench, not a production consumer.
- Evals are YAML cases in git; results go to New Relic as `EvalResult` events.
- No new hosted services until value is proven: New Relic custom events are
  the PoV's event stream.
- Own the formats (OTel, the event contract, eval cases); rent the engines.
- Per-person views are for that person; leadership sees team-level aggregates
  with suppression (hashed IDs, ≥ 3 distinct users per cell, complementary suppression).

## Layout

| Path | What |
|---|---|
| `contracts/events.md` | **The contract**: CloudEvents envelope, work-item IDs, aliases, event types |
| `obs/` | Shared core: ID minting, alias table, contract validation, New Relic sender |
| `collector/` | Collector config + APIM overlay + Langfuse (local) overlay + Dockerfile |
| `clients/claude-code/` | Managed settings, SessionStart hook (ps1 + py), identity scripts |
| `clients/copilot/` | How to capture Copilot (managed OTel, per-dev fallback, SDK) |
| `clients/apim/` | Gateway policy: Entra identity, per-user token quota (oid), token metrics, trace context |
| `adapters/jira/` | The only code that knows Jira: sync + mint IDs, Automation rule alternative |
| `adapters/github/` | CI / PR / QA gate events, Copilot usage (billing truth) |
| `newrelic/` | Dashboard NRQL, value-view export NRQL, org and price lookups |
| `valueview/value_view.py` | Power BI feed: leadership + eng + dev metrics with privacy by construction |
| `bench/` | Replay benchmark spec (Phase 7.2) and task template |
| `evals/` | Eval case format |
| `docs/` | Implementation plan, the leadership proposal, LiteLLM losses and alternatives |
| `deploy/`, `local/` | Container Apps deploy; local compose |
| `tools/` | Dev setup script; paste-block generator for the work VM |
| `tests/run_all.py` | All offline checks |

## Quick start

```bash
python tests/run_all.py                       # everything offline, no secrets
python valueview/value_view.py --selftest     # privacy properties on synthetic data
python tools/make_paste_blocks.py > paste-blocks.md   # for the work VM
```

Then follow `docs/IMPLEMENTATION.md`.

## Not verified here (do these first)

- `otelcol-contrib validate --config collector/otel-collector.yaml --config collector/otel-collector.apim.yaml`
  (the September config was validated against 0.135.0; `transform/harness`
  and the redaction patterns are new).
- Lines marked **VERIFY**: Copilot managed-settings keys, Jira search endpoint
  on your instance, Copilot usage API response shape, Claude Code
  managed-settings path on Windows, Jira Automation smart values.
- APIM `llm-*` policies against the Vertex backend.

## Cost source of truth

- Claude Code `cost.usage` is an estimate from list prices.
- APIM tokens × `model-prices.csv` (contract rates) is what finance will believe.
- Copilot: GitHub's AI credits per user (usage metrics API).
- Each `AIUsage` row carries `billing_source` so it reconciles against the right bill.
'@
W 'adapters\__init__.py' @'

'@
W 'adapters\github\__init__.py' @'

'@
~~~~

**Part 2 of 19**

~~~~powershell
# Part 2 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'adapters\github\copilot_usage.py' @'
#!/usr/bin/env python3
"""Pull GitHub Copilot per-user daily usage and emit AIUsage events
(billing_source=github). This is the billing truth for Copilot: its traffic
never touches APIM.

API: REST "Copilot usage metrics" (enterprise/org reports, API version
2026-03-10). Reports are returned as download links to NDJSON files.
VERIFY the exact response shape against your enterprise before relying on
it; the parser below is defensive and skips records it does not recognise.

Usage:
  python adapters/github/copilot_usage.py --enterprise hiscox --day 2026-11-17
  python adapters/github/copilot_usage.py --ndjson sample.ndjson --dry-run   # offline

Env: GH_TOKEN (enterprise admin / org owner token with Copilot metrics read)
"""
from __future__ import annotations

import argparse
import json
import os
import sys
import urllib.request

sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "..", "..")))

from obs.events import make_event  # noqa: E402
from obs.ids import WI_UNASSIGNED  # noqa: E402
from obs.nr import NewRelicSender  # noqa: E402

API = "https://api.github.com"
API_VERSION = "2026-03-10"


def _get(url: str, token: str | None):
    headers = {"Accept": "application/vnd.github+json", "X-GitHub-Api-Version": API_VERSION}
    if token:
        headers["Authorization"] = f"Bearer {token}"
    with urllib.request.urlopen(urllib.request.Request(url, headers=headers), timeout=60) as r:
        return r.read().decode()


def fetch_records(enterprise: str, day: str, token: str) -> list[dict]:
    meta = json.loads(_get(f"{API}/enterprises/{enterprise}/copilot/metrics/reports/users-1-day?day={day}", token))
    links = meta.get("download_links") or meta.get("links") or []
    records = []
    for link in links:
        url = link if isinstance(link, str) else link.get("url")
        if url:
            records += parse_ndjson(_get(url, None))
    return records


def parse_ndjson(text: str) -> list[dict]:
    out = []
    for line in text.splitlines():
        line = line.strip()
        if line:
            try:
                out.append(json.loads(line))
            except json.JSONDecodeError:
                continue
    return out


def to_events(records: list[dict], day: str) -> list[dict]:
    events = []
    for r in records:
        user = r.get("user_login") or r.get("user") or r.get("login")
        credits = r.get("ai_credits_used")
        if not user or credits is None:
            continue
        tokens = r.get("token_usage") or {}
        events.append(make_event("AIUsage", "/bu-ai-obs/adapters/copilot-usage", WI_UNASSIGNED, {
            "harness": "copilot",
            "gen_ai.request.model": r.get("model") or "copilot-mixed",
            "billing_source": "github",
            "enduser.id": str(user).lower(),
            "ai_credits_used": credits,
            "gen_ai.usage.input_tokens": tokens.get("prompt_tokens_sum"),
            "gen_ai.usage.output_tokens": tokens.get("output_tokens_sum"),
            "usage_date": r.get("date") or r.get("day") or day,
        }))
    return events


def main(argv=None) -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("--enterprise")
    ap.add_argument("--day", help="YYYY-MM-DD (defaults to the report's own date field)")
    ap.add_argument("--ndjson", help="parse a local NDJSON report instead of calling the API")
    ap.add_argument("--dry-run", action="store_true")
    a = ap.parse_args(argv)
    if a.ndjson:
        with open(a.ndjson, encoding="utf-8") as fh:
            records = parse_ndjson(fh.read())
    else:
        if not (a.enterprise and a.day):
            raise SystemExit("--enterprise and --day are required unless --ndjson is given")
        records = fetch_records(a.enterprise, a.day, os.environ["GH_TOKEN"])
    events = to_events(records, a.day or "")
    n = NewRelicSender(dry_run_path="events.ndjson" if a.dry_run else None).send(events)
    print(f"{len(records)} records -> {n} AIUsage events")
    return 0


if __name__ == "__main__":
    sys.exit(main())
'@
W 'adapters\github\delivery-events.yml' @'
# Copy to .github/workflows/delivery-events.yml in each squad repo.
# Emits CIRunCompleted for every finished workflow run and PRMerged on merge.
# Secrets: NEW_RELIC_ACCOUNT_ID, NEW_RELIC_LICENSE_KEY (ingest key).
# The QA gate workflow should call emit_event.py QAGatePassed / QABouncedBack
# at its own pass/fail points.
name: delivery-events

on:
  workflow_run:
    # List your CI workflows by name (workflow_run needs explicit names).
    workflows: ["CI", "QA gate"]
    types: [completed]
  pull_request:
    types: [closed]

permissions:
  contents: read

jobs:
  emit:
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_run' || github.event.pull_request.merged == true
    env:
      NEW_RELIC_ACCOUNT_ID: ${{ secrets.NEW_RELIC_ACCOUNT_ID }}
      NEW_RELIC_LICENSE_KEY: ${{ secrets.NEW_RELIC_LICENSE_KEY }}
      NEW_RELIC_REGION: eu
    steps:
      - name: Check out the observability adapters
        uses: actions/checkout@v4
        with:
          repository: REPLACE-org/bu-ai-observability
          path: obs
          sparse-checkout: |
            obs
            adapters/github

      - name: CI run finished
        if: github.event_name == 'workflow_run' && github.event.workflow_run.name != 'delivery-events'
        env:
          BRANCH: ${{ github.event.workflow_run.head_branch }}
          CONCLUSION: ${{ github.event.workflow_run.conclusion }}
          WORKFLOW: ${{ github.event.workflow_run.name }}
          RUN_ID: ${{ github.event.workflow_run.id }}
          ATTEMPT: ${{ github.event.workflow_run.run_attempt }}
        run: >
          python obs/adapters/github/emit_event.py CIRunCompleted --branch "$BRANCH"
          --set repo="$GITHUB_REPOSITORY" run_id="$RUN_ID" conclusion="$CONCLUSION"
          attempt="$ATTEMPT" workflow="$WORKFLOW"

      - name: PR merged
        if: github.event_name == 'pull_request'
        env:
          BRANCH: ${{ github.event.pull_request.head.ref }}
          PR: ${{ github.event.pull_request.number }}
          MERGED_AT: ${{ github.event.pull_request.merged_at }}
        run: >
          python obs/adapters/github/emit_event.py PRMerged --branch "$BRANCH"
          --set repo="$GITHUB_REPOSITORY" pr_number="$PR" merged_at="$MERGED_AT"
'@
~~~~

**Part 3 of 19**

~~~~powershell
# Part 3 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'adapters\github\emit_event.py' @'
#!/usr/bin/env python3
"""Emit one canonical delivery event from a GitHub Actions job.

The branch is turned into a tracker alias here; the work-item ID is resolved
downstream (CI has no alias table), so events go out with
subject=wi_unassigned and the alias in the `workitemalias` extension.
If --aliases points at a checked-out alias table, it resolves in place.

Examples (in a workflow step):
  python adapters/github/emit_event.py CIRunCompleted \
      --branch "$GITHUB_HEAD_REF" --set repo="$GITHUB_REPOSITORY" run_id="$GITHUB_RUN_ID" \
      conclusion=success attempt="$GITHUB_RUN_ATTEMPT"
  python adapters/github/emit_event.py QAGatePassed --branch "$GITHUB_HEAD_REF" --set attempt_n=1

Env: NEW_RELIC_ACCOUNT_ID, NEW_RELIC_LICENSE_KEY (repo secrets), NEW_RELIC_REGION=eu
"""
from __future__ import annotations

import argparse
import os
import sys

sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "..", "..")))

from obs.aliases import AliasTable, normalise  # noqa: E402
from obs.events import CONTRACT, make_event  # noqa: E402
from obs.ids import WI_UNASSIGNED  # noqa: E402
from obs.nr import NewRelicSender  # noqa: E402

NUMERIC = {"attempt", "attempt_n", "duration_s", "pr_number", "mutation_score", "task_size"}


def parse_sets(pairs: list[str]) -> dict:
    data = {}
    for p in pairs:
        k, _, v = p.partition("=")
        if not _:
            raise SystemExit(f"--set expects key=value, got {p!r}")
        if k in NUMERIC and v:
            try:
                v = float(v) if "." in v else int(v)
            except ValueError:
                raise SystemExit(f"{k} must be numeric, got {v!r}")
        data[k] = v
    return data


def main(argv=None) -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("type", choices=sorted(CONTRACT))
    ap.add_argument("--branch", default=os.environ.get("GITHUB_HEAD_REF") or os.environ.get("GITHUB_REF_NAME", ""))
    ap.add_argument("--aliases", help="alias table CSV, if available in the job")
    ap.add_argument("--set", nargs="*", default=[], metavar="key=value")
    ap.add_argument("--dry-run", action="store_true")
    a = ap.parse_args(argv)

    data = parse_sets(a.set)
    if a.branch and "branch" in CONTRACT[a.type][1] and "branch" not in data:
        data["branch"] = a.branch
    alias = normalise(a.branch)
    subject = AliasTable(a.aliases).resolve(alias) if (a.aliases and alias) else WI_UNASSIGNED
    ev = make_event(a.type, "/bu-ai-obs/adapters/github", subject, data, alias=alias)
    n = NewRelicSender(dry_run_path="events.ndjson" if a.dry_run else None).send([ev])
    print(f"emitted {n} {a.type} alias={alias or '-'} subject={subject}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
'@
W 'adapters\jira\__init__.py' @'

'@
W 'adapters\jira\automation-rule.md' @'
# Jira Automation rule (real-time alternative to polling)

`sync_work_items.py` polls and owns ID minting. For near-real-time status
changes without hosting anything, add a Jira Automation rule that posts
straight to New Relic. It sends the Jira key as an alias; the work-item ID is
resolved at query time through the `aliases` lookup table in New Relic, and by
the value view.

Run the poller at least daily anyway: it is what mints IDs for new issues.

## Rule

- **Trigger:** Issue transitioned (and optionally Issue created)
- **Condition:** Project = RE (and any squad filter)
- **Action:** Send web request
  - URL: `https://insights-collector.eu01.nr-data.net/v1/accounts/<ACCOUNT_ID>/events`
  - Method: `POST`
  - Headers: `Api-Key: <ingest licence key>` (store it as a secret/hidden header),
    `Content-Type: application/json`
  - Body (custom data):

```json
[{
  "eventType": "WorkItemStatusChanged",
  "work_item.id": "wi_unassigned",
  "work_item.alias": "jira:{{issue.key}}",
  "alias": "jira:{{issue.key}}",
  "from_status": "{{fieldChange.fromString}}",
  "to_status": "{{issue.status.name}}",
  "status_category": "{{issue.status.statusCategory.key}}",
  "changed_at": "{{now.jiraDateTime}}",
  "ce.source": "/bu-ai-obs/adapters/jira-automation",
  "org.bu": "hiscox-re",
  "telemetry.pipeline": "agentic-ops"
}]
```

VERIFY the smart-value names in your Jira instance's rule editor before
enabling. This mirrors the canonical `WorkItemStatusChanged` event in its flat
New Relic form (see `contracts/events.md`).
'@
W 'adapters\jira\fixture.json' @'
{
  "issues": [
    {"key": "RE-1201", "fields": {"issuetype": {"name": "Story"}, "labels": [], "created": "2026-10-01T09:00:00.000+0000", "updated": "2026-10-09T16:00:00.000+0000", "resolutiondate": "2026-10-09T16:00:00.000+0000", "status": {"name": "Done", "statusCategory": {"key": "done"}}, "customfield_10001": {"name": "Pricing"}, "customfield_10016": 3}},
    {"key": "RE-1202", "fields": {"issuetype": {"name": "Bug"}, "labels": ["data"], "created": "2026-10-03T09:00:00.000+0000", "updated": "2026-10-06T11:00:00.000+0000", "resolutiondate": null, "status": {"name": "In Progress", "statusCategory": {"key": "indeterminate"}}, "customfield_10001": {"name": "Pricing"}, "customfield_10016": 2}},
    {"key": "RE-1203", "fields": {"issuetype": {"name": "Task"}, "labels": ["platform"], "created": "2026-10-04T09:00:00.000+0000", "updated": "2026-10-04T09:00:00.000+0000", "resolutiondate": null, "status": {"name": "To Do", "statusCategory": {"key": "new"}}, "customfield_10001": null, "customfield_10016": null}}
  ]
}
'@
~~~~

**Part 4 of 19**

~~~~powershell
# Part 4 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'adapters\jira\sync_work_items.py' @'
#!/usr/bin/env python3
"""Jira adapter: the only component that knows Jira exists.

Polls Jira for issues changed since the last run, mints our work-item IDs for
new keys (alias table), and emits canonical work events. Also writes the
work_items.csv snapshot the value view reads.

Moving to Azure DevOps = write adapters/ado/ that emits the same events and
calls AliasTable.link('ado:AB#...', 'jira:RE-...') for migrated items.

Usage:
  python -m adapters.jira.sync_work_items --jql "project = RE" \
      --aliases data/aliases.csv --out data/work_items.csv
  # offline / test:
  python -m adapters.jira.sync_work_items --fixture adapters/jira/fixture.json --dry-run

Env: JIRA_BASE_URL, JIRA_EMAIL, JIRA_API_TOKEN,
     JIRA_TEAM_FIELD (e.g. customfield_10001), JIRA_POINTS_FIELD (e.g. customfield_10016)
"""
from __future__ import annotations

import argparse
import base64
import csv
import json
import os
import sys
import urllib.request
from datetime import datetime, timezone

sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "..", "..")))

from obs.aliases import AliasTable  # noqa: E402
from obs.events import make_event  # noqa: E402
from obs.nr import NewRelicSender  # noqa: E402

SOURCE = "/bu-ai-obs/adapters/jira"
SNAPSHOT_FIELDS = ["work_item_id", "alias", "task_class", "task_size", "team", "work_stream",
                   "status", "created_at", "done_at"]
DEFAULT_CLASS_MAP = {
    "issuetype": {"Bug": "bug", "Story": "feature", "Task": "feature", "Spike": "feature", "Sub-task": "feature"},
    "labels": {"refactor": "refactor", "tests": "test", "test": "test", "migration": "migration", "data": "data"},
    "work_stream_labels": {"platform": "platform", "rnd": "rnd", "r-and-d": "rnd"},
}


def fetch_issues(jql: str, fields: list[str]) -> list[dict]:
    """Jira Cloud enhanced search (POST /rest/api/3/search/jql, token paging).
    VERIFY against your Jira instance; Server/DC uses /rest/api/2/search."""
    base = os.environ["JIRA_BASE_URL"].rstrip("/")
    auth = base64.b64encode(f"{os.environ['JIRA_EMAIL']}:{os.environ['JIRA_API_TOKEN']}".encode()).decode()
    issues, token = [], None
    while True:
        body = {"jql": jql, "fields": fields, "maxResults": 100}
        if token:
            body["nextPageToken"] = token
        req = urllib.request.Request(f"{base}/rest/api/3/search/jql", data=json.dumps(body).encode(),
                                     method="POST", headers={"Authorization": f"Basic {auth}",
                                                             "Content-Type": "application/json",
                                                             "Accept": "application/json"})
        with urllib.request.urlopen(req, timeout=60) as resp:
            page = json.load(resp)
        issues.extend(page.get("issues", []))
        token = page.get("nextPageToken")
        if page.get("isLast", True) or not token:
            return issues


def classify(fields: dict, class_map: dict) -> tuple[str, str]:
    labels = [str(l).lower() for l in (fields.get("labels") or [])]
    work_stream = next((v for k, v in class_map["work_stream_labels"].items() if k in labels), "team_delivery")
    for label, cls in class_map["labels"].items():
        if label in labels:
            return cls, work_stream
    itype = (fields.get("issuetype") or {}).get("name", "")
    return class_map["issuetype"].get(itype, "other"), work_stream


def team_value(raw) -> str:
    if isinstance(raw, dict):
        return str(raw.get("name") or raw.get("value") or raw.get("title") or "")
    return str(raw or "")


def run(issues: list[dict], aliases: AliasTable, state: dict, class_map: dict,
        team_field: str, points_field: str) -> tuple[list[dict], list[dict]]:
    events, snapshot = [], []
    for issue in issues:
        f = issue.get("fields", {})
        alias = f"jira:{issue['key']}"
        is_new = aliases.resolve(alias) == "wi_unassigned"
        wi = aliases.ensure(alias)
        task_class, work_stream = classify(f, class_map)
        status = (f.get("status") or {}).get("name", "")
        category = ((f.get("status") or {}).get("statusCategory") or {}).get("key", "")
        points = f.get(points_field) if points_field else None
        team = team_value(f.get(team_field)) if team_field else ""
        done_at = f.get("resolutiondate") if category == "done" else None
        prev = state.get(alias, {})

        if is_new:
            events.append(make_event("WorkItemCreated", SOURCE, wi, {
                "alias": alias, "task_class": task_class, "task_size": points, "team": team,
                "work_stream": work_stream, "created_at": f.get("created")}))
        if prev.get("status") != status:
            events.append(make_event("WorkItemStatusChanged", SOURCE, wi, {
                "alias": alias, "from_status": prev.get("status"), "to_status": status,
                "status_category": category, "changed_at": f.get("updated")}))
        if done_at and not prev.get("done_at"):
            events.append(make_event("WorkItemDone", SOURCE, wi, {"alias": alias, "done_at": done_at}))

        state[alias] = {"status": status, "done_at": done_at}
        snapshot.append({"work_item_id": wi, "alias": alias, "task_class": task_class,
                         "task_size": points if points is not None else "", "team": team,
                         "work_stream": work_stream, "status": status,
                         "created_at": f.get("created", ""), "done_at": done_at or ""})
    return events, snapshot


def main(argv=None) -> int:
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    ap.add_argument("--jql", default="updated >= -7d ORDER BY updated ASC")
    ap.add_argument("--fixture", help="read issues from a JSON file instead of Jira")
    ap.add_argument("--aliases", default="data/aliases.csv")
    ap.add_argument("--state", default="data/jira-state.json")
    ap.add_argument("--out", default="data/work_items.csv")
    ap.add_argument("--class-map", help="JSON file overriding the task-class mapping")
    ap.add_argument("--dry-run", action="store_true", help="write events to events.ndjson instead of New Relic")
    a = ap.parse_args(argv)

    for p in (a.aliases, a.state, a.out):
        os.makedirs(os.path.dirname(os.path.abspath(p)), exist_ok=True)
    class_map = DEFAULT_CLASS_MAP
    if a.class_map:
        with open(a.class_map, encoding="utf-8") as fh:
            class_map = {**DEFAULT_CLASS_MAP, **json.load(fh)}
    team_field = os.environ.get("JIRA_TEAM_FIELD", "")
    points_field = os.environ.get("JIRA_POINTS_FIELD", "")

    if a.fixture:
        with open(a.fixture, encoding="utf-8") as fh:
            issues = json.load(fh)["issues"]
    else:
        fields = ["issuetype", "status", "labels", "created", "updated", "resolutiondate"]
        fields += [x for x in (team_field, points_field) if x]
        issues = fetch_issues(a.jql, fields)

    aliases = AliasTable(a.aliases)
    state = {}
    if os.path.exists(a.state):
        with open(a.state, encoding="utf-8") as fh:
            state = json.load(fh)

    events, snapshot = run(issues, aliases, state, class_map, team_field, points_field)

    sender = NewRelicSender(dry_run_path="events.ndjson" if a.dry_run else None)
    sent = sender.send(events)
    aliases.save()
    with open(a.state, "w", encoding="utf-8") as fh:
        json.dump(state, fh, indent=1)
    # merge snapshot into existing work_items.csv (latest wins)
    rows = {}
    if os.path.exists(a.out):
        with open(a.out, newline="", encoding="utf-8") as fh:
            rows = {r["work_item_id"]: r for r in csv.DictReader(fh)}
    rows.update({r["work_item_id"]: r for r in snapshot})
    with open(a.out, "w", newline="", encoding="utf-8") as fh:
        w = csv.DictWriter(fh, fieldnames=SNAPSHOT_FIELDS)
        w.writeheader()
        w.writerows(rows.values())
    print(f"{len(issues)} issues, {sent} events, {len(aliases)} aliases, "
          f"{len(rows)} work items -> {a.out}  ({datetime.now(timezone.utc):%Y-%m-%dT%H:%MZ})")
    return 0


if __name__ == "__main__":
    sys.exit(main())
'@
~~~~

**Part 5 of 19**

~~~~powershell
# Part 5 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'bench\README.md' @'
# Replay benchmark (Phase 7.2)

An internal SWE-bench-style suite on our own codebase. The authoritative
answer to "would a cheaper model do as well?".

1. **Corpus.** Closed tickets with a merged PR and passing tests. Stratify
   across `task_class` and size. Start with ~30, grow to 100+. One file per
   task: `bench/tasks/<work_item_id>.yaml` (see `TEMPLATE.yaml`).
2. **Isolation.** Each run in a fresh environment checked out at the base SHA —
   reuse the per-ticket slots script or a container. Reference tests are hidden
   from the agent and added only at scoring time.
3. **Run matrix.** Each task × each candidate model × k runs (k ≥ 3), same
   harness, prompt, tools and budget caps (max tokens, max wall-clock). Only
   the model changes.
4. **Scoring.** Run the existing QA gate (tests incl. hidden reference tests,
   mutation score, CRAP/complexity delta, lint/security). Record pass/fail plus
   each sub-score.
5. **Emit** a `BenchRun` event per run (contract fields: bench.version, model,
   outcome, task_class, task_size, attempt_n, cost_usd, turns, tool_calls,
   tool_errors, wallclock_s, mutation_score, crap_delta), and append a row to
   `bench_runs.csv` for the value view.
6. **Cadence.** On demand, and when a new model becomes available on
   Vertex/APIM. Pin `bench.version`; never edit tasks in place.
7. **Contamination.** Prefer recent tickets. Public models won't have trained on
   our private repo, but keep reference tests hidden regardless.

Acceptance: ≥ 30 tasks × 2 models × k=3 end to end, frontier chart with CIs,
and a one-paragraph finding per task class stating where a cheaper model is
statistically equivalent.
'@
W 'bench\tasks\TEMPLATE.yaml' @'
# bench/tasks/<work_item_id>.yaml — one closed ticket, frozen. Never edit in place.
bench_version: 1
work_item_id: wi_01JD0000000000000000000000
alias: jira:RE-1234
task_class: bug            # bug | feature | refactor | test | migration | data
task_size: 3               # story points
repo: hiscox/re-pricing-api
base_sha: 0000000000000000000000000000000000000000
brief: |
  The ticket description and acceptance criteria exactly as the agent
  originally received them.
hidden_tests:              # added only at scoring time
  - tests/Pricing/ReferenceTests.fs
gate: qa-gate.yml          # the QA gate config used for scoring
budget:
  max_tokens: 400000
  max_wallclock_min: 45
'@
W 'clients\apim\ai-gateway-policy.xml' @'
<!--
  APIM v2 inbound policy for the AI gateway API.
  Gives us identity, token metrics, per-user quotas and trace correlation.
  Test against the Vertex backend FIRST: the llm-* policies parse usage from
  OpenAI-compatible responses; confirm they read your backend's usage fields
  before building dashboards on them.

  Keyed on the Entra object ID (oid), not email: emails change, oids don't.
  Email is kept as a display dimension only.

  Prereqs:
  - Application Insights logger on the API with custom metrics enabled
  - Diagnostic setting: GatewayLlmLogs -> Event Hub (message logging OFF)
  - Backend auth via APIM managed identity, never provider keys in clients
-->
<policies>
  <inbound>
    <base />

    <!-- Identity: validate the caller's Entra token and keep the claims -->
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401"
                  output-token-variable-name="jwt">
      <openid-config url="https://login.microsoftonline.com/{{tenant-id}}/v2.0/.well-known/openid-configuration" />
      <audiences><audience>{{ai-gateway-audience}}</audience></audiences>
    </validate-jwt>
    <set-variable name="enduser-oid" value="@(((Jwt)context.Variables["jwt"]).Claims.GetValueOrDefault("oid", "unknown"))" />
    <set-variable name="enduser" value="@(((Jwt)context.Variables["jwt"]).Claims.GetValueOrDefault("preferred_username", "unknown").ToLower())" />

    <!-- Per-user token budget. Tokens, not money: convert the monthly budget
         into a token quota for the expected model mix. Money is computed
         downstream from tokens x newrelic/model-prices.csv. -->
    <llm-token-limit counter-key="@((string)context.Variables["enduser-oid"])"
                     tokens-per-minute="200000"
                     token-quota="{{monthly-token-quota}}"
                     token-quota-period="Monthly"
                     estimate-prompt-tokens="false"
                     remaining-quota-tokens-header-name="x-remaining-quota" />

    <!-- Token metrics to App Insights, dimensioned by the attribute contract.
         APIM allows a small number of custom dimensions; keep this list short. -->
    <llm-emit-token-metric namespace="agentic-ops">
      <dimension name="enduser.oid" value="@((string)context.Variables["enduser-oid"])" />
      <dimension name="enduser.id" value="@((string)context.Variables["enduser"])" />
      <dimension name="client" value="@(context.Request.Headers.GetValueOrDefault("x-client", "unknown"))" />
      <dimension name="API ID" />
      <dimension name="Backend ID" />
    </llm-emit-token-metric>

    <!-- Correlation: pass W3C trace context through so agent spans and
         APIM logs share a trace id. Generate one if the caller didn't. -->
    <choose>
      <when condition="@(!context.Request.Headers.ContainsKey("traceparent"))">
        <set-header name="traceparent" exists-action="skip">
          <value>@("00-" + Guid.NewGuid().ToString("N") + "-" + Guid.NewGuid().ToString("N").Substring(0, 16) + "-01")</value>
        </set-header>
      </when>
    </choose>
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
  <on-error><base /></on-error>
</policies>
'@
W 'clients\claude-code\managed-settings.json' @'
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "http/protobuf",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "https://REPLACE-collector-fqdn",
    "OTEL_EXPORTER_OTLP_HEADERS": "Authorization=Bearer REPLACE-ingest-token",
    "OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE": "delta",
    "OTEL_LOG_USER_PROMPTS": "0",
    "OTEL_METRICS_INCLUDE_SESSION_ID": "true",
    "AI_OBS_TICKET_REGEX": "([A-Z][A-Z0-9]+-[0-9]+|AB#[0-9]+)"
  },
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -ExecutionPolicy Bypass -File \"C:\\ProgramData\\ai-obs\\session-start.ps1\"",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
'@
W 'clients\claude-code\set-identity.ps1' @'
# Sets the per-user join keys that managed settings can't (they're global).
# Push via Intune as a user-context script. Values must be W3C baggage-safe:
# no spaces, comma-separated key=value.
param(
  [Parameter(Mandatory)] [string] $Team,     # e.g. pricing-platform
  [Parameter(Mandatory)] [string] $Bu        # e.g. hiscox-re
)
$upn = (whoami /upn).Trim().ToLower()
$attrs = "enduser.id=$upn,team=$Team,bu=$Bu,client=claude-code"
[Environment]::SetEnvironmentVariable("OTEL_RESOURCE_ATTRIBUTES", $attrs, "User")
Write-Output "OTEL_RESOURCE_ATTRIBUTES=$attrs"
'@
~~~~

**Part 6 of 19**

~~~~powershell
# Part 6 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'clients\claude-code\set-identity.sh' @'
#!/usr/bin/env bash
# macOS/Linux/WSL equivalent. Usage: ./set-identity.sh <team> <bu> <email>
set -euo pipefail
TEAM="$1"; BU="$2"; EMAIL="$(echo "$3" | tr '[:upper:]' '[:lower:]')"
LINE="export OTEL_RESOURCE_ATTRIBUTES=\"enduser.id=${EMAIL},team=${TEAM},bu=${BU},client=claude-code\""
for rc in ~/.zshrc ~/.bashrc; do
  [ -f "$rc" ] || continue
  grep -v '^export OTEL_RESOURCE_ATTRIBUTES=' "$rc" > "$rc.tmp" && mv "$rc.tmp" "$rc"
  echo "$LINE" >> "$rc"
done
echo "$LINE"
'@
W 'clients\claude-code\hooks\session-start.ps1' @'
# Claude Code SessionStart hook (Windows).
# Emits one OTLP log record, event.name = ai_session_started, carrying the
# session id and the git branch. Attribution happens downstream: usage joins
# to this record on session.id, and the branch resolves to a work item via the
# alias table. Nothing in the running Claude Code process has to change.
#
# Contract: never block, never fail the session. Any error -> exit 0 silently.
# Install to C:\ProgramData\ai-obs\session-start.ps1 (Intune), referenced by
# managed-settings.json.
$ErrorActionPreference = 'SilentlyContinue'
try {
  $raw = [Console]::In.ReadToEnd()
  $evt = $raw | ConvertFrom-Json
  $sessionId = [string]$evt.session_id
  $cwd = [string]$evt.cwd
  $source = [string]$evt.source
  if (-not $sessionId) { exit 0 }

  $branch = ''
  $repo = ''
  if ($cwd -and (Get-Command git -ErrorAction SilentlyContinue)) {
    $branch = (git -C $cwd branch --show-current 2>$null | Out-String).Trim()
    $remote = (git -C $cwd remote get-url origin 2>$null | Out-String).Trim()
    if ($remote) { $repo = ($remote -replace '\.git$', '') -replace '^.*[/:]', '' }
  }

  $pattern = $env:AI_OBS_TICKET_REGEX
  if (-not $pattern) { $pattern = '([A-Z][A-Z0-9]+-[0-9]+|AB#[0-9]+)' }
  $alias = ''
  if ($branch -match $pattern) {
    $key = $Matches[1]
    if ($key -like 'AB#*') { $alias = "ado:$key" } else { $alias = "jira:$key" }
  }

  $endpoint = $env:OTEL_EXPORTER_OTLP_ENDPOINT
  if (-not $endpoint) { exit 0 }
  $headers = @{}
  if ($env:OTEL_EXPORTER_OTLP_HEADERS) {
    foreach ($pair in $env:OTEL_EXPORTER_OTLP_HEADERS.Split(',')) {
      $kv = $pair.Split('=', 2)
      if ($kv.Count -eq 2) { $headers[$kv[0].Trim()] = [Uri]::UnescapeDataString($kv[1].Trim()) }
    }
  }

  function A($k, $v) { @{ key = $k; value = @{ stringValue = [string]$v } } }

  $resAttrs = @((A 'service.name' 'claude-code'))
  if ($env:OTEL_RESOURCE_ATTRIBUTES) {
    foreach ($pair in $env:OTEL_RESOURCE_ATTRIBUTES.Split(',')) {
      $kv = $pair.Split('=', 2)
      if ($kv.Count -eq 2) { $resAttrs += (A $kv[0].Trim() $kv[1].Trim()) }
    }
  }

  $now = [string]([DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds() * 1000000)
  $body = @{
    resourceLogs = @(@{
      resource = @{ attributes = $resAttrs }
      scopeLogs = @(@{
        scope = @{ name = 'ai-obs.session-start'; version = '1' }
        logRecords = @(@{
          timeUnixNano = $now
          body = @{ stringValue = 'ai_session_started' }
          attributes = @(
            (A 'event.name' 'ai_session_started'),
            (A 'session.id' $sessionId),
            (A 'session.source' $source),
            (A 'harness' 'claude-code'),
            (A 'git.branch' $branch),
            (A 'git.repo' $repo),
            (A 'work_item.alias' $alias)
          )
        })
      })
    })
  } | ConvertTo-Json -Depth 12 -Compress

  Invoke-RestMethod -Method Post -Uri ($endpoint.TrimEnd('/') + '/v1/logs') `
    -Headers $headers -ContentType 'application/json' -Body $body -TimeoutSec 3 | Out-Null
} catch { }
exit 0
'@
W 'clients\claude-code\hooks\session_start.py' @'
#!/usr/bin/env python3
"""Claude Code SessionStart hook (macOS / Linux / WSL). Same contract as
session-start.ps1: emit one OTLP log record (event.name = ai_session_started)
with session.id + git branch, never block, never fail the session.

Reference it from settings as:
  "command": "python3 /opt/ai-obs/session_start.py"
"""
import json
import os
import re
import subprocess
import sys
import time
import urllib.request

DEFAULT_PATTERN = r"([A-Z][A-Z0-9]+-[0-9]+|AB#[0-9]+)"


def git(cwd, *args):
    try:
        return subprocess.run(["git", "-C", cwd, *args], capture_output=True,
                              text=True, timeout=2).stdout.strip()
    except Exception:
        return ""


def alias_from_branch(branch, pattern=None):
    m = re.search(pattern or DEFAULT_PATTERN, branch or "")
    if not m:
        return ""
    key = m.group(1)
    return f"ado:{key}" if key.startswith("AB#") else f"jira:{key}"


def parse_kv(s):
    out = {}
    for pair in (s or "").split(","):
        if "=" in pair:
            k, v = pair.split("=", 1)
            out[k.strip()] = urllib.request.unquote(v.strip())
    return out


def build_body(session_id, source, branch, repo, alias, resource):
    def a(k, v):
        return {"key": k, "value": {"stringValue": str(v)}}
    res = [a("service.name", "claude-code")] + [a(k, v) for k, v in resource.items()]
    return {"resourceLogs": [{
        "resource": {"attributes": res},
        "scopeLogs": [{
            "scope": {"name": "ai-obs.session-start", "version": "1"},
            "logRecords": [{
                "timeUnixNano": str(time.time_ns()),
                "body": {"stringValue": "ai_session_started"},
                "attributes": [
                    a("event.name", "ai_session_started"), a("session.id", session_id),
                    a("session.source", source), a("harness", "claude-code"),
                    a("git.branch", branch), a("git.repo", repo), a("work_item.alias", alias),
                ],
            }],
        }],
    }]}


def main():
    try:
        evt = json.loads(sys.stdin.read() or "{}")
        session_id = evt.get("session_id")
        endpoint = os.environ.get("OTEL_EXPORTER_OTLP_ENDPOINT")
        if not session_id or not endpoint:
            return
        cwd = evt.get("cwd") or os.getcwd()
        branch = git(cwd, "branch", "--show-current")
        remote = git(cwd, "remote", "get-url", "origin")
        repo = re.sub(r"^.*[/:]", "", re.sub(r"\.git$", "", remote)) if remote else ""
        alias = alias_from_branch(branch, os.environ.get("AI_OBS_TICKET_REGEX"))
        body = build_body(session_id, evt.get("source", ""), branch, repo, alias,
                          parse_kv(os.environ.get("OTEL_RESOURCE_ATTRIBUTES")))
        headers = {"Content-Type": "application/json",
                   **parse_kv(os.environ.get("OTEL_EXPORTER_OTLP_HEADERS"))}
        req = urllib.request.Request(endpoint.rstrip("/") + "/v1/logs",
                                     data=json.dumps(body).encode(), headers=headers)
        urllib.request.urlopen(req, timeout=3)
    except Exception:
        pass


if __name__ == "__main__":
    main()
    sys.exit(0)
'@
~~~~

**Part 7 of 19**

~~~~powershell
# Part 7 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'clients\copilot\README.md' @'
# GitHub Copilot capture

Copilot traffic goes to GitHub, not through APIM. We can **see** it (OTel and
the usage metrics API) but only **limit** it with GitHub's own budgets and
policies. Lines marked VERIFY were not tested against our tenant.

## 1. Telemetry (all surfaces): enterprise-managed settings

GitHub lets enterprise admins mandate where Copilot sends OpenTelemetry, for
the VS Code Copilot Chat extension, the agent host behind Copilot CLI, and the
Copilot app. Devs then need no OTEL_* setup.

- Endpoint: our collector, `https://<collector-fqdn>` (OTLP/HTTP)
- Header: `Authorization: Bearer <ingest-token>`
- Content capture: **off**
- Service name: `github-copilot` (the collector maps anything containing
  "copilot" to `harness=copilot`)

VERIFY: the exact keys of the `telemetry` block in `managed-settings.json`.
Reference: https://docs.github.com/en/copilot/reference/enterprise-administrators/opentelemetry-reference

## 2. Per-dev fallback (before the enterprise setting is live)

VS Code `settings.json`:

```json
{
  "github.copilot.chat.otel.enabled": true,
  "github.copilot.chat.otel.exporterType": "otlp-http",
  "github.copilot.chat.otel.otlpEndpoint": "https://<collector-fqdn>"
}
```

VERIFY: the setting name for the auth header and for content capture.

Copilot CLI (PowerShell profile):

```powershell
$env:COPILOT_OTEL_ENABLED = "true"
$env:COPILOT_OTEL_EXPORTER_TYPE = "otlp-http"
$env:OTEL_EXPORTER_OTLP_ENDPOINT = "https://<collector-fqdn>"
$env:OTEL_EXPORTER_OTLP_HEADERS = "Authorization=Bearer <ingest-token>"
$env:OTEL_SERVICE_NAME = "github-copilot"
```

## 3. Copilot SDK (agents people build on Copilot)

Telemetry is set in code: pass a `TelemetryConfig` when constructing the
`CopilotClient`, OTLP/HTTP to the collector, `captureContent: false`. Put this
in the "how to build an agent" template; it cannot be forced centrally.

## 4. Billing truth: usage metrics API

`adapters/github/copilot_usage.py` pulls per-user daily usage, including
`ai_credits_used`, and turns it into `AIUsage` events with
`billing_source=github`. Run it daily (GitHub Action on a schedule, or by hand).

## Known gaps

- **Attribution.** Copilot has no SessionStart hook we can use (VERIFY), so its
  sessions are attributed by repo and time window, not by branch at start.
- **Claude inside Copilot.** Claude sessions run through VS Code Copilot are
  billed by GitHub. They show up as `gen_ai.agent.name = claude` in Copilot's
  telemetry. Count them under `billing_source=github`, not APIM.
- **Cloud coding agent.** No documented OTel path found; covered by the usage
  metrics API only.
'@
W 'collector\Dockerfile' @'
# Pin by digest in production (see docs/losses-and-alternatives.md, supply chain).
FROM otel/opentelemetry-collector-contrib:0.135.0
COPY otel-collector.yaml /etc/otelcol-contrib/config.yaml
COPY otel-collector.apim.yaml /etc/otelcol-contrib/apim.yaml
COPY otel-collector.langfuse.yaml /etc/otelcol-contrib/langfuse.yaml
EXPOSE 4318 13133
# Default: base only. Prod adds the APIM overlay via --args in deploy.sh.
# Local can add the Langfuse overlay via COLLECTOR_ARGS in docker-compose.
CMD ["--config=/etc/otelcol-contrib/config.yaml"]
'@
W 'collector\otel-collector.apim.yaml' @'
# Overlay: APIM v2 per-request logs via Event Hub.
# Loaded on top of otel-collector.yaml:
#   otelcol-contrib --config otel-collector.yaml --config otel-collector.apim.yaml
# Adds its own logs/apim pipeline, so the base pipelines are untouched.
#
# Scaling caveat: this receiver pulls from Event Hub. Multiple replicas will
# each read every partition (duplicates). Run max-replicas=1 for now, or split
# Event Hub ingestion into its own single-replica app later.

receivers:
  azureeventhub:
    connection: ${env:EVENTHUB_CONNECTION}   # includes EntityPath=<hub>
    format: azure                            # parse Azure resource-log envelope

processors:
  resource/apim:
    attributes:
      - key: client
        value: apim
        action: upsert
      - key: service.name
        value: apim-ai-gateway
        action: upsert
      - key: billing_source
        value: apim
        action: upsert

  # APIM LLM logs can include request/response bodies if message logging is on.
  # Keep message logging OFF in the diagnostic setting; this is the backstop.
  transform/apim-redact:
    error_mode: ignore
    log_statements:
      - context: log
        statements:
          - delete_matching_keys(log.attributes, "(?i)(prompt|completion|messages|request_body|response_body)")

service:
  pipelines:
    logs/apim:
      receivers: [azureeventhub]
      processors: [memory_limiter, resource/common, resource/apim, transform/apim-redact, transform/redact, batch]
      exporters: [otlphttp/newrelic]
'@
W 'collector\otel-collector.langfuse.yaml' @'
# Overlay: Langfuse as a personal learning / eval workbench. LOCAL ONLY.
# Decision (Sep 2026): Langfuse is not a production consumer. New Relic is
# the system of record. Load this only on your own machine:
#   otelcol-contrib --config otel-collector.yaml --config otel-collector.langfuse.yaml
#
# Traces sent here keep their bodies (no transform/redact), because that is
# what makes Langfuse useful for debugging and evals. Keep work traces that
# contain prompts or code on the personal instance until security has agreed
# data residency for eval datasets.

exporters:
  otlphttp/langfuse:
    endpoint: ${env:LANGFUSE_OTLP_ENDPOINT}    # e.g. http://host.docker.internal:3000/api/public/otel
    headers:
      Authorization: "Basic ${env:LANGFUSE_AUTH_B64}"   # base64("pk-lf-...:sk-lf-...")

service:
  pipelines:
    traces/langfuse:
      receivers: [otlp]
      processors: [memory_limiter, resource/common, transform/harness, batch]
      exporters: [otlphttp/langfuse]
'@
~~~~

**Part 8 of 19**

~~~~powershell
# Part 8 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'collector\otel-collector.yaml' @'
# BU AI observability collector (Hiscox RE)
# One funnel in. New Relic is the system of record:
#   metrics + logs + traces -> New Relic (redacted)
# Langfuse is optional and local only: load otel-collector.langfuse.yaml on top.
# APIM per-request logs: load otel-collector.apim.yaml on top.
#
# Stateless by design: clients emit DELTA temporality, so no
# cumulativetodelta processor is needed and replicas can scale out
# (except while the APIM Event Hub overlay is loaded, see that file).
#
# Sources that send OTLP here: Claude Code, GitHub Copilot (VS Code, CLI, app),
# own agents (OTel SDK / MAF), the Claude Code SessionStart hook.

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  bearertokenauth:
    token: ${env:INGEST_TOKEN}          # shared ingest token; rotate via Key Vault

receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
        auth:
          authenticator: bearertokenauth

processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 20

  # Stamp every signal with pipeline-level context.
  # org.bu lets Group tell our data apart once they collect from us.
  resource/common:
    attributes:
      - key: deployment.environment
        value: ${env:DEPLOY_ENV}
        action: upsert
      - key: telemetry.pipeline
        value: agentic-ops
        action: upsert
      - key: org.bu
        value: ${env:ORG_BU}
        action: upsert

  # Harness detection. Each client sets service.name; map it to one
  # low-cardinality 'harness' attribute the dashboards facet on.
  transform/harness:
    error_mode: ignore
    metric_statements:
      - context: resource
        statements:
          - set(resource.attributes["harness"], "claude-code") where resource.attributes["service.name"] == "claude-code"
          - set(resource.attributes["harness"], "copilot") where IsMatch(resource.attributes["service.name"], "(?i)copilot")
    log_statements:
      - context: resource
        statements:
          - set(resource.attributes["harness"], "claude-code") where resource.attributes["service.name"] == "claude-code"
          - set(resource.attributes["harness"], "copilot") where IsMatch(resource.attributes["service.name"], "(?i)copilot")
    trace_statements:
      - context: resource
        statements:
          - set(resource.attributes["harness"], "claude-code") where resource.attributes["service.name"] == "claude-code"
          - set(resource.attributes["harness"], "copilot") where IsMatch(resource.attributes["service.name"], "(?i)copilot")

  # Governance guardrails. Belt-and-braces: content capture is off at
  # every source too, but never trust a single control.
  transform/redact:
    error_mode: ignore
    log_statements:
      - context: log
        statements:
          - delete_key(log.attributes, "prompt")
          - delete_matching_keys(log.attributes, "(?i)^(gen_ai\\.)?(prompt|completion|input\\.messages|output\\.messages)")
          - truncate_all(log.attributes, 4095)          # New Relic attribute value limit
    trace_statements:
      - context: span
        statements:
          - delete_matching_keys(span.attributes, "^gen_ai\\.(prompt|completion|input\\.messages|output\\.messages|system_instructions)")
          - delete_matching_keys(span.attributes, "^llm\\.(prompts|completions)")
          - truncate_all(span.attributes, 4095)

  batch:
    send_batch_size: 1000
    timeout: 5s

exporters:
  otlphttp/newrelic:
    endpoint: ${env:NEW_RELIC_OTLP_ENDPOINT}   # https://otlp.eu01.nr-data.net (EU) or https://otlp.nr-data.net (US)
    headers:
      api-key: ${env:NEW_RELIC_LICENSE_KEY}
    compression: gzip

  debug:
    verbosity: ${env:DEBUG_VERBOSITY}          # basic locally, normal when debugging

service:
  extensions: [health_check, bearertokenauth]
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, resource/common, transform/harness, batch]
      exporters: [otlphttp/newrelic, debug]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, resource/common, transform/harness, transform/redact, batch]
      exporters: [otlphttp/newrelic, debug]

    traces:
      receivers: [otlp]
      processors: [memory_limiter, resource/common, transform/harness, transform/redact, batch]
      exporters: [otlphttp/newrelic, debug]
'@
~~~~

**Part 9 of 19**

~~~~powershell
# Part 9 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'contracts\events.md' @'
# Canonical event contract v0

The contract is the product. Sources and consumers can be swapped; this
cannot change without a version bump. The code version lives in
`obs/events.py` (`CONTRACT`), and `validate()` enforces it.

## Envelope: CloudEvents 1.0

| Field | Value |
|---|---|
| `specversion` | `1.0` |
| `id` | ULID |
| `source` | `/bu-ai-obs/<adapter>` e.g. `/bu-ai-obs/adapters/jira` |
| `type` | `com.hiscox.re.ai.<EventType>.v0` |
| `subject` | **our work-item ID** (`wi_` + ULID), or `wi_unassigned` |
| `time` | ISO 8601 UTC |
| `workitemalias` | extension, optional: raw tracker alias when the emitter can't resolve it (e.g. CI) |
| `data` | event fields below; **never** prompt, completion or message content |

In New Relic each event is flattened into one custom event:
`eventType` = the short type, `work_item.id` = subject, data fields as-is,
plus `org.bu` and `telemetry.pipeline`.

## Work-item IDs and aliases

- We mint the ID: `wi_01JD...` (ULID). Jira keys, Azure DevOps IDs and
  branch names are **aliases**.
- Alias forms: `jira:RE-1234`, `ado:AB#5678`, `git:<branch>` (resolved via the
  ticket key inside the branch name).
- The alias table (`data/aliases.csv`) is maintained by the Jira adapter and
  uploaded to New Relic as the `aliases` lookup table.
- Migration: `AliasTable.link("ado:AB#5678", "jira:RE-1234")` points the new
  key at the existing work item, so history stays joined.
- No ticket in the branch: `wi_unassigned`. Its share is tracked as a metric.

## Event types

| Type | Required | Optional | Emitted by |
|---|---|---|---|
| `WorkItemCreated` | alias | task_class, task_size, team, work_stream, created_at | Jira adapter |
| `WorkItemStatusChanged` | alias, to_status | from_status, status_category, changed_at | Jira adapter / Automation rule |
| `WorkItemDone` | alias, done_at | resolution | Jira adapter |
| `AISessionStarted` | session.id, harness | enduser.id, git.branch, git.repo, work_item.alias, session.source | Claude Code SessionStart hook (as an OTLP log) |
| `AIUsage` | harness, gen_ai.request.model, billing_source | session.id, enduser.id, gen_ai.usage.input_tokens, gen_ai.usage.output_tokens, cost_usd, ai_credits_used, usage_date | Copilot usage adapter; Claude Code via its own OTel metrics |
| `CIRunCompleted` | repo, run_id, conclusion | branch, attempt, duration_s, workflow | GitHub workflow |
| `PRMerged` | repo, pr_number | branch, merged_at | GitHub workflow |
| `QAGatePassed` | attempt_n | repo, branch, mutation_score | QA gate workflow |
| `QABouncedBack` | attempt_n | reason | QA gate workflow / Jira transition |
| `TaskOutcome` | outcome, attempt_n | gen_ai.request.model, task_class, task_size, cost_usd, wallclock_hours | emitted when a work item reaches a terminal state |
| `EvalResult` | suite, case_id, passed | gen_ai.request.model, score, harness_version | eval runner |
| `BenchRun` | bench.version, gen_ai.request.model, outcome | task_class, task_size, attempt_n, cost_usd, turns, tool_calls, tool_errors, wallclock_s, mutation_score, crap_delta | replay benchmark |

Enumerations:

- `outcome`: `pass` (QA gate passed on first run) · `reworked` (failed, then
  passed, or reopened) · `fail` · `abandoned` (closed without merge)
- `harness`: `claude-code`, `copilot`, `copilot-cli`, `copilot-sdk`, `own-agent`, `apim`, `unknown`
- `billing_source`: `apim`, `github`, `anthropic`, `vertex`, `unknown`.
  Each one reconciles against a different bill.
- `task_class`: `bug`, `feature`, `refactor`, `test`, `migration`, `data`, `other`
  (mapped from Jira issue type + labels in the Jira adapter)

## Resource attributes on OTel signals

| Attribute | Example | Set by |
|---|---|---|
| `enduser.id` | `mike@hiscox.com` (lowercase UPN) | identity script / APIM JWT |
| `enduser.oid` | Entra object ID | APIM (quota key) |
| `team`, `bu` | `pricing-platform`, `hiscox-re` | identity script |
| `harness` | `claude-code`, `copilot` | collector (`transform/harness`) |
| `org.bu` | `hiscox-re` | collector |
| `deployment.environment` | `local`, `prod` | collector |
| `telemetry.pipeline` | `agentic-ops` | collector |

Rules: lowercase, no spaces (W3C baggage format), UPN not display name.
Squad and budget live in the `org` lookup table, not on the wire, so reorgs
don't require re-instrumenting anyone.
'@
W 'deploy\deploy.sh' @'
#!/usr/bin/env bash
# max-replicas=1 while the Event Hub receiver lives in this app (see apim overlay).
# Deploys the collector to Azure Container Apps. Idempotent-ish: rerun to update.
# Langfuse is not deployed: it stays a local, personal workbench.
set -euo pipefail
: "${RG:?}" "${LOCATION:?}" "${ACR:?}" "${ENV_NAME:?}" "${KV_NAME:?}" "${IDENTITY_ID:?}"
APP=agentic-ops-collector
TAG=$(git rev-parse --short HEAD 2>/dev/null || date +%s)

az acr build -r "$ACR" -t "$APP:$TAG" ../collector

# Secrets come from Key Vault via the managed identity (needs 'Key Vault Secrets User').
kv() { echo "keyvaultref:https://${KV_NAME}.vault.azure.net/secrets/$1,identityref:${IDENTITY_ID}"; }

az containerapp create -g "$RG" -n "$APP" --environment "$ENV_NAME" \
  --image "$ACR.azurecr.io/$APP:$TAG" \
  --registry-server "$ACR.azurecr.io" --registry-identity "$IDENTITY_ID" \
  --user-assigned "$IDENTITY_ID" \
  --ingress external --target-port 4318 --transport http \
  --min-replicas 1 --max-replicas 1 --cpu 0.5 --memory 1Gi \
  --args "--config=/etc/otelcol-contrib/config.yaml" "--config=/etc/otelcol-contrib/apim.yaml" \
  --secrets \
    "ingest-token=$(kv otel-ingest-token)" \
    "nr-license=$(kv newrelic-license-key)" \
    "eventhub-conn=$(kv apim-eventhub-connection)" \
  --env-vars \
    DEPLOY_ENV=prod DEBUG_VERBOSITY=basic ORG_BU=hiscox-re \
    NEW_RELIC_OTLP_ENDPOINT=https://otlp.eu01.nr-data.net \
    INGEST_TOKEN=secretref:ingest-token \
    NEW_RELIC_LICENSE_KEY=secretref:nr-license \
    EVENTHUB_CONNECTION=secretref:eventhub-conn

az containerapp show -g "$RG" -n "$APP" --query properties.configuration.ingress.fqdn -o tsv
'@
~~~~

**Part 10 of 19**

~~~~powershell
# Part 10 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'docs\IMPLEMENTATION.md' @'
# Implementation plan (PoV, one squad)

Replaces `agentic-ops-implementation.md` (Oct 2026). Same phases in spirit;
the join key is now our own work-item ID with Jira as an alias, and Copilot is
in scope. The design rationale lives in the PoV doc ("BU AI Observability
Layer — PoV Plan"). Each phase has a "done when".

## Phase 0: Contract and foundations

- [ ] Agree `contracts/events.md` v0 with whoever will consume it.
- [ ] Pick the branch convention: ticket key in the name (`RE-1234-short-desc`).
      Set `AI_OBS_TICKET_REGEX` if keys look different.
- [ ] Run `python adapters/jira/sync_work_items.py --jql "project = RE"` once to
      mint IDs for existing issues (`data/aliases.csv`, `data/work_items.csv`).
- [ ] Upload `data/aliases.csv`, `newrelic/org-lookup.csv`, `newrelic/model-prices.csv`
      as New Relic lookup tables (`aliases`, `org`, `prices`).
- [ ] Deploy the collector (`deploy/deploy.sh`). Check `/` on port 13133 is healthy.

Done when: a test OTLP log posted to the collector shows up in New Relic with
`org.bu=hiscox-re` and `telemetry.pipeline=agentic-ops`.

## Phase 1: Capture

- [ ] **Claude Code.** IT pushes `clients/claude-code/managed-settings.json` and
      copies `hooks/session-start.ps1` to `C:\ProgramData\ai-obs\`. Identity via
      `set-identity.ps1` (Intune, user context).
- [ ] **Copilot.** Enterprise admin sets the managed OTel destination
      (`clients/copilot/README.md`). Schedule `adapters/github/copilot_usage.py` daily.
- [ ] **APIM.** Apply `clients/apim/ai-gateway-policy.xml` to a test API against
      the Vertex backend; confirm token metrics populate. Diagnostic setting to
      Event Hub with message logging OFF.
- [ ] **GitHub.** Add `adapters/github/delivery-events.yml` to the squad repos;
      the QA gate calls `emit_event.py QAGatePassed` / `QABouncedBack`.
- [ ] **Jira.** Schedule the sync daily; optionally add the Automation rule
      (`adapters/jira/automation-rule.md`) for real-time status.
- [ ] **Devs.** Each runs `tools/dev-setup.ps1` once.

Done when: for one ticket you can see, in New Relic, its session(s), cost,
CI runs, PR and QA gate result, all carrying the same alias.

## Phase 2: Consume

- [ ] Agentic Ops dashboard from `newrelic/dashboard-queries.nrql`. Per-user
      widgets go on a restricted page; team-level on the shared one.
- [ ] Export with `newrelic/value-view-exports.nrql` into `data/`, run
      `valueview/value_view.py`, load `out/value_view.csv` + `metric_catalogue.csv`
      into Power BI (star schema: metric table as fact; date, org, task class,
      model and catalogue as dimensions; filter `org_level` per page).
- [ ] Optional: Langfuse overlay on your own machine only.

Done when: one Power BI page shows cost per delivered work item next to
first-pass QA rate, with suppression visible (blank, not zero).

## Phase 3: Prove

- [ ] Run two to three weeks with the squad.
- [ ] Check the success criteria: coverage ≥ 90%, attribution ≥ 80%, value page live.
- [ ] Swap test: replace the Jira sync with the fixture (`--fixture`) and confirm
      nothing downstream changes.
- [ ] Review with the Engineering Manager and Head of Platform.

## Phase 7: Model effectiveness ("would another model do as well?")

Leadership question: which model gives the best task success per unit cost,
per kind of task?

### 7.0 The confound (read first)
Model choice is correlated with task difficulty — people pick bigger models
for harder work. **Never report raw pass rate by model from production
data.** Always compare within the same `task_class` × `task_size` bucket, and
treat the replay benchmark (7.2) as the authoritative answer.

### 7.1 Observational layer (from production telemetry)
- `task_class` from Jira issue type + labels (Jira adapter); `task_size` from story points.
- `outcome` per ticket: `pass` = reached Ready for QA via the QA gate on first
  gate run; `reworked` = gate failed then later passed, or ticket reopened;
  `fail` / `abandoned` = never passed / closed without merge.
- `attempt_n` = count of gate runs for the ticket.
- Emit a `TaskOutcome` event (model = dominant model by tokens on the ticket,
  task_class, task_size, outcome, attempt_n, cost_usd, wallclock_hours) when a
  ticket reaches a terminal state.

### 7.2 Replay benchmark (authoritative)
An internal SWE-bench-style suite on our own codebase. See `bench/README.md`.

### 7.3 Metrics
- **Headline:** cost per passed task = `sum(cost_usd) / count(outcome = pass)`, by model × task class.
- **Effectiveness:** pass@1, pass@k, human-intervention rate, rework/reopen rate, defect escape, review rejection rate.
- **Efficiency:** tokens/turns/tool calls per task, wall-clock, cache hit rate, stall/stuck-loop rate.
- **Output quality:** mutation score, CRAP/complexity delta, lint/security findings on AI changes.
- **Reliability:** tool-call error rate, retries, abandoned sessions.
- Always show **confidence intervals** (Wilson for pass rates) and n. Don't rank
  models whose intervals overlap — call them equivalent.

### 7.4 Dashboard
- **Model frontier:** cost per task (x) vs pass@1 (y), one point per model,
  faceted by task class. Models on the Pareto frontier are the recommendation.
- Table: model × task class with pass@1 (CI), cost per passed task, n.
- Production vs bench side by side, so drift between "lab" and "real" is visible.

### 7.5 Later: randomised routing
Once 7.2 shows equivalence for a low-risk task class, propose (to APIM owners)
routing a small random share of that class to the cheaper model and comparing
`TaskOutcome`. Requires sign-off; do not implement unilaterally.

## Open questions

1. Copilot session attribution: is there a session ID / hook to record the branch?
2. Does the SessionStart hook's session_id match Claude Code's OTel `session.id`?
3. Which user attribute does Copilot's OTel carry, and does it map to the Entra oid?
4. Can Claude Code on Vertex route through APIM without breaking auth or streaming?
5. Claude Code client-side cost vs the Vertex bill: agree a tolerance.
6. Group's LiteLLM: spend logs (Postgres), OTLP, or both? Which identity key?
7. Which models are available to benchmark via Vertex/APIM, and the bench budget.
8. Mapping from Jira issue types/labels to `task_class` (defaults in the Jira adapter).
9. New Relic AI Evaluation preview access.
'@
~~~~

**Part 11 of 19**

~~~~powershell
# Part 11 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'docs\ai-value-proposal.md' @'
# Measuring the Return on Agentic Engineering Spend

**A proposal for a three-tier measurement model**

Author: Mike — Software Engineering, Hiscox RE
Status: Draft for review

---

## 1. The ask

We are spending on agentic coding tooling and the platform that governs it. We currently have no agreed definition of what that spend is buying, which means the programme is defended on anecdote and defunded on intuition.

This proposal asks leadership to:

1. **Endorse a fixed set of five leadership metrics** (Section 4) and an explicit list of metrics we will *not* report (Section 6).
2. **Authorise a one-quarter measurement window** with the instrumentation described in Section 8.
3. **Pre-commit to the decision the data will inform** — scale, hold, or redirect — and to what "good" looks like *before* the first numbers arrive (Section 9).

Point 3 is the one that matters. A dashboard without a pre-agreed decision rule becomes a monthly argument about which chart is fair.

---

## 2. Why current measurement does not answer the leadership question

Engineering already has, or can shortly have, detailed telemetry: agent session traces, acceptance rates, escalation rates, PR cycle time, token spend. That instrumentation is well-suited to the squad and to the Scrum Master.

It does not answer the leadership question, for three reasons:

- **It measures the tool, not the outcome.** Escalation rate is an engineering diagnostic. It tells leadership nothing about whether the business is better off.
- **It is vulnerable to activity inflation.** Agentic tooling reliably increases commits, PRs and lines changed. Reporting those upward manufactures the appearance of return while saying nothing about qualified, durable output.
- **It has no denominator leadership recognises.** Token cost is a rounding error against engineer cost. A return story denominated in tokens is not a return story.

The fix is not more metrics. It is a smaller set, at a different altitude, with an explicit method for converting engineering signal into money and risk.

---

## 3. Three tiers, one data model

All three tiers are computed from the same underlying tables — agent sessions, pull requests, work items, deployments. The difference is aggregation and cadence, not collection.

| Tier | Audience | Question answered | Cadence |
|---|---|---|---|
| 1 — Squad | PO / Scrum Master | Is the flow healthy this week? | Weekly |
| 2 — Engineering | Engineering Manager | Where do agents work, where do they not? | Monthly |
| 3 — Leadership | BU / CTO level | What is the spend buying, and should we scale it? | Quarterly |

This matters operationally: **we instrument once**. Tier 3 is a view, not a second collection effort. That keeps the cost of measurement low and keeps the numbers reconcilable when someone challenges them.

---

## 4. The leadership metric set

Five metrics. The recommendation is to hold this list fixed for at least two quarters, because a metric set that changes every reporting cycle cannot show a trend, and a trend is the entire point.

### 4.1 Cost per production-qualified change (fully loaded)

**Definition:** total attributable delivery cost ÷ changes that reach production and survive 30 days.

Cost includes model and gateway spend, CI compute, and — critically — reviewer and rework time valued at fully loaded engineer cost.

**Why it is the headline:** it is the only metric that puts spend and output in the same fraction. It is also self-correcting: a tool that generates more but is trusted less will raise reviewer time and the metric will get worse, not better.

**Read the trend, not the level.** The absolute figure depends on cost assumptions nobody will fully agree on. The direction over four quarters is robust to those assumptions.

### 4.2 Engineer capacity redeployed

**Definition:** engineer hours released, expressed as FTE-equivalents, *plus a named statement of what that capacity moved to*.

**Why both halves are required:** released capacity is not value until it is redeployed. If the answer to "what did the freed capacity do" is "we don't know", the honest finding is that the saving has not yet been realised. Reporting the second half is what keeps this metric from becoming a fiction.

### 4.3 Quality differential

**Definition:** change failure rate and 30-day defect escape rate for agent-attributed changes, against the non-agent comparison cohort.

**Why it is non-negotiable:** it is the metric that proves we are not buying speed with future incidents. A report containing only favourable metrics will be discounted by any leader who has seen a technology programme before. Including the metric most likely to embarrass us is what makes the other four credible.

### 4.4 Governed path coverage

**Definition:** proportion of agent-originated changes that pass through the governed pipeline with a complete audit trail — construction-time constraints, CI policy gates, runtime guardrails, decision record.

**Why it belongs in a leadership pack at a reinsurer:** this is the metric that converts to regulatory and audit position rather than delivery speed. The question "can you demonstrate what the automated system did, under what constraints, and who approved it" has a cost attached to answering it badly. Coverage is the leading indicator of our ability to answer it well.

This metric is also the one that distinguishes our position from ungoverned adoption elsewhere in the market, and it is currently absent from every off-the-shelf measurement framework.

### 4.5 Adoption share

**Definition:** proportion of delivered work items with agent involvement.

**Why leadership needs it:** it is the multiplier on everything above, and it distinguishes two very different situations that otherwise look identical. Strong metrics at 15% adoption is an investment case. Strong metrics at 80% adoption is a mature capability. Weak metrics at low adoption is a pilot that has not been given a fair trial.

---

## 5. The value bridge: converting engineering signal to money

Three methods are available. The recommendation is to use the first as primary and the others as supporting colour, because only the first survives contact with Finance.

**Capacity method (recommended).** Hours released × fully loaded cost. Conservative, auditable, uses figures Finance already owns.
*Limitation, stated up front:* it counts input saved, not value created, and overstates return if capacity is not visibly redeployed.

**Throughput method.** Additional items delivered × value per item. Theoretically superior. Requires the PO to assign monetary value to work items, which in practice is rarely available and rarely defensible. Use only where a work item has a directly attributable revenue or cost line.

**Avoided-cost method.** Defects caught before production × average cost of a production defect; audit-preparation effort avoided. Useful for Section 4.4, but the counterfactual is weak — we are estimating incidents that did not happen. Present it as supporting evidence, never as the headline number.

---

## 6. What we will explicitly not report

Naming these up front is a deliberate move. It prevents a vanity metric being requested later, and it signals that the numbers we do report were not selected for flattery.

- **Percentage of code written by AI.** Measures volume of generated text. Rises when quality falls, because poor output is regenerated. It is the single most misleading metric in this space.
- **Token spend as a standalone figure.** Meaningless without a denominator; small enough against engineer cost to be a distraction either way.
- **Commit, PR or line counts.** Reliably inflated by agentic tooling. Reporting them upward is measurement theatre.
- **Per-engineer agent usage.** Creates an incentive to use agents where they do not help, and converts a capability question into a surveillance one.
- **Time saved, self-reported.** Survey-based estimates of hours saved are systematically optimistic and cannot be reconciled against any system of record.

---

## 7. Method, baselines, and where this could be wrong

Including this section is what makes the rest defensible under challenge.

**The counterfactual.** Comparison is against a concurrent non-agent cohort, drawn from the same data model via agent-involvement flagging on work items. Historical pre-adoption cycle time is available as a secondary check.

**The principal threat to validity — selection bias.** Engineers choose agents for tasks agents are good at. Well-specified, well-bounded, low-ambiguity work flows to the agent path; gnarly, under-specified work does not. This biases every agent-versus-human comparison in the agent's favour, and it is not a small effect.

*Mitigation:* segment all comparisons by work item type and size band, and treat any unsegmented headline comparison as inadmissible. Where segmentation leaves cells too small to be meaningful, say so rather than aggregating past it.

**Attribution decay.** Agent involvement is recorded at change level. A feature touched by twelve changes, three of them agent-assisted, is not cleanly attributable. Reported at the aggregate level this washes out; reported per-feature it does not. Leadership metrics are therefore aggregate only.

**Work stream separation.** Platform and harness construction must be reported separately from team delivery. Building the measurement and governance capability consumes engineering capacity and produces no delivered work items — folded into delivery metrics it makes the programme look worse the more we invest in it. It should be presented as capability investment, on its own line.

**Maturity effects.** Early numbers reflect a capability being learned, not a capability at steady state. Four quarters of trend is the minimum honest basis for a scaling decision. A single quarter should not be used to kill or scale the programme, and agreeing that now protects the decision from the first noisy data point.

---

## 8. Phasing and cost of measurement

**Phase 1 — weeks 1–4. Instrument and validate.**
Tier 1 collection into the existing reporting stack. Output: squad-level flow and agent-effectiveness metrics. Success condition is that the numbers reconcile against the systems of record, not that they look good.

**Phase 2 — weeks 5–8. Baseline.**
Establish the non-agent comparison cohort and historical baseline. Agree cost assumptions with Finance in writing. No leadership reporting during this phase — reporting before the baseline exists produces a number with nothing to compare it to, and first impressions of a metric are difficult to correct later.

**Phase 3 — weeks 9–12. First leadership pack.**
Five metrics, trend where available, method and limitations attached.

**Cost of the measurement itself** should be stated explicitly in the pack: engineering effort to build and maintain collection, plus the tooling line. A measurement programme that consumes a material share of the capacity it is measuring is not worth running, and leadership is entitled to see that trade.

---

## 9. The decision this informs

Proposed decision gate at the end of Q[X], with thresholds agreed *now*:

- **Scale** — cost per qualified change trending down, quality differential neutral or better, governed path coverage above the agreed floor.
- **Hold and improve** — mixed signal, with a named hypothesis about the constraint and a defined next measurement window.
- **Redirect** — quality differential materially worse, or cost per qualified change flat with adoption above the threshold at which we would expect leverage.

Agreeing the thresholds before the data arrives is the difference between measurement and justification.

---

## Appendix A — Metric lineage

| Leadership metric | Source tier-1 signals | System of record |
|---|---|---|
| Cost per production-qualified change | Session token cost; PR merge and survival; reviewer time | Observability platform, source control |
| Engineer capacity redeployed | Cycle time delta by cohort; work item throughput | Work tracking |
| Quality differential | Change failure rate; 30-day revert/defect rate | Pipeline, incident records |
| Governed path coverage | Policy gate outcomes; audit trail completeness | Governance platform |
| Adoption share | Agent involvement flag on delivered items | Work tracking, observability platform |

## Appendix B — Open items for review

- Fully loaded engineer cost figure, and whether it may be used in a pack at this distribution level — requires Finance.
- Depth of usable historical baseline in the work tracking system prior to adoption.
- Audience and distribution: engineering leadership only, or wider BU.
- Whether governed path coverage has a defensible floor already defined, or needs one setting.
- Whether the quality differential should be reported against an agreed tolerance band rather than as a raw comparison.
'@
~~~~

**Part 12 of 19**

~~~~powershell
# Part 12 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'docs\losses-and-alternatives.md' @'
# Dropping LiteLLM: what we lose, and the options

## Why it was dropped

In March 2026, two LiteLLM releases (1.82.7 and 1.82.8) were published to
**PyPI** with a credential stealer. They were live for under an hour before
quarantine. The entry point was upstream: the TeamPCP campaign compromised the
Trivy scanner used in LiteLLM's CI, took publishing credentials, and pushed
the malicious versions directly. The same campaign also hit npm, GitHub
Actions, Docker Hub and OpenVSX.

The lesson isn't really "LiteLLM is unsafe". Any gateway is a prime target:
it holds every provider credential and sees every prompt. So whatever sits
in that position needs the tightest supply-chain controls we have. That
applies to Claude Code too, which ships through npm.

## What we lose

| LiteLLM gave us | Status without it | Mitigation |
|---|---|---|
| **GenAI OTel spans per model call** (Langfuse trace trees for every call) | Lost. APIM emits logs and metrics, not spans. | Langfuse gets traces from instrumented agents (OpenInference/OpenLLMetry, MCP gateway, own harness). Gateway-level calls are visible in New Relic only. |
| **One OpenAI-compatible API over every provider** | Lost. Clients speak each provider's native API. | Vertex has an OpenAI-compatible endpoint for some models. Claude on Vertex stays Anthropic-format. Agent frameworks (MAF, Pydantic AI) already abstract providers. |
| **Per-user virtual keys with USD budgets** | Partly replaced. APIM `llm-token-limit` gives per-user **token** quotas keyed on Entra identity. | Convert £750/month into a token quota per model mix. Alert on spend in New Relic before the hard limit. |
| **Cost per call** (built-in model price map) | Lost at the gateway. | Tokens × `model-prices.csv` lookup in New Relic, using contract rates rather than list prices. |
| **Fallbacks, retries, load balancing** | Covered. | APIM backend pools with circuit breakers. |
| **Caching** | Covered, with limits. | APIM semantic cache policies. Check they support your backends. |
| **Guardrail hooks** | Partly covered. | APIM content safety policy. mlang/ACS remains the semantic layer. |
| **Admin UI for keys and spend** | Lost. | APIM portal plus New Relic dashboards. Less convenient for self-service. |

**Net:** cost, quotas, routing and health are recoverable in APIM + New
Relic. The real losses are **per-call spans** and **format translation**.

## Options

### A. APIM only (current)
Everything in `clients/apim/ai-gateway-policy.xml`.
- **For:** no third-party code in the request path. Microsoft-supported,
  already owned, Entra-native, managed identity to backends.
- **Against:** no GenAI spans, token-based budgets, no format translation.
  Verify the `llm-*` policies parse usage from the Vertex backend.
- **Fits when:** visibility and spend control matter more than a unified API.

### B. Compiled gateway behind APIM
APIM stays the perimeter; a small, single-binary gateway sits behind it for
translation and spans. Candidates worth evaluating:
- **agentgateway** (Rust, Linux Foundation, Apache-2.0): LLM, MCP and A2A
  in one proxy with OTel. Closest fit to your MCP gateway and agent-traffic
  direction.
- **Envoy AI Gateway** (Apache-2.0): reached v1.0 in June 2026. Strong
  lineage, but Kubernetes-first, so a poor fit for Container Apps.
- **Bifrost** (Go, Apache-2.0): OTel GenAI traces, virtual keys, budgets.
  Vendor-backed (Maxim), and most comparison articles ranking it first are
  Maxim's own. Judge it on a spike, not on those.
- **For:** get spans and translation back. A compiled binary has a much
  smaller runtime dependency surface than a Python package tree.
- **Against:** another component to patch and own. Same supply-chain risk
  class, just smaller.

### C. Build a thin one
A small .NET proxy (e.g. YARP) or an extension of your MCP gateway that
forwards to APIM and emits GenAI OTel spans.
- **For:** C# fits the org, and the dependency tree is yours. It's the
  "buy the perimeter, build the semantics" pattern.
- **Against:** you own maintenance, and provider format translation is more
  work than it looks. Scope it to spans only, not translation.

## Controls that apply to any option

- Pin container images **by digest**, not tag. Verify signatures where published.
- Pull packages through an internal mirror (Azure Artifacts upstreams) with a
  minimum-age delay, so a poisoned release has time to be caught before you pull it.
- No provider keys in the gateway's environment. Use APIM managed identity to
  backends so a compromised process has nothing to steal.
- Lock gateway egress to known model endpoints plus the collector.
- Keep the gateway image off the build hosts that hold publishing credentials.
'@
W 'evals\README.md' @'
# Evals

Decision (Sep 2026): own the formats, rent the engines.

- Eval cases live as YAML in git (`evals/cases/*.yaml`), written from
  **observed failures** (e.g. stalled sessions, QA bounce-backs), not imagined ones.
- Any runner (Langfuse locally, GCP Agent Platform offline, a script) can
  execute them. Results go to New Relic as `EvalResult` events:

```bash
python adapters/github/emit_event.py EvalResult \
  --set suite=pricing-agent case_id=stall-001 passed=true gen_ai.request.model=claude-sonnet
```

- Keep work traces that contain prompts or code on the personal Langfuse
  instance until security has agreed data residency for eval datasets.
'@
W 'evals\cases\example.yaml' @'
# One eval case, written from an observed failure.
suite: pricing-agent
case_id: stall-001
observed_in: wi_01JD0000000000000000000000   # the work item where it happened
failure: Agent looped on a failing migration test without reading the error output.
input: |
  Minimal reproduction of the situation the agent was in.
expect:
  - reads the test output before retrying
  - stops after 3 identical failures and asks for help
grader: rubric      # rubric | exact | script
'@
W 'local\.env.example' @'
DEPLOY_ENV=local
ORG_BU=hiscox-re
INGEST_TOKEN=local-dev-token
NEW_RELIC_OTLP_ENDPOINT=https://otlp.eu01.nr-data.net
NEW_RELIC_LICENSE_KEY=replace-me
DEBUG_VERBOSITY=basic
# Optional, only with the Langfuse overlay. If Langfuse runs in another compose
# project, use host.docker.internal.
LANGFUSE_OTLP_ENDPOINT=http://host.docker.internal:3000/api/public/otel
# echo -n "pk-lf-xxx:sk-lf-xxx" | base64
LANGFUSE_AUTH_B64=replace-me
'@
W 'local\docker-compose.yaml' @'
# Local mirror of production: same image, same config, different env.
# Add the Langfuse overlay locally by uncommenting the command below
# (assumes your local Langfuse is reachable at LANGFUSE_OTLP_ENDPOINT).
services:
  collector:
    build: ../collector
    env_file: .env
    # command: ["--config=/etc/otelcol-contrib/config.yaml", "--config=/etc/otelcol-contrib/langfuse.yaml"]
    ports:
      - "4318:4318"     # OTLP/HTTP in
      - "13133:13133"   # health
    restart: unless-stopped
'@
~~~~

**Part 13 of 19**

~~~~powershell
# Part 13 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'newrelic\dashboard-queries.nrql' @'
-- Agentic Ops starter queries. Paste into dashboard widgets.
-- Upload as lookup tables (Data > Lookup tables):
--   org-lookup.csv     -> "org"
--   model-prices.csv   -> "prices"
--   data/aliases.csv   -> "aliases"   (re-upload after each Jira sync, or automate via NerdGraph)
-- Syntax notes: verify the JOIN form in your account's query builder;
-- NRQL lookup joins are newer than most of the docs you'll find.

-- ===== Spend (Claude Code estimate; APIM and GitHub remain the billing truth) =====
-- Cost per user this period
FROM Metric SELECT sum(claude_code.cost.usage) AS 'USD' FACET enduser.id SINCE 1 month ago

-- Cost by team / BU (via lookup)
FROM Metric JOIN (FROM lookup(org) SELECT enduser_id AS enduser.id, team, bu) ON enduser.id
  SELECT sum(claude_code.cost.usage) FACET bu, team SINCE 1 month ago

-- Budget burn: users over 80% of a ~$1k monthly budget
FROM Metric SELECT sum(claude_code.cost.usage) AS spend FACET enduser.id
  SINCE 1 month ago HAVING spend > 800

-- Tokens by type (input / output / cacheRead / cacheCreation) and model
FROM Metric SELECT sum(claude_code.token.usage) FACET type, model TIMESERIES AUTO

-- Cache hit ratio (a cheap efficiency win to track)
FROM Metric SELECT filter(sum(claude_code.token.usage), WHERE type = 'cacheRead')
  / filter(sum(claude_code.token.usage), WHERE type IN ('input','cacheRead','cacheCreation')) * 100
  AS 'cache hit %' TIMESERIES AUTO

-- ===== All harnesses (Claude Code + Copilot + own agents) =====
-- Spend by harness and billing source (canonical AIUsage events + Claude Code metrics)
FROM AIUsage SELECT sum(cost_usd) FACET harness, billing_source SINCE 1 month ago
-- Copilot AI credits per user (GitHub is the billing truth for Copilot)
FROM AIUsage SELECT sum(ai_credits_used) WHERE billing_source = 'github' FACET enduser.id SINCE 1 month ago
-- Copilot agent activity from its own OTel (spans follow GenAI semantic conventions)
FROM Span SELECT count(*), sum(gen_ai.usage.input_tokens), sum(gen_ai.usage.output_tokens)
  WHERE harness = 'copilot' FACET gen_ai.agent.name, gen_ai.request.model SINCE 1 week ago

-- ===== Attribution (the work-item thread) =====
-- Sessions started, and how many resolved to a ticket
FROM Log SELECT count(*) AS sessions,
  percentage(count(*), WHERE work_item.alias IS NOT NULL AND work_item.alias != '') AS 'attributed %'
  WHERE event.name = 'ai_session_started' FACET harness SINCE 1 week ago
-- Cost per work item (Claude Code): join session cost to the session's alias
FROM Metric JOIN (FROM Log SELECT latest(work_item.alias) AS alias WHERE event.name = 'ai_session_started'
  FACET session.id LIMIT MAX) ON session.id
  SELECT sum(claude_code.cost.usage) FACET alias SINCE 1 month ago LIMIT 100
-- Unattributed spend (no ticket in branch): track its share, it is a metric in itself
FROM Log SELECT uniqueCount(session.id) WHERE event.name = 'ai_session_started'
  AND (work_item.alias IS NULL OR work_item.alias = '') FACET enduser.id SINCE 1 week ago

-- ===== Delivery and outcomes =====
FROM CIRunCompleted SELECT percentage(count(*), WHERE conclusion = 'success') AS 'green %',
  average(attempt) FACET repo SINCE 1 month ago
FROM QAGatePassed, QABouncedBack SELECT count(*) FACET eventType() TIMESERIES 1 week SINCE 3 months ago
FROM TaskOutcome SELECT count(*) FACET outcome, task_class SINCE 3 months ago
-- Cost per passed task by model, within a task class only (model choice correlates with difficulty)
FROM TaskOutcome SELECT sum(cost_usd) / filter(count(*), WHERE outcome = 'pass') AS 'USD per passed task',
  filter(count(*), WHERE outcome = 'pass') AS passed, count(*) AS n
  FACET task_class, gen_ai.request.model SINCE 3 months ago

-- ===== Output (the start of the value view) =====
FROM Metric SELECT sum(claude_code.lines_of_code.count) FACET type TIMESERIES 1 day
FROM Metric SELECT sum(claude_code.commit.count) AS commits, sum(claude_code.pull_request.count) AS PRs FACET team SINCE 1 week ago
FROM Metric SELECT sum(claude_code.cost.usage) / sum(claude_code.pull_request.count) AS 'USD per PR' FACET team SINCE 1 month ago

-- ===== Agent behaviour (Claude Code events land as Logs) =====
-- Tool reliability
FROM Log SELECT percentage(count(*), WHERE success = 'true') AS 'success %', count(*)
  WHERE event.name = 'tool_result' FACET tool_name SINCE 1 day ago
-- API errors
FROM Log SELECT count(*) WHERE event.name = 'api_error' FACET model, status_code TIMESERIES AUTO
-- Edit acceptance (proxy for trust in the agent)
FROM Metric SELECT sum(claude_code.code_edit_tool.decision) FACET decision SINCE 1 week ago

-- ===== Agent and MCP gateway health (spans) =====
FROM Span SELECT percentile(duration.ms, 50, 95), count(*)
  WHERE service.name = 'mcp-gateway' FACET name TIMESERIES AUTO
FROM Span SELECT percentage(count(*), WHERE otel.status_code = 'ERROR')
  WHERE service.name = 'mcp-gateway' TIMESERIES AUTO

-- ===== Evals =====
FROM EvalResult SELECT percentage(count(*), WHERE passed = 'true') FACET suite, gen_ai.request.model SINCE 1 month ago

-- ===== Pipeline self-monitoring =====
-- Is anything arriving, and from which client / harness?
FROM Metric, Log, Span SELECT count(*) WHERE telemetry.pipeline = 'agentic-ops' FACET harness, client, eventType() SINCE 1 hour ago
-- Coverage: distinct users emitting per harness (compare with licensed seats)
FROM Metric, Log SELECT uniqueCount(enduser.id) WHERE telemetry.pipeline = 'agentic-ops' FACET harness SINCE 1 week ago

-- ===== APIM (source of truth for all non-Copilot model traffic) =====
-- Upload model-prices.csv as lookup table "prices". Fill in your contract rates.
-- Exact APIM attribute names depend on the Azure integration / Event Hub
-- parsing; confirm them in the debug exporter or NR data explorer first.
FROM Log SELECT count(*) WHERE client = 'apim' FACET enduser.id SINCE 1 day ago

-- Coverage check: are callers bypassing APIM? (Claude Code sessions vs APIM calls)
FROM Metric, Log SELECT filter(count(*), WHERE client = 'claude-code') AS cc,
  filter(count(*), WHERE client = 'apim') AS apim TIMESERIES 1 hour
'@
W 'newrelic\model-prices.csv' @'
model,input_usd_per_mtok,output_usd_per_mtok
replace-with-your-vertex-model,0,0
copilot-mixed,0,0
'@
W 'newrelic\org-lookup.csv' @'
enduser_id,team,squad,bu,monthly_budget_gbp
mike@example.com,pricing-platform,governed-ai,hiscox-re,750
'@
~~~~

**Part 14 of 19**

~~~~powershell
# Part 14 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'newrelic\value-view-exports.nrql' @'
-- Exports that feed valueview/value_view.py. Run each, export as CSV
-- (or pull via NerdGraph), rename columns to match, save into data/.
-- Window: one reporting period at a time keeps row counts manageable.

-- sessions.csv: session_id, harness, user_id, started_at, work_item_alias
FROM Log SELECT session.id AS session_id, harness, enduser.id AS user_id,
  timestamp AS started_at, work_item.alias AS work_item_alias
  WHERE event.name = 'ai_session_started' SINCE 3 months ago LIMIT MAX

-- usage.csv (Claude Code): usage_date, harness, model, billing_source, user_id,
--   session_id, input_tokens, output_tokens, cost_usd, ai_credits_used
FROM Metric SELECT sum(claude_code.cost.usage) AS cost_usd,
  filter(sum(claude_code.token.usage), WHERE type = 'input') AS input_tokens,
  filter(sum(claude_code.token.usage), WHERE type = 'output') AS output_tokens
  FACET dateOf(timestamp) AS usage_date, session.id AS session_id, enduser.id AS user_id, model
  SINCE 3 months ago LIMIT MAX
-- (add harness=claude-code, billing_source=apim as constant columns)

-- usage.csv (Copilot): from the canonical AIUsage events
FROM AIUsage SELECT usage_date, harness, gen_ai.request.model AS model, billing_source,
  enduser.id AS user_id, gen_ai.usage.input_tokens AS input_tokens,
  gen_ai.usage.output_tokens AS output_tokens, cost_usd, ai_credits_used
  WHERE billing_source = 'github' SINCE 3 months ago LIMIT MAX

-- outcomes.csv: work_item_id, outcome, attempt_n, bounced_back, reverted_30d
FROM TaskOutcome SELECT latest(outcome) AS outcome, max(attempt_n) AS attempt_n
  FACET work_item.id AS work_item_id SINCE 6 months ago LIMIT MAX
-- bounced_back: 1 if any QABouncedBack exists for the work item
FROM QABouncedBack SELECT count(*) FACET work_item.id SINCE 6 months ago LIMIT MAX

-- work_items.csv and aliases.csv come from the Jira adapter (data/), not New Relic.
'@
W 'obs\__init__.py' @'
"""Shared core for the BU AI observability layer.

ids      -> our own work-item IDs (wi_ + ULID)
aliases  -> alias table: jira:RE-1234 / ado:AB#5678 / git:branch/... -> wi_...
events   -> canonical event contract (CloudEvents 1.0 envelope)
nr       -> sender for New Relic's Event API (the PoV's event stream)
"""
'@
W 'obs\aliases.py' @'
"""Alias table: every external key resolves to one of our work-item IDs.

Alias forms (system:key):
  jira:RE-1234        a Jira issue key
  ado:AB#5678         an Azure DevOps work item
  git:<branch name>   a branch; resolved through the ticket key inside it

Resolution happens at the edge (adapters, value view). Nothing downstream
ever sees a raw Jira key as an identifier: parse, don't validate.

Storage is a plain CSV (work_item_id, alias, created_at) so it can be uploaded
to New Relic as a lookup table and read by Power BI without a database.
"""
from __future__ import annotations

import csv
import os
import re
from datetime import datetime, timezone

from .ids import WI_UNASSIGNED, is_work_item_id, new_work_item_id

DEFAULT_TICKET_PATTERN = r"([A-Z][A-Z0-9]+-[0-9]+|AB#[0-9]+)"
FIELDS = ["work_item_id", "alias", "created_at"]


def ticket_key_from_text(text: str | None, pattern: str | None = None) -> str | None:
    m = re.search(pattern or DEFAULT_TICKET_PATTERN, text or "")
    return m.group(1) if m else None


def normalise(alias: str | None, pattern: str | None = None) -> str | None:
    """Turn any alias form into its canonical tracker alias.

    'jira:RE-1234' -> 'jira:RE-1234'
    'RE-1234'      -> 'jira:RE-1234'
    'AB#5678'      -> 'ado:AB#5678'
    'git:RE-1234-fix-pricing' or a bare branch -> 'jira:RE-1234'
    Anything without a ticket key -> None
    """
    if not alias:
        return None
    alias = alias.strip()
    system, _, rest = alias.partition(":")
    if rest and system in ("jira", "ado"):
        return f"{system}:{rest}"
    text = rest if (rest and system == "git") else alias
    key = ticket_key_from_text(text, pattern)
    if not key:
        return None
    return f"ado:{key}" if key.startswith("AB#") else f"jira:{key}"


class AliasTable:
    def __init__(self, path: str | None = None, pattern: str | None = None):
        self.path = path
        self.pattern = pattern or os.environ.get("AI_OBS_TICKET_REGEX") or DEFAULT_TICKET_PATTERN
        self._by_alias: dict[str, str] = {}
        self._rows: list[dict] = []
        if path and os.path.exists(path):
            with open(path, newline="", encoding="utf-8") as f:
                for row in csv.DictReader(f):
                    self._add(row["work_item_id"], row["alias"], row.get("created_at", ""))

    def _add(self, wi: str, alias: str, created_at: str) -> None:
        if not is_work_item_id(wi):
            raise ValueError(f"not a work-item id: {wi!r}")
        if alias in self._by_alias and self._by_alias[alias] != wi:
            raise ValueError(f"alias {alias} already points at {self._by_alias[alias]}")
        if alias not in self._by_alias:
            self._by_alias[alias] = wi
            self._rows.append({"work_item_id": wi, "alias": alias, "created_at": created_at})

    def resolve(self, alias_or_branch: str | None) -> str:
        """Any alias or branch -> work-item ID, or WI_UNASSIGNED. Never raises."""
        canon = normalise(alias_or_branch, self.pattern)
        return self._by_alias.get(canon, WI_UNASSIGNED) if canon else WI_UNASSIGNED

    def ensure(self, alias: str) -> str:
        """Return the work-item ID for a tracker alias, minting one if new."""
        canon = normalise(alias, self.pattern)
        if not canon:
            raise ValueError(f"no ticket key in alias {alias!r}")
        if canon in self._by_alias:
            return self._by_alias[canon]
        wi = new_work_item_id()
        self._add(wi, canon, datetime.now(timezone.utc).isoformat(timespec="seconds"))
        return wi

    def link(self, new_alias: str, existing_alias: str) -> str:
        """Migration: point a new tracker's key at an existing work item.
        e.g. link('ado:AB#5678', 'jira:RE-1234') keeps all history joined."""
        wi = self.resolve(existing_alias)
        if wi == WI_UNASSIGNED:
            raise ValueError(f"{existing_alias} is not in the table")
        canon = normalise(new_alias, self.pattern)
        if not canon:
            raise ValueError(f"no ticket key in alias {new_alias!r}")
        self._add(wi, canon, datetime.now(timezone.utc).isoformat(timespec="seconds"))
        return wi

    def aliases_of(self, wi: str) -> list[str]:
        return [r["alias"] for r in self._rows if r["work_item_id"] == wi]

    def save(self, path: str | None = None) -> None:
        target = path or self.path
        if not target:
            raise ValueError("no path to save to")
        tmp = target + ".tmp"
        with open(tmp, "w", newline="", encoding="utf-8") as f:
            w = csv.DictWriter(f, fieldnames=FIELDS)
            w.writeheader()
            w.writerows(self._rows)
        os.replace(tmp, target)

    def __len__(self) -> int:
        return len(self._rows)
'@
~~~~

**Part 15 of 19**

~~~~powershell
# Part 15 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'obs\events.py' @'
"""Canonical event contract v0 (see contracts/events.md).

Every event is a CloudEvents 1.0 envelope. `subject` is always our
work-item ID (or wi_unassigned), so every consumer can join on it.

Attribute names follow OTel GenAI semantic conventions where one exists;
in the flat New Relic form, dots are kept (gen_ai.request.model).
"""
from __future__ import annotations

from datetime import datetime, timezone
from typing import Any

from .ids import WI_UNASSIGNED, is_work_item_id, ulid

SPEC_VERSION = "1.0"
TYPE_PREFIX = "com.hiscox.re.ai."
CONTRACT_VERSION = "v0"

# type -> (required data fields, optional data fields)
CONTRACT: dict[str, tuple[tuple[str, ...], tuple[str, ...]]] = {
    # work
    "WorkItemCreated": (("alias",), ("task_class", "task_size", "team", "work_stream", "created_at")),
    "WorkItemStatusChanged": (("alias", "to_status"), ("from_status", "status_category", "changed_at")),
    "WorkItemDone": (("alias", "done_at"), ("resolution",)),
    # AI
    "AISessionStarted": (("session.id", "harness"), ("enduser.id", "git.branch", "git.repo", "work_item.alias", "session.source")),
    "AIUsage": (("harness", "gen_ai.request.model", "billing_source"),
                ("session.id", "enduser.id", "gen_ai.usage.input_tokens", "gen_ai.usage.output_tokens",
                 "cost_usd", "ai_credits_used", "usage_date")),
    # delivery
    "CIRunCompleted": (("repo", "run_id", "conclusion"), ("branch", "attempt", "duration_s", "workflow")),
    "PRMerged": (("repo", "pr_number"), ("branch", "merged_at")),
    "QAGatePassed": (("attempt_n",), ("repo", "branch", "mutation_score")),
    "QABouncedBack": (("attempt_n",), ("reason",)),
    # outcomes and evaluation
    "TaskOutcome": (("outcome", "attempt_n"), ("gen_ai.request.model", "task_class", "task_size", "cost_usd", "wallclock_hours")),
    "EvalResult": (("suite", "case_id", "passed"), ("gen_ai.request.model", "score", "harness_version")),
    "BenchRun": (("bench.version", "gen_ai.request.model", "outcome"),
                 ("task_class", "task_size", "attempt_n", "cost_usd", "turns", "tool_calls", "tool_errors",
                  "wallclock_s", "mutation_score", "crap_delta")),
}

OUTCOMES = {"pass", "reworked", "fail", "abandoned"}
HARNESSES = {"claude-code", "copilot", "copilot-cli", "copilot-sdk", "own-agent", "apim", "unknown"}
BILLING_SOURCES = {"apim", "github", "anthropic", "vertex", "unknown"}


class ContractError(ValueError):
    pass


def make_event(event_type: str, source: str, subject: str | None, data: dict[str, Any],
               time: datetime | None = None, event_id: str | None = None,
               alias: str | None = None) -> dict[str, Any]:
    """`alias` (CloudEvents extension `workitemalias`) carries the raw tracker
    key when the emitter cannot resolve it (e.g. a CI job without the alias
    table). Consumers re-resolve wi_unassigned events through it."""
    if event_type not in CONTRACT:
        raise ContractError(f"unknown event type {event_type!r}")
    ev = {
        "specversion": SPEC_VERSION,
        "id": event_id or ulid(),
        "source": source,
        "type": f"{TYPE_PREFIX}{event_type}.{CONTRACT_VERSION}",
        "subject": subject or WI_UNASSIGNED,
        "time": (time or datetime.now(timezone.utc)).isoformat(timespec="seconds"),
        "datacontenttype": "application/json",
        "data": {k: v for k, v in data.items() if v is not None and v != ""},
    }
    if alias:
        ev["workitemalias"] = alias
    validate(ev)
    return ev


def short_type(ev: dict[str, Any]) -> str:
    t = ev.get("type", "")
    if not t.startswith(TYPE_PREFIX):
        raise ContractError(f"type {t!r} is not ours")
    return t[len(TYPE_PREFIX):].rsplit(".", 1)[0]


def validate(ev: dict[str, Any]) -> None:
    for k in ("specversion", "id", "source", "type", "subject", "time", "data"):
        if k not in ev:
            raise ContractError(f"missing envelope field {k}")
    if ev["specversion"] != SPEC_VERSION:
        raise ContractError("specversion must be 1.0")
    if not is_work_item_id(ev["subject"]):
        raise ContractError(f"subject must be a work-item id, got {ev['subject']!r}")
    name = short_type(ev)
    if name not in CONTRACT:
        raise ContractError(f"unknown event type {name!r}")
    required, optional = CONTRACT[name]
    data = ev["data"]
    missing = [f for f in required if f not in data]
    if missing:
        raise ContractError(f"{name}: missing {missing}")
    unknown = [f for f in data if f not in required and f not in optional]
    if unknown:
        raise ContractError(f"{name}: fields not in contract {unknown}")
    for prompt_like in ("prompt", "completion", "messages", "gen_ai.prompt", "gen_ai.completion"):
        if prompt_like in data:
            raise ContractError("content fields are never allowed in events")
    if name in ("TaskOutcome", "BenchRun") and data["outcome"] not in OUTCOMES:
        raise ContractError(f"outcome must be one of {sorted(OUTCOMES)}")
    if name in ("AISessionStarted", "AIUsage") and data["harness"] not in HARNESSES:
        raise ContractError(f"harness must be one of {sorted(HARNESSES)}")
    if name == "AIUsage" and data["billing_source"] not in BILLING_SOURCES:
        raise ContractError(f"billing_source must be one of {sorted(BILLING_SOURCES)}")


def to_newrelic(ev: dict[str, Any], org_bu: str = "hiscox-re") -> dict[str, Any]:
    """Flatten an envelope into one New Relic custom event."""
    validate(ev)
    ts = datetime.fromisoformat(ev["time"])
    flat = {
        "eventType": short_type(ev),
        "timestamp": int(ts.timestamp() * 1000),
        "work_item.id": ev["subject"],
        "ce.id": ev["id"],
        "ce.source": ev["source"],
        "ce.type": ev["type"],
        "org.bu": org_bu,
        "telemetry.pipeline": "agentic-ops",
    }
    if ev.get("workitemalias"):
        flat["work_item.alias"] = ev["workitemalias"]
    for k, v in ev["data"].items():
        flat[k] = v if isinstance(v, (int, float, str, bool)) else str(v)
    return flat
'@
~~~~

**Part 16 of 19**

~~~~powershell
# Part 16 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'obs\ids.py' @'
"""Work-item IDs: 'wi_' + ULID (26 chars, Crockford base32, time-sortable).

We mint these ourselves. Jira keys, Azure DevOps IDs and branch names are
only aliases that point at one of them, so moving trackers never re-keys
history.
"""
from __future__ import annotations

import os
import time

_CROCKFORD = "0123456789ABCDEFGHJKMNPQRSTVWXYZ"
WI_PREFIX = "wi_"
WI_UNASSIGNED = "wi_unassigned"


def ulid(ts_ms: int | None = None, rand: bytes | None = None) -> str:
    ts = int(time.time() * 1000) if ts_ms is None else ts_ms
    if not 0 <= ts < 2 ** 48:
        raise ValueError("timestamp out of ULID range")
    r = os.urandom(10) if rand is None else rand
    if len(r) != 10:
        raise ValueError("ULID randomness must be 10 bytes")
    n = (ts << 80) | int.from_bytes(r, "big")
    out = []
    for _ in range(26):
        out.append(_CROCKFORD[n & 31])
        n >>= 5
    return "".join(reversed(out))


def new_work_item_id(ts_ms: int | None = None) -> str:
    return WI_PREFIX + ulid(ts_ms)


def is_work_item_id(value: str | None) -> bool:
    if not value or not value.startswith(WI_PREFIX):
        return False
    body = value[len(WI_PREFIX):]
    return body == "unassigned" or (len(body) == 26 and all(c in _CROCKFORD for c in body))
'@
W 'obs\nr.py' @'
"""Send canonical events to New Relic's Event API (the PoV's event stream).

Uses the ingest licence key (same key the collector uses). EU accounts use
the eu01 endpoint. With --dry-run / NR_DRY_RUN=1 nothing is sent; events are
appended to an NDJSON file instead, which is also handy for backfills.
"""
from __future__ import annotations

import gzip
import json
import os
import urllib.request
from typing import Any, Iterable

from .events import to_newrelic

ENDPOINTS = {
    "eu": "https://insights-collector.eu01.nr-data.net/v1/accounts/{account}/events",
    "us": "https://insights-collector.newrelic.com/v1/accounts/{account}/events",
}
MAX_EVENTS_PER_POST = 1000
MAX_BYTES_PER_POST = 900_000  # API limit is 1 MB compressed; stay under it uncompressed


class NewRelicSender:
    def __init__(self, account_id: str | None = None, key: str | None = None,
                 region: str | None = None, dry_run_path: str | None = None, org_bu: str | None = None):
        self.account_id = account_id or os.environ.get("NEW_RELIC_ACCOUNT_ID")
        self.key = key or os.environ.get("NEW_RELIC_LICENSE_KEY")
        self.region = (region or os.environ.get("NEW_RELIC_REGION") or "eu").lower()
        self.org_bu = org_bu or os.environ.get("ORG_BU") or "hiscox-re"
        env_dry = os.environ.get("NR_DRY_RUN") == "1"
        self.dry_run_path = dry_run_path or (os.environ.get("NR_DRY_RUN_PATH", "events.ndjson") if env_dry else None)
        if not self.dry_run_path and not (self.account_id and self.key):
            raise SystemExit("Set NEW_RELIC_ACCOUNT_ID and NEW_RELIC_LICENSE_KEY, or use a dry run")

    def send(self, envelopes: Iterable[dict[str, Any]]) -> int:
        flat = [to_newrelic(e, self.org_bu) for e in envelopes]
        if not flat:
            return 0
        if self.dry_run_path:
            with open(self.dry_run_path, "a", encoding="utf-8") as f:
                for e in flat:
                    f.write(json.dumps(e) + "\n")
            return len(flat)
        sent = 0
        for batch in _batches(flat):
            body = gzip.compress(json.dumps(batch).encode())
            req = urllib.request.Request(
                ENDPOINTS[self.region].format(account=self.account_id), data=body, method="POST",
                headers={"Api-Key": self.key, "Content-Type": "application/json", "Content-Encoding": "gzip"})
            with urllib.request.urlopen(req, timeout=30) as resp:
                if resp.status >= 300:
                    raise RuntimeError(f"New Relic returned {resp.status}")
            sent += len(batch)
        return sent


def _batches(events: list[dict]) -> Iterable[list[dict]]:
    batch, size = [], 2
    for e in events:
        n = len(json.dumps(e)) + 1
        if batch and (len(batch) >= MAX_EVENTS_PER_POST or size + n > MAX_BYTES_PER_POST):
            yield batch
            batch, size = [], 2
        batch.append(e)
        size += n
    if batch:
        yield batch
'@
~~~~

**Part 17 of 19**

~~~~powershell
# Part 17 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'tests\run_all.py' @'
#!/usr/bin/env python3
"""Run every offline check. No network, no secrets needed.
  python tests/run_all.py
"""
import io
import json
import os
import subprocess
import sys
import tempfile

ROOT = os.path.abspath(os.path.join(os.path.dirname(__file__), ".."))
sys.path.insert(0, ROOT)
os.chdir(tempfile.mkdtemp(prefix="obs_tests_"))

from obs.aliases import AliasTable, normalise  # noqa: E402
from obs.events import ContractError, make_event, to_newrelic, validate  # noqa: E402
from obs.ids import WI_UNASSIGNED, is_work_item_id, new_work_item_id, ulid  # noqa: E402

ok = []

# ids
a, b = ulid(1_700_000_000_000), ulid(1_700_000_000_001)
assert len(a) == 26 and a < b, "ULIDs must be fixed-length and time-sortable"
assert is_work_item_id(new_work_item_id()) and is_work_item_id(WI_UNASSIGNED) and not is_work_item_id("RE-1")
ok.append("work-item IDs")

# aliases
assert normalise("RE-1234") == "jira:RE-1234"
assert normalise("git:feature/RE-1234-fix-pricing") == "jira:RE-1234"
assert normalise("AB#5678") == "ado:AB#5678"
assert normalise("git:main") is None
t = AliasTable(os.path.join(os.getcwd(), "aliases.csv"))
wi = t.ensure("jira:RE-1234")
assert t.ensure("RE-1234") == wi and t.resolve("RE-1234-some-branch") == wi
assert t.resolve("main") == WI_UNASSIGNED
assert t.link("ado:AB#5678", "jira:RE-1234") == wi and t.resolve("AB#5678") == wi
t.save()
assert AliasTable(t.path).resolve("ado:AB#5678") == wi
ok.append("alias table: resolve, mint, migrate, persist")

# events
ev = make_event("AIUsage", "/test", wi, {"harness": "copilot", "gen_ai.request.model": "m",
                                         "billing_source": "github", "ai_credits_used": 3})
validate(ev)
flat = to_newrelic(ev)
assert flat["eventType"] == "AIUsage" and flat["work_item.id"] == wi and flat["org.bu"] == "hiscox-re"
for bad in ({"harness": "copilot", "gen_ai.request.model": "m", "billing_source": "github", "prompt": "x"},
            {"harness": "nope", "gen_ai.request.model": "m", "billing_source": "github"},
            {"harness": "copilot"}):
    try:
        make_event("AIUsage", "/test", wi, bad)
        raise AssertionError(f"should have rejected {bad}")
    except ContractError:
        pass
try:
    make_event("PRMerged", "/test", "RE-1234", {"repo": "r", "pr_number": 1})
    raise AssertionError("raw Jira key as subject must be rejected")
except ContractError:
    pass
ok.append("event contract: validation, content ban, raw keys rejected")


def run(*args, env=None):
    r = subprocess.run([sys.executable, *args], capture_output=True, text=True,
                       env={**os.environ, **(env or {})})
    if r.returncode != 0:
        raise AssertionError(f"{args}: {r.stderr or r.stdout}")
    return r.stdout


# Jira adapter (fixture, dry run), twice: second run must not re-emit
env = {"JIRA_TEAM_FIELD": "customfield_10001", "JIRA_POINTS_FIELD": "customfield_10016"}
fx = os.path.join(ROOT, "adapters", "jira", "fixture.json")
run(os.path.join(ROOT, "adapters", "jira", "sync_work_items.py"), "--fixture", fx, "--dry-run", env=env)
first = [json.loads(l) for l in open("events.ndjson")]
run(os.path.join(ROOT, "adapters", "jira", "sync_work_items.py"), "--fixture", fx, "--dry-run", env=env)
second = [json.loads(l) for l in open("events.ndjson")][len(first):]
types = sorted(e["eventType"] for e in first)
assert types.count("WorkItemCreated") == 3 and types.count("WorkItemDone") == 1, types
assert not second, "second sync with no changes must emit nothing"
created = {e["alias"]: e for e in first if e["eventType"] == "WorkItemCreated"}
assert created["jira:RE-1203"]["work_stream"] == "platform" and created["jira:RE-1202"]["task_class"] == "data"
ok.append("Jira adapter: mints IDs, emits once, classifies")

# GitHub emitter resolves through the alias table when it has one
os.remove("events.ndjson")
run(os.path.join(ROOT, "adapters", "github", "emit_event.py"), "CIRunCompleted", "--branch", "RE-1201-tidy",
    "--aliases", "data/aliases.csv", "--set", "repo=x/y", "run_id=1", "conclusion=success", "attempt=2", "--dry-run")
e = json.loads(open("events.ndjson").read().splitlines()[-1])
assert e["work_item.id"].startswith("wi_") and e["work_item.id"] != WI_UNASSIGNED and e["attempt"] == 2
ok.append("GitHub emitter: branch -> work item")

# Copilot usage parser
with open("copilot.ndjson", "w") as f:
    f.write(json.dumps({"user_login": "Dev1", "date": "2026-11-17", "ai_credits_used": 14,
                        "token_usage": {"prompt_tokens_sum": 1000, "output_tokens_sum": 200}}) + "\n")
    f.write("not json\n")
    f.write(json.dumps({"something": "else"}) + "\n")
out = run(os.path.join(ROOT, "adapters", "github", "copilot_usage.py"), "--ndjson", "copilot.ndjson", "--dry-run")
assert "3 records -> 1 AIUsage" in out or "2 records -> 1 AIUsage" in out, out
ok.append("Copilot usage: defensive parsing")

# Claude Code hook (python variant): body shape
sys.path.insert(0, os.path.join(ROOT, "clients", "claude-code", "hooks"))
import session_start  # noqa: E402
assert session_start.alias_from_branch("RE-77-x") == "jira:RE-77"
body = session_start.build_body("sess-1", "startup", "RE-77-x", "repo", "jira:RE-77", {"enduser.id": "a@b.c"})
attrs = {a["key"]: a["value"]["stringValue"] for a in body["resourceLogs"][0]["scopeLogs"][0]["logRecords"][0]["attributes"]}
assert attrs["session.id"] == "sess-1" and attrs["work_item.alias"] == "jira:RE-77"
ok.append("SessionStart hook payload")

# value view self-test
print(run(os.path.join(ROOT, "valueview", "value_view.py"), "--selftest"))
ok.append("value view self-test")

for o in ok:
    print("ok:", o)
print(f"\nall {len(ok)} checks passed")
'@
~~~~

**Part 18 of 19**

~~~~powershell
# Part 18 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'tools\dev-setup.ps1' @'
# One-time dev setup. Run once per machine:
#   powershell -ExecutionPolicy Bypass -File tools\dev-setup.ps1 -Team pricing-platform
# It sets your identity attributes, checks that IT's managed settings are in
# place, and sends one test event so you can see you're covered.
param(
  [Parameter(Mandatory)] [string] $Team,
  [string] $Bu = 'hiscox-re',
  [string] $Collector = $env:OTEL_EXPORTER_OTLP_ENDPOINT,
  [string] $Token
)
$here = Split-Path -Parent $MyInvocation.MyCommand.Path
$root = Split-Path -Parent $here

Write-Output '1/4 Identity attributes'
& (Join-Path $root 'clients\claude-code\set-identity.ps1') -Team $Team -Bu $Bu

Write-Output '2/4 Managed settings and hook'
# VERIFY the managed-settings path for your Claude Code version.
$managed = 'C:\Program Files\ClaudeCode\managed-settings.json'
$hook = 'C:\ProgramData\ai-obs\session-start.ps1'
foreach ($p in @($managed, $hook)) {
  if (Test-Path $p) { Write-Output "  ok  $p" } else { Write-Output "  MISSING  $p  (ask IT to push it, or copy it for a local test)" }
}

Write-Output '3/4 Branch convention'
$branch = (git branch --show-current 2>$null | Out-String).Trim()
$pattern = if ($env:AI_OBS_TICKET_REGEX) { $env:AI_OBS_TICKET_REGEX } else { '([A-Z][A-Z0-9]+-[0-9]+|AB#[0-9]+)' }
if (-not $branch) { Write-Output '  (not in a git repo; skipped)' }
elseif ($branch -match $pattern) { Write-Output "  ok  '$branch' -> $($Matches[1])" }
else { Write-Output "  WARN  '$branch' has no ticket key: sessions here count as unassigned" }

Write-Output '4/4 Test event'
if (-not $Collector) { Write-Output '  skipped: pass -Collector https://<fqdn> (or set OTEL_EXPORTER_OTLP_ENDPOINT)'; exit 0 }
if (-not $Token -and $env:OTEL_EXPORTER_OTLP_HEADERS -match 'Bearer\s+([^,]+)') { $Token = $Matches[1] }
$user = (whoami /upn).Trim().ToLower()
$payload = @{ session_id = "setup-test-$([guid]::NewGuid().ToString('N').Substring(0,8))"; cwd = (Get-Location).Path; source = 'dev-setup' } | ConvertTo-Json -Compress
$env:OTEL_EXPORTER_OTLP_ENDPOINT = $Collector
if ($Token) { $env:OTEL_EXPORTER_OTLP_HEADERS = "Authorization=Bearer $Token" }
$payload | powershell -NoProfile -ExecutionPolicy Bypass -File (Join-Path $root 'clients\claude-code\hooks\session-start.ps1')
Write-Output "  sent. In New Relic: FROM Log SELECT * WHERE event.name = 'ai_session_started' AND enduser.id = '$user' SINCE 10 minutes ago"
'@
W 'tools\make_paste_blocks.py' @'
#!/usr/bin/env python3
"""Turn the repo into PowerShell paste blocks, for machines where zip files
and GitHub can't reach (the work VM). Each part writes its files under
$HOME\\bu-ai-observability and prints 'wrote <path>' per file.

  python tools/make_paste_blocks.py > paste-blocks.md
"""
import os
import sys

ROOT = os.path.abspath(os.path.join(os.path.dirname(__file__), ".."))
SKIP_DIRS = {".git", "__pycache__", "data", "out", "dist"}
SKIP_FILES = {"events.ndjson", "paste-blocks.md"}
MAX_CHARS = 7000

HEADER = ("$root = Join-Path $HOME 'bu-ai-observability'\n"
          "$utf8 = New-Object System.Text.UTF8Encoding($false)\n"
          "function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force "
          "(Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace \"`r`n\", \"`n\") + \"`n\", $utf8); "
          "Write-Output \"wrote $rel\" }")


def files():
    out = []
    for d, dirs, fs in os.walk(ROOT):
        dirs[:] = sorted(x for x in dirs if x not in SKIP_DIRS)
        for f in sorted(fs):
            if f in SKIP_FILES or f.endswith(".pyc"):
                continue
            p = os.path.join(d, f)
            text = open(p, encoding="utf-8").read()
            if "\n'@" in text or text.startswith("'@"):
                sys.exit(f"{p} contains a here-string terminator; cannot embed safely")
            out.append((os.path.relpath(p, ROOT).replace("/", "\\"), text.rstrip("\n")))
    return out


def chunks(items):
    parts, cur, size = [], [], 0
    for rel, text in items:
        if cur and size + len(text) > MAX_CHARS:
            parts.append(cur)
            cur, size = [], 0
        cur.append((rel, text))
        size += len(text)
    if cur:
        parts.append(cur)
    return parts


def main():
    parts = chunks(files())
    print("Paste each part into PowerShell, in order. Each prints a 'wrote ...' line per file.\n")
    for i, part in enumerate(parts, 1):
        body = [f"# Part {i} of {len(parts)}", HEADER]
        body += [f"W '{rel}' @'\n{text}\n'@" for rel, text in part]
        print(f"**Part {i} of {len(parts)}**\n\n~~~~powershell\n" + "\n".join(body) + "\n~~~~\n")


if __name__ == "__main__":
    main()
'@
W 'valueview\__init__.py' @'

'@
~~~~

**Part 19 of 19**

~~~~powershell
# Part 19 of 19
$root = Join-Path $HOME 'bu-ai-observability'
$utf8 = New-Object System.Text.UTF8Encoding($false)
function W($rel, $txt) { $p = Join-Path $root $rel; New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null; [IO.File]::WriteAllText($p, ($txt -replace "`r`n", "`n") + "`n", $utf8); Write-Output "wrote $rel" }
W 'valueview\value_view.py' @'
#!/usr/bin/env python3
"""
value_view.py — spend next to what it bought, for devs and leadership.

Rebuilt (Nov 2026) from the September value view + leadership rollup, now
reading the canonical events (exported from New Relic) instead of the old
tier-1 CSVs. One script, one long-format output; Power BI pages filter on
audience and org level, so nobody runs a second collection.

Inputs (CSV, in --indir; see newrelic/value-view-exports.nrql):
  sessions.csv      session_id, harness, user_id, started_at, work_item_alias
  usage.csv         usage_date, harness, model, billing_source, user_id, session_id,
                    input_tokens, output_tokens, cost_usd, ai_credits_used
  work_items.csv    work_item_id, alias, task_class, task_size, team, work_stream,
                    status, created_at, done_at        (written by the Jira adapter)
  outcomes.csv      work_item_id, outcome, attempt_n, bounced_back, reverted_30d
  aliases.csv       work_item_id, alias               (the alias table)
  model-prices.csv  optional  model, input_usd_per_mtok, output_usd_per_mtok
  org_units.csv     optional  squad, team, bu, jira_team          (no people)
  org_members.csv   optional  RESTRICTED. user_hash, squad, valid_from, valid_to
  seats.csv         optional  user_id, harness                    (licensed seats)
  bench_runs.csv    optional  model, task_class, outcome, cost_usd (replay benchmark)

Output (--outdir):
  value_view.csv        period, org_level, org_unit, metric, cohort, segment_key,
                        segment_value, value, unit, n, ci_low, ci_high, suppressed, method_note
  metric_catalogue.csv  metric, audience, unit, how_to_read

Rules enforced in code, not left to the reporting layer:
  1. Individuals never reach the output. User IDs are HMAC-hashed on read with
     $VALUE_VIEW_SALT and the raw value is dropped; the schema has no user field.
     If org_members.csv is present and the salt is not, the script refuses to run.
  2. Cells with n < MIN_CELL_SIZE are emitted suppressed, never aggregated past.
  3. Any unit-period with fewer than MIN_DISTINCT_USERS distinct AI users has
     every AI-derived row suppressed, however many sessions it has.
  4. Complementary suppression: when a parent unit has exactly one hidden child,
     the smallest visible sibling is hidden too (no subtraction attacks).
  5. Squads come from the work item first (Jira team field), and only fall back
     to the squad of the people whose sessions served it.
  6. Only work_stream == team_delivery counts toward delivery metrics; platform
     and R&D spend is reported on its own line as capability investment.
  7. Agent vs non-agent comparisons are always segmented by task class;
     unsegmented headline comparisons are not emitted (selection bias).
  8. Unattributed spend is never called waste. It is reported on its own line.
  9. Cycle time is never converted to money.
 10. Raw pass rate by model is only compared within a task class (model choice
     correlates with difficulty); the replay benchmark is the authoritative view.

Self-test (synthetic data, asserts the privacy properties):
  python valueview/value_view.py --selftest
"""
from __future__ import annotations

import argparse
import csv
import hashlib
import hmac
import math
import os
import random
import secrets
import sys
import tempfile
from collections import Counter, defaultdict
from datetime import date, datetime, timedelta
from statistics import median

sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "..")))

from obs.aliases import AliasTable  # noqa: E402
from obs.ids import WI_UNASSIGNED  # noqa: E402

MIN_CELL_SIZE = 5
MIN_DISTINCT_USERS = 3
CURRENCY = "USD"
OUT_FIELDS = ["period", "org_level", "org_unit", "metric", "cohort", "segment_key", "segment_value",
              "value", "unit", "n", "ci_low", "ci_high", "suppressed", "method_note"]
LEVELS = ["squad", "team", "bu", "all"]
PARENT = {"squad": "team", "team": "bu", "bu": "all"}

CATALOGUE = [
    # metric, audience, unit, how to read
    ("cost_per_delivered_work_item", "leadership", CURRENCY, "Headline. AI spend on work items that reached Done and survived 30 days, per item. Read the trend, not the level."),
    ("first_pass_qa_rate", "leadership", "ratio", "Share of done items that passed the QA gate first time. Agent vs non-agent only within a task class."),
    ("change_failure_rate", "leadership", "ratio", "Quality differential: share of done items reverted within 30 days, by cohort and task class."),
    ("adoption_share", "leadership", "ratio", "Share of done items with any AI session. The multiplier on everything else."),
    ("capacity_redeployed", "leadership", "hours", "Not derivable from telemetry. Needs a named statement of what freed capacity moved to."),
    ("governed_path_coverage", "leadership", "ratio", "Not yet computed: no governance field in the events."),
    ("spend_total", "platform", CURRENCY, "All AI spend in the period, every harness and billing source."),
    ("spend_by_harness", "platform", CURRENCY, "Spend split by Claude Code, Copilot, own agents, APIM."),
    ("spend_by_billing_source", "platform", CURRENCY, "Which bill each dollar reconciles against (APIM, GitHub, ...)."),
    ("spend_by_model", "platform", CURRENCY, "Spend split by model."),
    ("spend_by_work_stream", "leadership", CURRENCY, "Team delivery vs capability investment (platform, R&D)."),
    ("spend_by_outcome", "eng", CURRENCY, "Attributed spend split into qualified, failed, no merged change, in progress; unattributed on its own line."),
    ("waste_share", "eng", "ratio", "Failed + no-merged-change spend over closed attributed spend. Only as trustworthy as attribution_rate."),
    ("attribution_rate", "platform", "ratio", "Sessions resolved to a work item. Below ~80% the value numbers are shaky."),
    ("attributed_spend_share", "platform", "ratio", "Share of spend that reached a work item."),
    ("session_success_rate", "dev", "ratio", "Attributed sessions whose work item passed the QA gate first time without bounce-back."),
    ("qa_bounce_back_rate", "eng", "ratio", "Share of done items bounced back from QA."),
    ("cycle_time_days", "eng", "days", "Median created -> done, by cohort within task class. Never converted to money."),
    ("pass_rate_by_model", "platform", "ratio", "Observed first-pass rate by dominant model, within task class. 95% Wilson CI; overlapping CIs = equivalent."),
    ("cost_per_passed_task", "leadership", CURRENCY, "Cost per passed task by model x task class. Bench cohort is authoritative."),
    ("coverage", "platform", "ratio", "Licensed users who emitted telemetry. Spend is understated by the gap."),
    ("org_attribution_source", "platform", "ratio", "How work items got their squad: Jira team field vs session users."),
    ("unpriced_usage_rows", "platform", "count", "Usage rows with no cost and no price to compute one. Should be 0."),
]
AI_DERIVED = {m for m, *_ in CATALOGUE} - {"capacity_redeployed", "governed_path_coverage"}
WORK_ONLY_COHORTS = {"all", "non_agent"}  # cohorts of work-only metrics that are not AI-derived


# --------------------------------------------------------------------------- helpers

def read_csv(path, required=True):
    if not os.path.exists(path):
        if required:
            raise SystemExit(f"missing input {path}")
        return None
    with open(path, newline="", encoding="utf-8") as f:
        return list(csv.DictReader(f))


def fnum(v, default=0.0):
    try:
        return float(v) if v not in (None, "") else default
    except ValueError:
        return default


def truthy(v):
    return str(v).strip().lower() in ("1", "true", "yes", "y")


def parse_day(v):
    if not v:
        return None
    return datetime.fromisoformat(str(v).replace("Z", "+00:00")[:19]).date() if "T" in str(v) else date.fromisoformat(str(v)[:10])


def period_of(d: date, grain: str) -> str:
    return f"{d.year}-Q{(d.month - 1) // 3 + 1}" if grain == "quarter" else f"{d.year}-{d.month:02d}"


def wilson(k, n, z=1.96):
    if n == 0:
        return None, None
    p = k / n
    den = 1 + z * z / n
    centre = (p + z * z / (2 * n)) / den
    half = z * math.sqrt(p * (1 - p) / n + z * z / (4 * n * n)) / den
    return max(0.0, centre - half), min(1.0, centre + half)


class Hasher:
    def __init__(self, salt: str | None, require: bool):
        if not salt and require:
            raise SystemExit("org_members.csv is present but $VALUE_VIEW_SALT is not set: refusing to run "
                             "rather than fall back to raw IDs.")
        self.key = (salt or secrets.token_hex(32)).encode()
        self.linkable = bool(salt)

    def __call__(self, uid: str | None) -> str | None:
        if not uid:
            return None
        return hmac.new(self.key, uid.strip().lower().encode(), hashlib.sha256).hexdigest()[:20]


def hash_members(raw_path, out_path, salt):
    """Convert an HR/Entra export (user_id, squad, valid_from, valid_to) to hashed form."""
    h = Hasher(salt, require=True)
    rows = read_csv(raw_path)
    with open(out_path, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=["user_hash", "squad", "valid_from", "valid_to"])
        w.writeheader()
        for r in rows:
            w.writerow({"user_hash": h(r["user_id"]), "squad": r["squad"],
                        "valid_from": r.get("valid_from", ""), "valid_to": r.get("valid_to", "")})
    print(f"hashed {len(rows)} members -> {out_path}")


# --------------------------------------------------------------------------- org model

class Org:
    def __init__(self, units, members):
        self.enabled = bool(units)
        self.squad_team, self.team_bu, self.jira_to_squad = {}, {}, {}
        for u in units or []:
            self.squad_team[u["squad"]] = u["team"]
            self.team_bu[u["team"]] = u["bu"]
            if u.get("jira_team"):
                self.jira_to_squad[u["jira_team"].strip().lower()] = u["squad"]
        self.members = defaultdict(list)
        for m in members or []:
            self.members[m["user_hash"]].append(
                (parse_day(m.get("valid_from")) or date.min, parse_day(m.get("valid_to")) or date.max, m["squad"]))

    def squad_of_user(self, user_hash, on: date | None):
        if not user_hash or on is None:
            return None
        for start, end, squad in self.members.get(user_hash, []):
            if start <= on <= end:
                return squad
        return None

    def chain(self, squad):
        """squad -> {level: unit} for every level, or just 'all'."""
        out = {"all": "all"}
        if squad and squad in self.squad_team:
            team = self.squad_team[squad]
            out.update(squad=squad, team=team, bu=self.team_bu.get(team, "unknown"))
        return out

    def parent(self, level, unit):
        if level == "squad":
            return "team", self.squad_team.get(unit)
        if level == "team":
            return "bu", self.team_bu.get(unit)
        if level == "bu":
            return "all", "all"
        return None, None


# --------------------------------------------------------------------------- load + attribute

def load(indir, grain, salt):
    p = lambda name: os.path.join(indir, name)  # noqa: E731
    members = read_csv(p("org_members.csv"), required=False)
    hasher = Hasher(salt, require=members is not None)
    org = Org(read_csv(p("org_units.csv"), required=False), members)
    aliases = AliasTable(p("aliases.csv"))

    work = {r["work_item_id"]: r for r in read_csv(p("work_items.csv"))}
    for r in read_csv(p("outcomes.csv")):
        if r["work_item_id"] in work:
            work[r["work_item_id"]].update(outcome=r.get("outcome", ""), attempt_n=r.get("attempt_n", ""),
                                           bounced_back=truthy(r.get("bounced_back")),
                                           reverted_30d=truthy(r.get("reverted_30d")))

    prices = {r["model"]: (fnum(r["input_usd_per_mtok"]), fnum(r["output_usd_per_mtok"]))
              for r in read_csv(p("model-prices.csv"), required=False) or []}

    sessions = {}
    for r in read_csv(p("sessions.csv")):
        d = parse_day(r.get("started_at"))
        sessions[r["session_id"]] = {
            "session_id": r["session_id"], "harness": r.get("harness") or "unknown", "user": hasher(r.get("user_id")),
            "day": d, "wi": aliases.resolve(r.get("work_item_alias")),
        }

    usage = []
    for r in read_csv(p("usage.csv")):
        s = sessions.get(r.get("session_id") or "")
        cost, priced = fnum(r.get("cost_usd"), None), True
        if cost is None:
            pin, pout = prices.get(r.get("model", ""), (None, None))
            if pin is not None:
                cost = (fnum(r.get("input_tokens")) * pin + fnum(r.get("output_tokens")) * pout) / 1e6
            else:
                cost, priced = 0.0, False
        usage.append({
            "day": parse_day(r.get("usage_date")) or (s["day"] if s else None),
            "harness": r.get("harness") or (s["harness"] if s else "unknown"),
            "model": r.get("model") or "unknown", "billing_source": r.get("billing_source") or "unknown",
            "user": hasher(r.get("user_id")) or (s["user"] if s else None),
            "wi": s["wi"] if s else WI_UNASSIGNED, "cost": cost, "priced": priced,
            "tokens": fnum(r.get("input_tokens")) + fnum(r.get("output_tokens")),
        })

    seats = [{"user": hasher(r["user_id"]), "harness": r.get("harness", "")}
             for r in read_csv(p("seats.csv"), required=False) or []]
    bench = read_csv(p("bench_runs.csv"), required=False) or []

    # ---- squad per work item: issue first, session users as fallback
    sessions_by_wi = defaultdict(list)
    for s in sessions.values():
        if s["wi"] != WI_UNASSIGNED:
            sessions_by_wi[s["wi"]].append(s)
    attr_source = Counter()
    for wi, w in work.items():
        squad = org.jira_to_squad.get((w.get("team") or "").strip().lower()) if org.enabled else None
        source = "jira_team" if squad else None
        if not squad and org.enabled:
            votes = Counter(org.squad_of_user(s["user"], s["day"]) for s in sessions_by_wi.get(wi, []))
            votes.pop(None, None)
            if votes:
                squad, source = votes.most_common(1)[0][0], "session_users"
        w["squad"], w["attr_source"] = squad, source or "none"
        attr_source[w["attr_source"]] += 1
        w["agent"] = bool(sessions_by_wi.get(wi))
        w["done_day"] = parse_day(w.get("done_at"))
        w["created_day"] = parse_day(w.get("created_at"))

    # ---- dominant model per work item (by tokens)
    tok = defaultdict(Counter)
    spend_wi = defaultdict(float)
    for u in usage:
        if u["wi"] != WI_UNASSIGNED:
            tok[u["wi"]][u["model"]] += u["tokens"]
            spend_wi[u["wi"]] += u["cost"]
    for wi, w in work.items():
        w["model"] = tok[wi].most_common(1)[0][0] if tok.get(wi) else None
        w["spend"] = spend_wi.get(wi, 0.0)

    # ---- squad for sessions and usage: the work item's squad, else the user's squad that day
    def squad_for(wi, user, day):
        if wi != WI_UNASSIGNED and wi in work and work[wi]["squad"]:
            return work[wi]["squad"]
        return org.squad_of_user(user, day) if org.enabled else None
    for s in sessions.values():
        s["squad"] = squad_for(s["wi"], s["user"], s["day"])
    for u in usage:
        u["squad"] = squad_for(u["wi"], u["user"], u["day"])
    for s in seats:
        s["squad"] = org.squad_of_user(s["user"], date.today()) if org.enabled else None

    return dict(org=org, work=work, sessions=list(sessions.values()), usage=usage, seats=seats,
                bench=bench, attr_source=attr_source, grain=grain, hasher=hasher)


# --------------------------------------------------------------------------- metrics

def outcome_bucket(w):
    if not w.get("done_day"):
        return "in_progress"
    if w.get("outcome") == "fail" or w.get("reverted_30d"):
        return "failed"
    if w.get("outcome") == "abandoned":
        return "no_merged_change"
    if w.get("outcome") in ("pass", "reworked"):
        return "qualified"
    return "no_outcome_recorded"


def compute(sessions, usage, work_items, seats, bench, is_global, attr_source):
    rows = []

    def add(metric, value, unit, n, cohort="all", seg_key="", seg_val="", note="", ci=(None, None)):
        rows.append(dict(metric=metric, value=value, unit=unit, n=n, cohort=cohort, segment_key=seg_key,
                         segment_value=seg_val, ci_low=ci[0], ci_high=ci[1], method_note=note))

    # spend
    total = sum(u["cost"] for u in usage)
    add("spend_total", total, CURRENCY, len(usage))
    for key, metric in (("harness", "spend_by_harness"), ("billing_source", "spend_by_billing_source"),
                        ("model", "spend_by_model")):
        groups = defaultdict(list)
        for u in usage:
            groups[u[key]].append(u)
        for k, us in sorted(groups.items()):
            add(metric, sum(x["cost"] for x in us), CURRENCY, len(us), seg_key=key, seg_val=k)
    unpriced = sum(1 for u in usage if not u["priced"])
    add("unpriced_usage_rows", unpriced, "count", len(usage), note="no cost_usd and no model price")

    by_wi = {w["work_item_id"]: w for w in work_items}
    ws_spend = defaultdict(lambda: [0.0, 0])
    for u in usage:
        w = by_wi.get(u["wi"])
        ws = (w.get("work_stream") or "team_delivery") if w else "unattributed"
        ws_spend[ws][0] += u["cost"]
        ws_spend[ws][1] += 1
    for ws, (v, n) in sorted(ws_spend.items()):
        add("spend_by_work_stream", v, CURRENCY, n, seg_key="work_stream", seg_val=ws)

    # attribution
    attributed = [s for s in sessions if s["wi"] != WI_UNASSIGNED]
    if sessions:
        add("attribution_rate", len(attributed) / len(sessions), "ratio", len(sessions))
    if total:
        att_spend = sum(u["cost"] for u in usage if u["wi"] != WI_UNASSIGNED)
        add("attributed_spend_share", att_spend / total, "ratio", len(usage))

    # outcome split (team delivery only); unattributed on its own line, never waste
    delivery = [w for w in work_items if (w.get("work_stream") or "team_delivery") == "team_delivery"]
    buckets = defaultdict(lambda: [0.0, 0])
    for w in delivery:
        if w["spend"]:
            b = outcome_bucket(w)
            buckets[b][0] += w["spend"]
            buckets[b][1] += 1
    unatt = [u for u in usage if u["wi"] == WI_UNASSIGNED]
    buckets["unattributed"] = [sum(u["cost"] for u in unatt), len(unatt)]
    for b, (v, n) in sorted(buckets.items()):
        add("spend_by_outcome", v, CURRENCY, n, seg_key="outcome", seg_val=b,
            note="unattributed is not waste; it is unknown" if b == "unattributed" else "")
    closed = sum(buckets[b][0] for b in ("qualified", "failed", "no_merged_change"))
    closed_n = sum(buckets[b][1] for b in ("qualified", "failed", "no_merged_change"))
    if closed:
        add("waste_share", (buckets["failed"][0] + buckets["no_merged_change"][0]) / closed, "ratio", closed_n,
            note="excludes unattributed spend; read next to attribution_rate")

    # delivery metrics
    done = [w for w in delivery if w.get("done_day")]
    qualified = [w for w in done if outcome_bucket(w) == "qualified" and w["agent"]]
    classes = sorted({w.get("task_class") or "other" for w in done})

    def cpdwi(items, seg_val):
        if items:
            add("cost_per_delivered_work_item", sum(w["spend"] for w in items) / len(items), CURRENCY,
                len(items), cohort="agent", seg_key="task_class" if seg_val else "", seg_val=seg_val or "")
    cpdwi(qualified, None)
    for c in classes:
        cpdwi([w for w in qualified if (w.get("task_class") or "other") == c], c)

    def rate(metric, items, pred, cohort, seg_val):
        if items:
            k = sum(1 for w in items if pred(w))
            add(metric, k / len(items), "ratio", len(items), cohort=cohort,
                seg_key="task_class" if seg_val else "", seg_val=seg_val or "", ci=wilson(k, len(items)))

    with_outcome = [w for w in done if w.get("outcome")]
    first_pass = lambda w: w.get("outcome") == "pass"  # noqa: E731
    rate("first_pass_qa_rate", with_outcome, first_pass, "all", None)
    rate("qa_bounce_back_rate", with_outcome, lambda w: w.get("bounced_back"), "all", None)
    rate("adoption_share", done, lambda w: w["agent"], "all", None)
    for c in classes:
        in_c = [w for w in with_outcome if (w.get("task_class") or "other") == c]
        for cohort, flag in (("agent", True), ("non_agent", False)):
            coh = [w for w in in_c if w["agent"] == flag]
            rate("first_pass_qa_rate", coh, first_pass, cohort, c)
            rate("change_failure_rate", coh, lambda w: w.get("reverted_30d"), cohort, c)
            ct = [(w["done_day"] - w["created_day"]).days for w in coh if w.get("created_day")]
            if ct:
                add("cycle_time_days", median(ct), "days", len(ct), cohort=cohort, seg_key="task_class", seg_val=c,
                    note="wall-clock; includes queue time; never converted to money")

    # session success
    ok_sessions = [s for s in attributed if by_wi.get(s["wi"], {}).get("outcome")]
    if ok_sessions:
        k = sum(1 for s in ok_sessions if by_wi[s["wi"]].get("outcome") == "pass" and not by_wi[s["wi"]].get("bounced_back"))
        add("session_success_rate", k / len(ok_sessions), "ratio", len(ok_sessions), cohort="agent",
            ci=wilson(k, len(ok_sessions)))

    # model effectiveness: only within task class
    for c in classes:
        in_c = [w for w in with_outcome if (w.get("task_class") or "other") == c and w.get("model")]
        for m in sorted({w["model"] for w in in_c}):
            ws = [w for w in in_c if w["model"] == m]
            k = sum(1 for w in ws if first_pass(w))
            add("pass_rate_by_model", k / len(ws), "ratio", len(ws), cohort="production", seg_key=f"model|task_class",
                seg_val=f"{m}|{c}", ci=wilson(k, len(ws)), note="observational; confounded by routing, see bench")
            if k:
                add("cost_per_passed_task", sum(w["spend"] for w in ws) / k, CURRENCY, k, cohort="production",
                    seg_key="model|task_class", seg_val=f"{m}|{c}")
    if is_global and bench:
        groups = defaultdict(list)
        for b in bench:
            groups[(b["model"], b.get("task_class") or "other")].append(b)
        for (m, c), bs in sorted(groups.items()):
            k = sum(1 for b in bs if b["outcome"] == "pass")
            add("pass_rate_by_model", k / len(bs), "ratio", len(bs), cohort="bench", seg_key="model|task_class",
                seg_val=f"{m}|{c}", ci=wilson(k, len(bs)), note="replay benchmark; authoritative")
            if k:
                add("cost_per_passed_task", sum(fnum(b.get("cost_usd")) for b in bs) / k, CURRENCY, k,
                    cohort="bench", seg_key="model|task_class", seg_val=f"{m}|{c}")

    # coverage
    if seats:
        emitting = {u["user"] for u in usage if u["user"]} | {s["user"] for s in sessions if s["user"]}
        licensed = {s["user"] for s in seats if s["user"]}
        if licensed:
            add("coverage", len(licensed & emitting) / len(licensed), "ratio", len(licensed))

    # placeholders: say so rather than vanish
    add("capacity_redeployed", None, "hours", 0,
        note="not derivable from telemetry; needs effort data plus a named statement of where capacity went")
    add("governed_path_coverage", None, "ratio", 0, note="no governance field in events yet")

    if is_global:
        tot = sum(attr_source.values())
        for src, n in sorted(attr_source.items()):
            add("org_attribution_source", n / tot, "ratio", tot, seg_key="source", seg_val=src)
    return rows


# --------------------------------------------------------------------------- run + suppression

def run(indir, outdir, grain="quarter", salt=None):
    d = load(indir, grain, salt)
    org = d["org"]

    def per(items, day_key):
        out = defaultdict(list)
        for x in items:
            if x.get(day_key):
                out[period_of(x[day_key], grain)].append(x)
        return out

    s_by_p, u_by_p = per(d["sessions"], "day"), per(d["usage"], "day")
    w_by_p = per(list(d["work"].values()), "done_day")
    # in-progress items still carry spend: bucket them by created day
    for w in d["work"].values():
        if not w.get("done_day") and w.get("created_day"):
            w_by_p[period_of(w["created_day"], grain)].append(w)
    periods = sorted(set(s_by_p) | set(u_by_p) | set(w_by_p))

    out = []
    unit_users = {}  # (period, level, unit) -> distinct AI users
    for period in periods:
        units = defaultdict(lambda: {"s": [], "u": [], "w": []})
        for key, coll in (("s", s_by_p[period]), ("u", u_by_p[period]), ("w", w_by_p[period])):
            for x in coll:
                for level, unit in org.chain(x.get("squad")).items():
                    units[(level, unit)][key].append(x)
        for (level, unit), g in sorted(units.items(), key=lambda kv: LEVELS.index(kv[0][0])):
            users = {x["user"] for x in g["s"] + g["u"] if x.get("user")}
            unit_users[(period, level, unit)] = len(users)
            for r in compute(g["s"], g["u"], g["w"], d["seats"] if level == "all" else
                             [s for s in d["seats"] if org.chain(s["squad"]).get(level) == unit],
                             d["bench"], level == "all", d["attr_source"]):
                r.update(period=period, org_level=level, org_unit=unit, suppressed=False)
                out.append(r)

    # rule 2: cell size
    for r in out:
        if r["value"] is not None and r["n"] < MIN_CELL_SIZE:
            r.update(value=None, ci_low=None, ci_high=None, suppressed=True,
                     method_note=f"n<{MIN_CELL_SIZE}; " + r["method_note"])

    def ai_row(r):
        return r["metric"] in AI_DERIVED and not (r["metric"] in ("first_pass_qa_rate", "change_failure_rate",
                                                                   "cycle_time_days", "qa_bounce_back_rate")
                                                  and r["cohort"] in WORK_ONLY_COHORTS)

    # rule 3: distinct users per unit-period
    hidden = {k for k, n in unit_users.items() if n < MIN_DISTINCT_USERS and k[1] != "all"}
    # rule 4: complementary suppression, bottom-up
    for level in ("squad", "team", "bu"):
        siblings = defaultdict(list)
        for (period, lvl, unit), n in unit_users.items():
            if lvl == level:
                siblings[(period, org.parent(level, unit))].append(((period, lvl, unit), n))
        for _, kids in siblings.items():
            hid = [k for k, _ in kids if k in hidden]
            vis = sorted([(n, k) for k, n in kids if k not in hidden])
            if len(hid) == 1 and vis:
                hidden.add(vis[0][1])
    for r in out:
        if (r["period"], r["org_level"], r["org_unit"]) in hidden and ai_row(r) and r["value"] is not None:
            r.update(value=None, ci_low=None, ci_high=None, suppressed=True,
                     method_note=f"unit has <{MIN_DISTINCT_USERS} AI users or protects a sibling; " + r["method_note"])

    os.makedirs(outdir, exist_ok=True)
    with open(os.path.join(outdir, "value_view.csv"), "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=OUT_FIELDS)
        w.writeheader()
        for r in out:
            w.writerow({k: ("" if r.get(k) is None else (round(r[k], 6) if isinstance(r[k], float) else r[k]))
                        for k in OUT_FIELDS})
    with open(os.path.join(outdir, "metric_catalogue.csv"), "w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["metric", "audience", "unit", "how_to_read"])
        w.writerows(CATALOGUE)
    return out, d


# --------------------------------------------------------------------------- self-test

def _write(path, fields, rows):
    with open(path, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=fields)
        w.writeheader()
        w.writerows(rows)


def selftest():
    rnd = random.Random(7)
    tmp = tempfile.mkdtemp(prefix="value_view_")
    indir, outdir = os.path.join(tmp, "in"), os.path.join(tmp, "out")
    os.makedirs(indir)
    salt = "selftest-salt"
    os.environ["VALUE_VIEW_SALT"] = salt

    units = [("sq-a", "pricing", "hiscox-re", "Pricing A"), ("sq-b", "pricing", "hiscox-re", "Pricing B"),
             ("sq-e", "claims", "hiscox-re", "Claims E"), ("sq-f", "claims", "hiscox-re", "Claims F"),
             ("sq-z", "core", "other-bu", "Core Z"), ("sq-y", "core", "other-bu", "Core Y")]
    _write(os.path.join(indir, "org_units.csv"), ["squad", "team", "bu", "jira_team"],
           [dict(zip(["squad", "team", "bu", "jira_team"], u)) for u in units])
    sizes = {"sq-a": 6, "sq-b": 5, "sq-e": 5, "sq-f": 2, "sq-z": 4, "sq-y": 4}
    people = {sq: [f"{sq}-dev{i}@example.com" for i in range(n)] for sq, n in sizes.items()}
    mover, move_day = "mover@example.com", date(2026, 8, 15)
    raw_members = [{"user_id": u, "squad": sq, "valid_from": "2026-01-01", "valid_to": ""}
                   for sq, us in people.items() for u in us]
    raw_members += [{"user_id": mover, "squad": "sq-a", "valid_from": "2026-01-01", "valid_to": "2026-08-14"},
                    {"user_id": mover, "squad": "sq-b", "valid_from": "2026-08-15", "valid_to": ""}]
    _write(os.path.join(tmp, "members_raw.csv"), ["user_id", "squad", "valid_from", "valid_to"], raw_members)
    hash_members(os.path.join(tmp, "members_raw.csv"), os.path.join(indir, "org_members.csv"), salt)

    aliases = AliasTable(os.path.join(indir, "aliases.csv"))
    work, outcomes, sessions, usage = [], [], [], []
    models = ["claude-sonnet", "claude-opus", "gemini-pro"]
    start = date(2026, 7, 1)
    sid = 0
    for i in range(240):
        sq = rnd.choice(list(sizes))
        key = f"RE-{1000 + i}"
        wi = aliases.ensure(f"jira:{key}")
        created = start + timedelta(days=rnd.randint(0, 150))
        done = created + timedelta(days=rnd.randint(2, 20)) if rnd.random() < 0.8 else None
        team_field = "" if rnd.random() < 0.15 else dict((u[0], u[3]) for u in units)[sq]
        work.append({"work_item_id": wi, "alias": f"jira:{key}", "task_class": rnd.choice(["bug", "feature"]),
                     "task_size": rnd.choice([1, 2, 3, 5]), "team": team_field,
                     "work_stream": "platform" if rnd.random() < 0.08 else "team_delivery",
                     "status": "Done" if done else "In Progress", "created_at": created.isoformat(),
                     "done_at": done.isoformat() if done else ""})
        if done:
            outcomes.append({"work_item_id": wi, "outcome": rnd.choices(["pass", "reworked", "fail", "abandoned"],
                                                                         [6, 2, 1, 1])[0],
                             "attempt_n": 1, "bounced_back": int(rnd.random() < 0.15),
                             "reverted_30d": int(rnd.random() < 0.05)})
        if rnd.random() < 0.7:  # agent involvement
            for _ in range(rnd.randint(1, 4)):
                sid += 1
                user = rnd.choice(people[sq])
                day = created + timedelta(days=rnd.randint(0, 3))
                sessions.append({"session_id": f"s{sid}", "harness": "claude-code", "user_id": user,
                                 "started_at": f"{day.isoformat()}T10:00:00Z",
                                 "work_item_alias": f"git:{key}-some-change"})
                m = rnd.choice(models)
                for _ in range(rnd.randint(1, 3)):
                    usage.append({"usage_date": day.isoformat(), "harness": "claude-code", "model": m,
                                  "billing_source": "apim", "user_id": user, "session_id": f"s{sid}",
                                  "input_tokens": rnd.randint(1000, 90000), "output_tokens": rnd.randint(100, 9000),
                                  "cost_usd": round(rnd.uniform(0.05, 4.0), 4), "ai_credits_used": ""})
    # unattributed sessions (no ticket in branch), including the mover before and after the move
    for day, who in [(date(2026, 8, 1), mover)] * 6 + [(date(2026, 9, 1), mover)] * 6 + \
            [(start + timedelta(days=rnd.randint(0, 150)), rnd.choice(people["sq-z"])) for _ in range(30)]:
        sid += 1
        sessions.append({"session_id": f"s{sid}", "harness": "claude-code", "user_id": who,
                         "started_at": f"{day.isoformat()}T09:00:00Z", "work_item_alias": "git:main"})
        usage.append({"usage_date": day.isoformat(), "harness": "claude-code", "model": "claude-sonnet",
                      "billing_source": "apim", "user_id": who, "session_id": f"s{sid}", "input_tokens": 5000,
                      "output_tokens": 500, "cost_usd": 1.0, "ai_credits_used": ""})
    # Copilot usage: no session, billed by GitHub, priced from the price table
    for sq, us in people.items():
        for u in us:
            usage.append({"usage_date": "2026-09-10", "harness": "copilot", "model": "copilot-mixed",
                          "billing_source": "github", "user_id": u, "session_id": "", "input_tokens": 20000,
                          "output_tokens": 2000, "cost_usd": "", "ai_credits_used": 12})
    aliases.save()
    _write(os.path.join(indir, "work_items.csv"), list(work[0]), work)
    _write(os.path.join(indir, "outcomes.csv"), list(outcomes[0]), outcomes)
    _write(os.path.join(indir, "sessions.csv"), list(sessions[0]), sessions)
    _write(os.path.join(indir, "usage.csv"), list(usage[0]), usage)
    _write(os.path.join(indir, "model-prices.csv"), ["model", "input_usd_per_mtok", "output_usd_per_mtok"],
           [{"model": "copilot-mixed", "input_usd_per_mtok": 1.0, "output_usd_per_mtok": 4.0}])
    _write(os.path.join(indir, "seats.csv"), ["user_id", "harness"],
           [{"user_id": u, "harness": "claude-code"} for us in people.values() for u in us] +
           [{"user_id": "never-used@example.com", "harness": "claude-code"}])
    _write(os.path.join(indir, "bench_runs.csv"), ["model", "task_class", "outcome", "cost_usd"],
           [{"model": m, "task_class": "bug", "outcome": rnd.choice(["pass", "pass", "fail"]),
             "cost_usd": rnd.uniform(0.5, 3)} for m in models for _ in range(30)])

    out, d = run(indir, outdir, "quarter", salt)
    text = open(os.path.join(outdir, "value_view.csv"), encoding="utf-8").read()
    rows = list(csv.DictReader(open(os.path.join(outdir, "value_view.csv"), encoding="utf-8")))
    checks = []

    # 1. no raw identities anywhere in the output
    assert "@example.com" not in text and "dev0" not in text
    checks.append("no raw user IDs in output")

    # 2. a two-person squad never shows an AI-derived value
    leaked = [r for r in rows if r["org_unit"] == "sq-f" and r["metric"] in AI_DERIVED and r["value"] != ""
              and not (r["metric"] in ("first_pass_qa_rate", "change_failure_rate", "cycle_time_days",
                                       "qa_bounce_back_rate") and r["cohort"] in WORK_ONLY_COHORTS)]
    assert not leaked, leaked[:2]
    checks.append("two-person squad never shows an AI-derived value")

    # 3. complementary suppression: wherever sq-f is hidden, its only sibling sq-e is hidden too
    def shown(unit, period, metric="spend_total"):
        return any(r["org_unit"] == unit and r["period"] == period and r["metric"] == metric and r["value"] != ""
                   for r in rows)
    for period in {r["period"] for r in rows}:
        if not shown("sq-f", period) and any(r["org_unit"] == "sq-f" and r["period"] == period for r in rows):
            assert not shown("sq-e", period), f"subtraction leak in {period}"
    checks.append("no subtraction leak between sibling squads")

    # 4. the mover's history stays with the old squad
    h = d["hasher"](mover)
    assert d["org"].squad_of_user(h, date(2026, 8, 1)) == "sq-a"
    assert d["org"].squad_of_user(h, date(2026, 9, 1)) == "sq-b"
    mover_usage = [u for u in d["usage"] if u["user"] == h and u["wi"] == WI_UNASSIGNED]
    assert {u["squad"] for u in mover_usage if u["day"] < move_day} == {"sq-a"}
    assert {u["squad"] for u in mover_usage if u["day"] >= move_day} == {"sq-b"}
    checks.append("a squad move does not rewrite history")

    # 5. BU spend reconciles to global wherever every BU is visible
    reconciled = 0
    for period in sorted({r["period"] for r in rows}):
        g = [r for r in rows if r["period"] == period and r["org_level"] == "all" and r["metric"] == "spend_total"]
        b = [r for r in rows if r["period"] == period and r["org_level"] == "bu" and r["metric"] == "spend_total"]
        unassigned = sum(u["cost"] for u in d["usage"] if period_of(u["day"], "quarter") == period and not u["squad"])
        if g and b and all(r["value"] != "" for r in b):
            assert abs(sum(float(r["value"]) for r in b) + unassigned - float(g[0]["value"])) < 0.05, period
            reconciled += 1
    assert reconciled
    checks.append(f"BU spend reconciles to global ({reconciled} periods)")

    # 6. unattributed spend has its own line and is never inside waste
    ua = [r for r in rows if r["metric"] == "spend_by_outcome" and r["segment_value"] == "unattributed"
          and r["org_level"] == "all"]
    assert ua and all("not waste" in r["method_note"] for r in ua)
    checks.append("unattributed spend reported separately, not as waste")

    # 7. CIs are well-formed
    for r in rows:
        if r["ci_low"] != "" and r["value"] != "":
            assert 0 <= float(r["ci_low"]) <= float(r["value"]) <= float(r["ci_high"]) <= 1, r
    checks.append("confidence intervals well-formed")

    # 8. no unsegmented agent vs non-agent comparison
    assert not [r for r in rows if r["metric"] in ("first_pass_qa_rate", "change_failure_rate", "cycle_time_days")
                and r["cohort"] in ("agent", "non_agent") and not r["segment_key"]]
    checks.append("agent vs non-agent only within task class")

    # 9. Copilot priced from the table; nothing unpriced; coverage below 1 because of the unused seat
    g_unpriced = [r for r in rows if r["org_level"] == "all" and r["metric"] == "unpriced_usage_rows"]
    assert all(r["value"] in ("0", "") for r in g_unpriced)
    cov = [r for r in rows if r["org_level"] == "all" and r["metric"] == "coverage" and r["value"] != ""]
    assert cov and all(float(r["value"]) < 1 for r in cov)
    checks.append("copilot priced, coverage counts unused seats")

    # 10. refusing to run without a salt when members exist
    del os.environ["VALUE_VIEW_SALT"]
    try:
        run(indir, outdir, "quarter", None)
        raise AssertionError("should have refused to run without a salt")
    except SystemExit:
        checks.append("refuses to run without $VALUE_VIEW_SALT when org members are present")

    print(f"selftest OK ({len(rows)} rows) -> {outdir}")
    for c in checks:
        print("  ok:", c)
    for r in rows:
        if r["org_level"] == "all" and r["metric"] in ("cost_per_delivered_work_item", "attribution_rate",
                                                       "first_pass_qa_rate", "adoption_share") \
                and not r["segment_key"]:
            print(f"  {r['period']} {r['metric']}: {r['value']} (n={r['n']})")


def main(argv=None):
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    ap.add_argument("--indir", default="data")
    ap.add_argument("--outdir", default="out")
    ap.add_argument("--period", choices=["quarter", "month"], default="quarter")
    ap.add_argument("--selftest", action="store_true")
    ap.add_argument("--hash-members", nargs=2, metavar=("RAW_CSV", "OUT_CSV"),
                    help="hash an HR/Entra export (user_id, squad, valid_from, valid_to)")
    a = ap.parse_args(argv)
    if a.selftest:
        selftest()
        return 0
    salt = os.environ.get("VALUE_VIEW_SALT")
    if a.hash_members:
        hash_members(a.hash_members[0], a.hash_members[1], salt)
        return 0
    out, _ = run(a.indir, a.outdir, a.period, salt)
    print(f"{len(out)} rows -> {os.path.join(a.outdir, 'value_view.csv')}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
'@
~~~~

