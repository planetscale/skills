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

## Live connection inspection

For live connection and session inventory, prefer the scriptable command:

```bash
pscale branch connections show <database> <branch> --org <org> --format json
```

`pscale branch connections top <database> <branch> --org <org>` opens the
interactive live view; when running headlessly, capture to a trace with
`--capture <file>` and `--duration <duration>`.

The command works for Postgres, Vitess, and Neki branches and adapts the
output to the engine. It requires a signed-in user (`pscale auth login`), not a
service token. For Vitess, pass `--keyspace` and `--shard` when automation
needs a non-interactive target. For Neki, results are client sessions on
routers only, not the connections between routers and each shard's Postgres
instances.

Action subcommands take effect immediately and do not prompt:

```bash
pscale branch connections kill <database> <branch> <connection-id> --org <org>
pscale branch connections kill <database> <branch> <query-id> --query --org <org>
pscale branch connections kill-transaction <database> <branch> <transaction-id> --org <org>
```

Agents should get IDs from `show --format json`, act on one session at a time
only after explicit approval, and run `show` again to verify the result. For
Neki, canceling a query leaves the router session open but aborts any open
transaction; terminating a session closes the client connection and rolls back
any open transaction.

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
