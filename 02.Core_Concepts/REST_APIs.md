# Elasticsearch REST APIs

> The API is the common interface behind Kibana, automation, and integrations.

## Table of contents

- [What the APIs are for](#what-the-apis-are-for)
- [Anatomy of a request](#anatomy-of-a-request)
- [Try it in Kibana Console](#try-it-in-kibana-console)
- [From exploration to automation](#from-exploration-to-automation)
- [Enterprise habits](#enterprise-habits)
- [References](#references)

## What the APIs are for

Elasticsearch exposes HTTP APIs to index and retrieve documents, search and aggregate data, manage mappings and index lifecycle, and inspect cluster state. Kibana uses APIs behind its interface; operators can also call them directly from Console, applications, or scripts.

An API is more than a URL: its documentation defines supported methods, request fields, privileges, response shape, and version behavior. Prefer the official API reference over assuming an endpoint behaves like a similarly named API in another product.

## Anatomy of a request

```text
HTTP method  /request/path?optional_parameter=value
{
  "optional": "JSON request body"
}
```

- **Method** indicates the operation, but does not alone guarantee whether it is safe or idempotent.
- **Path** selects the API and often an index, alias, or data stream.
- **Query parameters** adjust behavior, such as response formatting or result size.
- **Body** carries structured search, mapping, or settings content where the API requires it.
- **Response** includes a status and usually JSON; inspect error details and any per-item failures.

## Try it in Kibana Console

Confirm which cluster Console is connected to:

```http
GET /
```

Inspect cluster health:

```http
GET /_cluster/health
```

Search an index you own:

```http
GET logs-example/_search
{
  "size": 5,
  "query": {
    "match_all": {}
  }
}
```

Replace `logs-example` with a real index or data stream. A small `size` limits returned hits, but does not necessarily limit the work required to execute every query; use selective filters and time bounds for large datasets.

## From exploration to automation

Console is excellent for interactively learning an API. Before moving a request into an application or scheduled job:

1. Verify the API and version compatibility.
2. Define the least privileges the client needs.
3. Add authentication, TLS verification, timeouts, and structured error handling.
4. Decide how to handle retryable failures without duplicating non-idempotent operations.
5. Test against representative data and document expected response and failure cases.

Bulk APIs can return an HTTP-level response while individual items in the response contain errors. Automation must inspect item-level results rather than treating the request status as proof that every document succeeded.

## Enterprise habits

- Use TLS and least-privilege users or API keys; grant only required actions and data access.
- Keep credentials in a secret manager, never in source control, screenshots, shell history, or shared Console requests.
- Treat `PUT`, `POST`, and `DELETE` as potentially state-changing. Confirm target and impact before sending.
- Prefer documented APIs over CAT endpoints for integrations; CAT output is optimized for human inspection.
- Use aliases or data streams where they support a stable application boundary, rather than coupling every client to physical backing-index names.
- Record API version, request, privileges, expected response, and rollback or recovery considerations.

## References

- [Elasticsearch API reference](https://www.elastic.co/docs/api/doc/elasticsearch)
- [Run API requests with Console](https://www.elastic.co/docs/explore-analyze/query-filter/tools/console)
- [Cluster health API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-health)
- [Search API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-search)
