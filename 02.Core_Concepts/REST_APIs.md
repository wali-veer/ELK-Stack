# Elasticsearch REST APIs

> The API is the common interface behind Kibana, automation, and integrations.

## Table of contents

- [What a REST API gives you](#what-a-rest-api-gives-you)
- [Try it in Kibana Console](#try-it-in-kibana-console)
- [Enterprise habits](#enterprise-habits)
- [References](#references)

## What a REST API gives you

Elasticsearch exposes HTTP APIs for indexing and searching documents, managing indices, and observing cluster state. Kibana Console provides an interactive editor for these APIs; applications can call them using HTTP clients or supported client libraries.

An API request has a method, path, optional query parameters, and sometimes a JSON body. Read the API documentation for the exact method, privileges, and compatibility requirements—similar-looking endpoints do not necessarily have identical behavior.

## Try it in Kibana Console

Read cluster information:

```http
GET /
```

Check cluster health:

```http
GET /_cluster/health
```

Search a specific index:

```http
GET logs-example/_search
{
  "size": 5,
  "query": {
    "match_all": {}
  }
}
```

Replace `logs-example` with an index or data stream you own. Start with a small result size and a narrow time range where appropriate.

## Enterprise habits

- Use TLS and least-privilege users or API keys; grant only the actions and data access the integration needs.
- Keep credentials in a secret manager, not in source control, screenshots, shell history, or shared Console requests.
- Treat `PUT`, `POST`, and `DELETE` as potentially state-changing. Confirm the target and expected impact before sending.
- For scripts, handle errors, timeouts, retries, and partial results explicitly. A successful HTTP response alone does not guarantee every item in a bulk request succeeded.
- Prefer documented APIs over CAT endpoints for machine integrations; CAT output is optimized for human inspection.
- Record the API version, request, required privileges, and expected response so another operator can reproduce the work.

## References

- [Elasticsearch API reference](https://www.elastic.co/docs/api/doc/elasticsearch)
- [Run API requests with Console](https://www.elastic.co/docs/explore-analyze/query-filter/tools/console)
- [Cluster health API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-health)
- [Search API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-search)
