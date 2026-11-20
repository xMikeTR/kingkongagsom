# BU AI Observability Layer — PoV Plan

Nov 18, 2026 · @Mike

## Purpose and PoV scope

The PoV proves that every AI session in the BU can be attributed to a work item, so we can show what we spent next to what it delivered. It runs on one squad, covers Claude Code and GitHub Copilot, and is built so Group (LiteLLM) can collect from it later without rework.

**In scope**

- One squad, all roles that use AI (devs, data engineers, QA).
- Harnesses: Claude Code, Copilot (VS Code, CLI, app), plus any agent the squad builds.
- Sources: agent telemetry, APIM token metrics, GitHub CI and PRs, Jira.
- Outputs: a New Relic Agentic Ops dashboard, Langfuse session traces, and one Power BI value page.

**Out of scope for the PoV**

- Other squads and BU-wide rollout.
- Sending data to Group (designed for, not switched on).
- Enforcement changes beyond what APIM already does.
- Prompt or response content (metadata only, always).

**Proposed success criteria**

- Coverage: at least 90% of the squad's active AI users emit telemetry.
- Attribution: at least 80% of sessions resolve to a work item; the rest land in a visible "unassigned" bucket.
- Value: one Power BI page shows cost per work item next to its outcome (first-pass Ready for QA, cycle time).
- Replaceability: swapping the Jira adapter for a stub source needs no downstream change.

## Existing assets and decisions

Much of this is already built. The PoV reuses those assets and changes one design choice: the join key moves from the raw Jira key to our own work-item ID, with Jira as an alias.

| Asset | What it already does | Role in this PoV |
| --- | --- | --- |
| Agentic Ops repo (Sep) | Collector config with APIM Event Hub overlay, Claude Code managed settings and identity scripts, APIM policy XML, NRQL queries, org and price lookup tables, Container Apps deploy script | Starting point for Phases 0 and 1; add alias resolution and `org.bu` |
| `value_view.py` + `leadership_rollup.py` (Sep) | Spend split by outcome, CI and code-quality metrics, org segmentation with privacy by construction | The Power BI feed; switch its inputs to canonical events |
| `agent_traces.py`, `cobertura_delta.py` | Vendor-neutral agent trace CSV; complexity, CRAP and coverage deltas per PR | Session efficiency and quality metrics |
| `agentic-ops-implementation.md` (Oct) | Seven-phase brief: Claude Code telemetry, Jira Automation ingestion, NRQL dashboards, git signals, APIM logging, SaaS usage discovery, replay benchmark | Detailed task list; this doc replaces its join-key design |
| Preflight (in testing, key in hand) | Coding-assistant cost and efficiency | Optional dev-side feed |
| Slots script | Per-ticket local environments | Carries the dev setup step and Claude Code telemetry settings |

**Decisions carried forward**

- APIM is the only permitted route to model endpoints.
- New Relic is the operational system of record; Power BI is the leadership view.
- Langfuse stays a personal learning and eval workbench, not a production consumer.
- Evals live as YAML cases in git; results go to New Relic as `EvalResult` events.
- No new hosted services until value is proven. The PoV uses New Relic custom events rather than a new bus or store.
- Own the formats (OTel, the event contract, eval cases); rent the engines.

## Architecture

Sources never talk to consumers directly. Everything passes through an edge layer that turns any external key into our own work-item ID, so we can swap any source or consumer without touching the rest.

&#91;embedded content: BU AI observability · sources, edge, bus, consumers\]

The dashed box is designed for but not switched on during the PoV. For the PoV, the event bus is New Relic custom events, and the adapters are Jira Automation rules and a GitHub Actions step, so no new hosted service is needed. A dedicated bus comes only once value is proven, and the event contract stays the same.

- **Correlation identifier.** We mint the work-item ID (`wi_` + ULID). Jira keys, Azure DevOps IDs and branch names are only aliases for it.
- **Anti-corruption layer.** Adapters are the only components that know Jira or GitHub exist. A move to Azure DevOps means one new adapter that emits the same events.
- **Attribution per session.** The SessionStart hook records session.id with its branch. Usage events join to the work item on session.id, and Langfuse uses the same session ID.
- **Outside the request path.** The collector and bus only observe. If they go down, telemetry is lost, but nobody's work is blocked.

## Setup steps

Build it in four phases: agree the contract, capture, consume, then prove. Devs should only ever run one setup script; everything else is pushed centrally.

**Phase 0: Contract and foundations**

1. Write the canonical event contract (v0). Wrap every event in CloudEvents. Use OTel GenAI attribute names where they exist (`gen_ai.usage.input_tokens`, `gen_ai.request.model`). Extend the common SDLC schema from the October brief (user, role, team, stage, tool, work item, cost, outcome).
   - Work events: `WorkItemCreated`, `WorkItemStatusChanged`, `WorkItemDone`.
   - AI events: `AISessionStarted` (session.id, harness, user, branch), `AIUsage` (tokens, cost, model, billing\_source).
   - Delivery events: `CIRunCompleted`, `PRMerged`, `QAGatePassed`, `QABouncedBack`, `TaskOutcome`.
2. Define the work-item ID: a ULID with a `wi_` prefix, minted by us. Keep the alias table (`wi_… ↔ jira:RE-1234 ↔ git:branch/…`) as a lookup next to the existing org and price tables, with a reserved `wi_unassigned` row.
3. Deploy the existing Agentic Ops collector (config, APIM Event Hub overlay, Container Apps script). Add processors for `org.bu=hiscox-re`, alias resolution, dropping prompt content, and batch and retry. It sits outside the request path, so if it fails, devs are not blocked.

**Phase 1: Capture**

4. **Claude Code.** Push managed settings (via Intune or MDM): telemetry on, OTLP endpoint set to the collector, prompt logging off. Ship a SessionStart hook that emits `AISessionStarted` with the session ID and current branch. Attribution then joins on session.id downstream, so nothing has to change inside the running process.
5. **Copilot.** Set enterprise-managed OTel settings to point at the collector, with content capture off. Pull the Copilot usage metrics API daily for AI credits per user (the billing truth for Copilot).
6. **APIM.** Add the token-metric policy with dimensions for user object ID and a session header. Route Claude Code through APIM where the base-URL override works with our Vertex setup.
7. **GitHub.** Webhooks for workflow runs and PRs go to a GitHub adapter. Branch names are resolved to work items there.
8. **Jira.** A Jira Automation rule (Send web request) posts issue changes to the collector as canonical work events and creates alias rows. That rule is the only piece that knows Jira exists.
9. **Dev setup script.** One command (it could live in the slots script) installs the hooks and the branch naming check, then sends a test event so the dev can see they are covered.

**Phase 2: Consume**

10. **New Relic** is the system of record. Canonical events land as custom events. Build the Agentic Ops dashboard from the existing NRQL queries: cost by harness and model, coverage, error rates.
11. **Power BI** reads the output of `value_view.py` and `leadership_rollup.py`, extended to take canonical events (via NRQL export). Keep their privacy rules unchanged. Build the value page on top.
12. **Langfuse** stays optional, as a personal workbench. Evals run from YAML cases in git and post `EvalResult` events to New Relic.

**Phase 3: Prove**

13. Run with the squad for two to three weeks. Check the success criteria.
14. Run the swap test: replace the Jira adapter with a stub source and confirm nothing downstream changes.
15. Review the results with the Engineering Manager and the Head of Platform, and decide whether to scale.

## Metrics

The headline for leadership is cost per delivered work item, next to first-pass QA rate. Every other metric explains or qualifies those two numbers.

| Metric | Definition | Source | Audience |
| --- | --- | --- | --- |
| Cost per delivered work item | AI cost of all sessions attributed to a work item that reached Done | Usage events + work events | Leadership, team |
| First-pass QA rate | Share of work items that pass the QA gate without a bounce-back | QA gate, Jira adapter | Leadership, team |
| Session success rate | Share of sessions whose work item reached Ready for QA with CI green and no bounce-back | Sessions + delivery events | Team, dev |
| Model effectiveness | Cost per passed task by model, from observed sessions plus a replay benchmark on closed tickets (October brief, Phase 7) | Usage + outcomes | Leadership, platform |
| Session efficiency | Turns, tool error rate, retries and cost per session | Langfuse, Preflight (optional) | Dev |
| Cycle time | Work item created to Done, compared before and after AI use | Work events | Team, leadership |
| Spend by harness and model | Cost split by Claude Code, Copilot, own agents, and by model | Usage events, Copilot API, APIM | Platform, leadership |
| Coverage | Users emitting telemetry divided by licensed users | Collector + seat lists | Platform |
| Attribution rate | Sessions resolved to a work item divided by all sessions | Alias resolution | Platform |

Per-dev views are for the dev; leadership sees team-level aggregates. Keep the value view's privacy rules: hashed user IDs, no cell with fewer than 3 distinct users, and complementary suppression. That keeps people willing to instrument.

## Dependencies

Most of the blockers are admin access, not engineering work, so request them in week one.

| Dependency | Who holds it | Needed for |
| --- | --- | --- |
| Claude Code managed settings pushed via Intune or MDM | Endpoint / IT team | Telemetry on by default (step 4) |
| GitHub Enterprise admin: managed OTel settings and a usage metrics API token | GitHub enterprise or org owner | Copilot capture and AI credits (step 5) |
| APIM v2 policy changes | Head of Platform | Token metrics per user and session (step 6) |
| Azure Container Apps for the collector (deploy script exists) | Platform / subscription owner | Steps 3, 7, 8, 12 |
| A Jira Automation rule with Send web request (API token already held for backfill) | Jira admin | Jira adapter (step 8) |
| GitHub webhooks on the squad's repos | Repo admins | GitHub adapter (step 7) |
| New Relic OTLP licence key and dashboard permissions | New Relic account owner | Agentic Ops dashboard (step 10) |
| Nothing new for Langfuse: it stays a personal workbench | Platform | Session traces (step 11) |
| Power BI workspace and data connection | Management BI owners | Value page (step 12) |
| Security and privacy sign-off on per-user telemetry | Enterprise security, DPO / HR | Before any per-user data is collected |
| Branch naming convention with the ticket key | The squad | Attribution (steps 4, 7, 9) |

## Known unknowns

Each of these needs a spike or a conversation before the design is final. The first two decide how good attribution can be.

- [ ] **Copilot session attribution.** Does Copilot expose a session ID and hooks we can use to record the branch? If not, Copilot sessions can only be attributed by repo and time window.
- [ ] **Claude Code session join.** Confirm that the SessionStart hook sees the same session.id the OTel export carries, so the join works.
- [ ] **User identity in Copilot telemetry.** Which user attribute does Copilot's OTel carry, and does it map to the Entra object ID?
- [ ] **Claude Code through APIM on Vertex.** Can Claude Code be routed through APIM with our Vertex setup without breaking auth or streaming?
- [ ] **Cost accuracy.** How close is Claude Code's client-side cost estimate to the Vertex bill? Agree a tolerance and a monthly reconciliation.
- [ ] **Claude inside Copilot.** Claude sessions run inside VS Code Copilot are billed by GitHub. How are they tagged, and how do we avoid counting them twice?
- [ ] **Group's LiteLLM shape.** Does Group want spend logs (Postgres), OTLP, or both? Which identity do they key on?
- [ ] **Store choice.** Which store should Power BI read from: Azure SQL, Fabric or Postgres? Pick whatever management BI already supports.
- [ ] **Branch switching.** How often do sessions switch branch mid-session? Measure it with a `branch_changed` flag before building anything for it.
- [ ] **Azure DevOps timing.** When (if) does the move off Jira happen? That sets when the second work adapter is needed.
- [ ] **Preflight's role.** Do we keep it as an optional dev-side feed, given native OTel covers most of the same data?

## Risks and mitigations

The biggest risk is people, not technology. If per-user telemetry reads as surveillance, coverage drops and the numbers stop being true.

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Per-user telemetry seen as surveillance; EU employee-monitoring rules apply (Portugal and UK) | Pushback, lower coverage, possible legal blocker | Get a privacy assessment done early; collect metadata only; show leadership team-level views; devs see their own data |
| Partial coverage | Spend understated, value story loses credibility | Push settings centrally; publish coverage as a metric |
| Prompt or code content leaks into telemetry | Data exposure | Turn off content capture in every harness; the collector drops content attributes as a backstop |
| Cost figures disagree across sources (client estimate, APIM, Copilot API, Vertex bill, Group LiteLLM) | Trust in the numbers collapses | One price table; a named source of truth per `billing_source`; monthly reconciliation |
| Gaming the metrics (Goodhart's law) | Cheap sessions rewarded over good outcomes | Always pair cost with quality outcomes; no individual targets |
| Vendor telemetry schemas change (still evolving in Claude Code and Copilot) | Broken dashboards | Adapters normalise to our contract; contract tests on sample payloads |
| Preflight is experimental | Dependency risk | Keep it optional and off the critical path |
| Built by one person | Stalls if attention moves | Limit the PoV to one squad; use managed services; document the contract so others can add adapters |
| Chaining to Group's LiteLLM later (we dropped LiteLLM after its supply-chain breach) | Security exposure at the boundary | Keep APIM as the BU boundary; Group receives telemetry, not raw traffic, unless agreed |

## Group-readiness and next steps

Group can collect from this by adding one exporter to the collector. These design choices keep that true:

- Every event carries `org.bu=hiscox-re` and uses OTel GenAI attribute names.
- Cost uses the same model price map as Group's LiteLLM, so totals match.
- Only metadata leaves the BU; prompts and responses never do.
- Copilot and coding-agent usage, which Group's gateway cannot see, comes with the data. That is a reason for Group to adopt this pattern.

**Next steps**

- [ ] Review this plan with the Engineering Manager and the Head of Platform; pick the PoV squad.
- [ ] Send the admin requests in the Dependencies table.
- [ ] Start the privacy assessment for per-user telemetry.
- [ ] Write event contract v0 and create the alias table.
- [ ] Stand up the collector and connect Claude Code first (it has the most mature telemetry).
- [ ] Spike the two attribution unknowns: the Copilot session ID and the Claude Code session join.
- [ ] Have an informal chat with Group about their LiteLLM data shape.
