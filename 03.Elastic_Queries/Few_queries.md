# Elasticsearch Administration: Diagnostic and Change APIs

This guide expands the administration requests collected in `Few_queries`. It removes duplicate spellings, groups related operations, and distinguishes safe inspection from changes that can affect data or availability.

> **Scope:** Examples are for an Elasticsearch cluster you administer. Replace sample names with lab resources you own. Review the API reference and deployment-specific support before making changes.

## Table of contents

- [How to use this guide](#how-to-use-this-guide)
- [1. Read-only diagnostics](#1-read-only-diagnostics)
  - [Health report](#health-report)
  - [Cluster health, including index detail](#cluster-health-including-index-detail)
  - [CAT health and elected master](#cat-health-and-elected-master)
  - [Cluster statistics](#cluster-statistics)
  - [Cluster settings](#cluster-settings)
  - [Cluster state](#cluster-state)
  - [Node inventory and statistics](#node-inventory-and-statistics)
  - [Hot threads](#hot-threads)
  - [Task inventory](#task-inventory)
  - [Security inventory](#security-inventory)
- [2. Index lifecycle in a lab](#2-index-lifecycle-in-a-lab)
  - [Create and inspect an index](#create-and-inspect-an-index)
  - [Close and reopen an index](#close-and-reopen-an-index)
  - [Force merge: use sparingly](#force-merge-use-sparingly)
  - [Delete a test index](#delete-a-test-index)
- [3. Shard allocation and rerouting](#3-shard-allocation-and-rerouting)
- [4. Slow logs](#4-slow-logs)
- [5. Snapshots and restores](#5-snapshots-and-restores)
- [6. Task cancellation](#6-task-cancellation)
- [Queries consolidated or omitted](#queries-consolidated-or-omitted)
- [References](#references)
- [Before publishing](#before-publishing)

## How to use this guide

Run API examples in **Kibana → Dev Tools → Console**. Start with read-only checks, identify the exact index or task, and understand the expected result before making a change. Save the current state and define a rollback or recovery plan for operational changes.

For document-level Create, Read, Update, and Delete examples, see [Elasticsearch CRUD Operations](./CRUD.md). This guide focuses on additional diagnostics and safe lab examples of index, allocation, task, and snapshot administration.

## 1. Read-only diagnostics

### Health report

```http
GET /_health_report
```

Returns subsystem health indicators and diagnostic details. Use it when the broad cluster status is not enough to suggest which subsystem needs attention. It complements `GET /_cluster/health`; it does not replace checking application behavior, logs, or metrics.

The notes contained three spellings of this request (`GET /_health_report`, `GET _health_report`, and lowercase `get /_health_report`). They are the same endpoint; use the conventional uppercase method and leading slash shown above.

### Cluster health, including index detail

```http
GET /_cluster/health
```

This is the concise cluster-wide view. The optional `level` parameter expands the response:

```http
GET /_cluster/health?level=indices
```

Use the index level when you know the overall state needs to be traced to a particular index. Higher detail can produce a larger response on clusters with many indices. Green means primary and replica shards are allocated; yellow means primaries are allocated but one or more replicas are not; red means at least one primary is unallocated.

For a readable terminal-style summary, use `GET /_cat/health?v`, documented in [CAT health and elected master](#cat-health-and-elected-master). These APIs provide different representations of related health data.

### CAT health and elected master

```http
GET /_cat/health?v
GET /_cat/master?v
```

CAT health is a compact, human-readable status line. CAT master identifies the currently elected master node. These are useful quick checks during a cluster incident, but should not be used as machine-integration contracts; use JSON APIs and documented response fields for automation.

### Cluster statistics

```http
GET /_cluster/stats
```

Provides a cluster-level summary of nodes, indices, shards, and related statistics. It is useful for periodic capacity reviews or comparing broad cluster shape before and after a workload change. It is not a time series: take repeated samples or use monitoring to understand trends. If a node does not answer before the request timeout, inspect the `_nodes.failed` portion of the response rather than assuming the statistics are complete.

### Cluster settings

```http
GET /_cluster/settings?include_defaults=true&flat_settings=true
```

Returns explicit cluster settings along with defaults, in a flattened key format that is easier to scan or diff. This is the canonical version of the two overlapping settings queries in the source notes (`GET /_cluster/settings` and `GET /_cluster/settings?include_defaults=true`).

Use it to check whether a non-default setting is active before changing it. Defaults can depend on configuration and version; for node-local configuration, inspect node settings as well. Treat this output as configuration metadata and review it before sharing.

The source notes also include a persistent update to re-enable shard allocation:

```http
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}
```

This is **not** a generic fix for unassigned shards. Use it only when you have confirmed allocation was intentionally disabled and understand the resulting shard activity. Re-enabling allocation can trigger relocations and consume resources. If you need to remove this persistent override and return to the default behavior, reset it explicitly:

```http
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
```

The source's bare `PUT /_cluster/settings` without a JSON body is incomplete; the API requires a settings payload.

### Cluster state

```http
GET /_cluster/state/metadata,routing_table
```

Cluster state can be very large. Request only the sections needed: `metadata` helps inspect index metadata and settings, while `routing_table` shows shard routing. Avoid an unrestricted state dump during an incident unless you specifically need it; the response may be large and contain internal topology details.

### Node inventory and statistics

For a compact node inventory, use the existing CAT query:

```http
GET /_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load,node.role
```

![CAT nodes resource summary](./images/04.png)

For detailed metrics, scope the Nodes Stats API to relevant metric groups:

```http
GET /_nodes/stats/jvm,fs,process,thread_pool
```

The generic `GET /_nodes` and `GET /_nodes/stats` requests return much broader output. Scope by node ID/name and metric group where possible, especially when collecting results repeatedly. Use stats to investigate a hypothesis—such as heap pressure or disk capacity—not as a substitute for time-series monitoring.

### Hot threads

```http
GET /_nodes/hot_threads
```

Captures thread stacks that are consuming CPU or blocked, useful during a short-lived performance incident. It is a point-in-time diagnostic and can return a large text response. Run it only when needed, inspect the right node(s), and correlate the stack with workload and metrics; do not treat one capture as proof of a root cause.

### Task inventory

```http
GET /_tasks?detailed=true
```

Lists running tasks with additional details such as action, start time, and whether a task is cancellable. This can help find a long-running search, reindex, or other operation. The response may be large on a busy cluster.

To narrow the list by action, the notes include:

```http
GET /_tasks?detailed=true&actions=geoip*
```

Use a filter matching the operation you are investigating. A task may complete between inspection and follow-up; confirm its current status before attempting cancellation.

### Security inventory

```http
GET /_security/user
GET /_security/role
GET /_security/role_mapping
```

These APIs inspect users, roles, and role mappings, respectively. They require appropriate security privileges and may reveal sensitive account or authorization design. Limit access, avoid exporting real identities into public documentation, and prefer querying a specific user or role when possible.

## 2. Index lifecycle in a lab

The following examples use `test-index`. Run them only in a lab deployment where that index name is reserved for this exercise.

### Create and inspect an index

Create a simple test index:

```http
PUT /test-index
```

Inspect its effective settings and mappings:

```http
GET /test-index/_settings
GET /test-index/_mapping
```

Settings describe index behavior such as shard/replica configuration; mappings define how document fields are indexed and interpreted. For a useful experiment, create an index with an explicit mapping and settings rather than relying on implicit dynamic mapping.

### Close and reopen an index

```http
POST /test-index/_close
GET /_cat/indices/test-index?v
POST /test-index/_open
GET /_cat/indices/test-index?v
```

Closing an index removes it from normal search and indexing operations while retaining its data on disk. Reopening makes it available again. These are state-changing operations: check dependent applications and confirm the target before closing. A closed index does not mean data has been deleted or backed up.

### Force merge: use sparingly

```http
POST /test-index/_forcemerge?max_num_segments=1
```

Force merge combines Lucene segments and can reduce segment overhead for an index that is no longer receiving writes. It can consume significant I/O, CPU, and temporary disk space. Do not run it routinely on actively written indices or as a generic “cleanup” command; rollover/retention and lifecycle policies are usually the better operational controls. In Elastic Cloud, verify supported operations and resource implications for the deployment.

### Delete a test index

```http
PUT /delete-index
GET /_cat/indices/delete-index?v
DELETE /delete-index
GET /_cat/indices/delete-index?v
```

This sequence creates an empty index, confirms it exists, deletes it, and confirms it is gone. **The final DELETE is destructive and permanently removes the index and its data.** Never use a broad wildcard or substitute a real index name unless deletion is explicitly approved and required.

## 3. Shard allocation and rerouting

Start by inspecting shard state and allocation:

```http
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason
GET /_cat/allocation?v
```

The allocation overview is shown below; the shard listing is included in the unassigned-shard section.

![Full shard allocation overview](./images/07.png)

For the reason a particular shard is unassigned, ask the Allocation Explain API:

```http
POST /_cluster/allocation/explain
{
  "index": "test-index",
  "shard": 0,
  "primary": true
}
```

Replace the example with an actual unassigned shard. The explanation reports allocation decisions and deciders; use it to understand constraints such as disk thresholds, filters, or lack of eligible nodes before attempting remediation.

The source notes also contain a manual reroute example. First preview it with a dry run and explanation:

```http
POST /_cluster/reroute?dry_run=true&explain=true
{
  "commands": [
    {
      "move": {
        "index": "test-index",
        "shard": 0,
        "from_node": "NODE-NAME-1",
        "to_node": "NODE-NAME-2"
      }
    }
  ]
}
```

Only after reviewing the preview and confirming node names and shard constraints should an authorized operator consider the same request without `dry_run=true`. A reroute can affect availability and performance; avoid manual moves when automatic allocation can resolve the condition. Never use `allocate_empty_primary` casually: it can discard data and requires explicit acceptance of data loss.

## 4. Slow logs

`GET /_search/slowlog` is **not** the way to retrieve Elasticsearch search slow logs. Slow logs are written to Elasticsearch node log files or the configured logging destination; use the deployment's log viewer or logging pipeline to inspect them.

Configure per-index slow-log thresholds only when you have a clear diagnostic need and a plan to return to the previous values:

```http
PUT /test-index/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.fetch.warn": "1s"
}
```

These thresholds are examples, not recommended production values. A query threshold measures query-phase duration and a fetch threshold measures fetch-phase duration. Set thresholds based on the workload and desired log volume, and verify where logs are routed. Excessively low thresholds can generate large volumes of logs.

## 5. Snapshots and restores

A **repository** is the configured storage location; a **snapshot** is a named recovery point in that repository. The source query `PUT /_snapshot/test-index/snapshot_1...` mixes these concepts: `test-index` in that path is a repository name, not necessarily an Elasticsearch index.

List configured repositories and snapshots:

```http
GET /_snapshot
GET /_snapshot/my_backup/_all
```

Create a snapshot only after an administrator has configured and verified the repository:

```http
PUT /_snapshot/my_backup/snapshot_1?wait_for_completion=true
```

Wait-for-completion is convenient for a small lab; larger snapshots are often monitored asynchronously. Check the snapshot result for failures and partial state—an accepted request alone is not proof of a complete recovery point.

Restore selected indices with an explicit safe scope:

```http
POST /_snapshot/my_backup/snapshot_1/_restore
{
  "indices": "test-index",
  "include_global_state": false
}
```

Restoring is a change operation. Check for index-name conflicts, target-cluster compatibility, and the effect on templates/settings before proceeding. Restoring global state can replace or merge important cluster metadata; keep it disabled unless the recovery plan explicitly requires it. **A snapshot is only useful if restore has been tested.**

Repository registration and credentials are deployment-specific. In Elastic Cloud Hosted, use the supported repository configuration and secure settings; never put cloud credentials in a public Markdown file.

## 6. Task cancellation

Cancel a task only after identifying the exact task and confirming that interruption is acceptable:

```http
POST /_tasks/NODE_ID:TASK_NUMBER/_cancel
```

`NODE_ID:TASK_NUMBER` is a placeholder for the task ID returned by `GET /_tasks`; `task_id:12345` from the source notes is not a literal ID to reuse. Cancellation is best-effort and may not be supported for every task. It does not roll back work already completed. Check the task response and the target data afterward.

## Queries consolidated or omitted

- Duplicate `GET /_cluster/health`, `GET /_cluster/settings`, `GET /_cat/indices`, `GET /_cat/shards`, and `GET /_cat/nodes` entries were consolidated. Related CAT examples are retained in this guide where they support an administrative workflow.
- `GET /_cluster/settings` variants are represented by one query with defaults and flat keys.
- Variants of `GET /_health_report` differ only in method capitalization or slash formatting; one canonical form is retained.
- Generic broad queries such as `GET /_nodes`, `GET /_nodes/stats`, and `GET /_cluster/state` are discussed with scoped examples to avoid huge responses.
- `GET /_search/slowlog` was removed as an invalid slow-log retrieval approach; the guide explains where slow logs are found and how to set thresholds.
- CAT outputs are useful for interactive inspection, but automation should use documented JSON APIs.

## References

- [Health report API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-health-report)
- [Cluster health API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-health)
- [Cluster statistics API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-stats)
- [Cluster settings API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-get-settings)
- [Cluster state API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-state)
- [Nodes Stats API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-nodes-stats)
- [Hot threads API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-nodes-hot-threads)
- [Tasks API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-tasks-get)
- [Cancel task API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-tasks-cancel)
- [Index settings API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-get-settings)
- [Force merge API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-forcemerge)
- [Cluster Allocation Explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-allocation-explain)
- [Cluster Reroute API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-reroute)
- [Snapshot and restore](https://www.elastic.co/docs/deploy-manage/tools/snapshot-and-restore)
- [Security user API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-get-user)
- [CAT health API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cat-health)
- [CAT master API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cat-master)
- [Search slow logs](https://www.elastic.co/docs/deploy-manage/monitor/logging-configuration/slow-logs)

## Before publishing

The reused node screenshot and other operational screenshots expose deployment-specific node/index names, internal addresses, and cluster identifiers. Redact identifying details and any sensitive configuration before publishing them publicly.
