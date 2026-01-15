---
type: "docs"
title: "Live Order Dashboard"
linkTitle: "Live Order Dashboard"
weight: 30
description: "Build a real-time operations dashboard with aggregated order status views"
related:
  concepts:
    - title: "Continuous Queries"
      url: "/concepts/continuous-queries/"
    - title: "Reactions"
      url: "/concepts/reactions/"
  tutorials:
    - title: "SLA Breach Alerting"
      url: "/drasi-server/tutorials/sla-breach-alerting/"
    - title: "Smart Reorder Recommendations"
      url: "/drasi-server/tutorials/smart-reorder-recommendations/"
  howto:
    - title: "Configure SSE Reaction"
      url: "/drasi-server/how-to-guides/configuration/configure-reactions/configure-sse-reaction/"
  reference:
    - title: "Configuration Reference"
      url: "/drasi-server/reference/configuration/"
---

# Live Order Dashboard

In this tutorial, you'll build a real-time operations dashboard that displays live order statistics, payment status, and stuck order alerts. You'll learn how to coordinate multiple continuous queries and consume them in a browser-based UI.

## The Business Problem

Your operations team monitors order fulfillment throughout the day. They need to see:

- **Order counts by status** (pending, processing, shipped, delivered)
- **Revenue by status** (how much money is in each stage)
- **Orders stuck in a status** (orders that haven't moved in 2+ hours)
- **Payment failures** that need attention

Currently, they refresh a report page manually. The data is always stale, and by the time they spot a problem, customers are already complaining. They need a dashboard that updates in real-time as orders flow through the system.

## Why Traditional Approaches Fall Short

### AJAX Polling

```javascript
// Poll every 10 seconds
setInterval(async () => {
  const stats = await fetch('/api/order-stats');
  updateDashboard(stats);
}, 10000);
```

**Problems:**
- Every browser tab hitting the API every 10 seconds scales poorly
- Data is always up to 10 seconds stale
- Wasted requests when nothing has changed
- Server load increases linearly with users

### WebSocket with Custom Backend

**Problems:**
- Need to build a service that watches database changes
- Maintain client connection state
- Aggregate data for each connected client
- Handle reconnection, backpressure, and edge cases
- Significant development and maintenance effort

### Traditional CDC

**Problems:**
- Debezium gives you row-level change events
- You need another layer to aggregate these into dashboard metrics
- Joining multiple tables requires stateful processing
- Complexity compounds with each dashboard widget

## Enter Drasi

With Drasi, each dashboard widget is a continuous query. The SSE reaction pushes updates to all connected browsers simultaneously. When an order changes status, all affected metrics update instantly.

```cypher
-- Order counts by status
MATCH (o:orders)
WITH o.status AS status, count(o) AS orderCount, sum(o.total) AS totalValue
RETURN status, orderCount, totalValue

-- Stuck orders
MATCH (o:orders)
WHERE o.status IN ['processing', 'packing']
  AND drasi.trueFor(o.status = o.status, duration({hours: 2}))
RETURN o.id, o.status, o.customer_name, drasi.changeDateTime(o) AS stuckSince
```

One Drasi Server. Multiple queries. Real-time updates to every connected dashboard.

## What You'll Build

- PostgreSQL database with orders and payments
- Multiple Drasi queries for different dashboard views
- SSE reaction streaming to browsers
- HTML/JavaScript dashboard with live updates

## Prerequisites

- Docker and Docker Compose
- A web browser
- Completed the previous tutorials

## Step 1: Create the Project

```bash
mkdir drasi-order-dashboard
cd drasi-order-dashboard
mkdir -p config scripts public
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

  # Simple web server for the dashboard
  web:
    image: nginx:alpine
    ports:
      - "3000:80"
    volumes:
      - ./public:/usr/share/nginx/html:ro
```

## Step 3: Create the Database Schema

Create `scripts/init.sql`:

```sql
-- Customers
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL
);

-- Orders with status tracking
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    customer_name VARCHAR(100) NOT NULL,
    status VARCHAR(30) DEFAULT 'pending',
    total DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Payment tracking
CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    status VARCHAR(30) DEFAULT 'pending',
    amount DECIMAL(10,2) NOT NULL,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Trigger to update updated_at
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_updated
    BEFORE UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION update_timestamp();

CREATE TRIGGER payments_updated
    BEFORE UPDATE ON payments
    FOR EACH ROW EXECUTE FUNCTION update_timestamp();

-- Create publication
CREATE PUBLICATION drasi_pub FOR ALL TABLES;

-- Sample customers
INSERT INTO customers (name, email) VALUES
    ('Alice Johnson', 'alice@example.com'),
    ('Bob Smith', 'bob@example.com'),
    ('Carol Williams', 'carol@example.com'),
    ('David Brown', 'david@example.com'),
    ('Eve Davis', 'eve@example.com');

-- Sample orders in various states
INSERT INTO orders (customer_id, customer_name, status, total, created_at) VALUES
    (1, 'Alice Johnson', 'pending', 129.99, NOW() - INTERVAL '10 minutes'),
    (2, 'Bob Smith', 'processing', 249.99, NOW() - INTERVAL '30 minutes'),
    (3, 'Carol Williams', 'processing', 89.99, NOW() - INTERVAL '3 hours'),
    (4, 'David Brown', 'packing', 199.99, NOW() - INTERVAL '1 hour'),
    (5, 'Eve Davis', 'shipped', 159.99, NOW() - INTERVAL '2 hours'),
    (1, 'Alice Johnson', 'delivered', 79.99, NOW() - INTERVAL '1 day'),
    (2, 'Bob Smith', 'delivered', 299.99, NOW() - INTERVAL '2 days');

-- Sample payments
INSERT INTO payments (order_id, status, amount) VALUES
    (1, 'pending', 129.99),
    (2, 'completed', 249.99),
    (3, 'completed', 89.99),
    (4, 'completed', 199.99),
    (5, 'completed', 159.99),
    (6, 'completed', 79.99),
    (7, 'completed', 299.99);
```

## Step 4: Configure Drasi

Create `config/server.yaml`:

```yaml
id: order-dashboard
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
      - public.payments
    slot_name: drasi_dashboard_slot
    publication_name: drasi_pub
    bootstrap_provider:
      type: postgres

queries:
  # Aggregated order stats by status
  - id: order-stats
    query: |
      MATCH (o:orders)
      WITH o.status AS status,
           count(o) AS orderCount,
           sum(o.total) AS totalValue
      RETURN status, orderCount, totalValue
    sources:
      - source_id: ecommerce-db
        nodes: [orders]
    auto_start: true

  # Orders stuck in processing/packing for over 1 hour (demo: 1 min)
  - id: stuck-orders
    query: |
      MATCH (o:orders)
      WHERE o.status IN ['processing', 'packing']
        AND drasi.trueFor(o.status = o.status, duration({minutes: 1}))
      RETURN
        o.id AS orderId,
        o.customer_name AS customerName,
        o.status AS status,
        o.total AS total,
        drasi.changeDateTime(o) AS lastUpdate
    sources:
      - source_id: ecommerce-db
        nodes: [orders]
    auto_start: true

  # Failed or pending payments
  - id: payment-issues
    query: |
      MATCH (p:payments)-[:FOR_ORDER]->(o:orders)
      WHERE p.status IN ['pending', 'failed']
      RETURN
        o.id AS orderId,
        o.customer_name AS customerName,
        p.status AS paymentStatus,
        p.amount AS amount,
        p.error_message AS errorMessage
    sources:
      - source_id: ecommerce-db
        nodes: [payments, orders]
    joins:
      - id: FOR_ORDER
        keys:
          - label: payments
            property: order_id
          - label: orders
            property: id
    auto_start: true

  # Recent orders (last 10)
  - id: recent-orders
    query: |
      MATCH (o:orders)
      RETURN
        o.id AS orderId,
        o.customer_name AS customerName,
        o.status AS status,
        o.total AS total,
        o.created_at AS createdAt
      ORDER BY o.created_at DESC
      LIMIT 10
    sources:
      - source_id: ecommerce-db
        nodes: [orders]
    auto_start: true

reactions:
  # SSE stream for dashboard
  - kind: sse
    id: dashboard-stream
    queries: [order-stats, stuck-orders, payment-issues, recent-orders]
    host: 0.0.0.0
    port: 8081
    sse_path: /events
    heartbeat_interval_ms: 30000
    auto_start: true

  # Console logging
  - kind: log
    id: dashboard-log
    queries: [stuck-orders, payment-issues]
    routes:
      stuck-orders:
        added:
          template: "[STUCK] Order #{{after.orderId}} for {{after.customerName}} stuck in {{after.status}}"
      payment-issues:
        added:
          template: "[PAYMENT] Order #{{after.orderId}} has {{after.paymentStatus}} payment"
    auto_start: true
```

## Step 5: Create the Dashboard

Create `public/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Order Operations Dashboard</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background: #0f172a;
            color: #e2e8f0;
            padding: 20px;
            min-height: 100vh;
        }
        .header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 24px;
        }
        h1 { color: #38bdf8; font-size: 24px; }
        .status-badge {
            padding: 6px 12px;
            border-radius: 20px;
            font-size: 12px;
            font-weight: 600;
        }
        .status-badge.connected { background: #22c55e; color: #000; }
        .status-badge.disconnected { background: #ef4444; }

        .grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
            gap: 20px;
            margin-bottom: 24px;
        }

        .card {
            background: #1e293b;
            border-radius: 12px;
            padding: 20px;
            border: 1px solid #334155;
        }
        .card h2 {
            color: #94a3b8;
            font-size: 12px;
            text-transform: uppercase;
            letter-spacing: 0.5px;
            margin-bottom: 16px;
        }

        .stat-grid {
            display: grid;
            grid-template-columns: repeat(2, 1fr);
            gap: 12px;
        }
        .stat {
            background: #0f172a;
            padding: 16px;
            border-radius: 8px;
            text-align: center;
        }
        .stat-value {
            font-size: 28px;
            font-weight: 700;
            color: #f8fafc;
        }
        .stat-label {
            font-size: 11px;
            color: #64748b;
            text-transform: uppercase;
            margin-top: 4px;
        }
        .stat-revenue {
            font-size: 14px;
            color: #22c55e;
            margin-top: 4px;
        }

        .stat.pending .stat-value { color: #fbbf24; }
        .stat.processing .stat-value { color: #38bdf8; }
        .stat.packing .stat-value { color: #a78bfa; }
        .stat.shipped .stat-value { color: #22c55e; }

        .alert-list { max-height: 200px; overflow-y: auto; }
        .alert-item {
            background: #0f172a;
            padding: 12px;
            border-radius: 8px;
            margin-bottom: 8px;
            border-left: 3px solid;
        }
        .alert-item.stuck { border-color: #f97316; }
        .alert-item.payment { border-color: #ef4444; }
        .alert-item .title { font-weight: 600; margin-bottom: 4px; }
        .alert-item .detail { font-size: 13px; color: #94a3b8; }
        .no-alerts {
            text-align: center;
            padding: 20px;
            color: #22c55e;
        }

        .order-list { max-height: 300px; overflow-y: auto; }
        .order-item {
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 12px;
            background: #0f172a;
            border-radius: 8px;
            margin-bottom: 8px;
        }
        .order-item .info { flex: 1; }
        .order-item .order-id { font-weight: 600; color: #f8fafc; }
        .order-item .customer { font-size: 13px; color: #94a3b8; }
        .order-item .amount { font-weight: 600; color: #22c55e; }
        .order-status {
            padding: 4px 10px;
            border-radius: 12px;
            font-size: 11px;
            font-weight: 600;
            text-transform: uppercase;
        }
        .order-status.pending { background: #fbbf24; color: #000; }
        .order-status.processing { background: #38bdf8; color: #000; }
        .order-status.packing { background: #a78bfa; color: #000; }
        .order-status.shipped { background: #22c55e; color: #000; }
        .order-status.delivered { background: #6b7280; color: #fff; }

        .full-width { grid-column: 1 / -1; }

        .event-log {
            max-height: 150px;
            overflow-y: auto;
            font-family: monospace;
            font-size: 11px;
            background: #0f172a;
            padding: 12px;
            border-radius: 8px;
        }
        .event { padding: 4px 0; border-bottom: 1px solid #1e293b; }
        .event-time { color: #64748b; }
        .event-type { margin-left: 8px; }
        .event-type.add { color: #22c55e; }
        .event-type.update { color: #fbbf24; }
        .event-type.delete { color: #ef4444; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Order Operations Dashboard</h1>
        <span id="connection-status" class="status-badge disconnected">Disconnected</span>
    </div>

    <div class="grid">
        <div class="card">
            <h2>Orders by Status</h2>
            <div id="order-stats" class="stat-grid">
                <div class="stat pending">
                    <div class="stat-value" id="pending-count">-</div>
                    <div class="stat-label">Pending</div>
                    <div class="stat-revenue" id="pending-value">$0</div>
                </div>
                <div class="stat processing">
                    <div class="stat-value" id="processing-count">-</div>
                    <div class="stat-label">Processing</div>
                    <div class="stat-revenue" id="processing-value">$0</div>
                </div>
                <div class="stat packing">
                    <div class="stat-value" id="packing-count">-</div>
                    <div class="stat-label">Packing</div>
                    <div class="stat-revenue" id="packing-value">$0</div>
                </div>
                <div class="stat shipped">
                    <div class="stat-value" id="shipped-count">-</div>
                    <div class="stat-label">Shipped</div>
                    <div class="stat-revenue" id="shipped-value">$0</div>
                </div>
            </div>
        </div>

        <div class="card">
            <h2>Stuck Orders</h2>
            <div id="stuck-orders" class="alert-list">
                <div class="no-alerts">No stuck orders</div>
            </div>
        </div>

        <div class="card">
            <h2>Payment Issues</h2>
            <div id="payment-issues" class="alert-list">
                <div class="no-alerts">No payment issues</div>
            </div>
        </div>

        <div class="card full-width">
            <h2>Recent Orders</h2>
            <div id="recent-orders" class="order-list">
                Loading...
            </div>
        </div>

        <div class="card full-width">
            <h2>Live Event Stream</h2>
            <div id="event-log" class="event-log"></div>
        </div>
    </div>

    <script>
        // State
        const state = {
            stats: {},
            stuckOrders: new Map(),
            paymentIssues: new Map(),
            recentOrders: new Map()
        };

        // Format currency
        const formatCurrency = (val) => '$' + (parseFloat(val) || 0).toFixed(2);

        // Update stats display
        function updateStatsUI() {
            const statuses = ['pending', 'processing', 'packing', 'shipped'];
            statuses.forEach(status => {
                const data = state.stats[status] || { orderCount: 0, totalValue: 0 };
                document.getElementById(`${status}-count`).textContent = data.orderCount;
                document.getElementById(`${status}-value`).textContent = formatCurrency(data.totalValue);
            });
        }

        // Update stuck orders display
        function updateStuckOrdersUI() {
            const container = document.getElementById('stuck-orders');
            const orders = Array.from(state.stuckOrders.values());

            if (orders.length === 0) {
                container.innerHTML = '<div class="no-alerts">No stuck orders</div>';
                return;
            }

            container.innerHTML = orders.map(o => `
                <div class="alert-item stuck">
                    <div class="title">Order #${o.orderId} - ${o.customerName}</div>
                    <div class="detail">Stuck in ${o.status} | $${o.total}</div>
                </div>
            `).join('');
        }

        // Update payment issues display
        function updatePaymentIssuesUI() {
            const container = document.getElementById('payment-issues');
            const issues = Array.from(state.paymentIssues.values());

            if (issues.length === 0) {
                container.innerHTML = '<div class="no-alerts">No payment issues</div>';
                return;
            }

            container.innerHTML = issues.map(p => `
                <div class="alert-item payment">
                    <div class="title">Order #${p.orderId} - ${p.customerName}</div>
                    <div class="detail">${p.paymentStatus} | ${formatCurrency(p.amount)}</div>
                </div>
            `).join('');
        }

        // Update recent orders display
        function updateRecentOrdersUI() {
            const container = document.getElementById('recent-orders');
            const orders = Array.from(state.recentOrders.values())
                .sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt))
                .slice(0, 10);

            if (orders.length === 0) {
                container.innerHTML = '<div class="no-alerts">No orders yet</div>';
                return;
            }

            container.innerHTML = orders.map(o => `
                <div class="order-item">
                    <div class="info">
                        <div class="order-id">Order #${o.orderId}</div>
                        <div class="customer">${o.customerName}</div>
                    </div>
                    <span class="order-status ${o.status}">${o.status}</span>
                    <div class="amount">${formatCurrency(o.total)}</div>
                </div>
            `).join('');
        }

        // Add event to log
        function logEvent(type, queryId, data) {
            const log = document.getElementById('event-log');
            const time = new Date().toLocaleTimeString();
            const event = document.createElement('div');
            event.className = 'event';
            event.innerHTML = `
                <span class="event-time">${time}</span>
                <span class="event-type ${type}">[${type.toUpperCase()}]</span>
                <span>${queryId}</span>
            `;
            log.insertBefore(event, log.firstChild);
            while (log.children.length > 50) log.removeChild(log.lastChild);
        }

        // Process SSE event
        function processEvent(event) {
            const data = JSON.parse(event.data);
            const queryId = data.queryId || data.query_id;

            // Process added items
            if (data.added) {
                data.added.forEach(item => {
                    const record = item.after || item;
                    handleAdd(queryId, record);
                    logEvent('add', queryId, record);
                });
            }

            // Process updated items
            if (data.updated) {
                data.updated.forEach(item => {
                    const record = item.after || item;
                    handleAdd(queryId, record);  // Same as add for our UI
                    logEvent('update', queryId, record);
                });
            }

            // Process deleted items
            if (data.deleted) {
                data.deleted.forEach(item => {
                    const record = item.before || item;
                    handleDelete(queryId, record);
                    logEvent('delete', queryId, record);
                });
            }
        }

        function handleAdd(queryId, record) {
            switch (queryId) {
                case 'order-stats':
                    state.stats[record.status] = record;
                    updateStatsUI();
                    break;
                case 'stuck-orders':
                    state.stuckOrders.set(record.orderId, record);
                    updateStuckOrdersUI();
                    break;
                case 'payment-issues':
                    state.paymentIssues.set(record.orderId, record);
                    updatePaymentIssuesUI();
                    break;
                case 'recent-orders':
                    state.recentOrders.set(record.orderId, record);
                    updateRecentOrdersUI();
                    break;
            }
        }

        function handleDelete(queryId, record) {
            switch (queryId) {
                case 'order-stats':
                    delete state.stats[record.status];
                    updateStatsUI();
                    break;
                case 'stuck-orders':
                    state.stuckOrders.delete(record.orderId);
                    updateStuckOrdersUI();
                    break;
                case 'payment-issues':
                    state.paymentIssues.delete(record.orderId);
                    updatePaymentIssuesUI();
                    break;
                case 'recent-orders':
                    state.recentOrders.delete(record.orderId);
                    updateRecentOrdersUI();
                    break;
            }
        }

        // Load initial state
        async function loadInitialState() {
            const queries = ['order-stats', 'stuck-orders', 'payment-issues', 'recent-orders'];

            for (const query of queries) {
                try {
                    const response = await fetch(`http://localhost:8080/api/v1/queries/${query}/results`);
                    const data = await response.json();
                    if (data.success && data.data) {
                        const results = Array.isArray(data.data) ? data.data : [];
                        results.forEach(record => handleAdd(query, record));
                    }
                } catch (e) {
                    console.error(`Failed to load ${query}:`, e);
                }
            }
        }

        // Connect to SSE
        function connect() {
            const statusEl = document.getElementById('connection-status');
            const eventSource = new EventSource('http://localhost:8081/events');

            eventSource.onopen = () => {
                statusEl.textContent = 'Connected';
                statusEl.className = 'status-badge connected';
            };

            eventSource.onerror = () => {
                statusEl.textContent = 'Disconnected';
                statusEl.className = 'status-badge disconnected';
                eventSource.close();
                setTimeout(connect, 3000);
            };

            eventSource.onmessage = (event) => {
                try {
                    processEvent(event);
                } catch (e) {
                    console.error('Event processing error:', e);
                }
            };
        }

        // Initialize
        loadInitialState().then(() => connect());
    </script>
</body>
</html>
```

## Step 6: Start the Environment

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

## Step 7: Open the Dashboard

Open your browser to: **http://localhost:3000**

You should see the dashboard with:
- Order counts and revenue by status
- Any stuck orders (orders in processing/packing for over 1 minute in demo mode)
- Payment issues (if any)
- Recent orders list
- Live event stream

## Step 8: Make Changes and Watch Updates

Connect to PostgreSQL:

```bash
docker compose exec postgres psql -U postgres -d ecommerce
```

Try these changes and watch the dashboard update in real-time:

```sql
-- Create a new order
INSERT INTO orders (customer_id, customer_name, status, total)
VALUES (1, 'Alice Johnson', 'pending', 449.99);

-- Watch the pending count increase!

-- Move an order through the pipeline
UPDATE orders SET status = 'processing' WHERE id = 1;

-- Watch pending decrease, processing increase!

-- Ship an order
UPDATE orders SET status = 'shipped' WHERE id = 4;

-- Create a payment failure
INSERT INTO payments (order_id, status, amount, error_message)
VALUES (1, 'failed', 449.99, 'Card declined');

-- Watch the payment issues panel update!

-- Fix the payment
UPDATE payments SET status = 'completed', error_message = NULL
WHERE order_id = 1;

-- Watch the payment issue disappear!
```

## Step 9: Test Stuck Order Detection

Create an order and leave it in processing:

```sql
INSERT INTO orders (customer_id, customer_name, status, total, created_at)
VALUES (3, 'Carol Williams', 'processing', 199.99, NOW());
```

Wait 1 minute (our demo threshold). Watch the "Stuck Orders" panel. The order will appear automatically when it crosses the threshold.

Move the order forward to clear it:

```sql
UPDATE orders SET status = 'packing' WHERE customer_name = 'Carol Williams' AND status = 'processing';
```

The stuck order alert clears, but now the order is in packing. Wait another minute and it will appear as stuck again.

## What You Learned

1. **Multiple coordinated queries**: Different dashboard widgets use different queries, but they all share the same source and react to changes together.

2. **Real-time aggregations**: `count()` and `sum()` work continuously. When an order moves from pending to processing, both stats update instantly.

3. **Time-based alerts with `drasi.trueFor()`**: Detect when orders are stuck without polling or maintaining external state.

4. **SSE for multi-query dashboards**: A single SSE connection delivers updates for all queries. The frontend routes events to the right widget.

5. **Initial state + streaming**: Load current state via REST API, then subscribe to SSE for updates. This pattern ensures you never miss events.

## Production Considerations

### Scaling to Multiple Dashboard Users

- SSE connections are lightweight; Drasi Server can handle many concurrent connections
- Consider a CDN or SSE proxy for very high user counts
- Use connection pooling in the browser (most browsers limit concurrent connections)

### Dashboard Authentication

- Add authentication to the SSE endpoint
- Use token-based auth passed as a query parameter or header
- Consider API gateway integration for enterprise environments

### Error Handling

- Implement exponential backoff for reconnection
- Show a "last updated" timestamp so users know data freshness
- Consider a "refresh all" button for manual recovery

### Performance Optimization

- Use the profiler reaction to monitor query performance
- For high-volume systems, consider separate queries with filters rather than one large query
- Tune SSE heartbeat interval based on your network requirements

## Cleanup

```bash
docker compose down -v
cd ..
rm -rf drasi-order-dashboard
```

## Congratulations!

You've completed all three Drasi Server tutorials. You've learned how to:

1. **SLA Breach Alerting**: Use `drasi.trueFor()` for time-based conditions and multi-table joins
2. **Smart Reorder Recommendations**: Build complex multi-table queries with aggregations and computed fields
3. **Live Order Dashboard**: Coordinate multiple queries and build real-time UIs with SSE

## Next Steps

- Explore the [How-to Guides](/drasi-server/how-to-guides/) for specific configuration tasks
- Read about [Source configuration](/drasi-server/how-to-guides/configuration/configure-sources/) for other databases
- Learn about [Production deployment](/drasi-server/how-to-guides/operations/) patterns
