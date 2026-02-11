---
type: "docs"
title: "Getting Started"
linkTitle: "Getting Started"
weight: 5
no_list: true
hide_readingtime: true
description: "Build your first change-driven solution with Drasi Server"
---

This Getting Started tutorial teaches you how to use Drasi Server by demonstrating how to create {{< term "Source" "Sources" >}}, {{< term "Continuous Query" "Continuous Queries" >}}, and {{< term "Reaction" "Reactions" >}}. You'll use the `drasi-server init` command to create an initial simple configuration file, then progressively extend the configuration to explore more Drasi Server functionality. 

**Time**: ~30 minutes

**What you'll learn**:
- How to use `drasi-server init` to scaffold your initial configuration, then iteratively edit, validate, and extend it
- How to create Sources that connect Drasi to PostgreSQL and HTTP data sources
- How to write Continuous Queries that detect specific changes, track aggregations, identify time-based data patterns, and join data across multiple disparate Sources
- How to configure Reactions that distribute notifications of query result changes to downstream systems

## Step 1: Set Up Your Environment {#setup}

Choose your preferred environment for working through the Getting Started tutorial. Each approach gets you to the same starting point with Drasi Server installed and ready to run the tutorial.

<div class="card-grid">
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
  <a href="download-binary/">
    <div class="unified-card unified-card--tutorials">
      <div class="unified-card-icon"><i class="fas fa-download"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Download Binary</h3>
        <p class="unified-card-summary">Download a prebuilt binary for macOS or Linux. The fastest way to get started.</p>
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

The tutorial uses a PostgreSQL database as a data source. Start the database container using Docker Compose:

```bash
docker compose -f examples/getting-started/database/docker-compose.yml up -d
```

Verify the database container is running:

```bash
docker compose -f examples/getting-started/database/docker-compose.yml ps
```

You should see the `getting-started-postgres` container with a status of `Up`:

```text
NAME                       IMAGE                COMMAND                  SERVICE    CREATED          STATUS                    PORTS
getting-started-postgres   postgres:14-alpine   "docker-entrypoint.s…"   postgres   31 seconds ago   Up 30 seconds (healthy)   0.0.0.0:5432->5432/tcp
```

If the container shows a different status or you see errors, check the container logs with `docker compose -f examples/getting-started/database/docker-compose.yml logs`. See the [Docker Compose documentation](https://docs.docker.com/compose/) for additional troubleshooting help.

### Initialize the Database

Once the container is up, initialize the database schema and sample data.

The tutorial uses a simple `Message` table with the following schema:

| Field | Type | Description |
|-------|------|-------------|
| MessageId | integer | Unique message identifier |
| From | varchar(50) | Who sent the message |
| Message | varchar(200) | The message content |
| CreatedAt | timestamp | When the message was sent |

<div style="margin-top: 1.5rem;"></div>

The `Message` table is initially populated with these messages:

| MessageId | From | Message |
|-----------|------|---------|
| 1 | Buzz Lightyear | To infinity and beyond! |
| 2 | Brian Kernighan | Hello World |
| 3 | Antoninus | I am Spartacus |
| 4 | David | I am Spartacus |

<div style="margin-top: 1.5rem;"></div>

Run the database initialization script:

```bash
docker exec -i getting-started-postgres psql -U postgres -d getting_started < examples/getting-started/database/init.sql
```

You should see:

```text
NOTICE:  Getting Started database initialized successfully!
NOTICE:  Tables: Message
NOTICE:  Publication: drasi_pub
NOTICE:  Replication slot: drasi_slot
```

Verify the sample data was loaded:

```bash
docker exec getting-started-postgres psql -U drasi_user -d getting_started -c 'SELECT * FROM "Message";'
```

You should see the 4 sample messages:

```text
 MessageId |      From       |         Message          |         CreatedAt          
-----------+-----------------+--------------------------+----------------------------
         1 | Buzz Lightyear  | To infinity and beyond!  | 2026-02-10 21:30:08.123456
         2 | Brian Kernighan | Hello World              | 2026-02-10 21:30:08.123456
         3 | Antoninus       | I am Spartacus           | 2026-02-10 21:30:08.123456
         4 | David           | I am Spartacus           | 2026-02-10 21:30:08.123456
(4 rows)
```

---

## Step 3: Create Your First Configuration {#phase-1}

Now you'll create your initial Drasi Server configuration using the interactive `drasi-server init` command.

The `init` command walks you through an interactive wizard that will assist you in creating a correctly formatted Drasi Server config file. The wizard will only write the config file at the end, so if you make a mistake just break out of the wizard using `ctrl-c` and run `drasi-server init` again. 


> **Note:** The `init` command cannot be used to edit existing config files, you must edit them in your preferred text editor.


### Create the Drasi Server Configuration

From the tutorial root folder, run the following command:

```bash
./bin/drasi-server init --output getting-started.yaml
```

Here's what to enter at each prompt:

#### 1. Server Settings

Configuration starts with general Drasi Server settings.

| Prompt | Enter | Notes |
|--------|-------|-------|
| **Server host** | `0.0.0.0` (default) | Press Enter to accept |
| **Server port** | `8080` (default) | Press Enter to accept |
| **Log level** | `info` | Use arrow keys to select |
| **Enable persistent indexing (RocksDB)?** | `No` (default) | Press Enter to accept |
| **State store** | `None` | Use arrow keys to select "None - In-memory state" |

<div style="margin-top: 1.5rem;"></div>

Your terminal should show this once you have completed the Server Settings section of the wizard:

```text
Server Settings
---------------
> Server host: 0.0.0.0
> Server port: 8080
> Log level: info
> Enable persistent indexing (RocksDB)? No
> State store (for plugin state persistence): None - In-memory state (lost on restart)
```

#### 2. Data Sources

After configuring server settings, you'll add a data source. For this tutorial, use the arrow keys to highlight **PostgreSQL**, press Space to select the source, then Enter.

After selecting PostgreSQL, you'll configure the database connection settings:

| Prompt | Enter | Notes |
|--------|-------|-------|
| **Source ID** | `my-postgres` | A unique name for this source |
| **Database host** | `${DB_HOST:-localhost}` | Uses env var with localhost as default (see note below) |
| **Database port** | `5432` (default) | Press Enter to accept |
| **Database name** | `getting_started` | The tutorial database |
| **Database user** | `drasi_user` | |
| **Database password** | `drasi_password` | Type the password (characters won't display) and press Enter |
| **Tables to monitor** | `Message` | The table we'll query |
| **Configure table keys for tables without primary keys??** | `Yes` | Required for CDC change tracking |
| **Does table 'Message' need key columns specified?** | `Yes` | Need to configure tableKey for `Message` table |
| **Key columns for 'Message'** | `MessageId` | The Message table's primary key |
| **Bootstrap provider** | `PostgreSQL` | Use arrow keys to select "PostgreSQL - Load initial data" |

> **Why `${DB_HOST:-localhost}`?** This uses Drasi Server's environment variable interpolation. When running locally (Download Binary or Build from Source), it defaults to `localhost`. In a Dev Container or Codespace, the `DB_HOST` environment variable is automatically set to `getting-started-postgres` (the container name on the shared Docker network).


<div style="margin-top: 1.5rem;"></div>

Your terminal should show this once you complete the Data Source section of the wizard:

```text
Data Sources
------------
Select one or more data sources for your configuration.

> Select sources (space to select, enter to confirm): PostgreSQL - CDC from PostgreSQL database

Configuring PostgreSQL Source
------------------------------
> Source ID: my-postgres
> Database host: ${DB_HOST:-localhost}
> Database port: 5432
> Database name: getting_started
> Database user: drasi_user
> Database password: ********
> Tables to monitor (comma-separated): Message
> Configure table keys for tables without primary keys? Yes
> Does table 'Message' need key columns specified? Yes
> Key columns for 'Message' (comma-separated): MessageId
> Bootstrap provider (for initial data loading): PostgreSQL - Load initial data from PostgreSQL
```

#### 3. Reactions

Finally, you will add a Reaction to process changes to the Continuous Query results. 

Use the arrow keys to highlight **Log**, press Space to select the Reaction, then Enter.

After selecting Log, you'll configure the following settings:

| Prompt | Enter | Notes |
|--------|-------|-------|
| **Reaction ID** | `log-reaction` (default) | Press Enter to accept |
 
After completing the Reactions section of the wizard, your terminal will show the following:

```text
Reactions
---------
Select how you want to receive query results.

> Select reactions (space to select, enter to confirm): Log - Write query results to console

Configuring Log Reaction
------------------------
> Reaction ID: log-reaction


Configuration saved to: getting-started.yaml

Next steps:
  1. Review and edit getting-started.yaml as needed
  2. Run: drasi-server --config getting-started.yaml
```

### Update the Default Continuous Query

The wizard created a default Continuous Query that selects all nodes from the `my-postgres` Source. Now you'll edit the Continuous Query to select only `message` nodes and to rename some of their fields for clarity.

Open `getting-started.yaml` in your preferred editor and find the `queries` section. The wizard's default Continuous Query looks like this:

```yaml
queries:
- id: my-query
  autoStart: true
  query: MATCH (n) RETURN n
  queryLanguage: GQL
  middleware: []
  sources:
  - sourceId: my-postgres
    nodes: []
    relations: []
    pipeline: []
  enableBootstrap: true
  bootstrapBufferSize: 10000
```

Replace the `id` and `query` settings as shown here:

```yaml
queries:
  - id: all-messages
    autoStart: true
    query: |
      MATCH (m:Message)
      RETURN m.MessageId AS MessageId, m.From AS From, m.Message AS Message
    queryLanguage: GQL
    ...
```

The `|` character allows you to write the query across multiple lines for readability. The `Message` label in the `Match` clause must match the table name exactly (labels are case-sensitive). Leave the other fields (`queryLanguage`, `sources`, etc.) as they are.

### Update the Log Reaction
Because you changed the Continuous Query's `id` from `my-query` to `all-messages`, you need to update the Log Reaction's configuration to subscribe to the new Continuous Query ID.

Find the `reactions` section in your config file and update the `queries` field to reference the new query ID as shown here:

```yaml
reactions:
  - kind: log
    id: log-reaction
    queries:
      - all-messages    # Update this from my-query to all-messages
    autoStart: true
```

### Run Drasi Server

```bash
./bin/drasi-server --config getting-started.yaml
```

You'll see detailed startup logs as Drasi Server initializes all configured Sources, Continuous Queries, and Reactions. There's a lot of output, so look for these key lines:

```
Starting Drasi Server
  Config file: getting-started.yaml
  API Port: 8080
  Log level: info
```

This shows the name of the config file being used, the log level that controls the output to the console, and the port on which the Drasi Server management API is accessible.

```
[log-reaction] Started - receiving results from queries: ["all-messages"]
```

This confirms that the `log-reaction` Reaction is subscribed to Query Result Change notifications from the `all-messages` Continuous Query.

```
Drasi Server started successfully with API on port 8080
```

Shortly after, the bootstrap process loads the initial data from the `Messages` table and the Log Reaction outputs the 4 messages representing additions to the `all-messages` query's result set:

```
[log-reaction] Query 'all-messages' (1 items):
[log-reaction]   [ADD] {"From":"Buzz Lightyear","Message":"To infinity and beyond!","MessageId":"1"}
[log-reaction] Query 'all-messages' (1 items):
[log-reaction]   [ADD] {"From":"Brian Kernighan","Message":"Hello World","MessageId":"2"}
[log-reaction] Query 'all-messages' (1 items):
[log-reaction]   [ADD] {"From":"Antoninus","Message":"I am Spartacus","MessageId":"3"}
[log-reaction] Query 'all-messages' (1 items):
[log-reaction]   [ADD] {"From":"David","Message":"I am Spartacus","MessageId":"4"}
```

> **Note:** The messages may appear in a different order and may be interleaved with other log lines. This is normal — bootstrapped data is processed asynchronously.

> **Tip:** You can customize the log output format using templates. See [Configure Log Reaction](../how-to-guides/configuration/configure-reactions/configure-log-reaction/) for details.

### Test Real-Time Changes

Open a **new terminal** and run the following command to insert a record into the `message` table:

```bash
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO \"Message\" (\"From\", \"Message\") VALUES ('You', 'My first message!');"
```

Watch the Drasi Server console — notification of a change to the `all-messages` Continuous Query result appears instantly thanks to the Log Reaction!

```text
[log-reaction] Query 'all-messages' (1 items):
[log-reaction]   [ADD] {"n":"Element(Node { metadata: ElementMetadata { reference: ElementReference { source_id: \"my-postgres\", element_id: \"Message:5\" }, labels: [\"Message\"], effective_from: 1770778392526 }, properties: ElementPropertyMap { values: {\"CreatedAt\": String(\"2026-02-11 02:53:12.522474\"), \"From\": String(\"You\"), \"Message\": String(\"My first message!\"), \"MessageId\": Integer(5)} } })"}
```

### View Continuous Query Results

Drasi Server provides a [REST API](../reference/rest-api/) through which you can view the current result set of any Continuous Query. Click the following URL to view the current result set of the `all-messages` Continuous Query in your browser:

<a href="http://localhost:8080/api/v1/queries/all-messages/results" target="_blank">http://localhost:8080/api/v1/queries/all-messages/results</a>

Or use `curl` from your second terminal:

```bash
curl -s http://localhost:8080/api/v1/queries/all-messages/results
```

Either way, you should see the current result set for the `all-messages` query, including the message you just inserted:

```json
[
    {
        "From": "Buzz Lightyear",
        "Message": "To infinity and beyond!",
        "MessageId": "1"
    },
    {
        "From": "Brian Kernighan",
        "Message": "Hello World",
        "MessageId": "2"
    },
    {
        "From": "Antoninus",
        "Message": "I am Spartacus",
        "MessageId": "3"
    },
    {
        "From": "David",
        "Message": "I am Spartacus",
        "MessageId": "4"
    },
    {
        "From": "You",
        "Message": "My first message!",
        "MessageId": "5"
    }
]
```

> **Tip:** The Drasi Server REST API also provides a Swagger UI at **http://localhost:8080/api/v1/docs/** where you can explore all available endpoints interactively.

<div style="margin-top: 1.5rem;"></div>

**✅ Checkpoint**: You've created your first Source, Continuous Query, and Reaction. Changes in the database flow through Drasi Server and notification of data changes appear in the console instantly. And you can view the current state of the Continuous Query's result set at any time through the Drasi Server REST API.

---

## Step 4: Add a Query with Criteria {#phase-2}

Now you'll edit your configuration to add a query that only includes specific data in its result set.

### Edit Your Config

Stop the running Drasi Server (`Ctrl+C`). 

Open `getting-started.yaml` in your editor and add a second query. Find the `queries:` section and add:

```yaml
queries:
  # ... your existing query ...
  
  - id: hello-world-senders
    autoStart: true
    sources:
      - sourceId: my-postgres
    query: |
      MATCH (m:Message)
      WHERE m.Message = 'Hello World'
      RETURN m.MessageId AS Id, m.From AS Sender
    queryLanguage: Cypher
```

Notice that the new `hello-world-senders` Continuous Query defines the same `my-postgres` Source used by the original `all-messages` Continuous Query meaning both Continuous Queries share the same Source. Also the `hello-world-senders` Continuous Query has the setting `queryLanguage: Cypher`; Drasi Server supports Continuous Queries written in both GQL and openCypher.

Then update the `reactions:` section to configure `log-reaction` to also subscribe to the new `hello-world-senders` query:

```yaml
reactions:
  - kind: log
    id: log-reaction
    queries:
      - all-messages
      - hello-world-senders  # Add the ID of the new query
    autoStart: true
```

### Validate Your Drasi Server Config 

Drasi Server provides a `validate` command that checks your configuration for errors before you run it. This is especially helpful as you add more components to your config file. 

Validate your configuration file with the following command:

```bash
./bin/drasi-server validate --config getting-started.yaml
```

If there are no errors, you will see:

```text
Validating configuration: getting-started.yaml

[OK] Configuration file is valid

Summary:
  Instances: 1
  Sources: 1
  Queries: 2
  Reactions: 1
```

### Test the hello-world-senders Query

Start Drasi Server with your updated configuration:

```bash
./bin/drasi-server --config getting-started.yaml
```

Now you have two queries running. The new `hello-world-senders` query only shows Brian Kernighan (the one who sent "Hello World").

### Test the Query Criteria

```bash
# This WILL appear in hello-world-senders (meets the WHERE criteria)
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO \"Message\" (\"From\", \"Message\") VALUES ('Alice', 'Hello World');"
```

Watch the console:
```
[all-messages] + {"MessageId":6,"From":"Alice","Message":"Hello World"}
[hello-world-senders] + {"Id":6,"Sender":"Alice"}
```

Now try a message that doesn't match the criteria:

```bash
# This will NOT appear in hello-world-senders (different message text)
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO \"Message\" (\"From\", \"Message\") VALUES ('Bob', 'Goodbye World');"
```

The console shows:
```
[all-messages] + {"MessageId":7,"From":"Bob","Message":"Goodbye World"}
```

Notice that `hello-world-senders` didn't produce any output — the new message doesn't meet the `WHERE` criteria, so it isn't part of the query's result set. The Reaction is only notified of changes to the result set.

**✅ Checkpoint**: You understand how Cypher `WHERE` clauses define which data is meaningful to a Continuous Query. Changes that don't affect the query's result set don't generate notifications.

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
      MATCH (m:Message)
      RETURN m.Message AS MessageText, count(m) AS Count
    queryLanguage: Cypher
```

Update the `reactions:` section to include the new query:

```yaml
reactions:
  - kind: log
    id: log-reaction
    queries:
      - all-messages
      - hello-world-senders
      - message-counts  # Add the new query
    autoStart: true
```

### Validate and Run

```bash
./bin/drasi-server validate --config getting-started.yaml
./bin/drasi-server --config getting-started.yaml
```

You'll see the aggregated counts in the initial output:
```
[message-counts] + {"MessageText":"To infinity and beyond!","Count":1}
[message-counts] + {"MessageText":"Hello World","Count":2}
[message-counts] + {"MessageText":"I am Spartacus","Count":2}
[message-counts] + {"MessageText":"My first message!","Count":1}
[message-counts] + {"MessageText":"Goodbye World","Count":1}
```

### Test Aggregation Updates

```bash
docker exec -it getting-started-postgres psql -U drasi_user -d getting_started -c \
  "INSERT INTO \"Message\" (\"From\", \"Message\") VALUES ('Eve', 'Hello World');"
```

Watch the count update automatically:
```
[message-counts] - {"MessageText":"Hello World","Count":2}
[message-counts] + {"MessageText":"Hello World","Count":3}
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
      MATCH (m:Message)
      WITH m.From AS Sender, max(m.CreatedAt) AS LastSeen
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
      - all-messages
      - hello-world-senders
      - message-counts
      - inactive-senders  # Add the new query
    autoStart: true
```

### Validate and Run

```bash
./bin/drasi-server validate --config getting-started.yaml
./bin/drasi-server --config getting-started.yaml
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
  "INSERT INTO \"Message\" (\"From\", \"Message\") VALUES ('Alice', 'Still here!');"
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
./bin/drasi-server validate --config getting-started.yaml
./bin/drasi-server --config getting-started.yaml
```

### View in Browser

Open **http://localhost:8081** in your browser. You'll see query result set changes appearing as JSON in real time.

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
  "INSERT INTO \"Message\" (\"From\", \"Message\") VALUES ('Charlie', 'Hello World');"
```

**✅ Checkpoint**: You understand that queries can feed multiple reactions, and reactions can output to different destinations (console, browser, webhooks, etc.).

---

## Step 8: Add Cross-Source Joins {#phase-6}

So far you've used a single PostgreSQL source. Now you'll add an HTTP source and join data across both sources.

**Scenario**: Track where message senders are currently located and their availability status. Imagine the HTTP source receives location updates from a mobile app or badge system.

### Add an HTTP Source

Edit `getting-started.yaml` and add a second source:

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
      MATCH (m:Message)-[:FROM_USER]->(u:UserLocation)
      RETURN m.MessageId AS Id, m.Message AS Message, 
             m.From AS Sender, u.location AS Location, u.status AS Status
    queryLanguage: Cypher
    joins:
      - id: FROM_USER
        keys:
          - label: Message
            property: From
          - label: UserLocation
            property: name
```

The `joins` section creates a virtual relationship `FROM_USER` that connects `Message.From` to `UserLocation.name`.

Update the `log-reaction` to include the new query:

```yaml
reactions:
  - kind: log
    id: log-reaction
    queries:
      - all-messages
      - hello-world-senders
      - message-counts
      - inactive-senders
      - messages-with-location  # Add the new query
    autoStart: true
```

### Validate and Run

```bash
./bin/drasi-server validate --config getting-started.yaml
./bin/drasi-server --config getting-started.yaml
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
  "INSERT INTO \"Message\" (\"From\", \"Message\") VALUES ('Alice', 'Good morning!');"

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
| **Sources** | Created PostgreSQL and HTTP sources to connect Drasi to different data systems |
| **Queries** | Wrote 5 Continuous Queries: simple change detection, criteria-based selection, aggregation, time-based detection, and cross-source joins |
| **Reactions** | Configured Log and SSE reactions to distribute notifications of query result changes to console and browser |
| **Joins** | Connected data across sources using virtual relationships |
| **Configuration** | Used `drasi-server init` to scaffold, then manually edited to add components |
| **Validation** | Used `drasi-server validate` to catch errors before running |

The core Drasi pattern: **Sources connect to data → Continuous Queries detect changes → Reactions distribute notifications of result set changes**.

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