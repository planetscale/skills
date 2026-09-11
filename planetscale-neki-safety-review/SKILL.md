---
name: planetscale-neki-safety-review
description: Review a PlanetScale Neki database for sharded Postgres topology, managed DDL, preview limitations, backups/PITR, router behavior, Query Insights, and safe agent operation.
---

# Neki safety review

## Purpose

Recommend best practices for a PlanetScale Neki database. Focus on sharded
Postgres topology, data placement, managed schema changes, preview
limitations, recovery, observability, and agent-safe automation. Do not apply
changes.

## Product posture

Neki is PlanetScale's sharded Postgres engine and is currently in Platform
Preview. Platform Preview features are Beta Features under PlanetScale's terms
or the customer's applicable agreement and are not covered by a service level
agreement.

To create a Neki database, an organization administrator must opt in to the
Platform Preview from the organization dashboard or Settings -> Platform
Preview. After opt-in, Neki appears as a database engine during database
creation.

Record Platform Preview status in the report when the reviewed database is
Neki. Treat it as a fit and limitation consideration, not as a finding by
itself.

Neki uses standard Postgres clients over the Postgres wire protocol. Clients
connect to Neki routers, and routers plan statements against one or more real
Postgres shards. A Neki database can start as a single unsharded cluster and
later add shards while keeping one connection string.

## Topology and data placement

Check:

- Shard count and whether the database is still single-shard.
- Data topology: unsharded tables, sharded tables, shard keys, reference
  tables, global secondary indexes, and authoritative shard groups.
- Whether latency-sensitive queries and transactions include shard-key
  predicates.
- Whether joins and multi-statement transactions stay within one shard where
  possible.
- Whether scatter queries appear in `EXPLAIN`, Query Insights, or query-plan
  evidence.
- Whether heavy tenants or workload classes should have separate shard groups,
  resources, replicas, parameters, extensions, or router groups.

Recommend:

- Design shard keys from the application's query and transaction shape before
  adding shards.
- Use reference tables for small shared datasets that need local joins.
- Use global secondary indexes only with a disabled, backfill, verify, and
  enable process. An enabled but incomplete GSI can return incomplete results.
- Keep transactions on one shard where possible. Cross-shard transactions do
  not provide atomic commit across all shards.
- Add routing predicates or revise data topology before accepting unexpected
  scatter queries as normal.

Do not change data topology, enable a GSI, add shards, move tables, or reshard
without approval.

## Branch and schema workflow

Neki does not use Vitess deploy requests. Schema changes use Postgres DDL
through routers, with two broad paths:

- **Native DDL**: a supported statement such as `ALTER TABLE` is sent through a
  router and executed immediately across managed shards. It has no workflow
  record, progress tracking, readiness gate, or explicit completion step.
- **Managed DDL**: Neki records a workflow, coordinates every managed shard,
  reports progress, waits for readiness, and requires explicit completion and
  cleanup. Online DDL uses a shadow table and streaming catch-up for disruptive
  live-table changes. Managed direct DDL uses the same workflow coordination
  for changes that do not use the shadow-table path.

Check:

- Whether production DDL is performed natively or through managed DDL.
- Whether large or busy table changes use managed Online DDL.
- Whether each workflow affects one table and one execution category; mixed or
  multi-table changes should be split.
- Whether operators wait for router schema visibility after native DDL. The
  router that accepts native DDL emits the exact
  `__neki.wait_for_ddl(schema_version, cluster_version)` call to run before
  dependent SQL goes through other routers.
- Whether failed, canceled, or partially completed managed workflows are
  recovered forward to a consistent table definition rather than hidden with
  blind artifact cleanup.

Recommend:

- Use managed Online DDL when direct execution could block traffic, rewrite a
  large live table, or build an index aggressively on the serving table.
- Use managed direct DDL when the statement cannot use the shadow-table process,
  when no copy is needed, or when workflow coordination is still valuable.
- Complete a managed workflow only after every shard reports readiness.
- After completion, run cleanup as a separate step; after failure, clean up the
  failed attempt and reissue the workflow before completing remaining shards.
- Treat production native DDL and managed DDL completion as production
  availability-impacting operations requiring explicit approval.

Do not create, complete, cancel, clean up, or retry managed DDL workflows
without approval.

## Platform Preview limitations

Review the current Platform Preview limitations before recommending a Neki
migration, topology change, or query rewrite. During Platform Preview, notable
limits include:

- No single-node, non-high-availability configuration.
- Router rejection of some SQL shapes, including `SELECT ... INTO`, selected
  `INTERSECT` and `EXCEPT` plans, `COMMIT AND CHAIN`, `ROLLBACK AND CHAIN`,
  cross-database object references, inheritance-parent reads without `ONLY`,
  and `search_path` including `pg_temp`.
- `COPY` protocol and shape limits: `COPY` must use the simple query protocol
  and be the only statement; `COPY TO` and file-based `COPY` support unsharded
  tables only; sharded `COPY FROM` requires the single-column shard key in the
  copied columns.
- User-defined text-search configuration and dictionary DDL is rejected by the
  router.
- Cluster-level objects depending on storage or code local to one Postgres host
  are rejected, including tablespaces, large objects, custom procedural
  language handlers, and `LOAD`.
- Externally managed Postgres sources cannot be attached directly to Neki
  migration workflows during Platform Preview; use the supported import path
  into a Neki-managed unsharded database.
- Cross-shard transactions do not provide atomic commit across all shards.

Report only limitations that intersect the customer's application, schema,
queries, import plan, or operating model.

## Connections, routers, and workload isolation

Check:

- Whether applications connect to routers using TLS with certificate and host
  verification.
- Which router group each workload uses.
- Whether read-only workloads route to replicas with `__neki.target =
  'replica'`, connection options, or `pscale shell --replica`.
- Whether router errors, connection counts, CPU, memory, storage, WAL growth,
  archive failures, and replication lag are monitored.
- Whether private connectivity is configured where required.

Recommend:

- Use one role per application or permission boundary.
- Use router groups for workloads needing independent router sizing or
  autoscaling.
- Route read-only agent, BI, analytics, and reporting queries to replicas when
  stale-read tolerance permits.
- Wait for asynchronous profile, router, and Admin changes to finish before
  submitting dependent changes.

Do not change router groups, profiles, replicas, network access, or roles
without approval.

## Backups and recovery

Neki backups cover schema and data on every managed shard in the branch. A
backup succeeds only after every shard backup succeeds. Restore creates a new
Neki branch; it does not replace the source branch.

Check:

- Required backup schedules, custom schedules, manual backups, and retention.
- Last successful backup and whether each managed shard succeeded.
- PITR window and whether the requested restore time is covered by a backup
  that includes the relevant shard set.
- Restore drill history and application cutover plan.
- Whether manual backups that must be retained have deletion protection.

Recommend:

- Verify required 12-hour backups and retention against RPO/RTO.
- Use restore drills into disposable branches to validate recovery.
- Account for shard-set changes: a PITR point after adding a shard may be
  unavailable until a later successful backup includes every shard in the new
  set.
- Treat emergency manual backups as operational actions that may affect
  performance.

Do not create backups, restore branches, or change schedules or retention
without approval.

## Query Insights and observability

Review:

- Slow, expensive, high-frequency, and erroring query patterns.
- Query tags and deploy correlation.
- Scatter query evidence and shard-call counts.
- Active anomalies.
- Schema recommendations.
- Metrics for routers, shards, storage, WAL, archive failures, replication lag,
  and connection pressure.

Recommend:

- Use SQLCommenter-style tags for application, service, route/job, feature,
  source, environment, and release SHA.
- Use Query Insights and query plans to find unexpected scatter queries before
  changing topology.
- Schedule agent review of Query Insights, query errors, and schema
  recommendations; agents produce reports, issues, or PRs by default.

## Webhook recommendations for Neki

Evaluate and recommend webhooks for:

- `branch.anomaly`
- `branch.out_of_memory`
- `branch.primary_promoted`
- `branch.ready`
- `branch.sleeping`
- `branch.start_maintenance`
- `cluster.storage`
- `database.access_request`
- `branch.schema_recommendation` if available
- `webhook.test` for setup validation

Recommended automation behavior:

- Alerts: anomaly, out-of-memory, primary promotion, storage, and maintenance.
- Agent intake: anomaly, query errors, schema recommendations, and capacity
  signals.
- Human approval: any generated topology, schema workflow, role, backup,
  router, profile, replica, or network change.

## Output

Return:

- Current Neki safety posture.
- Platform Preview limitations that affect the workload.
- Topology and data-placement gaps.
- Recommended managed DDL workflow.
- Recommended backup/PITR plan.
- Recommended connection, router, and role plan.
- Recommended query tagging and observability plan.
- Proposed changes requiring approval.

End with:

"No Neki changes have been applied."
