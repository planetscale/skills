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

## Read-only evidence commands

Use these with `--format json` and the normal `--org <org>` flag placement:

- Regions: `pscale database regions list <database>` and, for Vitess default
  branches, `pscale database read-only-regions list <database>`.
- Postgres switchover follow-up:
  `pscale branch switchover list <database> <branch>` and
  `pscale branch switchover show <database> <branch> <id>`.
- Metrics: `pscale metrics queries`, `pscale metrics tables`, and
  `pscale metrics tags` on Postgres or Vitess; Vitess also has
  `pscale metrics tablets` and `pscale metrics keyspace-tables`.
- Schema recommendations:
  `pscale insights recommendations show <database> <number>` fetches one
  recommendation, including full ready-to-apply DDL, for review.
- Query details: `pscale insights queries samples/show/summary` and
  `pscale insights queries traffic-budgets`; `show` takes a sample ID, while
  `samples`, `summary`, and `traffic-budgets` take a fingerprint.
- Traffic Control inventory:
  `pscale traffic-control budget list <database> <branch>` and
  `--fingerprint <fingerprint>` when narrowing to budgets that affect a query.
- Service-token metadata: `pscale service-token show <token-id>`. Treat this
  as credential-administration scope; do not copy token values into logs or
  reports if any are returned.
- Billing invoices: `pscale billing invoice list/show/line-items --org <org>`.
  Treat invoice and payment details as sensitive and only collect them when
  billing is explicitly in scope.
- Organization teams: `pscale org team list --org <org>` for read-only team
  inventory.

## Approval-gated commands

The commands below are available to the CLI, but they change state and must be
handled under `../11-change-gates-and-approval-contract/SKILL.md`:

- Postgres point-in-time restore branches: use `pscale branch create` with
  `--from <source-branch> --restore-point <timestamp>` (Postgres-only, cannot
  be combined with `--seed-data`). Creating the branch is a restore operation.
- Backup protection:
  `pscale backup update <database> <branch> <backup-id> --protected` or
  `--protected=false`.
- Organization billing/role settings:
  `pscale org update --org <org> ...`.
- Team and team-member management:
  `pscale org team create ...`, `pscale org team member add ...`, and related
  update/delete commands.
- Any credential, payment-method, branch restore, switchover start, backup,
  role, network, Traffic Control, deploy, or schema mutation.

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
