---
type: "docs"
title: "Smart Reorder Recommendations"
linkTitle: "Smart Reorder Recommendations"
weight: 20
description: "Trigger inventory reorders based on real-time sales velocity across multiple data sources"
related:
  concepts:
    - title: "Continuous Queries"
      url: "/concepts/continuous-queries/"
    - title: "Reactions"
      url: "/concepts/reactions/"
  tutorials:
    - title: "SLA Breach Alerting"
      url: "/drasi-server/tutorials/sla-breach-alerting/"
    - title: "Live Order Dashboard"
      url: "/drasi-server/tutorials/live-order-dashboard/"
  howto:
    - title: "Configure SSE Reaction"
      url: "/drasi-server/how-to-guides/configuration/configure-reactions/configure-sse-reaction/"
  reference:
    - title: "Configuration Reference"
      url: "/drasi-server/reference/configuration/"
---

# Smart Reorder Recommendations

In this tutorial, you'll build an intelligent inventory reorder system that considers sales velocity, not just static thresholds. You'll learn how Drasi's multi-table reactive joins can solve complex problems that would require significant stream processing infrastructure with traditional approaches.

## The Business Problem

Your warehouse team needs to reorder products before they stock out. The naive approach is simple: alert when inventory drops below a fixed threshold. But this doesn't account for how fast items are selling.

**Consider two products:**
- **Product A**: 50 units in stock, sells 2 per day. You have 25 days of inventory.
- **Product B**: 50 units in stock, sells 30 per day. You have less than 2 days of inventory.

A static threshold of "alert below 50 units" treats these identically. But Product B is a crisis while Product A is fine.

**What you really need:**
- Calculate sales velocity from recent orders
- Compare velocity against current inventory
- Factor in supplier lead times
- Alert when "days until stockout" is less than lead time

## Why Traditional Approaches Fall Short

### Simple Database Triggers

```sql
-- Alert when quantity drops below threshold
CREATE TRIGGER low_inventory_alert
AFTER UPDATE ON inventory
FOR EACH ROW
WHEN (NEW.quantity < NEW.min_threshold)
EXECUTE FUNCTION send_alert();
```

**Problems:**
- No velocity calculation. Can't tell if you're selling 2/day or 200/day.
- Static thresholds miss fast-moving items and create noise for slow-moving items.
- Need external system to analyze sales trends.

### Scheduled Analytics Job

```sql
-- Run nightly to calculate reorder recommendations
WITH sales_velocity AS (
    SELECT product_id, COUNT(*) / 7.0 as daily_rate
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.id
    WHERE o.created_at > NOW() - INTERVAL '7 days'
    GROUP BY product_id
)
SELECT p.name, i.quantity, sv.daily_rate,
       i.quantity / sv.daily_rate as days_remaining
FROM products p
JOIN inventory i ON p.id = i.product_id
JOIN sales_velocity sv ON p.id = sv.product_id
JOIN suppliers s ON p.supplier_id = s.id
WHERE i.quantity / sv.daily_rate < s.lead_time_days * 1.5;
```

**Problems:**
- Runs once per day (or hour). A product could sell out between runs.
- Doesn't react to sudden demand spikes (flash sales, viral moments).
- Complex to maintain as business rules evolve.

### Stream Processing (Kafka + Flink)

**Problems:**
- Requires maintaining sliding window aggregates across multiple streams.
- Joining 4 tables (products, inventory, order_items, suppliers) in a stream processor is complex.
- Late-arriving data handling adds significant complexity.
- Substantial operational overhead for what should be a business rule.

## Enter Drasi

With Drasi, you write a declarative query that joins 4 tables, calculates velocity, and computes days-until-stockout. The query continuously maintains this computation as orders flow in and inventory changes.

```cypher
MATCH (p:products)<-[:CONTAINS]-(oi:order_items)-[:PART_OF]->(o:orders),
      (p)-[:STOCKED_AT]->(inv:inventory),
      (p)-[:SUPPLIED_BY]->(s:suppliers)
WHERE o.created_at > datetime() - duration({days: 7})
WITH p, inv, s,
     count(oi) AS units_sold_7d,
     (count(oi) / 7.0) AS daily_velocity
WHERE daily_velocity > 0
WITH p, inv, s, daily_velocity,
     (inv.quantity / daily_velocity) AS days_until_stockout
WHERE days_until_stockout < s.lead_time_days * 1.5
RETURN
  p.id AS productId,
  p.name AS productName,
  inv.quantity AS currentStock,
  round(daily_velocity * 100) / 100 AS salesPerDay,
  round(days_until_stockout * 10) / 10 AS daysRemaining,
  s.lead_time_days AS supplierLeadTime,
  s.name AS supplierName
```

Every time an order is placed or inventory changes, Drasi re-evaluates which products need reordering. No polling. No batch jobs. Real-time recommendations.

## What You'll Build

- E-commerce database with products, inventory, orders, and suppliers
- Drasi query that calculates velocity-based reorder recommendations
- SSE reaction to stream recommendations to a procurement dashboard
- Test scenarios with varying sales velocities

## Prerequisites

- Docker and Docker Compose
- curl (for API testing)
- Completed [SLA Breach Alerting](/drasi-server/tutorials/sla-breach-alerting/) tutorial

## Step 1: Create the Project

```bash
mkdir drasi-reorder-system
cd drasi-reorder-system
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
      - "8081:8081"
    volumes:
      - ./config:/config:ro
    environment:
      - RUST_LOG=info
    depends_on:
      postgres:
        condition: service_healthy
    command: ["--config", "/config/server.yaml"]
```

## Step 3: Create the Database Schema

Create `scripts/init.sql`:

```sql
-- Suppliers
CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    lead_time_days INTEGER DEFAULT 5
);

-- Products
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    sku VARCHAR(50) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    supplier_id INTEGER REFERENCES suppliers(id)
);

-- Inventory
CREATE TABLE inventory (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER DEFAULT 0,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Customers
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL
);

-- Orders
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    status VARCHAR(30) DEFAULT 'completed',
    total DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Order Items
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL
);

-- Create publication for CDC
CREATE PUBLICATION drasi_pub FOR ALL TABLES;

-- Sample data: Suppliers
INSERT INTO suppliers (name, lead_time_days) VALUES
    ('FastShip Co', 3),
    ('Reliable Imports', 7),
    ('Budget Supplies', 14);

-- Sample data: Products
INSERT INTO products (name, sku, price, supplier_id) VALUES
    ('Wireless Mouse', 'WM-001', 29.99, 1),
    ('USB-C Hub', 'HUB-002', 49.99, 1),
    ('Mechanical Keyboard', 'KB-003', 129.99, 2),
    ('Monitor Stand', 'MS-004', 79.99, 2),
    ('Desk Lamp', 'DL-005', 39.99, 3);

-- Sample data: Initial inventory
INSERT INTO inventory (product_id, quantity) VALUES
    (1, 100),  -- Wireless Mouse: plenty of stock
    (2, 25),   -- USB-C Hub: moderate stock
    (3, 50),   -- Mechanical Keyboard: good stock
    (4, 15),   -- Monitor Stand: low stock
    (5, 200);  -- Desk Lamp: lots of stock

-- Sample data: Customers
INSERT INTO customers (name, email) VALUES
    ('Alice Johnson', 'alice@example.com'),
    ('Bob Smith', 'bob@example.com'),
    ('Carol Williams', 'carol@example.com');

-- Sample data: Historical orders (past 7 days) to establish velocity
-- USB-C Hub: High velocity (selling ~10/day)
INSERT INTO orders (customer_id, status, total, created_at)
SELECT
    (id % 3) + 1,
    'completed',
    49.99 * 2,
    NOW() - (id || ' hours')::interval
FROM generate_series(1, 70) AS s(id);

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT id, 2, 1, 49.99 FROM orders WHERE id <= 70;

-- Mechanical Keyboard: Low velocity (selling ~2/day)
INSERT INTO orders (customer_id, status, total, created_at)
SELECT
    (id % 3) + 1,
    'completed',
    129.99,
    NOW() - ((id * 8) || ' hours')::interval
FROM generate_series(71, 84) AS s(id);

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT id, 3, 1, 129.99 FROM orders WHERE id BETWEEN 71 AND 84;

-- Monitor Stand: Medium velocity (selling ~5/day)
INSERT INTO orders (customer_id, status, total, created_at)
SELECT
    (id % 3) + 1,
    'completed',
    79.99,
    NOW() - ((id * 4) || ' hours')::interval
FROM generate_series(85, 119) AS s(id);

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT id, 4, 1, 79.99 FROM orders WHERE id BETWEEN 85 AND 119;
```

## Step 4: Configure Drasi

Create `config/server.yaml`:

```yaml
id: reorder-system
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
      - public.products
      - public.inventory
      - public.orders
      - public.order_items
      - public.suppliers
    slot_name: drasi_reorder_slot
    publication_name: drasi_pub
    bootstrap_provider:
      type: postgres

queries:
  # Smart reorder recommendations based on velocity
  - id: reorder-recommendations
    query: |
      MATCH (p:products)<-[:CONTAINS]-(oi:order_items)-[:PART_OF]->(o:orders),
            (p)-[:STOCKED_AT]->(inv:inventory),
            (p)-[:SUPPLIED_BY]->(s:suppliers)
      WHERE o.created_at > datetime() - duration({days: 7})
      WITH p, inv, s,
           count(oi) AS units_sold_7d,
           (toFloat(count(oi)) / 7.0) AS daily_velocity
      WHERE daily_velocity > 0
      WITH p, inv, s, units_sold_7d, daily_velocity,
           (toFloat(inv.quantity) / daily_velocity) AS days_until_stockout
      WHERE days_until_stockout < toFloat(s.lead_time_days) * 1.5
      RETURN
        p.id AS productId,
        p.name AS productName,
        p.sku AS sku,
        inv.quantity AS currentStock,
        units_sold_7d AS unitsSoldLast7Days,
        round(daily_velocity * 100.0) / 100.0 AS salesPerDay,
        round(days_until_stockout * 10.0) / 10.0 AS daysRemaining,
        s.lead_time_days AS supplierLeadTime,
        s.name AS supplierName
    sources:
      - source_id: ecommerce-db
        nodes: [products, inventory, orders, order_items, suppliers]
    joins:
      - id: CONTAINS
        keys:
          - label: order_items
            property: product_id
          - label: products
            property: id
      - id: PART_OF
        keys:
          - label: order_items
            property: order_id
          - label: orders
            property: id
      - id: STOCKED_AT
        keys:
          - label: products
            property: id
          - label: inventory
            property: product_id
      - id: SUPPLIED_BY
        keys:
          - label: products
            property: supplier_id
          - label: suppliers
            property: id
    auto_start: true

reactions:
  # SSE stream for real-time dashboard
  - kind: sse
    id: reorder-stream
    queries: [reorder-recommendations]
    host: 0.0.0.0
    port: 8081
    sse_path: /events
    auto_start: true

  # Log for visibility
  - kind: log
    id: reorder-log
    queries: [reorder-recommendations]
    routes:
      reorder-recommendations:
        added:
          template: |
            [REORDER NEEDED] {{after.productName}} ({{after.sku}})
            Stock: {{after.currentStock}} | Velocity: {{after.salesPerDay}}/day | Days left: {{after.daysRemaining}}
            Supplier: {{after.supplierName}} ({{after.supplierLeadTime}} day lead time)
        deleted:
          template: |
            [RESOLVED] {{before.productName}} no longer needs reorder
    auto_start: true
```

## Step 5: Start the Environment

```bash
docker compose up -d
```

Wait for services:

```bash
until curl -s http://localhost:8080/health > /dev/null; do
    echo "Waiting for Drasi Server..."
    sleep 2
done
echo "Drasi Server is ready!"
```

## Step 6: Check Initial Recommendations

The historical data we seeded should already trigger some recommendations:

```bash
curl -s http://localhost:8080/api/v1/queries/reorder-recommendations/results | jq
```

You should see products like the USB-C Hub (high velocity, limited stock) appearing in the recommendations.

Watch the logs to see the recommendations:

```bash
docker compose logs -f drasi-server
```

## Step 7: Simulate a Sales Spike

Let's simulate a sudden spike in Mechanical Keyboard sales. Connect to PostgreSQL:

```bash
docker compose exec postgres psql -U postgres -d ecommerce
```

Add a burst of keyboard orders:

```sql
-- Simulate 20 keyboard orders in the past few hours
INSERT INTO orders (customer_id, status, total, created_at)
SELECT
    (id % 3) + 1,
    'completed',
    129.99,
    NOW() - ((id * 10) || ' minutes')::interval
FROM generate_series(1, 20) AS s(id);

-- Get the IDs of the new orders
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT id, 3, 1, 129.99
FROM orders
WHERE id > (SELECT MAX(id) - 20 FROM orders);
```

Watch the Drasi logs. The Mechanical Keyboard should now appear in recommendations because the sales velocity just spiked.

## Step 8: Restock and Watch It Clear

Simulate receiving a shipment:

```sql
-- Restock the Monitor Stand
UPDATE inventory SET quantity = 100, updated_at = NOW()
WHERE product_id = 4;
```

Watch the logs. The Monitor Stand should disappear from recommendations because it now has plenty of stock relative to its velocity.

## Step 9: Connect to the SSE Stream

Open a new terminal and subscribe to the live recommendations stream:

```bash
curl -N http://localhost:8081/events
```

In the PostgreSQL terminal, make changes and watch them stream in real-time:

```sql
-- Create more orders for the USB-C Hub
INSERT INTO orders (customer_id, status, total, created_at)
VALUES (1, 'completed', 99.98, NOW());

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES ((SELECT MAX(id) FROM orders), 2, 2, 49.99);

-- Or restock something
UPDATE inventory SET quantity = 50, updated_at = NOW()
WHERE product_id = 2;
```

## What You Learned

1. **Multi-table reactive joins**: The query joins 5 tables (products, inventory, orders, order_items, suppliers) and maintains these relationships reactively. When any table changes, affected recommendations update immediately.

2. **Aggregations in continuous queries**: `count()` and arithmetic operations (`quantity / velocity`) work continuously as data changes.

3. **Business logic in queries**: Complex business rules (velocity calculation, lead time comparison) are expressed declaratively rather than in application code.

4. **SSE for real-time dashboards**: The SSE reaction enables building live dashboards that update without polling.

## Production Considerations

### Handling Seasonality

Sales velocity varies by day of week and season. Consider:
- Using weighted averages (recent days count more)
- Separate queries for different time windows
- External system for seasonal adjustment factors

### Debouncing Recommendations

Recommendations might jitter as orders come in. Consider:
- Adding a minimum time between recommendation updates
- Requiring the condition to be true for N minutes before alerting
- Batching recommendations for procurement review

### Multiple Warehouses

Extend the schema to include warehouse locations:

```cypher
MATCH (p:products)-[:STOCKED_AT]->(inv:inventory)-[:LOCATED_AT]->(w:warehouses),
      ...
WHERE inv.quantity / daily_velocity < w.reorder_threshold
RETURN p.name, w.name, ...
```

### Performance at Scale

For catalogs with thousands of products:
- Consider partitioning by category or supplier
- Use the profiler reaction to monitor query performance
- Tune the 7-day window based on your data patterns

## Cleanup

```bash
docker compose down -v
cd ..
rm -rf drasi-reorder-system
```

## Next Steps

Continue to the final tutorial to learn about building complete dashboards:

- [Live Order Dashboard](/drasi-server/tutorials/live-order-dashboard/) - Build a real-time operations dashboard with multiple coordinated queries
