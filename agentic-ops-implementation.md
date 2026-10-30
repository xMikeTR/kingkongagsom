# Agentic Ops on New Relic — Implementation Brief

> Brief for an implementing agent. Read fully before starting. Work phase by phase; stop at each **Checkpoint** and report back before continuing. Where this doc says **VERIFY**, check current vendor docs before relying on the detail — some syntax and endpoints here are from memory.

## 1. Goal

Capture **how and where AI is used across the whole SDLC** (not just developers) and roll it up to **cost and value** per work item (story → epic → feature), team, and SDLC stage. Everything lands in **New Relic** (we have a contract), which becomes our single "Agentic Ops" pane. Leadership dashboards are built on top in NRQL.

Non-goals for now: replacing self-hosted Langfuse (decided later after piloting NR AI Evaluation), capturing prompt/response content, per-person leaderboards.

## 2. Constraints

- **No Jira API token.** Jira access is MCP-only for humans. Do not design anything that requires a personal Jira API token. Jira data must be *pushed* by Jira Automation.
- **Hosting must be Azure-native** (Functions or Container Apps) if anything needs hosting. Prefer no hosting at all.
- **Models are served via Vertex AI (GCP)** behind **Azure APIM v2**. NR's native Claude Platform trace export does not cover Vertex yet — instrument on our side with OTel.
- **Codebases:** C# + TypeScript, monorepo, GitHub + GitHub Actions.
- **Secrets:** never commit keys. NR keys go in GitHub Secrets / Key Vault / Jira Automation secret fields only. Use a **dedicated, revocable NR ingest key** for each source.
- **Privacy:** metadata only. No prompts, completions, file contents, or tool arguments leave the machine. Leadership views aggregate to team/stage; individual views are visible only to the individual.

## 3. Canonical event schema

Every AI-related event, whatever the source, should carry as many of these attributes as the source allows. Use these exact names so NRQL joins work across sources.

| Attribute | Example | Notes |
|---|---|---|
| `user.email` | `jane@…` | Join key to org lookup table |
| `jira.key` | `RE-1234` | Work item; primary join key |
| `sdlc.stage` | `refine` / `design` / `build` / `test` / `review` / `release` / `operate` | Explicit if known; else derived from role (see §4) |
| `ai.tool` | `claude-code` / `copilot` / `apim-app:<name>` / `m365-copilot` / `rovo` | |
| `ai.model` | `claude-sonnet-…` | Where available |
| `cost.usd` | `0.42` | Numeric |
| `tokens.input` / `tokens.output` | | Numeric |
| `source` | `otel-claude-code` / `jira-automation` / `gh-action` / `apim` / `replay-bench` | Provenance |
| `task.class` | `bug` / `feature` / `refactor` / `test` / `migration` / `data` | Needed to compare models fairly (see Phase 7) |
| `task.size` | `3` | Story points; numeric |
| `outcome` | `pass` / `fail` / `reworked` / `abandoned` | From QA gate / CI / Jira |
| `attempt.n` | `1` | Attempt number for this task; numeric |

Org lookup table (uploaded to NR as a lookup table, CSV): `user.email, role, team, squad, bu`. **VERIFY** NR lookup table upload + `lookup()` NRQL syntax.

## 4. Stage derivation rule

1. If an event carries an explicit `sdlc.stage`, use it.
2. Else derive from role via the org table: `BA/PM → refine`, `Dev → build`, `QA → test`, `Data Eng → build`, `SRE/Ops → operate`.
3. Else `unknown` (show it on dashboards — the size of `unknown` is a quality metric for this system).

---

## Phase 1 — Coding assistant telemetry (Claude Code → NR)

**Note:** a mechanism already exists that joins Jira ticket keys onto Claude Code sessions. **Find and reuse it** before building anything new; extend rather than replace.

Tasks:
1. Configure Claude Code OTel export to NR's OTLP endpoint (EU region: `https://otlp.eu01.nr-data.net:4318`, header `api-key=<ingest key>`; **VERIFY** region and endpoint). Relevant env vars: `CLAUDE_CODE_ENABLE_TELEMETRY=1`, `OTEL_METRICS_EXPORTER=otlp`, `OTEL_LOGS_EXPORTER=otlp`, `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS`. **VERIFY** against current Claude Code monitoring docs.
2. Set `OTEL_RESOURCE_ATTRIBUTES` at session launch with `jira.key`, `user.email`, and optionally `sdlc.stage`. Resource attributes are fixed per process, so set them in the launcher. The existing per-ticket **"slots" PowerShell script** is the natural place: derive `jira.key` from the branch name / slot and export it before launching Claude Code.
3. Ship this as a shareable config (managed settings or a checked-in script) so QA and data engineers get it too, not just devs.
4. Confirm prompt logging stays **off** (`OTEL_LOG_USER_PROMPTS` unset).
5. Optional, local-only first: evaluate `newrelic-experimental/preflight` for efficiency scoring/anti-patterns. It is experimental and captures tool-call content — keep it in `local` mode unless security signs off.

Acceptance:
- `FROM Metric SELECT sum(claude_code.cost.usage) FACET jira.key SINCE 1 day ago` returns rows with real ticket keys. (**VERIFY** metric name as it lands in NR.)
- A QA engineer's session shows up with their email and ticket key.

**Checkpoint 1.**

## Phase 2 — Jira data into NR (no API token)

Use **Jira Automation → "Send web request" → NR Event API**.

Rule:
- **Trigger:** Issue created; Issue transitioned; Field value changed (Parent, Story Points, Status, Sprint).
- **Action:** Send web request
  - `POST https://insights-collector.eu01.nr-data.net/v1/accounts/{ACCOUNT_ID}/events` (**VERIFY** region)
  - Headers: `Content-Type: application/json`, `Api-Key: <dedicated ingest key>` (use the rule's secret/hidden header option if available)
  - Body:

```json
{
  "eventType": "JiraIssue",
  "key": "{{issue.key}}",
  "issueType": "{{issue.issueType.name}}",
  "parent": "{{issue.parent.key}}",
  "status": "{{issue.status.name}}",
  "statusCategory": "{{issue.status.statusCategory.name}}",
  "points": {{issue.Story Points|0}},
  "project": "{{issue.project.key}}",
  "assignee": "{{issue.assignee.emailAddress}}",
  "updated": "{{issue.updated.jqlDateTime}}",
  "trigger": "{{triggerIssue.key}}"
}
```

Notes:
- The Story Points field name differs per instance — find the real field name. Test empty-value handling so the JSON never breaks.
- Events are append-only. Always read latest state with `latest(...) FACET key`.
- Hierarchy: story → `parent` = epic; the epic's own event → `parent` = feature. Resolve with two joins (or materialise later).
- Check the instance's **Automation run limits** before enabling on high-volume projects. Start with one project.
- **Backfill:** a one-off script using the existing Jira MCP to emit historical `JiraIssue` events for open epics/features. One-off only; not a scheduled dependency.

Acceptance:
- Transitioning a test ticket produces a `JiraIssue` event in NR within ~1 minute.
- `FROM JiraIssue SELECT latest(parent), latest(status) FACET key` shows correct hierarchy for a known epic.

**Checkpoint 2.**

## Phase 3 — Join + first dashboards

NRQL sketch — cost per epic (**VERIFY** NRQL `JOIN` subquery syntax and limits):

```sql
FROM Metric
JOIN (
  FROM JiraIssue SELECT latest(parent) AS epic FACET key AS jira.key LIMIT MAX
) ON jira.key
SELECT sum(claude_code.cost.usage) AS 'USD'
FACET epic SINCE 30 days ago
```

Then a second join from epic → feature using the epics' own `JiraIssue.parent`.

Dashboards (create via NerdGraph/Terraform so they're versioned in the repo):
1. **Leadership — Cost & Value by Feature/Epic:** spend, AI-assisted share, cycle time. Team/stage aggregates only.
2. **AI across the SDLC:** spend and active usage by `sdlc.stage` × `team`; size of `unknown`.
3. **Personal:** filtered to the viewer; not shared upward.

Acceptance: leadership dashboard renders for at least one real feature with cost and cycle time side by side.

**Checkpoint 3.**

## Phase 4 — Git signal (GitHub Action)

- Workflow on PR merge: read commits in the PR, count those with a `Co-Authored-By:` trailer naming an AI tool vs total, extract `jira.key` from branch name / PR title, then POST an `AiCommitStats` event to the NR Event API (key in GitHub Secrets).
- Attributes: `jira.key`, `repo`, `pr.number`, `commits.total`, `commits.ai`, `lines.added`, `lines.removed`, `author.email`.

Acceptance: merging a PR produces one `AiCommitStats` event with the correct ticket key.

## Phase 5 — APIM gateway (all in-house AI apps)

Depends on the Head of Platform / APIM v2 owners. Prepare the design and raise it; don't change shared APIM config unilaterally.
- Convention: callers send `x-work-item` and `x-sdlc-stage` headers.
- APIM policy logs user (from Entra token), app, model, tokens, cost estimate, plus those headers.
- Route to NR: preferred is APIM diagnostics → Event Hub → NR's Azure integration or a small Azure Function forwarding to the Event API. **VERIFY** which path NR supports natively for APIM v2.

## Phase 6 — SaaS AI usage outside the gateway

Discovery only for now. List which admin usage exports / metrics APIs exist and who owns them: Claude Enterprise admin analytics, GitHub Copilot metrics API, M365 Copilot usage reports, Atlassian Rovo admin analytics. Output: a short table (tool, data available, owner, granularity, can it carry a user email). These are how non-dev stages (BA/PM refinement) become visible.

## Phase 7 — Model effectiveness: "would another model do as well?"

Leadership question: which model gives the best task success per unit cost, per kind of task?

### 7.0 The confound (read first)
Model choice is correlated with task difficulty — people pick bigger models for harder work. **Never report raw pass rate by model from production data.** Always compare within the same `task.class` × `task.size` bucket, and treat the replay benchmark (7.2) as the authoritative answer.

### 7.1 Observational layer (from production telemetry)
- Derive `task.class` from Jira issue type + labels/components (via the Phase 2 `JiraIssue` events); `task.size` from story points.
- Derive `outcome` per ticket: `pass` = reached Ready for QA via the QA gate on first gate run; `reworked` = gate failed then later passed, or ticket reopened; `fail`/`abandoned` = never passed / closed without merge.
- `attempt.n` = count of gate runs for the ticket.
- Emit a `TaskOutcome` event (`jira.key`, `ai.model` = dominant model by tokens on the ticket, `task.class`, `task.size`, `outcome`, `attempt.n`, `cost.usd` total, `tokens.*`, `wallclock.hours`) when a ticket reaches a terminal state. Source: CI (gate results) + Jira Automation transition.

### 7.2 Replay benchmark (authoritative)
An internal SWE-bench-style suite on our own codebase.
1. **Corpus:** select closed tickets with a merged PR and passing tests. Stratify across `task.class` and size. Start with ~30, grow to 100+. Store as `bench/tasks/<jira.key>.yaml`: base commit SHA, ticket description/acceptance criteria (as given to the agent originally), the hidden reference tests, the gate config.
2. **Isolation:** each run in a fresh environment checked out at the base SHA — reuse the per-ticket **slots** script or a container. Reference tests are hidden from the agent and added only at scoring time.
3. **Run matrix:** each task × each candidate model × k runs (k ≥ 3) with the same harness, prompt, tools and budget caps (max tokens, max wall-clock). Only the model changes.
4. **Scoring:** run the existing **QA gate** (tests incl. hidden reference tests, mutation score, CRAP/complexity delta, lint/security). Record pass/fail plus each sub-score.
5. **Emit** a `BenchRun` event per run: `jira.key`, `ai.model`, `task.class`, `task.size`, `attempt.n`, `outcome`, `cost.usd`, `tokens.*`, `turns`, `tool.calls`, `tool.errors`, `wallclock.sec`, `mutation.score`, `crap.delta`, `bench.version`.
6. **Cadence:** on demand + when a new model becomes available on Vertex/APIM. Pin `bench.version`; never edit tasks in place.
7. **Contamination:** prefer recent tickets; note that public-model training data won't include our private repo, but keep reference tests hidden regardless.

### 7.3 Metrics
- **Headline:** cost per passed task = `sum(cost.usd) / count(outcome = pass)`, by `ai.model` × `task.class`.
- **Effectiveness:** pass@1, pass@k, human-intervention rate (from production), rework/reopen rate, defect escape, review rejection rate.
- **Efficiency:** tokens/turns/tool calls per task, wall-clock, cache hit rate, stall/stuck-loop rate.
- **Output quality:** mutation score, CRAP/complexity delta, lint/security findings on AI changes.
- **Reliability:** tool-call error rate, retries, abandoned sessions.
- Always show **confidence intervals** (e.g. Wilson interval for pass rates) and n. Don't rank models whose intervals overlap — call them equivalent.

### 7.4 Dashboard
- **Model frontier:** scatter of cost per task (x) vs pass@1 (y), one point per model, faceted by `task.class`. Models on the Pareto frontier are the recommendation.
- Table: model × task class with pass@1 (CI), cost per passed task, n.
- Production vs bench side by side, so drift between "lab" and "real" is visible.

### 7.5 Later: randomised routing
Once 7.2 shows candidate equivalence for a low-risk task class, propose (to APIM owners) routing a small random share of that class to the cheaper model and comparing `TaskOutcome`. Requires sign-off; do not implement unilaterally.

Acceptance:
- Bench runs ≥ 30 tasks × 2 models × k=3 end to end and emits `BenchRun` events.
- Frontier chart renders with CIs.
- One-paragraph written finding per task class, stating where a cheaper model is statistically equivalent.

**Checkpoint 7.**

## Value metrics (pair usage with outcomes)

Do not report tokens as value. For each stage, show an existing outcome next to AI usage, comparing AI-heavy vs other work:
- refine → reopened stories / stories bounced back from dev
- build → cycle time, CRAP score / complexity trend
- test → defect escape rate
- review → PR time-to-merge, rework commits
- operate → time to resolve incidents

## Open questions (flag, don't guess)

1. NR account region (US/EU) and who issues ingest keys.
2. Who has Jira project admin to create the Automation rule.
3. Exact Story Points / Feature hierarchy field names in our Jira.
4. Security sign-off on the ingest key living in a Jira Automation rule.
5. APIM v2 logging ownership and timeline.
7. Which models are available to benchmark via Vertex/APIM, and the bench compute/token budget.
8. Mapping from Jira issue types/labels to `task.class`.
9. NR AI Evaluation preview access (planned pilot on one agent) and Vertex support timeline — ask the NR account team.
