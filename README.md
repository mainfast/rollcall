# Mainfast Rollcall

Required-check integrity audit. Reads the required status checks from classic
branch protection **and** rulesets, then checks them against what recently
merged **and** open PRs actually reported, and against the workflow files that
are supposed to produce them. Posts a check run and keeps **one tracking issue**
open while something is wrong (closed automatically once it is fixed).

Licence: MIT; free for public repos and private repos with up to 5 active contributors ([Free and Pro](https://github.com/mainfast/mainfast/blob/main/README.md#free-and-pro)).

## Why

A required check protects nothing when:

- it is **skipped**: GitHub treats `skipped` (and `neutral`) as passing, so a job
  with a false `if:`, or one whose `needs` failed, lets the PR through;
- the job was **renamed**: the required list still names the old job, the new
  name is not required, and merges go through by bypass;
- it is a **matrix** job: the check is `test (20)`, never `test`;
- its workflow is **path- or branch-filtered**: PRs outside the filter wait on
  "Expected — Waiting for status to be reported" until someone bypasses;
- the ruleset and classic protection **drift** apart, or the ruleset is still in
  `evaluate` mode.

## Findings

Each finding in the check run, job summary and issue names the PRs that show it
and carries its fix.

| Code | Severity | Meaning | Fix |
|---|---|---|---|
| `never-ran` | error | Required, but no sampled PR reported it (and no similar name exists). Says whether any workflow job produces the name. | Remove the stale requirement or restore the job; merges need a bypass until then. |
| `renamed` | error | The required name stopped reporting (or never did) and a similar, non-required name took its place. | Put the new job name in the required list and drop the old one. |
| `stopped-reporting` | error | Missing on the newest N sampled PRs in a row after reporting on older ones, no rename candidate, and no path filter that explains it. | Find the commit that removed or renamed the job; restore it or update the list. |
| `matrix-name-mismatch` | error | Required `test`, the job reports `test (…)`. | Require each leg by full name, or add an `if: always()` gate job with the required name. |
| `skipped-as-success` | error | Concluded `skipped` on sampled PRs (merged or open), which GitHub counts as passing. | Make it a gate job with `if: always()` that fails unless its dependencies succeeded. |
| `path-filtered` | error | The producing workflow has `paths` / `paths-ignore` on its PR trigger. | Drop the filter and skip inside the job, or require a gate job that always runs. |
| `branch-filtered` | error | The producing workflow does not run for PRs into the audited branch. | Add the branch to the workflow's `pull_request.branches` (or remove the filter). |
| `not-on-pull-request` | error | The producing workflow has no `pull_request` / `pull_request_target` / `merge_group` trigger. | Add `pull_request` (and `merge_group` with a merge queue). |
| `app-mismatch` | error | Requirement pinned to app X, the runs come from app Y. | Pin it to the app that reports it (GitHub Actions is 15368), or to any source. |
| `required-workflow-missing` | error | A ruleset `workflows` rule points at a file that is gone. | Point the rule at an existing file, or remove it. |
| `conditional-job` | warning | The producing job has an `if:` without `always()`; a false condition reports skipped. | Move the condition into the steps, or make sure every skip is intended. |
| `skips-when-needs-fail` | warning | The producing job `needs` jobs that are not required; their failure skips it. | Add `if: always()` and fail on `contains(needs.*.result, 'failure')`, or require them too. |
| `neutral-as-success` | warning | Concluded `neutral`, which satisfies a required check. | Have the check conclude success or failure. |
| `merged-without` | warning | Merged PRs with no result for the check (bypass, or required later). | Review who bypassed (ruleset insights / audit log) and trim the bypass list. |
| `merged-with-failure` | warning | Merged while the check had failed. | Review the bypass; fix a flaky check instead of merging over it. |
| `merged-while-pending` | warning | Merged while the check was still running, even if it finished later, or it never finished (`stale`). A check still queued at merge time is only caught if it hasn't finished by the audit. | Wait for required checks; limit bypass, or use the merge queue. |
| `protection-drift` | warning | Context, explicit app pins that disagree, or "up to date" policy differs between classic protection and rulesets. | Keep one source of truth (move everything into the ruleset). |
| `ruleset-not-enforced` | warning | A ruleset with required checks is in `evaluate` or `disabled`. | Switch it to Active once its checks report reliably, or delete it. |
| `ambiguous-name` | warning | Two workflows produce the same check name. | Give each job a unique name. |
| `workflow-unparsable` | warning | A workflow file is not valid YAML. | Fix the YAML; GitHub does not run it either. |

Open PRs count for "does the check report at all" (never-ran, renamed, matrix,
stopped-reporting, skipped, neutral, app pin). A failing or pending check on an
open PR is normal and never a finding; the `merged-*` findings look at merged
PRs only. An open PR whose head has reported nothing yet (CI not started) is
left out and mentioned in the notes.

## GitHub Action

```yaml
on:
  schedule: [{ cron: "17 6 * * 1" }]   # weekly; also catches ruleset edits (see below)
  workflow_dispatch:
  branch_protection_rule:
  push:
    branches: [main]
    paths: [".github/workflows/**", ".mainfast.yml"]
permissions:
  contents: read
  pull-requests: read
  statuses: read
  checks: write
  issues: write
concurrency: { group: mainfast-rollcall, cancel-in-progress: true }
jobs:
  rollcall:
    runs-on: ubuntu-latest
    steps:
      - uses: mainfast/rollcall@v0
```

No `actions/checkout` step is needed: `.mainfast.yml` is read through the API
from the **default branch** (with `extends:` applied) and `.github/workflows/*`
from the **audited branch**, never from a checkout, so a `workflow_dispatch` run
on a feature branch audits the same thing as the scheduled one. A push to any
other branch than the audited one is a no-op. Full example:
[`examples/rollcall.yml`](examples/rollcall.yml) (if you add a checkout step
elsewhere, pin it: `actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09 # v5.1.0`).

**When it runs.** Weekly, by hand, on a push to main that touches workflows or
`.mainfast.yml`, and on `branch_protection_rule` (classic protection created,
edited or deleted; it runs on the default branch and only if the workflow file
is on the default branch). **Ruleset changes trigger no workflow**: GitHub has a
`repository_ruleset` webhook for apps, but no workflow event for it (checked
against docs.github.com "Events that trigger workflows", 28 Oct 2026). After
editing a ruleset, run the workflow by hand (`gh workflow run rollcall.yml`) or
wait for the weekly run.

**What it writes.**

- Job summary: findings with fixes, then a tally per required check.
- Outputs: `ok`, `conclusion`, `issue` (number), `report` (JSON).
- Check run `Mainfast Rollcall` on the audited branch's head commit:
  `failure` at the `fail-on` level, `neutral` with lower findings, else `success`.
- **One tracking issue** (`issue: true`, the default): opened when a finding
  reaches `issue-on` (default `error`), updated in place on later runs, closed as
  completed when the audit drops below that level. The issue is found again by
  its marker **and** its author: only an issue opened by the token's own identity
  (`comment-author` input, else `GET /user`, else `github-actions[bot]`) is ever
  edited or closed, so a marker pasted into someone else's issue is ignored. Set
  `comment-author` when the token is an app token (`my-app[bot]`).
- The job fails at the `fail-on` level (default `error`).

## Permissions

| Permission | Why | Without it |
|---|---|---|
| `contents: read` | `.mainfast.yml`, workflow files, "Get a branch" (classic required checks) | config and workflows unreadable: the run stops on the config read |
| `pull-requests: read` | merged and open PRs to sample; exact plan count | the run stops |
| `statuses: read` | commit statuses on each sampled head | statuses treated as empty (a note); commit-status-based checks look missing |
| `checks: write` | read check runs; post the result | no check run (a warning annotation); check runs cannot be read either |
| `issues: write` | the tracking issue | no issue (a warning annotation) |
| *(metadata: read, always granted)* | `GET /rules/branches/{b}`, `GET /rulesets`, `GET /rulesets/{id}` | — |

**No `administration` permission is needed**, and `GITHUB_TOKEN` cannot have
one. Active rulesets come from `GET /repos/{o}/{r}/rules/branches/{b}` and
`/rulesets` (metadata read; org rulesets included). Classic required checks come
from "Get a branch" (`protection.required_status_checks`, contents read). Only
if that response has no checks for a protected branch does Rollcall try the
admin endpoint `GET .../branches/{b}/protection/required_status_checks`; a 403
there becomes a note in the report ("classic required checks unknown") and the
audit carries on with the rulesets. Any other source that answers 403/404 is
also a note, treated as empty, so a partial audit still runs. A ruleset's
`bypass_actors` are only returned with write access to the ruleset; Rollcall
does not read them.

## REST endpoints used (API version 2022-11-28)

| What | Endpoint |
|---|---|
| Default branch | `GET /repos/{o}/{r}` (or the event payload) |
| Config | `GET /repos/{o}/{r}/contents/.mainfast.yml?ref={default branch}` |
| Classic required checks | `GET /repos/{o}/{r}/branches/{branch}`; fallback `GET .../branches/{branch}/protection/required_status_checks` (admin) |
| Active rules for the branch | `GET /repos/{o}/{r}/rules/branches/{branch}` (`required_status_checks`, `workflows`, with `ruleset_id`) |
| All rulesets incl. evaluate/disabled | `GET /repos/{o}/{r}/rulesets?includes_parents=true`, then `GET /repos/{o}/{r}/rulesets/{id}` |
| Merged and open PRs | `GET /repos/{o}/{r}/pulls?state=closed&base={branch}&sort=updated&direction=desc` (filtered on `merged_at`), `...?state=open...` |
| What each PR head reported | `GET /repos/{o}/{r}/commits/{sha}/check-runs?filter=latest`, `GET /repos/{o}/{r}/commits/{sha}/status` |
| Workflow files | `GET /repos/{o}/{r}/contents/.github/workflows?ref={branch}` + each file |
| Result | `POST /repos/{o}/{r}/check-runs` (summary ≤ 65535 chars) |
| Tracking issue | `GET /repos/{o}/{r}/issues?state=open`, `POST /repos/{o}/{r}/issues`, `PATCH /repos/{o}/{r}/issues/{n}` |
| Comment author | `GET /user` (403 for `GITHUB_TOKEN`: falls back to `github-actions[bot]`) |

Required-check resolution (classic + active rulesets + evaluate-mode rulesets +
app pins) and the reading of a check run or status as passed / failed / pending
/ skipped / neutral / stale / missing for one commit live in
`@mainfast/shared/required-checks`; Mainfast Evidence uses the same code for a
single PR head.

## Untrusted text

Check names (set by other apps), workflow paths and job names, `if:`
expressions, ruleset names, branch and repository names and GitHub error text
are escaped with the suite's helpers (`@mainfast/shared/markdown`) in the check
run, the job summary and the issue: no HTML, no links, no `@` mentions (a
zero-width space follows the `@`), no table breakouts, no forged marker.
Annotations and titles are plain text.

## Config

`.mainfast.yml`, section `rollcall:` ([reference](https://github.com/mainfast/mainfast/blob/main/docs/config.md#rollcall),
[example](https://github.com/mainfast/mainfast/blob/main/rollcall/.mainfast.example.yml)): `branch`, `pulls`, `openPulls`, `ignore`,
`stoppedAfter`, `failOn`, `issueOn`. Inputs override: `branch`, `pulls`,
`open-pulls`, `fail-on`, `issue-on`; `config:` reads a file from the workspace
instead of the default branch.

## CLI

```sh
GITHUB_TOKEN=... npx mainfast-rollcall audit --github-repo owner/name --pulls 30 --open-pulls 10 --format md
npx mainfast-rollcall audit --github-repo owner/name --workflows-from . --save-snapshot snap.json
npx mainfast-rollcall audit --snapshot snap.json --format json        # offline
```

Exit 1 on an error-level finding. The CLI only reads; it posts no check run and
no issue.

## Development

```sh
npm run all        # at the suite root: brand check, typecheck, every test, build
npm test -w @mainfast/rollcall
```

`test/fixtures/acme-web.routes.json` is a fake repository's API responses
(branch, rules, rulesets, 8 merged PRs with check runs, workflow contents) with
one planted case of most findings; tests serve it through the shared
`mockFetch`. `test/setup.ts` replaces the global `fetch` with one that throws,
so no test can reach the network.

## Limits

- Check runs from the PR's test merge commit are not read; only the head SHA.
- Matrix names are matched by pattern (`name (…)`); `${{ }}` expressions in a
  job name become wildcards.
- Rulesets that target by repository property or that only apply to some
  branches are taken as returned by `rules/branches/{branch}`, which already
  resolves them.
- Reusable-workflow producers are matched as `caller / …`; the called workflow
  is not opened.
- Up to 1000 open issues are scanned for the tracking issue.
