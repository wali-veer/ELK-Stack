# Elasticsearch CRUD Operations

> CRUD stands for Create, Read, Update, and Delete. This guide follows the `electronics_store` examples from the notes and explains how Elasticsearch document APIs behave.

## Table of contents

- [Before you start: choose a mapping](#before-you-start-choose-a-mapping)
- [Create an index](#create-an-index)
- [Create documents](#create-documents)
- [Read documents](#read-documents)
- [Update a document](#update-a-document)
- [Delete a document or index](#delete-a-document-or-index)
- [What the original examples demonstrate](#what-the-original-examples-demonstrate)
- [Operational habits](#operational-habits)
- [References](#references)

## Before you start: choose a mapping

The notes first create `electronics_store` without an explicit mapping, then index a numeric price (`2000`) and later send decimal prices as strings (`"999.99"`). Dynamic mapping may infer `price` as a whole-number type from the first document. A later decimal can then fail to parse; sending numbers as strings also makes the intended data type less clear.

For a fresh lab index, define the fields first:

```http
PUT /electronics_store
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "price": {
        "type": "double"
      }
    }
  }
}
```

Use JSON numbers for prices (`2000.00`), not quoted strings. This mapping supports text search on `name`, exact matching/sorting through `name.keyword`, and decimal numeric operations on `price`.

If `electronics_store` already exists, this create-index request will return an error rather than change its mapping. Inspect existing mappings first; changing an existing field's type generally requires a new index and reindexing. Do not delete data just to make a tutorial request succeed.

## Create an index

The source notes use:

```http
PUT /electronics_store
```

This creates an empty index with default settings and dynamically inferred mappings as documents arrive. It is suitable for a quick disposable experiment, but it does not demonstrate deliberate schema design. For repeatable use, define mappings and relevant settings at index creation as shown above.

Check that the index exists:

```http
GET /_cat/indices/electronics_store?v
```

CAT APIs are convenient for people inspecting results in Console. For automation, use documented JSON APIs rather than parsing CAT table output.

## Create documents

### Let Elasticsearch generate the document ID

```http
POST /electronics_store/_doc
{
  "name": "wireless headphones",
  "price": 2000.00
}
```

The `_doc` endpoint with `POST` generates an ID. The response includes that ID and a result such as `created`. Save the returned `_id` if you want to retrieve, update, or delete this specific document later.

Repeating this request creates another document with a new ID; it is not an idempotent retry. If your application needs safe retries, choose a stable ID or use an appropriate create/index operation.

### Supply a known ID and replace or create

```http
PUT /electronics_store/_doc/1
{
  "name": "Bluetooth Speaker",
  "price": 999.99
}
```

With the default `index` operation, a `PUT` to `/_doc/1` creates the document if ID `1` does not exist, or replaces its entire `_source` if it does. Fields omitted from the new body are removed from the stored document. This is different from a partial update.

The notes send another `PUT` to ID `1` with a different product:

```http
PUT /electronics_store/_doc/1
{
  "name": "Microphone",
  "price": 1000.99
}
```

After this request, ID `1` represents the microphone; it is not a second document. The previous speaker source is replaced. Use a new ID if both products should exist.

### Create only if the ID is unused

```http
PUT /electronics_store/_create/2
{
  "name": "Smart Watch",
  "price": 20000.00
}
```

The `_create` endpoint fails with a conflict if document ID `2` already exists. Use it when overwriting an existing document would be incorrect. This is the explicit create-only alternative to the default index behavior of `PUT /_doc/{id}`.

## Read documents

### Retrieve a document by ID

```http
GET /electronics_store/_doc/1
```

The response contains metadata such as `_index`, `_id`, version information, a `found` flag, and (when found) the document under `_source`. A direct GET by ID is the clearest way to verify a known document.

### Search documents

```http
GET /electronics_store/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "price": "asc"
    }
  ]
}
```

This returns matching documents ordered by price. In a real index, use a selective query and a suitable result size; `match_all` is useful for a tiny lab, not a default for large datasets.

Elasticsearch search is near-real-time: a newly indexed or updated document may not appear in search results until a refresh. A direct GET by ID is real-time by default, so it may show the write before a search does. Avoid forcing refreshes on every production write; they can reduce indexing throughput.

## Update a document

The notes use a partial document update:

```http
POST /electronics_store/_update/1
{
  "doc": {
    "price": 599.00
  }
}
```

The `doc` object is merged into the existing `_source`, so the document's `name` remains unchanged while `price` is updated. This differs from `PUT /electronics_store/_doc/1`, which replaces the complete source.

By default, `_update` expects the document to exist; it does not silently create a missing document. If an upsert is intended, explicitly configure one using the Update API options. Consider optimistic concurrency controls (`if_seq_no` and `if_primary_term`) when multiple writers could update the same document and lost updates matter.

## Delete a document or index

### Delete one document

```http
DELETE /electronics_store/_doc/1
```

This deletes document `1` but leaves the `electronics_store` index and its other documents intact. The response result indicates whether a document was deleted or not found.

Verify the result with `GET /electronics_store/_doc/1` (which should report `found: false`) or search the index. The index itself remains available.

### Delete an entire index — destructive

The source notes also contain:

```http
DELETE /test-index
```

This deletes the **entire `test-index` index**, including all its documents, mappings, and settings. It is not part of the `electronics_store` CRUD sequence and must not be run as a document-delete example. Only delete an index that you own, after confirming the exact target and ensuring any required data is backed up.

For a safe lab verification, inspect the target first:

```http
GET /_cat/indices/test-index?v
```

If this is not the disposable index you intended to remove, do not proceed. Never replace the name with a wildcard.

## What the original examples demonstrate

| Original operation | CRUD concept | Important behavior |
|---|---|---|
| `PUT electronics_store` | Create index | Creates an index; without a mapping, field types are inferred as data is indexed. |
| `POST electronics_store/_doc` | Create document | Generates an ID; repeating can create duplicates. |
| `PUT electronics_store/_doc/1` | Create/replace | Uses a known ID; replaces the full document if it already exists. |
| `PUT electronics_store/_create/2` | Create only | Fails if ID `2` already exists. |
| `GET electronics_store/_doc/1` | Read by ID | Retrieves one known document. |
| `GET electronics_store/_search` | Search/read | Finds matching documents; search visibility is near-real-time. |
| `POST electronics_store/_update/1` | Partial update | Merges fields from `doc` into an existing document. |
| `DELETE electronics_store/_doc/1` | Delete document | Removes one document, not the index. |
| `DELETE test-index` | Delete index | Removes a whole index and its contents; unrelated to the store example and destructive. |

## Operational habits

- Define mappings for fields with known types, especially dates, identifiers, and numeric values.
- Use numeric JSON values for numeric fields; avoid sending numbers as quoted strings.
- Use stable IDs when updates or retries must target the same logical entity.
- Know whether a request creates, replaces, partially updates, or deletes before executing it.
- Test destructive operations only on disposable data and verify the target path carefully.
- Keep credentials and real customer/product data out of screenshots and public examples.
- Use the response's `result`, `_id`, and error details to verify what happened.

## References

- [Create or update a document API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-index)
- [Create document API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-create)
- [Get document API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-get)
- [Search API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-search)
- [Update document API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-update)
- [Delete document API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-delete)
- [Delete index API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-delete)
- [Mappings](https://www.elastic.co/docs/manage-data/data-store/mapping)
