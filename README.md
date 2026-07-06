# PlanetScale Skills

A skill pack that turns your coding agent into a PlanetScale database
reviewer and operator. Point it at an organization and it audits
configuration, query behavior, open recommendations, and safety posture —
then, with your approval, does the work: reviewable schema changes,
instrumentation PRs, webhooks, and scheduled automation.

Works with any agent that reads `SKILL.md` files — Cursor, Claude Code,
and anything following the open agent-skills convention. The skills are
plain markdown instructions; all output is markdown and text. Nothing is
agent-specific: no plugins, no custom tools, no proprietary rendering.

## What it does

- **Assess** — a read-only, evidence-backed review of one database or a
  whole organization: branch and safety workflow, backups and restore
  posture, credentials, Query Insights, anomalies, SQL comment / query tag
  coverage, open schema recommendations, webhooks, and Traffic Control.
- **Report** — a single markdown report where every claim carries the
  command that produced it and every recommendation carries a measured
  finding, a gate, and a rollback path.
- **Execute** — on your approval, changes ship through PlanetScale's own
  safety mechanisms: development branches, pull requests, deploy requests
  with revert windows, warn-before-enforce Traffic Control budgets.
- **Automate** — scheduled agent loops that sweep recommendations into
  ready-to-ship PRs, watch for query regressions, and detect posture
  drift.

## Setup

### Prerequisites

1. **PlanetScale CLI**, authenticated for automation:

   ```sh
   brew install planetscale/tap/pscale   # or see planetscale.com/cli
   pscale agent-guide --format json      # CLI conventions (or load skill 14-pscale-cli-automation)
   pscale auth check --format json       # verify auth; follow next_steps if action_required
   pscale org list --format json         # verify access
   ```

   Agents should always pass `--format json` on `pscale` commands. Use `pscale sql`
   for non-interactive queries, not `pscale shell`. See skill `14-pscale-cli-automation`
   or the CLI repo `AGENTS.md` for full conventions.

2. **Optional: PlanetScale MCP server.** The skills prefer the
   insights-only MCP server for telemetry analysis when it is available,
   and fall back to `pscale` / `pscale api` otherwise. Nothing requires
   MCP.

3. **Recommended: run from your application repository.** The skills
   work anywhere, but they are significantly better when the agent runs
   inside the repo that contains the code talking to the database. See
   [Run it from the repo that uses the database](#run-it-from-the-repo-that-uses-the-database).

### Install the skills

The skills work with any agent that reads `SKILL.md` files. Pick the
route that fits:

**Setup script (recommended).** Detects installed agents and installs to
each of them:

```sh
git clone https://github.com/planetscale/skills.git
cd skills && script/setup
```

The script installs to every agent it finds (`~/.cursor/skills/`,
`~/.claude/skills/`, `~/.agents/skills/`). For any other agent, pass its
skills directory explicitly:

```sh
script/setup ~/.codex/skills          # or wherever your agent reads skills
```

**Skills CLI.** If you use the `npx skills` package manager:

```sh
npx skills add planetscale/skills -g -y
```

**Manual.** Copy the numbered directories into your agent's skills
location:

| Agent | User-level | Project-level |
|---|---|---|
| Cursor | `~/.cursor/skills/` | `<repo>/.cursor/skills/` |
| Claude Code | `~/.claude/skills/` | `<repo>/.claude/skills/` |
| Open agent-skills convention | `~/.agents/skills/` | `<repo>/.agents/skills/` |
| Anything else | the directory your agent documents for `SKILL.md` files | |

One constraint in all cases: the directories must stay siblings — skills
reference each other by relative path
(`../11-change-gates-and-approval-contract/SKILL.md`). Some installers
(including the Skills CLI) rename the folders to their frontmatter names
(`planetscale-change-gates-and-approval-contract`); agents should resolve
a missing numbered path by locating the sibling skill with the matching
frontmatter `name` instead.

## Usage

### Run it from the repo that uses the database

You can run these skills from anywhere, but they are at their best when
the agent runs inside the repository that contains the code talking to
the database. With the codebase in view, the agent can:

- map expensive query patterns in Insights back to the routes, jobs, and
  ORM calls that produce them
- add SQLCommenter query tags to your framework so future load is
  attributed automatically
- fix the code behind findings (N+1s, missing pagination, unindexed
  lookups) instead of only reporting them
- turn schema recommendations into migrations in your framework's own
  migration system, ready for review as a normal PR

Without the repository, the assessment still covers everything on the
database side; it just stops at recommendations where the fix lives in
code. If your agent supports project-level skills, installing the pack
into the application repo (for example `<repo>/.cursor/skills/`) gets
you this by default.

### Run an assessment

Ask in plain language. The orchestrator skill picks up the request and
runs every phase. For the full treatment, start with a prompt that puts
everything in scope:

> Run the full PlanetScale best-practices assessment on my-org/my-db.
> Review the whole setup against best practices: branch topology and
> schema-change workflow, safe migrations and deploy requests, backups
> and restore posture, roles and credentials, connection pooling and
> network configuration, Query Insights behavior, anomalies, SQL comment
> / query tag coverage, open schema recommendations, webhooks, and
> Traffic Control. My application repository is in this workspace:
> include query attribution and instrumentation recommendations for it,
> and map findings to the code that produces them.
> Produce the complete evidence-backed report with IDs for every
> recommendation. Read-only: propose everything, change nothing.

Everything in that prompt beyond the first sentence is optional — the
orchestrator covers all phases by default. Scope it down whenever you
want less:

> Check the larger databases in my-org.
> Audit our query tag coverage and open recommendations.

The first pass is always read-only. You get the report; nothing changes.

### Approve changes

Every recommendation in the report has an ID. Approve the ones you want:

> Approve VIT-1 and WEB-1.

The agent shows the exact commands, expected effect, and rollback plan
for each approved ID, executes them one at a time, verifies each by
reading state back, and reports.

Work that lives inside a review workflow — development branches, pull
requests, deploy requests — does not need approval to *prepare*. The
agent's default deliverable for a schema recommendation is the complete
reviewable unit: a branch with the DDL applied, a PR with the evidence,
and an open deploy request. Your action is the merge/deploy decision.

### Full autonomy (opt-in)

If you want the agent to execute the whole plan without per-change
approval, say so explicitly. Three elements are required — an
unambiguous risk acknowledgment, a named scope, and whether production
is included:

> I accept the risk — apply all report recommendations to my-db,
> production included.

The agent then plans in dependency order, executes one atomic change at
a time, verifies each step, streams status the entire way, and halts on
failure with staged rollbacks. "Go ahead" and "fix everything" do not
activate this mode; the acknowledgment must be explicit, and it covers
one run and one scope only.

### Scheduled automation

For unattended runs (cron, CI, agent schedulers), interactive
acknowledgment is replaced by a **standing authorization**: a committed
file naming the owner, scope, allowed operations, numeric bounds, expiry
date, and a status delivery channel. Details and templates are in
`13-autonomous-execution-mode/SKILL.md`. Recommended loops (see
`09-mcp-agent-operating-model/SKILL.md`):

- **Recommendation-to-PR** (daily) — open schema recommendations become
  ready-to-ship branch + PR + deploy request units. No authorization
  needed; it only produces proposals.
- **Recommendation deployer** (daily) — deploys approved deploy requests
  within authorized bounds. Requires a standing authorization.
- **Regression watch** (hourly or per-deploy) — diffs query patterns
  against a baseline and reports regressions with the responsible deploy.
- **Posture drift check** (daily) — reports changes to safety flags,
  webhooks, roles, and backup posture since the last assessment.

## Safety model

Every operation the agent might take is classified before it happens
(`11-change-gates-and-approval-contract/SKILL.md`):

| Class | Scope | Default |
|---|---|---|
| A | Read-only: list, inspect, report | Runs freely |
| B | Proposals: dev branches, PRs, deploy requests, restore tests | Runs freely — the review workflow is the gate |
| C | Behavior-changing: safety flags, webhooks, roles, budgets | Approval or authorization required |
| D | Production data/availability: deploys, DDL, deletions, credentials | Approval + named target + rollback plan |
| E | Destructive without recourse: drop production data, disable all safety, expose secrets | Never executed autonomously, under any phrasing |

Two properties hold in every mode, including full autonomy: a change is
verified by reading state back (not by command exit codes), and every
run produces an audit log. Reports end with the same contract line:

> No changes have been applied. Approve specific change IDs before any
> mutation.

## The skills

| # | Skill | Role |
|---|---|---|
| 00 | safe-orchestrator | Runs the full assessment end to end; the usual entry point |
| 01 | readonly-inventory | Evidence collection: org, branches, backups, webhooks, roles |
| 02 | vitess-safety-review | Vitess: safe migrations, deploy requests, revert, sharding |
| 03 | postgres-safety-review | Postgres: roles, pg_strict, Traffic Control, PITR, pooling |
| 04 | query-insights-and-tags | Query behavior, anomaly review, tag coverage and cardinality |
| 05 | traffic-control-recommendations | Warn-first budget plans for Postgres traffic slices |
| 06 | webhook-automation-recommendations | Event routing, receiver requirements, automation flows |
| 07 | schema-recommendations-agent-loop | Recommendation triage into reviewable units |
| 08 | codebase-sqlcommenter-instrumentation | Repository review and query-tagging PR plans |
| 09 | mcp-agent-operating-model | Agent/MCP configuration and scheduled loop catalog |
| 10 | customer-report-template | Report format, tone, and evidence requirements |
| 11 | change-gates-and-approval-contract | The Class A–E permission model |
| 12 | best-practices-matrix | Per-engine coverage checklist so assessments miss nothing |
| 13 | autonomous-execution-mode | Risk-acknowledged autonomy: contract, status protocol, halt rules |

Each skill is a standalone `SKILL.md` — readable as documentation,
executable by the agent. Start with `00-safe-orchestrator/SKILL.md` to
see how a full run fits together.

## Disclaimer

These skills direct AI agents that read from and — with your approval —
make changes to databases and infrastructure. Agent behavior is not
deterministic, and no safety model eliminates risk. You are responsible
for reviewing changes before they are applied, for maintaining backups,
and for the consequences of enabling autonomous execution or scheduled
automation.

This software is provided "as is", without warranty of any kind.
PlanetScale, Inc. accepts no liability for any damages, data loss,
downtime, or costs arising from its use, as set out in the
[LICENSE](LICENSE). These skills are not part of the PlanetScale
service and are not covered by any service agreement or SLA.

## License

MIT — see [LICENSE](LICENSE).
