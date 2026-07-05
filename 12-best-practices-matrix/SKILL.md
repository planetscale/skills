---
name: planetscale-best-practices-matrix
description: A concise feature matrix for deciding which PlanetScale safety, observability, and automation recommendations apply by engine.
---

# Best-practices matrix

## Purpose

Map database findings to recommended PlanetScale features. Use this to ensure the assessment does not miss major safety and operational surfaces.

## Cross-engine recommendations

### Query Insights

Recommend for every production database:

- Review slow, expensive, high-frequency, and erroring query patterns.
- Correlate regressions with deploys.
- Use tags/comments to map queries back to code.
- Use anomalies as alert and automation inputs.

### Webhooks

Recommend for operational events:

- Anomalies.
- Storage pressure.
- Branch readiness/sleeping.
- Primary promotion.
- Maintenance.
- Schema recommendations where available.
- Deploy request lifecycle for Vitess.

### MCP and agents

Recommend:

- Use insights-only MCP for most autonomous analysis.
- Use full MCP only with narrow scopes and read-only default.
- Put database targeting and safety rules in the project's agent instructions file (`AGENTS.md`, `CLAUDE.md`, or equivalent — use whichever the project already has).
- Agents generate PRs/issues/change plans; humans approve database changes.

### SQLCommenter / query tags

Recommend:

- Add framework-native SQLCommenter-compatible instrumentation.
- Use low-cardinality tags.
- Include application, service, route/job, feature, source, and release SHA.
- Avoid PII and unbounded IDs.

### Schema recommendation workflow

Recommend:

- Triage open recommendations.
- Correlate with code and Insights.
- Convert into migrations or branch changes.
- Test before production.
- Apply only through approved workflow.

## Vitess-specific recommendations

These features do not exist on PlanetScale Postgres. For a Postgres database, mark every item in this section "Applies: no — Vitess-only feature" and never carry them into the recommendations.

### Safe migrations

Recommend for production branches and staging branches that accept deploy requests.

### Deploy requests

Recommend for schema changes into protected branches.

### Admin approval

Recommend for production deploy requests in multi-admin organizations.

### Gated deployments

Recommend when cutover timing and human control matter.

### Schema revert runbook

Recommend documenting revert responsibilities and the application rollback relationship.

### Branch strategy

Recommend production, staging, and short-lived development branches with safe migrations on protected targets.

### Sharding/keyspace review

Recommend when query patterns or growth suggest shard-awareness problems. Do not reshard automatically.

## Postgres-specific recommendations

### User-defined roles

Recommend for application servers instead of default role.

### pg_strict

Recommend for application roles after evaluation, especially to block accidental full-table update/delete mistakes.

### Traffic Control

Recommend for resource isolation of agents, exports, reports, workers, integrations, BI, and known expensive fingerprints.

### Backups and PITR

Recommend verifying retention and restore drill coverage.

### PgBouncer and connection pooling

Recommend where connection churn or serverless/edge behavior creates pressure, subject to transaction-pooling limitations.

### Private connectivity and IP restrictions

Recommend for customers requiring private network posture or reduced public exposure. Treat changes as production-risking.

### Extensions

Recommend only when use case is clear and restart/activation impact is accepted.

## Output

For each matrix item, mark:

- Applies: yes/no/unknown.
- Current state.
- Gap.
- Value: the specific measured finding this feature addresses (query
  fingerprint, anomaly count, incident, metric). State the mechanism and
  the measurement. Gaps are capability gaps with quantified impact, not
  risks safely avoided.
- Recommendation ID.
- Approval requirement.

End with:

“No changes have been applied.”
