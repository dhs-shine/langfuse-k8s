# ClickHouse Subchart Values Mapping (Bitnami -> ClickHouse)

This document tracks how Langfuse chart values are interpreted after moving away from the Bitnami ClickHouse chart.

## Compatibility Goals

- Keep top-level `clickhouse.*` values stable for Langfuse users.
- Keep `clickhouse.deploy=false` external ClickHouse flow unchanged.
- Keep env var rendering (`CLICKHOUSE_URL`, `CLICKHOUSE_MIGRATION_URL`, `CLICKHOUSE_USER`, `CLICKHOUSE_PASSWORD`) unchanged.

## Value Mapping

| Langfuse value | Previous (Bitnami chart) | New behavior |
|---|---|---|
| `clickhouse.deploy` | Enabled Bitnami clickhouse dependency | Enables ClickHouse dependency from `oci://ghcr.io/clickhouse/charts` |
| `clickhouse.host` | Host override used by Langfuse templates | Same |
| `clickhouse.serviceName` | N/A | Optional DNS service override when `deploy=true` |
| `clickhouse.httpPort` / `clickhouse.nativePort` | Used by Langfuse env var generation | Same |
| `clickhouse.auth.*` | Used by Langfuse env var generation and Bitnami defaults | Used by Langfuse env var generation |
| `clickhouse.image.repository` | Overrode Bitnami image repository | Defaults to `clickhouse/clickhouse-server` |
| `clickhouse.image.tag` | N/A | Optional tag override |
| `clickhouse.shards` / `clickhouse.replicaCount` | Bitnami cluster topology inputs | Kept for compatibility + Langfuse cluster-mode logic |
| `clickhouse.resourcesPreset` | Bitnami preset | Deprecated (no-op compatibility field) |
| `clickhouse.zookeeper.*` | Bitnami zookeeper config | Deprecated (no-op compatibility field) |

## Notes

- `clickhouse.resourcesPreset` and `clickhouse.zookeeper` are intentionally retained as compatibility placeholders.
- The Langfuse chart still enforces `clickhouse.shards=1` because Langfuse does not support sharding.
