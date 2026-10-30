# Elasticsearch Configuration Basics

> Configuration is deployment-specific: distinguish node bootstrap settings from live cluster settings.

## Table of contents

- [Choose the right configuration surface](#choose-the-right-configuration-surface)
- [Self-managed configuration files](#self-managed-configuration-files)
- [Cluster settings API](#cluster-settings-api)
- [Change-management checklist](#change-management-checklist)
- [References](#references)

## Choose the right configuration surface

How you configure Elasticsearch depends on where it runs:

| Deployment | Typical configuration path |
|---|---|
| Self-managed | `elasticsearch.yml` for node/startup settings; APIs for supported dynamic cluster settings. |
| Elastic Cloud Hosted | Deployment configuration and supported settings through the Cloud Console/API. You do not edit the managed hosts' local files. |
| Elastic Cloud Serverless | Service-managed configuration; use the supported project settings and APIs. Do not assume node-level controls are exposed. |

Always check that a setting is supported for the deployment type and current version before planning a change.

## Self-managed configuration files

Self-managed Elasticsearch commonly uses:

- `elasticsearch.yml` for Elasticsearch configuration.
- `jvm.options` for JVM options.
- `log4j2.properties` for logging configuration.

For example, node-specific bootstrap configuration may include:

```yaml
node.name: es-lab-01
```

Use the official setting reference for valid names, types, defaults, and whether a restart is required. Protect configuration files and avoid committing environment-specific secrets.

## Cluster settings API

Supported dynamic cluster settings can be read or changed through the Cluster Update Settings API. For example, inspect current settings:

```http
GET /_cluster/settings?include_defaults=true
```

The API distinguishes persistent settings (survive restarts) and transient settings. Elastic no longer recommends transient settings; prefer persistent settings when the chosen setting is dynamic and appropriate for the deployment. Never assume every setting is dynamically changeable.

## Change-management checklist

1. State the problem and expected measurable result.
2. Verify scope, setting type, version support, and deployment support.
3. Capture the current value and define a rollback.
4. Change one setting at a time in a controlled environment.
5. Verify behavior and resource impact; document owner and expiration/review date.

Configuration is an operational contract: an undocumented override can outlive the incident that motivated it and surprise the next operator.

## References

- [Configure Elasticsearch](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/configure-elasticsearch)
- [Elasticsearch configuration reference](https://www.elastic.co/docs/reference/elasticsearch/configuration-reference)
- [Cluster Update Settings API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-put-settings)
- [Elastic Cloud Hosted settings](https://www.elastic.co/docs/deploy-manage/stack-settings)
