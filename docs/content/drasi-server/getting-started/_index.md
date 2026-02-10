---
type: "docs"
title: "Getting Started"
linkTitle: "Getting Started"
weight: 5
no_list: true
hide_readingtime: true
description: "Build your first change-driven solution with Drasi Server"
---

This Getting Started tutorial teaches you how to use Drasi Server by demonstrating how to create {{< term "Source" "Sources" >}}, {{< term "Continuous Query" "Continuous Queries" >}}, and {{< term "Reaction" "Reactions" >}}. You'll use the `drasi-server init` command to create your initial configuration, then progressively add queries and reactions to learn each concept.

**Time**: ~30 minutes

**What you'll learn**:
- How to create a Source that connects to PostgreSQL
- How to write Continuous Queries that filter, aggregate, and detect patterns over time
- How to configure Reactions that output to console and browser

## Step 1: Set Up Your Environment {#setup}

Choose your preferred environment for working through the Getting Started tutorial. Each approach gets you to the same starting point: Drasi Server ready to run and a PostgreSQL database setup to use as a data source in the tutorial.

<div class="card-grid">
  <a href="download-binary/">
    <div class="unified-card unified-card--tutorials">
      <div class="unified-card-icon"><i class="fas fa-download"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Download Binary</h3>
        <p class="unified-card-summary">Download a prebuilt binary for macOS or Linux. The fastest way to get started.</p>
      </div>
    </div>
  </a>
  <a href="github-codespace/">
    <div class="unified-card unified-card--tutorials">
      <div class="unified-card-icon"><i class="fab fa-github"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">GitHub Codespace</h3>
        <p class="unified-card-summary">One-click cloud environment. No local installation needed.</p>
      </div>
    </div>
  </a>
  <a href="dev-container/">
    <div class="unified-card unified-card--tutorials">
      <div class="unified-card-icon"><i class="fas fa-cube"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Dev Container</h3>
        <p class="unified-card-summary">VS Code Dev Container with all dependencies preconfigured.</p>
      </div>
    </div>
  </a>
  <a href="build-from-source/">
    <div class="unified-card unified-card--tutorials">
      <div class="unified-card-icon"><i class="fas fa-hammer"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Build from Source</h3>
        <p class="unified-card-summary">Clone and build Drasi Server yourself. Ideal for contributors.</p>
      </div>
    </div>
  </a>
</div>

<div style="margin-top: 2rem;"></div>

After completing your preferred setup, return here to continue with the tutorial.

---

## Step 2: Setup the Tutorial Database {#database}

The tutorial uses a PostgreSQL database. Start the database container using Docker Compose:

```bash
docker compose -f examples/getting-started/database/docker-compose.yml up -d
```

Verify the database container is running:

```bash
docker compose -f examples/getting-started/database/docker-compose.yml ps
```

You should see the `getting-started-postgres` container with a status of `running`:

```
NAME                       IMAGE         COMMAND                  SERVICE    CREATED          STATUS          PORTS
getting-started-postgres   postgres:16   "docker-entrypoint.s…"   postgres   10 seconds ago   Up 9 seconds    0.0.0.0:5432->5432/tcp
```

If the container shows a different status or you see errors, check the container logs with `docker compose -f examples/getting-started/database/docker-compose.yml logs`. See the [Docker Compose documentation](https://docs.docker.com/compose/how-tos/troubleshoot/) for additional troubleshooting help.

### Initialize the Database

Once the container is running, initialize the database schema and sample data.

The tutorial uses a simple `Message` table with the following schema:

| Field | Type | Description |
|-------|------|-------------|
| MessageId | integer | Unique message identifier |
| From | varchar(50) | Who sent the message |
| Message | varchar(200) | The message content |
| created_at | timestamp | When the message was sent |

<div style="margin-top: 1.5rem;"></div>

The `Message` table is initially populated with these messages:

| MessageId | From | Message |
|-----------|------|---------|
| 1 | Buzz Lightyear | To infinity and beyond! |
| 2 | Brian Kernighan | Hello World |
| 3 | Antoninus | I am Spartacus |
| 4 | David | I am Spartacus |

<div style="margin-top: 1.5rem;"></div>

Copy the `init.sql` script into the container:

```bash
docker cp examples/getting-started/database/init.sql getting-started-postgres:/tmp/
```

Run the initialization script inside the container:

```bash
docker exec getting-started-postgres psql -U postgres -d getting_started -f /tmp/init.sql
```

You should see output ending with:
```
NOTICE:  Getting Started database initialized successfully!
NOTICE:  Tables: message
NOTICE:  Publication: drasi_pub
NOTICE:  Replication slot: drasi_slot
```

---

## Step 3: Create Your First Configuration {#phase-1}

Now you'll create your own Drasi Server configuration using the interactive `drasi-server init` command.
 
### Create the Drasi Server Configuration

Make sure you're in the tutorial root folder and run the following command:

```bash
./bin/drasi-server init --output my-config.yaml
```

The `init` command walks you through an interactive wizard that will assist you in creating a correctly formatted Drasi Server config file. Here's what to enter at each prompt:

#### 1. Server Settings

Configuration starts with general Drasi Server settings.

| Prompt | Enter | Notes |
|--------|-------|-------|
| **Server host** | `0.0.0.0` (default) | Press Enter to accept |
| **Server port** | `8080` (default) | Press Enter to accept |
| **Log level** | `info` | Use arrow keys to select |
| **Enable persistent indexing (RocksDB)?** | `No` (default) | Press Enter to accept |
| **State store** | `None` | Use arrow keys to select "None - In-memory state" |

#### 2. Data Sources

After configuring server settings, you'll add a data source. For this tutorial, use the arrow keys to highlight **PostgreSQL**, press Space to select the source, then Enter.

After selecting PostgreSQL, you'll configure the database connection settings:

| Prompt | Enter | Notes |
|--------|-------|-------|
| **Source ID** | `my-postgres` | A unique name for this source |
| **Database host** | `getting-started-postgres` | The PostgreSQL container name |
| **Database port** | `5432` (default) | Press Enter to accept |
| **Database name** | `getting_started` | The tutorial database |
| **Database user** | `drasi_user` | |
| **Database password** | `drasi_password` | Type the password (characters won't display) and press Enter |
| **Tables to monitor** | `message` | The table we'll query |
| **Configure table keys for tables without primary keys??** | `Yes` | Required for CDC change tracking |
| **Does table 'message' need key columns specified?** | `Yes` | Need to configure tableKey for `message` table |
| **Key columns for 'message'** | `messageid` | The message table's primary key |
| **Bootstrap provider** | `PostgreSQL` | Use arrow keys to select "PostgreSQL - Load initial data" |

#### 3. Reactions

Finally, add a Reaction to see query results. Use the arrow keys to highlight **Log**, press Space to select the Reaction, then Enter.

After selecting Log, you'll configure the following settings:

| Prompt | Enter | Notes |
|--------|-------|-------|
| **Reaction ID** | `log-reaction` (default) | Press Enter to accept |

After completing the wizard, you'll see the following output:

```
Configuration saved to: my-config.yaml

Next steps:
  1. Review and edit my-config.yaml as needed
  2. Run: drasi-server --config my-config.yaml
```

Before running Drasi Server, you need to update the default query to return messages from our tutorial database.

### Update the Query

Open `my-config.yaml` in your editor and find the `queries` section. The wizard created a default query that looks like this:

```yaml
queries:
  - id: my-query
    autoStart: true
    query: MATCH (n) RETURN n
    ...
```

Replace the entire `query:` line with a multi-line query that returns messages:

```yaml
queries:
  - id: my-query
    autoStart: true
    query: |
      MATCH (m:message)
      RETURN m.messageid AS messageid, m.from AS from, m.message AS message
    queryLanguage: GQL
    ...
```

The `|` character allows you to write the query across multiple lines for readability. The label `message` must match the table name exactly (labels are case-sensitive). Leave the other fields (`queryLanguage`, `sources`, etc.) as they are.

### Run Drasi Server

```bash
./bin/drasi-server --config my-config.yaml
```

You should see startup logs followed by the initial query results:

```
[INFO  drasi_server] Starting Drasi Server
[INFO  drasi_server::server] Drasi Server started successfully with API on port 8080
[my-query] + {"messageid":1,"from":"Buzz Lightyear","message":"To infinity and beyond!"}
[my-query] + {"messageid":2,"from":"Brian Kernighan","message":"Hello World"}
[my-query] + {"messageid":3,"from":"Antoninus","message":"I am Spartacus"}
[my-query] + {"messageid":4,"from":"David","message":"I am Spartacus"}
```

### Test Real-Time Changes

Open a **new terminal** and insert a message:

```bash
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO message (\"from\", message) VALUES ('You', 'My first message!');"
```

Watch the Drasi Server console — your message appears instantly!

```
[my-query] + {"messageid":5,"from":"You","message":"My first message!"}
```

**✅ Checkpoint**: You've created your first Source, Query, and Reaction. Changes in the database flow through Drasi and appear in the console in real-time.

---

## Step 4: Add a Filtered Query {#phase-2}

Now you'll edit your configuration to add a query that filters the change stream.

### Edit Your Config

Open `my-config.yaml` in your editor and add a second query. Find the `queries:` section and add:

```yaml
queries:
  # ... your existing query ...
  
  - id: hello-world-senders
    autoStart: true
    sources:
      - sourceId: my-postgres  # Use the source ID you created
    query: |
      MATCH (m:message)
      WHERE m.message = 'Hello World'
      RETURN m.messageid AS Id, m.from AS Sender
    queryLanguage: Cypher
```

Then update the `reactions:` section to subscribe to the new query:

```yaml
reactions:
  - kind: log
    id: log-reaction
    queries:
      - my-query
      - hello-world-senders  # Add the new query
    autoStart: true
```

### Validate Your Changes

Before running, validate the configuration to catch any errors:

```bash
./bin/drasi-server validate --config my-config.yaml
```

If there are errors (typos, invalid syntax), you'll see helpful messages. Fix them before proceeding.

### Restart and Observe

Stop the running server (`Ctrl+C`) and restart:

```bash
./bin/drasi-server --config my-config.yaml
```

Now you have two queries running. The new `hello-world-senders` query only shows Brian Kernighan (the one who sent "Hello World").

### Test Filtering

```bash
# This WILL appear in hello-world-senders (matches the filter)
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO message (\"from\", message) VALUES ('Alice', 'Hello World');"
```

Watch the console:
```
[my-query] + {"messageid":6,"from":"Alice","message":"Hello World"}
[hello-world-senders] + {"Id":6,"Sender":"Alice"}
```

Now try a message that doesn't match:

```bash
# This will NOT appear in hello-world-senders (different message)
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO message (\"from\", message) VALUES ('Bob', 'Goodbye World');"
```

The console shows:
```
[my-query] + {"messageid":7,"from":"Bob","message":"Goodbye World"}
```

Notice that `hello-world-senders` didn't output anything — the `WHERE` clause filtered it out.

**✅ Checkpoint**: You understand how Cypher `WHERE` clauses filter which changes trigger reactions. Only matching changes produce output.

---

## Step 5: Add an Aggregation Query {#phase-3}

Drasi maintains state, so you can run aggregations that update automatically as data changes.

### Edit Your Config

Add a query that counts how many times each message has been sent:

```yaml
queries:
  # ... existing queries ...
  
  - id: message-counts
    autoStart: true
    sources:
      - sourceId: my-postgres
    query: |
      MATCH (m:message)
      RETURN m.message AS Message, count(m) AS Count
    queryLanguage: Cypher
```

Update the `reactions:` section to include the new query:

```yaml
reactions:
  - kind: log
    id: log-reaction
    queries:
      - my-query
      - hello-world-senders
      - message-counts  # Add the new query
    autoStart: true
```

### Validate and Run

```bash
./bin/drasi-server validate --config my-config.yaml
./bin/drasi-server --config my-config.yaml
```

You'll see the aggregated counts in the initial output:
```
[message-counts] + {"Message":"To infinity and beyond!","Count":1}
[message-counts] + {"Message":"Hello World","Count":2}
[message-counts] + {"Message":"I am Spartacus","Count":2}
[message-counts] + {"Message":"My first message!","Count":1}
[message-counts] + {"Message":"Goodbye World","Count":1}
```

### Test Aggregation Updates

```bash
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO message (\"from\", message) VALUES ('Eve', 'Hello World');"
```

Watch the count update automatically:
```
[message-counts] - {"Message":"Hello World","Count":2}
[message-counts] + {"Message":"Hello World","Count":3}
```

The `-` shows the old value being removed, and `+` shows the new value. The count for "Hello World" incremented from 2 to 3.

**✅ Checkpoint**: You understand that Drasi tracks state — aggregations update in real-time as data changes.

---

## Step 6: Add Time-Based Detection {#phase-4}

Drasi can detect patterns over time, including the *absence* of activity.

### Edit Your Config

Add a query that identifies senders who haven't sent a message in the last 20 seconds:

```yaml
queries:
  # ... existing queries ...
  
  - id: inactive-senders
    autoStart: true
    sources:
      - sourceId: my-postgres
    query: |
      MATCH (m:message)
      WITH m.from AS Sender, max(m.created_at) AS LastSeen
      WHERE LastSeen < datetime() - duration('PT20S')
      RETURN Sender, LastSeen
    queryLanguage: Cypher
```

Update the `reactions:` section to include the new query:

```yaml
reactions:
  - kind: log
    id: log-reaction
    queries:
      - my-query
      - hello-world-senders
      - message-counts
      - inactive-senders  # Add the new query
    autoStart: true
```

### Validate and Run

```bash
./bin/drasi-server validate --config my-config.yaml
./bin/drasi-server --config my-config.yaml
```

### Wait and Observe

After about 20 seconds of inactivity, senders will start appearing in the `inactive-senders` output:

```
[inactive-senders] + {"Sender":"Buzz Lightyear","LastSeen":"2024-..."}
[inactive-senders] + {"Sender":"Brian Kernighan","LastSeen":"2024-..."}
```

### Reactivate a Sender

```bash
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO message (\"from\", message) VALUES ('Alice', 'Still here!');"
```

Watch Alice disappear from the inactive list (she just sent a message). After 20 more seconds of inactivity, she'll reappear.

**✅ Checkpoint**: You understand that Drasi can detect the *absence* of activity over time — a powerful capability for monitoring and alerting.

---

## Step 7: Add a Browser Reaction {#phase-5}

So far you've used the Log reaction. Now add an SSE (Server-Sent Events) reaction to view results in a browser.

### Edit Your Config

Add an SSE reaction:

```yaml
reactions:
  # ... existing log reaction ...
  
  - kind: sse
    id: browser-stream
    autoStart: true
    queries:
      - hello-world-senders
      - message-counts
      - inactive-senders
    host: 0.0.0.0
    port: 8081
    ssePath: /events
```

### Validate and Run

```bash
./bin/drasi-server validate --config my-config.yaml
./bin/drasi-server --config my-config.yaml
```

### View in Browser

Open **http://localhost:8081** in your browser. You'll see a live stream of query results as JSON.

For a friendlier view, you can use the SSE viewer from the examples:

```bash
cd examples/getting-started
python3 -m http.server 8000 --directory viewer &
```

Then open **http://localhost:8000** to see a dashboard with all three queries.

### Test Live Updates

Insert more messages and watch the browser update in real-time — no page refresh needed!

```bash
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO message (\"from\", message) VALUES ('Charlie', 'Hello World');"
```

**✅ Checkpoint**: You understand that queries can feed multiple reactions, and reactions can output to different destinations (console, browser, webhooks, etc.).

---

## Step 8: Add Cross-Source Joins {#phase-6}

So far you've used a single PostgreSQL source. Now you'll add an HTTP source and join data across both sources.

**Scenario**: Track where message senders are currently located and their availability status. Imagine the HTTP source receives location updates from a mobile app or badge system.

### Add an HTTP Source

Edit `my-config.yaml` and add a second source:

```yaml
sources:
  # ... existing postgres source ...
  
  - kind: http
    id: location-tracker
    autoStart: true
    host: 0.0.0.0
    port: 9000
    bootstrapProvider:
      kind: script
      path: examples/getting-started/locations.json
```

The `bootstrapProvider` loads initial location data from a JSON file on startup.

### Add a Join Query

Add a query that joins messages with user locations:

```yaml
queries:
  # ... existing queries ...
  
  - id: messages-with-location
    autoStart: true
    sources:
      - sourceId: my-postgres
      - sourceId: location-tracker
    query: |
      MATCH (m:message)-[:FROM_USER]->(u:UserLocation)
      RETURN m.messageid AS Id, m.message AS Message, 
             m.from AS Sender, u.location AS Location, u.status AS Status
    queryLanguage: Cypher
    joins:
      - id: FROM_USER
        keys:
          - label: message
            property: from
          - label: UserLocation
            property: name
```

The `joins` section creates a virtual relationship `FROM_USER` that connects `message.from` to `UserLocation.name`.

Update the `log-reaction` to include the new query:

```yaml
reactions:
  - kind: log
    id: log-reaction
    queries:
      - my-query
      - hello-world-senders
      - message-counts
      - inactive-senders
      - messages-with-location  # Add the new query
    autoStart: true
```

### Validate and Run

```bash
./bin/drasi-server validate --config my-config.yaml
./bin/drasi-server --config my-config.yaml
```

On startup, the bootstrap loads location data for 4 users. You'll see the joined output:

```
[messages-with-location] + {"Id":2,"Message":"Hello World","Sender":"Brian Kernighan","Location":"Building A, Floor 3","Status":"online"}
[messages-with-location] + {"Id":1,"Message":"To infinity and beyond!","Sender":"Buzz Lightyear","Location":"Space Station","Status":"away"}
```

### Update Location in Real-Time

Simulate Brian moving to a new location by sending an update to the HTTP source:

```bash
curl -X POST http://localhost:9000 -H "Content-Type: application/json" -d '{
  "op": "update", "label": "UserLocation", "id": "brian",
  "properties": {"name": "Brian Kernighan", "location": "Conference Room B", "status": "away"}
}'
```

Watch the console — Brian's messages now show the new location:

```
[messages-with-location] - {"Id":2,"Message":"Hello World","Sender":"Brian Kernighan","Location":"Building A, Floor 3","Status":"online"}
[messages-with-location] + {"Id":2,"Message":"Hello World","Sender":"Brian Kernighan","Location":"Conference Room B","Status":"away"}
```

### Add a New User Location

When a new user starts sending messages, add their location:

```bash
# First, send a message from a new user
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO message (\"from\", message) VALUES ('Alice', 'Good morning!');"

# Then add Alice's location
curl -X POST http://localhost:9000 -H "Content-Type: application/json" -d '{
  "op": "insert", "label": "UserLocation", "id": "alice",
  "properties": {"name": "Alice", "location": "Home Office", "status": "online"}
}'
```

Alice's message appears in the joined query once her location is added.

**✅ Checkpoint**: You understand how to join data across multiple sources using virtual relationships. Changes to either source propagate through the join in real-time.

---

## What You've Learned {#summary}

You built a complete change-driven solution from scratch:

| Concept | What You Did |
|---------|-------------|
| **Sources** | Created PostgreSQL and HTTP sources to capture changes from different systems |
| **Queries** | Wrote 5 Continuous Queries: simple retrieval, filtering, aggregation, time-based detection, and cross-source joins |
| **Reactions** | Configured Log and SSE reactions to output to console and browser |
| **Joins** | Connected data across sources using virtual relationships |
| **Configuration** | Used `drasi-server init` to scaffold, then manually edited to add components |
| **Validation** | Used `drasi-server validate` to catch errors before running |

The core Drasi pattern: **Sources feed data → Queries detect changes → Reactions act on them**.

---

## Cleanup {#cleanup}

Stop Drasi Server with `Ctrl+C`.

Stop the tutorial database:

```bash
docker compose -f examples/getting-started/database/docker-compose.yml down -v
```

The `-v` flag removes the persistent volume. Without it, the database data persists after the container is removed and can cause confusion if you restart the tutorial later.

---

## Next Steps

<div class="card-grid">
  <a href="/concepts/overview/">
    <div class="unified-card unified-card--concepts">
      <div class="unified-card-icon"><i class="fas fa-lightbulb"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Understand Drasi Concepts</h3>
        <p class="unified-card-summary">Understand how Drasi works under the hood</p>
      </div>
    </div>
  </a>
  <a href="../how-to-guides/configuration/configure-sources/">
    <div class="unified-card unified-card--howto">
      <div class="unified-card-icon"><i class="fas fa-database"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Configure Sources</h3>
        <p class="unified-card-summary">Detect changes in PostgreSQL, HTTP, gRPC, and more</p>
      </div>
    </div>
  </a>
  <a href="../how-to-guides/configuration/configure-queries/">
    <div class="unified-card unified-card--howto">
      <div class="unified-card-icon"><i class="fas fa-search"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Configure Queries</h3>
        <p class="unified-card-summary">Write advanced Continuous Queries in GQL and openCypher</p>
      </div>
    </div>
  </a>
  <a href="../how-to-guides/configuration/configure-reactions/">
    <div class="unified-card unified-card--howto">
      <div class="unified-card-icon"><i class="fas fa-bolt"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Configure Reactions</h3>
        <p class="unified-card-summary">React to changes using SSE, gRPC, Stored Procedures, and more</p>
      </div>
    </div>
  </a>
</div>