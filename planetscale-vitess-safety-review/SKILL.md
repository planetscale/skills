---
name: planetscale-vitess-safety-review
description: Review a PlanetScale Vitess database for safe migrations, deploy requests, schema recommendations, Insights, webhooks, and operational safety.
---

# Vitess safety review

## Purpose

Recommend best practices for a PlanetScale Vitess database. Focus on production safety, deployability, observability, and agent-safe automation. Do not apply any changes.

## Primary safety features to evaluate

### Safe migrations

Check whether safe migrations are enabled on production and staging branches.

Recommend enabling safe migrations when:

- The branch receives production traffic.
- The team runs DDL outside deploy requests.
- There is no protected branch workflow.
- The application is expected to evolve schema frequently.

Explain the tradeoff: safe migrations reject direct DDL on protected branches and force schema changes through deploy requests. That is a feature, but it can break teams relying on direct production DDL. Treat enablement as a behavior-changing change requiring approval.

### Deploy requests

Check:

- Whether deploy requests are used for schema changes.
- Whether administrator approval is required.
- Whether deploy requests are reviewed for data loss, conflicts, lint errors, foreign key problems, charset issues, and shard impact.
- Whether teams use gated deployments for cutover control.
- Whether “deploy instantly” is used and whether the team understands it removes the gated-deployment/revert shape.
- Whether cutover is regularly delayed by long-running transactions.
- Whether deploy request events are subscribed to via webhooks.

Recommend:

- Require deploy requests for production schema changes.
- Enable administrator approval for production deploy requests when there is more than one administrator.
- In a single-admin organization, administrator approval does not stop an agent that acts as that admin:
  the admin who opens a deploy request can also approve it. If the goal is to keep agents from
  self-approving, give the agent a separate user, or a service token without deploy request
  approval permission.
- Prefer normal safe deployments over instant deployments unless the migration is known to be instant-safe and the rollback story is acceptable.
- Use gated deployment when cutover timing matters.
- Treat “force cutover now” as an operator-controlled action for delayed
  cutovers: it aggressively stops running transactions to complete schema
  cutover. Recommend reviewing the blocking workload and incident context
  before use, and only recommend the database-level aggressive cutover default
  when frequent cutover blocking is understood and accepted.

### Schema revert

Check whether the team knows the revert window and whether their incident runbook includes it.

Recommend documenting:

- How to identify a bad schema migration.
- How to revert within the supported window.
- Who is authorized to revert.
- Which application deploy should be rolled back together with the schema revert.

### Branch strategy

Recommend a branch topology:

- `main` or equivalent production branch with safe migrations enabled.
- `staging` branch based from production with safe migrations enabled.
- Short-lived development branches based from staging.
- Deploy requests from development to staging, then staging to production when appropriate.

Do not create branches without approval.

### Query Insights

Review Insights for:

- Slow queries.
- Queries reading too many rows.
- Erroring queries.
- Queries with poor index usage.
- For sharded databases, whether query patterns use relevant vindexes and how
  vindex usage changes after index or routing changes.
- Unusual query volume.
- Missing SQL comment tags.
- Tag breakdowns when built-in metadata or SQLCommenter tags are present:
  `tag:key:value` filtering in the dashboard, and the `insights/tags` and
  `insights/tags/summaries` API endpoints for programmatic breakdowns.
- Deploy correlation data.

Recommend enabling or improving application query tagging so Insights can attribute queries to app, route, controller, action, job, deployment SHA, and feature.

### Prometheus reparent and VTOrc metrics

When Prometheus metrics are available, review primary-change and VTOrc recovery
signals:

- `planetscale_vitess_planned_reparents_total` counts planned primary
  changes, such as maintenance or operator-requested reparents.
- `planetscale_vitess_emergency_reparents_total` counts emergency reparents
  where a replica was promoted to recover from a failed primary.
- Both counters include successes and failures via
  `planetscale_reparent_result`; `planetscale_component` identifies whether
  `vtorc` or `vtctld` ran the reparent.
- For VTOrc recovery counters, `planetscale_vtorc_recovery_type` contains
  recovery names such as `RecoverDeadPrimary`, `ElectNewPrimary`, or
  `FixReplica`; do not filter for older documented placeholder values like
  `planned` or `unplanned`.
- On Vitess 23 and later, `planetscale_vtorc_failed_recoveries_total` and
  `planetscale_vtorc_successful_recoveries_total` include
  `planetscale_keyspace` and `planetscale_shard` labels.
- Use `planetscale_vtorc_detected_problems` with the recovery counters to see
  which problems VTOrc detected and what recovery action followed.

Recommend alerts and runbook checks that distinguish planned maintenance from
emergency primary promotion, correlate failures to keyspace/shard when labels
are present, and avoid stale Prometheus queries that filter
`planetscale_vtorc_recovery_type` as `planned` or `unplanned`.

### Anomalies

Review active and recent anomalies.

Recommend:

- Subscribe `branch.anomaly` webhooks to the team’s alerting or automation system.
- Route anomaly payloads to a triage workflow that opens an issue or agent task.
- Correlate anomalies with deploy requests, application deploys, and query tags.
- Do not automatically apply schema or code changes from anomaly events; generate a proposal or PR only.

### Schema recommendations

Review open schema recommendations.

For each recommendation, capture:

- Type: add index, remove redundant index, primary key exhaustion, unused table, legacy charset/collation, or other.
- Affected table and keyspace.
- Supporting query telemetry.
- DDL.
- Expected benefit.
- Risk.
- Test plan.

Recommend implementation path:

- Convert the recommendation into an application migration or PlanetScale branch schema change.
- Open a deploy request.
- Review generated DDL and shard impact.
- Benchmark or validate on a branch.
- Deploy with safe migrations.
- Monitor Insights and anomaly state after deployment.

Do not apply recommendations directly.

### Backups and restore

Check backup posture and restore runbooks.

Recommend:

- Verify automated backups exist.
- Run a non-production restore drill periodically.
- Document restore target, RPO/RTO expectation, and application cutover plan.
- For sharded databases, document shard-aware restore expectations.

### Sharding and keyspace safety

If the database is sharded, review:

- Keyspaces and shards.
- VTTablet and MySQL keyspace parameter overrides, default values, and any
  in-progress rollout changes from `pscale keyspace parameters list` and
  `pscale keyspace parameters changes list`.
- Vschema.
- Cross-shard query patterns.
- Whether schema deploy requests show per-shard impact.
- Whether queries use shard-friendly access paths.

Recommend an agent-safe sharding review only as a proposal. Never reshard,
change vschema, alter routing, or change/reset keyspace parameters
automatically. Parameter tuning for imports or replication workload changes
must include the current value, default value, target value or reset, rollout
status, expected effect, and rollback plan.

## Webhook recommendations for Vitess

Evaluate and recommend webhooks for:

- `branch.anomaly`
- `branch.primary_promoted`
- `branch.ready`
- `branch.sleeping`
- `cluster.storage`
- `keyspace.storage`
- `deploy_request.opened`
- `deploy_request.queued`
- `deploy_request.in_progress`
- `deploy_request.pending_cutover`
- `deploy_request.schema_applied`
- `deploy_request.errored`
- `deploy_request.reverted`
- `deploy_request.closed`
- `branch.schema_recommendation` if available in the webhook API for the customer’s database

Recommended destinations:

- Alerting for anomaly, primary promotion, storage, and deploy errors.
- Slack or internal notifications for deploy request lifecycle.
- Agent intake queue for schema recommendations and anomalies, with PR-only output by default.

## Output

Return:

- Current Vitess safety posture.
- Missing safety features.
- Recommended workflow.
- Recommended webhook subscriptions.
- Schema recommendation triage table.
- Deploy safety gaps.
- Proposed changes requiring approval.

End with:

“No Vitess changes have been applied.”
