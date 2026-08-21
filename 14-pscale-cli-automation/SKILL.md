---
name: planetscale-pscale-cli-automation
description: >-
  Use the PlanetScale CLI (pscale) from automated agents with --format json,
  auth check, pscale sql, and per-command --force. Run before other PlanetScale
  skills when driving pscale directly. Use when the user asks to automate
  pscale, run CLI commands headless, or verify pscale auth from an agent.
---

# PlanetScale CLI automation

## Purpose

Teach agents how to invoke `pscale` non-interactively. This skill covers **CLI
conventions only**. Operational workflows (inventory, safety review, schema
recommendations) use the other skills in this repo — start with
`../00-safe-orchestrator/SKILL.md` for a full assessment.

## Two AGENTS.md files (do not confuse them)

| Document | Where | Purpose |
|----------|-------|---------|
| **CLI agent guide** | Shipped with `pscale` (`AGENTS.md` in the CLI repo, or `pscale agent-guide`) | How to call `pscale`: auth, `--format json`, flag placement, `pscale sql` |
| **Project agent guide** | Your application repository's `AGENTS.md` | Which org, database, branch, engine, prod branch, MCP scope, approval rules |

Do not edit project `AGENTS.md` without operator approval (see
`../09-mcp-agent-operating-model/SKILL.md`).

## Bootstrap (always start here)

```bash
pscale agent-guide --format json
pscale auth check --format json
```

If `auth check` returns `"status": "action_required"`, follow `issues` and
`next_steps` in the JSON. For login, the human may need to approve in the
browser; use `pscale auth login --format json`.

## Conventions

- Always pass **`--format json`** for automation.
- Put **`--org <org>`** on resource subcommands (`database`, `branch`, `sql`,
  `api`, …) — not on root `pscale`.
- Put **positional arguments before flags** (`pscale sql mydb main --org bb …`).
- Use **`pscale sql`**, not `pscale shell` (shell requires a TTY).
- Default SQL role is **reader**; pass `--role admin` (or writer/readwriter) for
  writes. Match `pscale shell` semantics for `--role` and `--replica`.
- **`--force`** is per subcommand only (e.g. `database delete … --force`, `pscale
  sql … --force`). There is no global `--force` or `PSCALE_FORCE`.
- **`--format json` alone never skips confirmations** — add `--force` on the
  destructive subcommand after explicit user approval.

## Typical workflow

```bash
pscale auth check --format json
pscale org list --format json
pscale database list --org <org> --format json
pscale branch list <database> --org <org> --format json
pscale sql <database> <branch> --org <org> --format json --query "SELECT 1"
```

MySQL uses `@primary` by default (same as `pscale shell`); pass `--keyspace` only
for multi-keyspace databases.

## Current command surfaces to prefer

Use the highest-level `pscale` subcommand before falling back to `pscale api`.
These commands are useful in automated assessments when available in the local
CLI:

Read-only inventory and drill-down:

- `pscale traffic-control budget list <database> <branch> --org <org>
  [--fingerprint <fingerprint>]` — list Postgres Traffic Control budgets, or
  budgets with a rule for one query fingerprint.
- `pscale branch query-patterns list <database> <branch> --org <org>` and
  `pscale branch query-patterns show <database> <branch> <report-id>
  --org <org>` — inspect generated query-pattern reports.
- `pscale role default <database> <branch> --org <org>` — view the default
  Postgres role without rotating credentials.
- `pscale password show <database> <branch> <password-id> --org <org>` or
  `--name <name>` — inspect Vitess password metadata and IP allowlists without
  rotating or emitting the secret.
- `pscale insights errors show <database> <branch> <fingerprint> --org <org>`
  and `pscale insights anomalies show <database> <branch> <anomaly-id>
  --org <org>` — inspect one Insights error fingerprint or anomaly.
- `pscale database aggressive-cutover show <database> --org <org>` — inspect
  the Vitess aggressive-cutover default for future deploy requests.
- `pscale branch extensions list <database> <branch> --org <org>` — list
  Postgres extensions available on the branch cluster image.
- `pscale org member list --org <org>` and
  `pscale org member show <email-or-user-id> --org <org>` — inspect
  organization membership when the operator's requested scope includes org
  administration.

Mutation or destructive commands require the approval contract in
`../11-change-gates-and-approval-contract/SKILL.md` before execution:

- `pscale branch query-patterns delete <database> <branch> <report-id>
  --org <org>`.
- `pscale password update <database> <branch> --name <name> --cidrs <cidrs>
  --org <org>`; this changes metadata/IP allowlists but does not rotate the
  secret.
- `pscale keyspace delete <database> <branch> <keyspace> --org <org>`.
- `pscale deploy-request unblock <database> <number> --org <org>` and
  `pscale deploy-request update <database> <number> ... --org <org>`.
- `pscale database aggressive-cutover enable <database> --org <org>`.
- `pscale branch update <database> <branch> --new-name <name>
  --deletion-protected --org <org>`; only flags passed are sent.
- `pscale org member update <email-or-user-id> --role <role> --org <org>` and
  `pscale org member remove <email-or-user-id> --org <org>`.

Dashboard-only defaults can affect what humans do even when automation uses the
CLI. The Vitess **Prefer instant** setting changes the dashboard deploy button
default when instant deployment is available; CLI/API deploys still require
explicit `--instant` or `instant_ddl`.

## MCP vs CLI

- **MCP clients** — use the hosted PlanetScale MCP server (see `pscale agent-guide
  --format json` for the current URL).
- **Shell scripts and coding agents** — use `pscale` with `--format json` as above.

## When this skill is not enough

Install the full PlanetScale skills pack (if not already):

```sh
git clone https://github.com/planetscale/skills.git && cd skills && script/setup
# or: npx skills add planetscale/skills -g -y
```

Then run sub-skills or `../00-safe-orchestrator/SKILL.md` for database operations
beyond basic CLI invocation.

## Current conventions source of truth

Prefer live output over memorized flag syntax:

```bash
pscale agent-guide --format json
```

The embedded `guide` field contains the full CLI agent guide shipped with your
`pscale` binary.
