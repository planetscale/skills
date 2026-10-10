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
`../planetscale-safe-orchestrator/SKILL.md` for a full assessment.

## Two AGENTS.md files (do not confuse them)

| Document | Where | Purpose |
|----------|-------|---------|
| **CLI agent guide** | Shipped with `pscale` (`AGENTS.md` in the CLI repo, or `pscale agent-guide`) | How to call `pscale`: auth, `--format json`, flag placement, `pscale sql` |
| **Project agent guide** | Your application repository's `AGENTS.md` | Which org, database, branch, engine, prod branch, MCP scope, approval rules |

Do not edit project `AGENTS.md` without operator approval (see
`../planetscale-mcp-agent-operating-model/SKILL.md`).

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

## Operational command notes

Vitess deploy requests that partially fail can retry only the failed tables:

```bash
pscale deploy-request retry <database> <number> --org <org> --format json
```

Use this after inspecting the failed tables and error details. It is a schema
deployment recovery action, not a generic transient retry; for production deploy
requests it requires the same approval or standing authorization as applying
production DDL.

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

Then run sub-skills or `../planetscale-safe-orchestrator/SKILL.md` for database operations
beyond basic CLI invocation.

## Current conventions source of truth

Prefer live output over memorized flag syntax:

```bash
pscale agent-guide --format json
```

The embedded `guide` field contains the full CLI agent guide shipped with your
`pscale` binary.
