# Elasticsearch Configuration Basics

> Configuration is deployment-specific: know what changes at startup, what can change live, and who owns the control plane.

## Table of contents

- [Choose the right configuration surface](#choose-the-right-configuration-surface)
- [Self-managed configuration files](#self-managed-configuration-files)
- [Cluster settings and precedence](#cluster-settings-and-precedence)
- [A safe settings workflow](#a-safe-settings-workflow)
- [Configuration as code](#configuration-as-code)

## Choose the right configuration surface

The deployment model determines where settings are applied and which ones are available:

| Deployment | Typical configuration path |
|---|---|
| Self-managed | Local configuration files for node/startup settings; APIs for supported dynamic cluster settings. |
| Elastic Cloud Hosted | Deployment configuration and supported settings through the Cloud Console/API; operators do not edit the managed hosts' local files. |
| Elastic Cloud Serverless | Service-managed infrastructure; configure only the project settings and APIs exposed by the service. |

Do not copy an `elasticsearch.yml` recipe into a managed deployment without checking its supported settings page. A setting's existence does not mean every deployment exposes it or that a restart is avoidable.

## Self-managed configuration files

Common Elasticsearch files include:

- `elasticsearch.yml`: Elasticsearch node and startup configuration.
- `jvm.options`: JVM options.
- `log4j2.properties`: Elasticsearch logging configuration.

The config directory depends on the installation method and can be set using `ES_PATH_CONF`. For example:

```yaml
cluster.name: search-lab
node.name: es-lab-01
```

This is illustrative only; actual cluster discovery, networking, security, and storage settings depend on deployment design. Use the configuration reference to check valid names, types, defaults, and whether a restart is required. Protect files and environment variables as operational assets; do not commit secrets.

## Cluster settings and precedence

Elasticsearch settings are generally categorized as **static** or **dynamic**:

- **Static settings** must be configured before a node starts and typically require a restart to change.
- **Dynamic settings** can be updated while the cluster is running, when supported by the specific setting.

For self-managed deployments, dynamic cluster settings are commonly managed through the Cluster Update Settings API. Cluster settings can be persistent (survive restarts) or transient (cleared on restart). Elastic no longer recommends transient settings; prefer persistent settings when appropriate.

When a setting is defined in multiple places, precedence matters. Transient settings override persistent settings, which override `elasticsearch.yml`, which overrides defaults. This can make local file edits appear ineffective when a higher-precedence cluster setting remains set.

Inspect current settings, including defaults:

```http
GET /_cluster/settings?include_defaults=true
```

Always consult the setting-specific reference for its scope and update semantics before changing it.

## A safe settings workflow

1. **Define the issue:** State the operational symptom and measurable outcome expected from the change.
2. **Check eligibility:** Confirm setting name, type, supported version, scope, and deployment model.
3. **Record baseline:** Capture current values and relevant health/performance metrics.
4. **Plan rollback:** Decide how to restore the prior value and how to verify recovery.
5. **Change one variable:** Use a controlled environment and the least disruptive supported interface.
6. **Validate:** Compare behavior and resource impact; do not rely only on an HTTP success response.
7. **Document ownership:** Record approver, reason, date, and review/expiry if the change is temporary.

Configuration is an operational contract. An undocumented override can outlive the incident that motivated it and surprise the next operator.

## Configuration as code

Treat repeatable settings as reviewed change artifacts: keep sanitized templates in version control, parameterize environment-specific values, and separate secrets into an approved secrets manager. Where configuration is managed through a provider or deployment API, track the desired state and verify that the live deployment has converged to it.
