# Elasticsearch Architecture: Clusters, Nodes, and Shards

> A mental model for understanding where data lives, how it is copied, and how requests are served.

## Table of contents

- [The building blocks](#the-building-blocks)
- [How a request flows](#how-a-request-flows)
- [Availability is not the same as backup](#availability-is-not-the-same-as-backup)
- [Deployment perspective](#deployment-perspective)
- [References](#references)

## The building blocks

- **Cluster:** A group of Elasticsearch nodes that work together and expose a shared data and API surface.
- **Node:** An Elasticsearch instance that contributes roles and resources to a cluster.
- **Index:** A logical collection of documents.
- **Primary shard:** A partition of an index that owns the primary copy of its documents.
- **Replica shard:** A copy of a primary shard, placed to provide redundancy and serve search requests.

An index is divided into shards so work can be distributed. A replica is not a substitute for a backup: it helps tolerate node-level failures, but it does not protect against accidental deletion, logical corruption, or every cluster-wide failure.

## How a request flows

```mermaid
flowchart LR
    C[Client request] --> N[Coordinating node]
    N --> P[Primary shard]
    P --> R[Replica shard copies]
    R --> P
    P --> N
    N --> C
```

The coordinating node routes work to the shard copies. Writes are handled by a primary shard and replicated to in-sync copies before acknowledgment; searches can be served from shard copies and their results are coordinated.

## Availability is not the same as backup

Replicas can improve resilience and search capacity, but their value depends on placement and topology. A single-node cluster cannot place a replica on another node. In production, plan for failure domains, shard sizing, recovery time, capacity headroom, and tested snapshots/restores—not just a green health status.

Shard counts are design choices with long-lived consequences. Primary shard count is set when an index is created; replica counts can be adjusted later. Avoid creating many tiny shards, and validate sizing against data volume and query patterns.

## Deployment perspective

In self-managed and Elastic Cloud Hosted deployments, operators need a working understanding of nodes, roles, shards, and replicas. Elastic Cloud Serverless abstracts much of this infrastructure and does not expose the same node/shard operations. Learn the concepts, but apply only controls supported by your deployment model.

## References

- [Clusters, nodes, and shards](https://www.elastic.co/docs/deploy-manage/distributed-architecture/clusters-nodes-shards)
- [Reading and writing documents](https://www.elastic.co/docs/deploy-manage/distributed-architecture/reading-and-writing-documents)
- [Shard sizing guidance](https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/size-shards)
- [Snapshot and restore](https://www.elastic.co/docs/deploy-manage/tools/snapshot-and-restore)
