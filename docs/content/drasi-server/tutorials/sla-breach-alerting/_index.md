---
type: "docs"
title: "SLA Breach Alerting"
linkTitle: "SLA Breach Alerting"
weight: 10
description: "Detect orders at risk of missing fulfillment SLAs using time-based reactive queries"
related:
  concepts:
    - title: "Continuous Queries"
      url: "/concepts/continuous-queries/"
    - title: "Reactions"
      url: "/concepts/reactions/"
  tutorials:
    - title: "Smart Reorder Recommendations"
      url: "/drasi-server/tutorials/smart-reorder-recommendations/"
    - title: "Live Order Dashboard"
      url: "/drasi-server/tutorials/live-order-dashboard/"
  howto:
    - title: "Configure PostgreSQL Source"
      url: "/drasi-server/how-to-guides/configuration/configure-sources/configure-postgresql-source/"
    - title: "Configure Webhook Reaction"
      url: "/drasi-server/how-to-guides/configuration/configure-reactions/configure-webhook-reaction/"
  reference:
    - title: "Drasi Custom Functions"
      url: "/reference/query-language/drasi-custom-functions/"
---

# SLA Breach Alerting

In this tutorial, you'll build a real-time SLA monitoring system that detects orders at risk of missing fulfillment deadlines. You'll learn how Drasi's time-based queries elegantly solve a problem that's surprisingly difficult with traditional approaches.

## The Business Problem

Your e-commerce company promises same-day shipping for orders placed before 2 PM. The operations team needs to know immediately when an order hasn't been shipped within 4 hours of being placed, so they can intervene before customers start complaining.

**The stakes are real:**
- Late shipments generate support tickets costing $15-25 each to resolve
- VIP customers who experience delays have a 40% higher churn rate
- Operations managers need advance warning, not post-mortem reports

Currently, a scheduled job runs every 15 minutes querying for overdue orders. By the time an SLA breach is detected, it's often too late to recover. The team needs real-time alerts the moment an order becomes at-risk.

## Why Traditional Approaches Fall Short

### Polling Every N Minutes

```sql
-- Run this every 15 minutes via cron
SELECT o.id, o.created_at, c.name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status != 'shipped'
  AND o.created_at < NOW() - INTERVAL '4 hours';
```

**Problems:**
- You detect breaches up to 15 minutes late
- As order volume grows, this query hammers your database every 15 minutes
- You need to maintain the cron job, handle failures, and manage duplicates
- Edge cases: What if the job takes longer than 15 minutes? What if it fails silently?

### Database Triggers

```sql
CREATE TRIGGER check_sla_on_update
AFTER UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION check_order_sla();
```

**Problems:**
- Triggers fire on every order update, not just SLA-relevant ones
- Time-based conditions (`created_at < NOW() - INTERVAL '4 hours'`) don't trigger automatically
- You need a separate service to poll for time-based transitions
- Joining with customer data inside a trigger adds latency to every order update

### CDC + Stream Processing (Kafka, Flink)

**Problems:**
- Requires Debezium, Kafka, and a stream processor (Flink/Spark)
- Time-based windowing in stream processors is complex to get right
- Joining order data with customer data requires stateful stream processing
- Significant operational overhead for what should be a simple requirement

## Enter Drasi

With Drasi, you write a single declarative query that continuously monitors for SLA breaches. The `drasi.trueFor()` function handles the time-based logic, and the multi-table join is automatic.

```cypher
MATCH (o:orders)-[:PLACED_BY]->(c:customers)
WHERE o.status != 'shipped'
  AND drasi.trueFor(o.status != 'shipped', duration({hours: 4}))
RETURN
  o.id AS orderId,
  o.created_at AS orderTime,
  c.name AS customerName,
  c.email AS customerEmail,
  c.tier AS customerTier
```

When an order crosses the 4-hour threshold without being shipped, Drasi fires an event. No polling. No complex stream processing. Just a query.

## What You'll Build

- PostgreSQL database with orders and customers tables
- Drasi Server monitoring for SLA breaches
- Webhook reaction that fires when orders become at-risk
- Test scenarios to see the system in action

## Prerequisites

- Docker and Docker Compose
- curl (for API testing)
- Completed the [Getting Started](/drasi-server/getting-started/) guide

## Step 1: Create the Project

```bash
mkdir drasi-sla-alerting
cd drasi-sla-alerting
mkdir -p config scripts
```

## Step 2: Create Docker Compose

Create `docker-compose.yaml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: ecommerce
    ports:
      - "5432:5432"
    volumes:
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    command: >
      postgres
      -c wal_level=logical
      -c max_replication_slots=4
      -c max_wal_senders=4
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  drasi-server:
    image: ghcr.io/drasi-project/drasi-server:latest
    ports:
      - "8080:8080"
    volumes:
      - ./config:/config:ro
    environment:
      - RUST_LOG=info
    depends_on:
      postgres:
        condition: service_healthy
    command: ["--config", "/config/server.yaml"]

  # Simple webhook receiver to see alerts
  webhook-receiver:
    image: mendhak/http-https-echo:latest
    ports:
      - "8888:8080"
```

## Step 3: Create the Database Schema

Create `scripts/init.sql`:

```sql
-- Customers table
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL,
    tier VARCHAR(20) DEFAULT 'standard'
);

-- Orders table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    status VARCHAR(30) DEFAULT 'pending',
    total DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Create publication for CDC
CREATE PUBLICATION drasi_pub FOR ALL TABLES;

-- Sample customers
INSERT INTO customers (name, email, tier) VALUES
    ('Alice Johnson', 'alice@example.com', 'vip'),
    ('Bob Smith', 'bob@example.com', 'standard'),
    ('Carol Williams', 'carol@example.com', 'premium');

-- Note: We'll create orders during the tutorial to demonstrate time-based behavior
```

## Step 4: Configure Drasi

Create `config/server.yaml`:

```yaml
id: sla-monitor
host: 0.0.0.0
port: 8080
log_level: info

sources:
  - kind: postgres
    id: ecommerce-db
    host: postgres
    port: 5432
    database: ecommerce
    user: postgres
    password: postgres
    tables:
      - public.customers
      - public.orders
    slot_name: drasi_sla_slot
    publication_name: drasi_pub
    bootstrap_provider:
      type: postgres

queries:
  # Alert when an order hasn't shipped within the SLA window
  # Using a shorter duration (30 seconds) for demo purposes
  - id: sla-breach
    query: |
      MATCH (o:orders)-[:PLACED_BY]->(c:customers)
      WHERE o.status != 'shipped'
        AND drasi.trueFor(o.status != 'shipped', duration({seconds: 30}))
      RETURN
        o.id AS orderId,
        o.status AS orderStatus,
        o.total AS orderTotal,
        o.created_at AS orderTime,
        c.name AS customerName,
        c.email AS customerEmail,
        c.tier AS customerTier
    sources:
      - source_id: ecommerce-db
        nodes: [orders, customers]
    joins:
      - id: PLACED_BY
        keys:
          - label: orders
            property: customer_id
          - label: customers
            property: id
    auto_start: true

reactions:
  # Send webhook when SLA breach detected
  - kind: webhook
    id: sla-alerts
    queries: [sla-breach]
    url: http://webhook-receiver:8080/sla-alert
    method: POST
    headers:
      Content-Type: application/json
    auto_start: true

  # Also log to console for visibility
  - kind: log
    id: console-alerts
    queries: [sla-breach]
    routes:
      sla-breach:
        added:
          template: |
            [SLA BREACH] Order #{{after.orderId}} for {{after.customerName}} ({{after.customerTier}})
            has not shipped. Total: ${{after.orderTotal}}. Customer email: {{after.customerEmail}}
    auto_start: true
```

{{% alert title="Demo Timing" color="info" %}}
For this tutorial, we use `duration({seconds: 30})` so you can see results quickly. In production, you'd use `duration({hours: 4})` or whatever your actual SLA requires.
{{% /alert %}}

## Step 5: Start the Environment

```bash
docker compose up -d
```

Wait for services to be ready:

```bash
# Check Drasi Server health
until curl -s http://localhost:8080/health > /dev/null; do
    echo "Waiting for Drasi Server..."
    sleep 2
done
echo "Drasi Server is ready!"
```

Verify the query is running:

```bash
curl -s http://localhost:8080/api/v1/queries/sla-breach | jq '.data.status'
```

## Step 6: Create Test Orders

Open a terminal to watch the Drasi Server logs:

```bash
docker compose logs -f drasi-server
```

In another terminal, connect to PostgreSQL and create some orders:

```bash
docker compose exec postgres psql -U postgres -d ecommerce
```

```sql
-- Create an order that will breach SLA (we won't ship it)
INSERT INTO orders (customer_id, status, total)
VALUES (1, 'pending', 299.99);

-- Create another order
INSERT INTO orders (customer_id, status, total)
VALUES (2, 'pending', 149.99);

-- Check current orders
SELECT o.id, o.status, o.total, c.name, o.created_at
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

## Step 7: Watch the SLA Breach Detection

Wait 30 seconds (our demo SLA window). Watch the Drasi Server logs. You should see alerts like:

```
[SLA BREACH] Order #1 for Alice Johnson (vip) has not shipped. Total: $299.99. Customer email: alice@example.com
[SLA BREACH] Order #2 for Bob Smith (standard) has not shipped. Total: $149.99. Customer email: bob@example.com
```

Check the webhook receiver to see the alerts that were sent:

```bash
docker compose logs webhook-receiver
```

## Step 8: Ship an Order and See It Clear

Back in the PostgreSQL terminal:

```sql
-- Ship order #1
UPDATE orders SET status = 'shipped', updated_at = NOW() WHERE id = 1;
```

Watch the logs. Order #1 is no longer in the SLA breach results because it's been shipped. Drasi automatically detected the status change and removed it from the query results.

## Step 9: Query Current Breaches via API

```bash
curl -s http://localhost:8080/api/v1/queries/sla-breach/results | jq
```

This returns the current set of orders in SLA breach, useful for dashboards or integration with other systems.

## What You Learned

1. **Time-based reactive queries**: `drasi.trueFor()` monitors for conditions that remain true for a specified duration, perfect for SLA monitoring without polling.

2. **Multi-table reactive joins**: The query automatically joins orders with customers and maintains this relationship reactively. When either table changes, affected results update immediately.

3. **Webhook reactions**: Drasi can push alerts to external systems the moment conditions are met, enabling integration with Slack, PagerDuty, or custom alerting systems.

4. **Declarative simplicity**: What would require a cron job, state management, and careful duplicate handling is expressed in a single Cypher query.

## Production Considerations

### Scaling
- For high-volume systems, consider separate queries for different customer tiers (VIP customers might have a 2-hour SLA)
- Use the profiler reaction to monitor query performance

### Alert Fatigue
- Consider adding a debounce or digest pattern for high-volume alerts
- Use the webhook reaction to send to an aggregation service rather than directly to chat

### Error Handling
- The webhook reaction retries on failure
- Monitor Drasi Server logs for connection issues
- Consider a dead-letter queue for failed webhooks

### Real SLA Windows
Replace the demo duration with your actual SLA:

```cypher
drasi.trueFor(o.status != 'shipped', duration({hours: 4}))
```

Or for VIP customers with faster SLAs:

```cypher
MATCH (o:orders)-[:PLACED_BY]->(c:customers)
WHERE o.status != 'shipped'
  AND c.tier = 'vip'
  AND drasi.trueFor(o.status != 'shipped', duration({hours: 2}))
RETURN ...
```

## Cleanup

```bash
docker compose down -v
cd ..
rm -rf drasi-sla-alerting
```

## Next Steps

Continue to the next tutorial to learn about more complex multi-table joins and aggregations:

- [Smart Reorder Recommendations](/drasi-server/tutorials/smart-reorder-recommendations/) - Build velocity-based inventory alerts using 4-table joins
