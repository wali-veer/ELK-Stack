# Event Query Language (EQL)

> EQL is for finding relationships and sequences in time-stamped event data.

## Table of contents

- [When EQL helps](#when-eql-helps)
- [A simple event query](#a-simple-event-query)
- [A sequence query](#a-sequence-query)
- [Data requirements and tuning](#data-requirements-and-tuning)
- [References](#references)

## When EQL helps

Use EQL when the question is about a **pattern across events**, not just whether one document matches. EQL supports event queries and ordered sequences, making it useful for threat hunting and other event-centric investigations.

EQL is not a replacement for every Elasticsearch query language. For ordinary filtering and aggregation, Query DSL or Kibana Query Language may be a better fit. Choose the language that makes the intent clearest.

## A simple event query

This example finds process events for a given process name:

```eql
process where process.name == "my-test-process"
```

Run it through the EQL search API in Console:

```http
POST logs-endpoint.events.process-*/_eql/search
{
  "query": "process where process.name == \"my-test-process\"",
  "size": 10
}
```

Replace the index pattern and process name with values that exist in your lab. EQL returns events in time order; set a suitable time range and result size for the investigation.

## A sequence query

Sequences can express ordered events. This illustrative example looks for a process event followed by a network event associated with the same process entity:

```eql
sequence by process.entity_id with maxspan=5m
  [process where process.name == "my-test-process"]
  [network where network.direction == "outgoing"]
```

The sequence is meaningful only if the data has consistent timestamps and `process.entity_id` is populated on both event types. Adapt the fields to the schema and event relationships in your data; a syntactically valid query with unsuitable fields can return no useful matches.

## Data requirements and tuning

- EQL generally expects a timestamp field (by default ECS `@timestamp`) and an event category field (by default ECS `event.category`). `any` can be used when intentionally matching without a category.
- ECS is recommended because EQL's defaults align with ECS fields, but a mapping/schema plan is still essential.
- Use a bounded index pattern and time range; long sequence windows and broad data scans can be expensive.
- Validate the query on representative data, including expected missing/null fields and timestamp quality.
- Measure both detection quality and cost: useful detections should be explainable, reproducible, and tuned to reduce irrelevant matches.

## References

- [EQL overview](https://www.elastic.co/docs/explore-analyze/query-filter/languages/eql)
- [EQL syntax reference](https://www.elastic.co/docs/reference/query-languages/eql)
- [EQL search API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-eql-search)
- [Elastic Common Schema (ECS)](https://www.elastic.co/docs/reference/ecs)
