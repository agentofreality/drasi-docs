---
type: "docs"
title: "Configure Sources"
linkTitle: "Configure Sources"
weight: 20
no_list: true
notoc: true
hide_readingtime: true
description: "Connect Drasi Server to databases, APIs, and data streams"
---

{{< term "Source" "Sources" >}} connect {{< term "Drasi Server" >}} to your data systems and stream changes to {{< term "Continuous Query" "queries" >}}.

Drasi Server currently supports these source kinds:
- `postgres`
- `http`
- `grpc`
- `platform`
- `mock`

{{< alert title="Configuration keys are camelCase" color="warning" >}}
Source configuration is parsed with strict **camelCase** keys (for example: `autoStart`, `bootstrapProvider`, `timeoutMs`). Unknown keys are rejected.
{{< /alert >}}

<div class="card-grid">
  <a href="configure-postgresql-source/">
    <div class="unified-card unified-card--howto">
      <div class="unified-card-icon"><i class="fas fa-database"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">PostgreSQL</h3>
        <p class="unified-card-summary">Stream changes from PostgreSQL using logical replication (WAL)</p>
      </div>
    </div>
  </a>
  <a href="configure-http-source/">
    <div class="unified-card unified-card--howto">
      <div class="unified-card-icon"><i class="fas fa-globe"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">HTTP</h3>
        <p class="unified-card-summary">Receive events via HTTP endpoints</p>
      </div>
    </div>
  </a>
  <a href="configure-grpc-source/">
    <div class="unified-card unified-card--howto">
      <div class="unified-card-icon"><i class="fas fa-network-wired"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">gRPC</h3>
        <p class="unified-card-summary">Receive events via gRPC streaming</p>
      </div>
    </div>
  </a>
  <a href="configure-mock-source/">
    <div class="unified-card unified-card--howto">
      <div class="unified-card-icon"><i class="fas fa-flask"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Mock</h3>
        <p class="unified-card-summary">Generate test data for development</p>
      </div>
    </div>
  </a>
  <a href="configure-platform-source/">
    <div class="unified-card unified-card--howto">
      <div class="unified-card-icon"><i class="fas fa-stream"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Platform</h3>
        <p class="unified-card-summary">Consume from Redis Streams for Drasi Platform integration</p>
      </div>
    </div>
  </a>
</div>
