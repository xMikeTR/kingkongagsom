# QA Gate — setup instructions for Claude Code

**Audience:** a Claude Code agent (Sonnet or Haiku) wiring this kit into Mike's work repo.

**Goal:** Mike's Jira tickets reach *Ready for QA* only with proof. A ticket can move there only when CI has proven that:
- every acceptance criterion is tested on video,
- visuals match the approved baselines,
- mutation testing proves the tests actually catch bugs,
- the anti-gaming guard is clean,
- an independent reviewer agent passes,
- and Mike has approved the run.

**How Jira is reached:** there is **no Jira API token**. Jira is reached only through the Atlassian MCP in Mike's Claude Code session. CI never touches Jira. CI proves and records; Mike's session moves the ticket through MCP. A hook allows the move to *Ready for QA* only after checking, against Azure DevOps directly, that a succeeded and approved CI run exists for that exact commit.

## Rules for you, the setup agent
1. Work through the steps **in order**. Each step ends with a **Check**. Do not start the next step until the check passes.
2. When a step says **ASK MIKE**, stop and ask. Do not guess repo structure, status names, URLs, IDs or secrets.
3. Never commit secrets. Secrets go in the Azure DevOps variable group, or in `.claude/settings.local.json` (not committed).
4. Do not "fix" a failing check by weakening the kit: lowering thresholds, deleting guard rules, adding retries, loosening screenshot tolerance or editing hooks. Report the failure to Mike instead.
5. Keep a running log of what you changed in `qa-gate-setup-log.md` at the repo root, for Mike to review.
6. If a command or flag in this file doesn't exist in the installed version, check `--help` or the tool's docs. If the fix isn't obvious, report it to Mike. Don't improvise a replacement.

## Which model for which step

| Step | Who | Why |
|---|---|---|
| 0 Discover, 1 Place files | **Sonnet** | Reading an unfamiliar repo and merging configs needs judgement. |
| 2 Install, 3 Conventions, 4 Jira + ADO access | Haiku is fine | Mechanical, with clear checks. |
| 5 Azure DevOps pipeline | **Sonnet** | Pipeline templating and stage outputs are easy to get subtly wrong. |
| 6 Guardrails | Haiku is fine | Copy, merge, configure, test. |
| 7 Baselines | Mike + any model | Mike approves the images. |
| 8 Dry run + sabotage tests | **Sonnet** | Interpreting failures matters. |
| 8b, 8c, 9 | Haiku is fine | Following procedures. |

If Haiku fails the same check twice, stop and hand the step to Sonnet. Don't keep retrying.

---

## What's in the kit

| Path | Purpose |
|---|---|
| `tools/ticket/cli.ts` | `ticket` CLI: `snapshot`, `acs`, `verify`, `ci-check`, `promote`. Never talks to Jira. |
| `tools/ticket/acs.ts` | AC parsing and the per-ticket AC snapshot (`<KEY>.acs.json`), captured once from Jira. |
| `tools/ticket/evidence.ts` | `Evidence` type with a private constructor. It only exists when every check passes. |
| `tools/ticket/ado.ts` | Finds the succeeded, approved CI run for a ticket and commit via the `az` CLI, and downloads its evidence. |
| `tools/ticket/guard.ts` | Anti-gaming checks on the PR diff: skipped tests, removed assertions, changed baselines, fixtures, AC snapshots or gate config. |
| `app/src/ui/e2e/acceptance/fixtures.ts` | Shared Playwright fixture. Fails on console errors, failed API calls and serious a11y issues. Provides `mockApi`, `apiJson`, `expectSameShape`. |
| `app/src/ui/e2e/acceptance/PROJ-123.spec.ts` | Example spec showing the conventions. Delete it once real specs exist. |
| `app/src/ui/e2e/acceptance/api-fixtures/` | Frozen API responses, so screenshots don't change with Snowflake data. |
| `app/src/ui/e2e/playwright.config.ts` | Two viewports, video and traces in CI, zero retries, strict visual diff. |
| `web/stryker.conf.json` | StrykerJS config for the React app (vitest runner). |
| `api/stryker-config.json` | Stryker.NET config for the C# API. |
| `scripts/review-agent.sh` | Fresh-context reviewer using `claude -p`. Writes `evidence/review.json`. |
| `scripts/draft-acs.sh` | Drafts AC lines from a ticket (read via MCP), or functional parity ACs from the legacy repo and website. Prints only. |
| `pipeline/qa-gate.yml` | Azure Pipelines template: Verify stage, then ApproveForQA behind Mike's approval, which publishes a `promotion` record. |
| `.claude/settings.json` | Permission rules and hook registration. |
| `.claude/hooks/jira-gate.mjs` | Gates Jira MCP tools. Allows the start transition, allows the *Ready for QA* transition only after `ci-check` passes, blocks other transitions and ticket edits. |
| `.claude/hooks/legacy-readonly.mjs` | Blocks any write or mutating command in the legacy repo. |
| `.claude/skills/ironclad-delivery/SKILL.md` | Day-to-day workflow skill for the coding agent. |

---

## Step 0 — Discover the repo (read-only)

Find and record in the setup log:
- Where the React app lives (the folder whose `package.json` has `react`), its test runner (vitest or jest), and whether `@testing-library/react` is present.
- Where the C# API lives (the folder with the `.sln` or API `.csproj`) and its test project(s).
- The existing Azure Pipelines YAML for PRs, and the stage or job that deploys the PR **slot**. Note how it exposes the slot URL.
- The API route prefix the React app calls (e.g. `/api/`).
- Node version (`node -v`), .NET SDK version (`dotnet --version`), and whether the `az` CLI is installed (`az --version`).

**ASK MIKE** to confirm what you found before continuing, especially the slot deploy stage and how its URL is exposed.

**Check:** the setup log lists the React dir, API dir, test runners, pipeline file, slot stage name, slot URL variable and API prefix.

---

## Step 1 — Place the kit files

Copy the kit into the repo root, keeping paths, **except the `.claude/` folder**. Leave `.claude/` out until Step 6: its permission rules block edits to `pipeline/`, `tools/ticket/` and `.claude/`, which you still need to adapt in Steps 1–5.

Playwright lives in Mike's repo at `app/src/ui/e2e`, and the kit already targets that folder: specs go in `app/src/ui/e2e/acceptance/`.

**Do not overwrite Mike's existing Playwright setup.** If `app/src/ui/e2e` already has a `playwright.config.*`:
- **ASK MIKE** which way to go:
  - merge the kit's settings into it as a separate project (`retries: 0`, `forbidOnly`, the evidence reporters and `outputDir` from `EVIDENCE_DIR`, video and trace in CI, two viewports, screenshot tolerance, timezone and locale), or
  - keep the kit's config as `playwright.qa-gate.config.ts` and run it with `--config`. If so, update the pipeline's Playwright step to pass `--config playwright.qa-gate.config.ts`.
- Existing e2e tests stay where they are. The gate only runs tests tagged `@<TICKET-KEY>`.
- If the e2e folder has its own `package.json`, install the Playwright dependencies (Step 2) there too.

Then adapt:
- Move `web/stryker.conf.json` into the real React folder. Move `api/stryker-config.json` next to the real API `.sln` or test project.
- If the React app uses **jest** instead of vitest: in `stryker.conf.json`, set `"testRunner": "jest"` and install `@stryker-mutator/jest-runner` instead of the vitest runner.
- If the React source isn't under `src/`, fix the `mutate` globs in `stryker.conf.json` and the `src/**` paths in the pipeline's React mutation step.
- If the repo root already has a `package.json`, **ASK MIKE** whether to merge the kit's devDependencies and scripts into it, or keep the kit in a `qa-gate/` subfolder. If it's a subfolder, update every path in `pipeline/qa-gate.yml`, `.claude/settings.json`, the hooks, and `guard.ts` (the `PROTECTED` list) to match.
- Add the kit's `.gitignore` entries to the repo's `.gitignore`.

**Check:** `git status` shows the new files in the agreed locations, and nothing outside them changed except `.gitignore`.

---

## Step 2 — Install dependencies

At the kit root (wherever its `package.json` ended up):
```bash
npm install -D @playwright/test @axe-core/playwright tsx typescript @types/node
npx playwright install --with-deps chromium
```

In the React folder:
```bash
npm install -D @stryker-mutator/core @stryker-mutator/vitest-runner @stryker-mutator/typescript-checker
```
Use `@stryker-mutator/jest-runner` instead if it's jest.

For the API:
```bash
dotnet new tool-manifest   # only if .config/dotnet-tools.json does not exist
dotnet tool install dotnet-stryker
```

Playwright CLI for agent exploration, on Mike's machine only:
```bash
npm install -g @playwright/cli@latest
playwright-cli install --skills
```

**Check:** `npx tsc --noEmit` passes at the kit root, `npx stryker --version` works in the React folder, and `dotnet stryker --help` works in the API folder.

---

## Step 3 — Configure the conventions

These values have defaults. **ASK MIKE** for the real ones, then set them.

| Setting | Where | Default |
|---|---|---|
| Jira status for QA (used in messages) | env `TO_STATUS` | `Ready for QA` |
| API path prefix | env `API_PATH_PREFIX` | `/api/` |
| Mutation threshold | pipeline parameter `mutationThreshold` | `80` |
| Branch naming | must contain the ticket key | `feature/PROJ-123-...` |
| AC format in Jira | lines in the description starting `AC-1`, `AC-2`, ... | — |
| Timezone/locale for screenshots | `playwright.config.ts` `use` | `Europe/London`, `en-GB` |
| Viewports | `playwright.config.ts` `projects` | 1440×900 and 1280×720 |

If Mike's Jira tickets don't use `AC-<n>` lines, **ASK MIKE** whether to adopt that convention or adapt `parseAcceptanceCriteria` in `tools/ticket/acs.ts`. Don't silently change the parser.

**Check:** the setup log records every value, and anything non-default is applied.

---

## Step 4 — Jira (MCP) and Azure DevOps (az CLI) access

### Jira via the Atlassian MCP
1. Confirm the Atlassian MCP is connected in Mike's Claude Code (`/mcp`). Note the server's exact name, e.g. `atlassian`. The tool names will be `mcp__<server>__<tool>`.
2. Find the tool names for: reading an issue, listing transitions, transitioning an issue, and adding a comment. Record them in the setup log.
3. On one of Mike's real tickets, use the MCP to **list its transitions** (read-only). Record:
   - the transition **id(s)** that lead to *In Progress*, the "start" transitions,
   - the transition **id(s)** that lead to *Ready for QA*.

   Transition ids are per workflow. If Mike's tickets use more than one workflow, collect the ids from each.

### Azure DevOps via the az CLI
Mike runs these himself, since they use his login:
```bash
az login
az extension add --name azure-devops
az devops configure --defaults organization=https://dev.azure.com/<org> project=<project>
```
**ASK MIKE** for the org and project.

### Local config
Put this in `.claude/settings.local.json`, which is not committed. Create the file now; the hooks that read it are installed in Step 6.
```json
{
  "env": {
    "JIRA_MCP_PREFIX": "mcp__<server>",
    "JIRA_START_TRANSITION_IDS": "<id>[,<id>]",
    "JIRA_QA_TRANSITION_IDS": "<id>[,<id>]",
    "QA_GATE_PIPELINE_ID": "<filled in at Step 5>",
    "LEGACY_REPO_PATH": "<filled in at Step 8b>"
  }
}
```

**Check:**
- Reading a ticket via MCP works.
- `az pipelines list -o table` lists the repo's pipelines.
- The transition ids are recorded.

---

## Step 5 — Azure DevOps pipeline (Mike or an admin does the clicking; you prepare the instructions)

1. **Variable group** `qa-gate-secrets` (Pipelines → Library):
   - `ANTHROPIC_API_KEY` (secret), or whatever env vars the `claude` CLI needs to reach Hiscox's AI gateway. **ASK MIKE** which.
   - `ALLOW_BASELINE_UPDATE` = `false` and `ALLOW_GATE_CHANGES` = `false`, both marked *settable at queue time*.
2. **Environment** `qa-gate` (Pipelines → Environments → New → None): Approvals and checks → Approvals → approver = **Mike only**.
3. **Pipeline:** include `pipeline/qa-gate.yml` as a template after the slot deploy stage, passing `frontendDir`, `apiDir` and `e2eDir`. Make sure:
   - The slot deploy stage is named `DeploySlot`, with a job `Deploy` and a step `slot` that sets an output variable `url`. If it's named differently, update the `stageDependencies.DeploySlot.Deploy.outputs['slot.url']` reference rather than renaming Mike's stage.
   - The `claude` CLI is available on the agent. Add `npm install -g @anthropic-ai/claude-code` before the review step if it isn't.
4. Find the pipeline's numeric id (`az pipelines list -o table`) and set `QA_GATE_PIPELINE_ID` in `.claude/settings.local.json`.

**Check:** a PR pipeline run shows the Verify stage, then ApproveForQA waiting for Mike's approval. After he approves, the run has `evidence` and `promotion` artifacts.

---

## Step 6 — Claude Code guardrails

**Do this step last among the file changes.** Once these rules are in place, they block you from editing `pipeline/`, `tools/ticket/` and `.claude/`. Finish any remaining edits in those folders first. If you need to change them after this step, stop and ask Mike to make the edit himself.

1. Copy `.claude/hooks/` and `.claude/skills/` from the kit. Merge `.claude/settings.json` into the repo's existing `.claude/settings.json`, if any. Keep every `deny` rule and both `hooks` entries.
2. If Mike keeps personal skills elsewhere (his own harness), **ASK MIKE** where to copy the skill as well.
3. Restart Claude Code so the hooks load.

**Check** in a fresh Claude Code session in the repo:
- Asking the agent to move a real ticket to *Ready for QA* via MCP is **blocked** with "no proof for <KEY>", because no approved run exists yet.
- Asking it to edit a ticket's description via MCP is **blocked**.
- Asking it to read a ticket via MCP works.
- Asking it to edit `tools/ticket/evidence.ts` is denied.

---

## Step 7 — Seed visual baselines (Mike must be present)

Baselines are the "approved look". They must come from a slot Mike has personally checked.
1. **ASK MIKE** for a slot URL he's happy with.
2. Run:
   ```bash
   SLOT_URL=<url> npx playwright test --update-snapshots
   ```
3. Show Mike the generated `app/src/ui/e2e/acceptance/__screenshots__/` images. Commit them only after he approves.

The agent's settings deny `--update-snapshots`, so Mike runs this command himself. The same applies whenever a ticket adds new screenshots: Mike updates, reviews and commits the new baselines, and queues that run with `ALLOW_BASELINE_UPDATE=true`.

**Check:** baseline PNGs are committed for both viewports, and a normal run against the same slot passes.

---

## Step 8 — Dry run end to end

On a real ticket of Mike's:
1. **Start.** Via MCP, read the ticket and transition it to *In Progress* (allowed by the hook).
2. **Capture the ACs.** Pipe the ticket description, exactly as read from MCP, into:
   ```bash
   npm run ticket -- snapshot <KEY>
   ```
   Show Mike the printed ACs. Once he confirms, commit the `<KEY>.acs.json` on its own.
3. **Tests first.** Write `app/src/ui/e2e/acceptance/<KEY>.spec.ts` following the skill. Watch the tests fail, then implement.
4. **Local checks.**
   - `npm run guard` should print `GUARD: clean`.
   - `cd <react dir> && npx stryker run --mutate <changed files>` should score at or above the threshold.
5. **Push and open the PR.** Verify should go green, then ApproveForQA should wait. Mike approves.
6. **Promote.** Run `npm run ticket -- promote <KEY>`. It verifies the approved run and prints the evidence comment. Post the comment via MCP, then transition to *Ready for QA*. The hook re-checks and allows it.

Then prove the gate holds. Run each of these on a throwaway branch; every one must block promotion:
- Add `test.skip` to an AC test → guard fails.
- Delete an `expect` from an existing test → guard fails.
- Edit an existing file in `api-fixtures/` or the `<KEY>.acs.json` → guard fails.
- Remove an AC test while the AC is still in the snapshot → `verify` fails ("AC-n has no test").
- Change a component's logic without a component test → mutation score drops and `verify` fails.
- Try the MCP transition to *Ready for QA* before Mike approves the run → hook blocks it.

**Check:** the happy path promotes, and all six sabotage cases are blocked. Record the results in the setup log.

---

## Step 8b — Drafting ACs, and legacy parity

Tickets without `AC-n` lines can't be snapshotted, by design. `npm run draft-acs` drafts ACs for Mike to review. It **only prints**: it reads the ticket via MCP and never writes to Jira.

1. **ASK MIKE** for:
   - the local path to the legacy system's repo,
   - the legacy website URL. Prefer a UAT or non-production instance if one exists,
   - whether a **read-only** legacy account exists. Strongly recommend one: the drafting agent explores the legacy site, and a read-only account is the real guarantee it can't change data. The prompt-level rules are a second layer, not the main one.
2. Make sure `playwright-cli` is installed (Step 2), and logged into the legacy site if it needs auth. Mike does the login himself.
3. Test both modes on real tickets:
   ```bash
   npm run draft-acs -- <KEY-with-description>
   npm run draft-acs -- <PARITY-KEY> --parity <legacy-repo> <legacy-url> "<screen name>"
   ```

**Check:**
- The description mode outputs `AC-n:` lines using only the ticket's content, plus PO questions.
- The parity mode outputs functional ACs, each citing `[legacy: file:line]`, and saves screenshots in `app/src/ui/e2e/acceptance/legacy-reference/<KEY>/`.
- Mike confirms that nothing on the legacy site changed.

Mike pastes the reviewed ACs into the Jira ticket himself. Then the snapshot is captured as in Step 8.

**The legacy repo is read-only for every agent.** Set up all three layers:
1. **Hook (installed in Step 6).** `.claude/hooks/legacy-readonly.mjs` blocks Edit/Write on any file in the legacy repo, and any Bash command touching it that isn't on a read-only list (grep, cat, ls, find without -delete/-exec, `git log/show/diff/blame`…). Set `LEGACY_REPO_PATH` in `.claude/settings.local.json`.
2. **Drafting run.** `draft-acs.sh` only allows read tools, explicitly denies write tools, and sets `LEGACY_REPO_PATH` for the hook automatically.
3. **OS backstop.** Make the clone itself unwritable, so even a mistake outside Claude can't change it:
   ```bash
   chmod -R a-w /absolute/path/to/legacy-repo
   ```
   To pull updates later: `chmod -R u+w`, `git pull`, then `chmod -R a-w` again.

**Check** in a Claude Code session in the work repo:
- Asking the agent to edit a legacy file is blocked with "the legacy repo is read-only".
- Asking it to run `git -C <legacy> checkout main` is blocked.
- `grep -rn <something> <legacy>/src` works.

Mike reviews the legacy reference screenshots before committing them. CI copies them into the evidence artifact, so they sit next to the new app's evidence.

---

## Step 8c — Backfill tickets already in Ready for QA

Mike has tickets that moved to *Ready for QA* before the gate existed. They go through the same pipeline, with the same checks and the same approval. At the end, `ticket promote` prints the evidence comment as usual; post it via MCP and **skip the transition**, since the ticket is already there.

For each ticket:
1. Make sure it has `AC-n` lines in Jira. If not, use `draft-acs`, and Mike pastes them in. Then snapshot as in Step 8.
2. On its branch, write the acceptance specs, then push. The pipeline runs Verify.
3. **If Verify passes:** Mike approves, `ticket promote` prints the comment, and you post it via MCP.
4. **If Verify fails:** stop and show Mike the failure list. Do not weaken anything to make it pass. Mike decides whether to fix it quickly before QA reaches it, or move the ticket back himself.

**Check:** each backfilled ticket either has an evidence comment, or is on a list for Mike with its failures.

---

## Step 9 — Trial period, then less manual

Mike manually checks the first 3–5 tickets. During the trial, `manualSignoff` stays `true` (the default).

For each trial ticket, before approving the run, Mike checks the slot himself and writes in the setup log:
- what he checked by hand,
- anything he found that the gate did NOT catch.

Anything the gate missed must become a check before the trial ends: a new fixture rule, a guard rule, a skill instruction or a reviewer prompt line. **ASK MIKE** which. A trial ticket where Mike found something the gate missed doesn't count towards the 3–5.

Once Mike confirms the trial is done:
1. Create a second environment `qa-gate-auto` with **no** approval checks.
2. Set the pipeline parameter `manualSignoff: false`.
3. The evidence comment then says the visual sign-off was automated against approved baselines, so nobody is told a human looked when none did.

Promotion itself still happens from Mike's Claude session via MCP, since CI has no Jira access.

**Check:** the next ticket's run completes without pausing, and its evidence comment shows the automated sign-off line.

---

## Known things to verify against installed versions

These depend on tool versions. Check each and adapt if the syntax differs:
- **Stryker.NET** `--since` flag syntax and the `since` block in `stryker-config.json` (`dotnet stryker --help`).
- **StrykerJS** `--mutate` accepts a comma-separated file list (`npx stryker run --help`).
- **az CLI** `pipelines runs list` flags (`--pipeline-ids`, `--result`, `--status`, `--top`) and `pipelines runs artifact download` (`--run-id`, `--artifact-name`, `--path`).
- **Claude Code** permission rule wildcard syntax, the hook `matcher` regex, and that hooks receive `tool_name`/`tool_input` on stdin (current docs).
- **Atlassian MCP** transition tool input: the hook looks for an issue key field and a transition `id`. If the tool's arguments differ, tell Mike rather than editing the hook.
- **`claude -p`** reads the prompt from stdin, and `--output-format json` returns `{ "result": ... }`.
- **Jira workflow**: a transition to *Ready for QA* must exist from *In Progress*. If it doesn't, tell Mike.

## Done means
- All step checks pass, and the setup log is complete.
- Mike has seen one ticket promoted with full evidence.
- All six sabotage cases are blocked.
