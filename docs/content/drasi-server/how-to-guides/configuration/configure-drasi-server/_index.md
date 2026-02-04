---
type: "docs"
title: "Configure Drasi Server"
linkTitle: "Configure Drasi Server"
weight: 10
description: "Understanding the Drasi Server YAML configuration file structure"
---

{{< term "Drasi Server" >}} uses YAML configuration files to define {{< term "Source" "sources" >}}, {{< term "Continuous Query" "queries" >}}, {{< term "Reaction" "reactions" >}}, and server settings. This guide explains the configuration structure and options.

## Configuration File Location

By default, Drasi Server looks for configuration at `config/server.yaml`. Override with the `--config` flag:

```bash
drasi-server --config /path/to/my-config.yaml
```

## Basic Structure

A complete configuration file has this structure.

{{< alert color="warning" >}}
Drasi Server config keys are **camelCase** and unknown fields are rejected. For example, use `logLevel` (not `log_level`) and `autoStart` (not `auto_start`).
{{< /alert >}}

```yaml
# Server identification
id: my-server-instance

# Server settings
host: 0.0.0.0
port: 8080
logLevel: info

# Persistence settings
persistConfig: true
persistIndex: false

# State store for plugin data (optional)
stateStore:
  kind: redb
  path: ./data/state.redb

# Performance tuning (optional)
defaultPriorityQueueCapacity: 10000
defaultDispatchBufferCapacity: 1000

# Data sources
sources:
  - kind: postgres
    id: my-source
    # ... source-specific configuration

# Continuous queries
queries:
  - id: my-query
    query: "MATCH (n) RETURN n"
    queryLanguage: GQL
    autoStart: false
    enableBootstrap: true
    sources:
      - sourceId: my-source

# Reactions to query changes
reactions:
  - kind: log
    id: my-reaction
    queries: [my-query]
    autoStart: true

# Multi-instance configuration (optional)
instances: []
```

## Server Settings

Top-level settings in `server.yaml`:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | string | Auto-generated UUID | Unique server identifier |
| `host` | string | `0.0.0.0` | Server bind address |
| `port` | integer | `8080` | REST API port |
| `logLevel` | string | `info` | Log level: `trace`, `debug`, `info`, `warn`, `error` |
| `persistConfig` | boolean | `true` | Persist API changes back to the config file (if writable) |
| `persistIndex` | boolean | `false` | Enable persistent indexes using RocksDB (stored under `./data/<instanceId>/index`) |
| `stateStore` | object | None | Persist plugin state across restarts (see below) |
| `defaultPriorityQueueCapacity` | integer | `10000` | Default event queue capacity for queries/reactions (optional override) |
| `defaultDispatchBufferCapacity` | integer | `1000` | Default dispatch buffer capacity for sources/queries (optional override) |
| `sources` | array | `[]` | Source plugin instances |
| `queries` | array | `[]` | Continuous queries |
| `reactions` | array | `[]` | Reactions to query changes |
| `instances` | array | `[]` | Optional multi-instance mode (see below) |

### Example

```yaml
id: production-server
host: 0.0.0.0
port: 8080
logLevel: info
persistConfig: true
persistIndex: true
```

## State Store Configuration

The state store persists plugin state across server restarts.

### REDB State Store

File-based persistent storage:

```yaml
stateStore:
  kind: redb
  path: ./data/state.redb
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kind` | string | Yes | Must be `redb` |
| `path` | string | Yes | Path to database file |

### In-Memory (Default)

If `stateStore` is not configured, an in-memory store is used and state is lost on restart.

## Performance Tuning

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `defaultPriorityQueueCapacity` | integer | `10000` | Default event priority queue capacity for queries/reactions |
| `defaultDispatchBufferCapacity` | integer | `1000` | Default dispatch buffer capacity for sources/queries |

```yaml
defaultPriorityQueueCapacity: 50000
defaultDispatchBufferCapacity: 5000
```

These can be overridden per-query:

```yaml
queries:
  - id: high-volume-query
    query: "MATCH (n) RETURN n"
    sources:
      - sourceId: my-source
    priorityQueueCapacity: 100000
    dispatchBufferCapacity: 10000
```

## Environment Variable Interpolation

Many scalar configuration values support environment variable substitution.

### Syntax

| Pattern | Behavior |
|---------|----------|
| `${VAR}` | Required - fails if not set |
| `${VAR:-default}` | Optional with default value |

### Examples

```yaml
host: ${SERVER_HOST:-0.0.0.0}
port: ${SERVER_PORT:-8080}

sources:
  - kind: postgres
    id: production-db
    host: ${DB_HOST}           # Required
    port: ${DB_PORT:-5432}     # Optional with default
    password: ${DB_PASSWORD}   # Required
```

### Using .env Files

Drasi Server loads a `.env` file **from the same directory as your config file** (if present). This is primarily for local development.

```bash
# config/.env
DB_HOST=postgres.example.com
DB_PASSWORD=secretpassword
```

## Creating Configuration Interactively

Use the `init` command to create a configuration file interactively:

```bash
drasi-server init --output config/server.yaml
```

Options:
- `--output`, `-o`: Output path (default: `config/server.yaml`)
- `--force`: Overwrite existing file

## Validating Configuration

Validate before starting the server:

```bash
# Basic validation
drasi-server validate --config config/server.yaml

# Show resolved environment variables
drasi-server validate --config config/server.yaml --show-resolved
```

## Multi-Instance Configuration

For advanced use cases, configure multiple isolated DrasiLib instances:

```yaml
host: 0.0.0.0
port: 8080
logLevel: info

instances:
  - id: analytics
    persistIndex: true
    stateStore:
      kind: redb
      path: ./data/analytics-state.redb
    sources:
      - kind: postgres
        id: analytics-db
        host: analytics-db.example.com
        # ...
    queries:
      - id: analytics-query
        query: "MATCH (n:Order) WHERE n.total > 1000 RETURN n"
        sources:
          - sourceId: analytics-db
    reactions:
      - kind: log
        id: analytics-log
        queries: [analytics-query]

  - id: monitoring
    persistIndex: false
    sources:
      - kind: http
        id: metrics-api
        host: 0.0.0.0
        port: 9001
    queries:
      - id: alert-threshold
        query: "MATCH (m:Metric) WHERE m.value > m.threshold RETURN m"
        sources:
          - sourceId: metrics-api
    reactions:
      - kind: sse
        id: alert-stream
        queries: [alert-threshold]
        port: 8082
```

Each instance has:
- Isolated namespace for sources, queries, reactions
- Optional separate state store
- API access via `/api/v1/instances/{instanceId}/...`

## Complete Example

```yaml
# Server identification and settings
id: drasi-production
host: 0.0.0.0
port: 8080
logLevel: info

# Enable persistence
persistConfig: true
persistIndex: true

# State store
stateStore:
  kind: redb
  path: ${DATA_PATH:-./data}/state.redb

# Performance settings
# (optional; defaults are 10000 and 1000)
defaultPriorityQueueCapacity: ${PRIORITY_QUEUE_CAPACITY:-10000}
defaultDispatchBufferCapacity: ${DISPATCH_BUFFER_CAPACITY:-1000}

# Sources
sources:
  - kind: postgres
    id: orders-db
    autoStart: true
    host: ${DB_HOST}
    port: ${DB_PORT:-5432}
    database: ${DB_NAME}
    user: ${DB_USER}
    password: ${DB_PASSWORD}
    sslMode: prefer
    tables:
      - public.orders
      - public.customers
    slotName: drasi_slot
    publicationName: drasi_publication
    tableKeys:
      - table: public.orders
        keyColumns: [id]
    bootstrapProvider:
      kind: postgres

  - kind: http
    id: webhook-receiver
    autoStart: true
    host: 0.0.0.0
    port: 9000
    timeoutMs: 10000

# Queries
queries:
  - id: high-value-orders
    autoStart: true
    queryLanguage: GQL
    query: |
      MATCH (o:orders)
      WHERE o.total > 1000
      RETURN o.id, o.customer_id, o.total, o.status
    sources:
      - sourceId: orders-db
    enableBootstrap: true

  - id: pending-orders
    autoStart: true
    queryLanguage: GQL
    query: |
      MATCH (o:orders)-[:CUSTOMER]->(c:customers)
      WHERE o.status = 'pending'
      RETURN o.id, c.name, c.email, o.total
    sources:
      - sourceId: orders-db
    joins:
      - id: CUSTOMER
        keys:
          - label: orders
            property: customer_id
          - label: customers
            property: id

# Reactions
reactions:
  - kind: log
    id: console-output
    queries: [high-value-orders, pending-orders]
    autoStart: true
    defaultTemplate:
      added:
        template: "[NEW] Order {{after.id}}: ${{after.total}}"
      updated:
        template: "[UPDATE] Order {{after.id}}: ${{before.total}} -> ${{after.total}}"
      deleted:
        template: "[DELETED] Order {{before.id}}"

  - kind: http
    id: webhook-notification
    queries: [high-value-orders]
    autoStart: true
    baseUrl: ${WEBHOOK_URL}
    token: ${WEBHOOK_TOKEN}
    timeoutMs: 5000
    routes:
      high-value-orders:
        added:
          url: /orders/high-value
          method: POST
          body: |
            {
              "order_id": "{{after.id}}",
              "total": {{after.total}},
              "customer_id": "{{after.customer_id}}"
            }
          headers:
            Content-Type: application/json

  - kind: sse
    id: live-updates
    queries: [pending-orders]
    autoStart: true
    host: 0.0.0.0
    port: 8081
    ssePath: /events
    heartbeatIntervalMs: 30000
```

## Configuration File Formats

Drasi Server supports both YAML and JSON configuration files. It tries to parse the file as YAML first and falls back to JSON if YAML parsing fails.

### YAML (Recommended)

```yaml
host: 0.0.0.0
port: 8080
sources:
  - kind: mock
    id: test
```

### JSON

```json
{
  "host": "0.0.0.0",
  "port": 8080,
  "sources": [
    {
      "kind": "mock",
      "id": "test"
    }
  ]
}
```

## Next Steps

- [Configure Sources](/drasi-server/how-to-guides/configuration/configure-sources/) - Connect to data sources
- [Configure Reactions](/drasi-server/how-to-guides/configuration/configure-reactions/) - Define what happens on changes
- [Configuration Reference](/drasi-server/reference/configuration/) - Complete configuration schema
