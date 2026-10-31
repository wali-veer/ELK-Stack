# Event Query Language (EQL)

> EQL makes event relationships and sequences explicit—especially useful when the order of activity matters.

## Table of contents

- [When EQL helps](#when-eql-helps)
- [Event queries](#event-queries)
- [Sequences and join keys](#sequences-and-join-keys)
- [Run EQL through the API](#run-eql-through-the-api)
- [Data requirements and query quality](#data-requirements-and-query-quality)
- [EQL in an enterprise detection workflow](#eql-in-an-enterprise-detection-workflow)

## When EQL helps

Many search languages are good at asking whether one event matches conditions. EQL adds syntax for asking whether a **series of events** occurred in a particular order, optionally within a time window and for the same entity.

That makes EQL useful for threat hunting, detection engineering, and other event-centric investigations. It is not the best tool for every query: use Query DSL for general search and aggregations, or KQL for filtering in Kibana, when those express the question more directly.

## Event queries

An event query selects events from a category:

```eql
process where process.name == "my-test-process"
```

The category (`process`) and field (`process.name`) need to match the actual event schema and mappings. ECS conventions are supported by EQL defaults, but ECS is a schema recommendation—not an automatic guarantee that every incoming event has those fields.

## Sequences and join keys

A sequence states ordered event clauses. This example looks for a process event followed by an outgoing network event associated with the same process entity:

```eql
sequence by process.entity_id with maxspan=5m
  [process where process.name == "my-test-process"]
  [network where network.direction == "outgoing"]
```

Read this as: within five minutes, find the first event, then a later event matching the second clause, with the same `process.entity_id`.

The `by` key is central. Without a reliable key, events from unrelated processes could be correlated—or valid activity could be missed. Confirm that the key is populated consistently in every relevant event category. Sequence order follows event timestamps, so timestamp accuracy and clock synchronization matter.

## Run EQL through the API

In Console, the EQL search endpoint accepts the query in a JSON body:

```http
POST logs-endpoint.events.process-*/_eql/search
{
  "query": "process where process.name == \"my-test-process\"",
  "size": 10
}
```

Replace the example index pattern and event values with those in your environment. Use a bounded index pattern and, where appropriate, a time range:

```http
POST logs-endpoint.events.process-*/_eql/search
{
  "query": "process where process.name == \"my-test-process\"",
  "filter": {
    "range": {
      "@timestamp": {
        "gte": "now-1h",
        "lte": "now"
      }
    }
  },
  "size": 10
}
```

Check the current API reference for exact request options and response fields supported by your Elasticsearch version.

## Data requirements and query quality

- EQL searches generally need a timestamp field (default `@timestamp`) and an event-category field (default `event.category`). The `any` category can be used when intentionally matching without a category.
- Confirm field types and mappings; values that look similar can have different mappings across data streams.
- Bound both the data scope and sequence duration. Wide searches and long `maxspan` windows can increase resource cost and produce ambiguous correlations.
- Test against known positive examples and realistic benign activity. A query returning zero results may indicate no matching behavior, poor data coverage, or a schema mismatch.
- Check timestamp quality, entity identifiers, null fields, and event ordering before tuning the condition itself.
- Choose `size` intentionally; it controls returned results, not necessarily the total query work.

## EQL in an enterprise detection workflow

Treat a detection as an operational artifact, not just a query string. Document the behavior it represents, required fields and data sources, expected false positives, time bounds, owner, and response action. Version it in source control, validate it on representative data, and monitor both detection quality and query cost over time.
