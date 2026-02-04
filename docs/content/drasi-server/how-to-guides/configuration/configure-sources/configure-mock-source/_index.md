---
type: "docs"
title: "Configure Mock Source"
linkTitle: "Mock"
weight: 30
description: "Generate synthetic node inserts for development and testing"
related:
  concepts:
    - title: "Sources"
      url: "/concepts/sources/"
  howto:
    - title: "Configure Bootstrap Providers"
      url: "/drasi-server/how-to-guides/configuration/configure-bootstrap-providers/"
  reference:
    - title: "Configuration Reference"
      url: "/drasi-server/reference/configuration/"
---

The Mock {{< term "Source" >}} generates synthetic data on a timer, so you can test queries and reactions without connecting to a real system.

## When to use the Mock source

- Developing and testing continuous queries locally.
- Demo environments where you want predictable, infrastructure-free data generation.
- Load/throughput testing (within reason).

## Quick example (Drasi Server config)

Drasi Server source configuration uses **camelCase** keys.

```yaml
sources:
  - kind: mock
    id: demo-sensors
    autoStart: true

    # Optional
    dataType: sensor
    intervalMs: 1000
```

## Configuration reference (Drasi Server)

| Field | Type | Default | Description |
|---|---:|---:|---|
| `kind` | string | required | Must be `mock`. |
| `id` | string | required | Unique source identifier. |
| `autoStart` | boolean | `true` | Whether Drasi Server starts the source on startup. |
| `bootstrapProvider` | object | none | Optional bootstrap provider for initial state (usually not needed for mock data). |
| `dataType` | string | `generic` | Data generation mode: `counter`, `sensor`, or `generic`. |
| `intervalMs` | integer | `5000` | Interval between generated events in milliseconds (must be `> 0`). |

Fields support Drasi Server config references like `${ENV_VAR}` / `${ENV_VAR:-default}`.

## Data types and generated nodes

The Mock source generates **insert** events for nodes.

### Counter (dataType: counter)

- **Label**: `Counter`
- **Element ID**: `counter_{sequence}`
- **Properties**: `value` (sequential integer), `timestamp` (RFC3339 string)

### Sensor (dataType: sensor)

- **Label**: `SensorReading`
- **Element ID**: `reading_{sensor_id}_{sequence}`
- **Properties**: `sensor_id`, `temperature` (20..30), `humidity` (40..60), `timestamp`

### Generic (dataType: generic) (default)

- **Label**: `Generic`
- **Element ID**: `generic_{sequence}`
- **Properties**: `value` (random i32), `message` ("Generic mock data"), `timestamp`

## Performance tuning notes

- Decrease `intervalMs` to generate more data (higher load).
- Use multiple mock sources with different `id`s if you want independent streams.

## Troubleshooting

**Startup error: "Validation error: data_type '...' is not valid"**
- `dataType` must be exactly one of: `counter`, `sensor`, `generic`.

**Startup error: "interval_ms cannot be 0"**
- Set `intervalMs` to a positive integer.

## Known limitations

- Generates inserts only (no updates / deletes).
- Sequence counters reset on restart; data is not persisted.
- No relationships are generated.

## Documentation resources

<div class="card-grid card-grid--2">
  <a href="https://github.com/drasi-project/drasi-core/blob/main/components/sources/mock/README.md" target="_blank" rel="noopener">
    <div class="unified-card unified-card--tutorials">
      <div class="unified-card-icon"><i class="fab fa-github"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Mock Source README</h3>
        <p class="unified-card-summary">Generated schemas, validation rules, and behavior details</p>
      </div>
    </div>
  </a>
  <a href="https://crates.io/crates/drasi-source-mock" target="_blank" rel="noopener">
    <div class="unified-card unified-card--howto">
      <div class="unified-card-icon"><i class="fas fa-box"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">drasi-source-mock on crates.io</h3>
        <p class="unified-card-summary">Package info and release history</p>
      </div>
    </div>
  </a>
</div>
