---
type: "docs"
title: "Setup: Dev Container"
linkTitle: "Dev Container"
weight: 30
description: "Run Drasi Server in a VS Code Dev Container"
---

Use VS Code Dev Containers for a consistent development environment with all dependencies pre-installed.

## Prerequisites

- **Git** — [Install Git](https://git-scm.com/downloads)
- **Docker Desktop** — [Install Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker Engine on Linux)
- **VS Code** — [Install Visual Studio Code](https://code.visualstudio.com/)
- **Dev Containers extension** — Install from the VS Code Extensions marketplace

### Recommended Docker Resources

For optimal performance, allocate to Docker:
- **CPU**: 4+ cores
- **Memory**: 8+ GB

### Verify Docker is Running

```bash
docker ps
```

If Docker is running, you'll see a table header (even if no containers are running):

```
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

If you see an error like `Cannot connect to the Docker daemon`, Docker isn't running. Start Docker Desktop and wait for it to fully initialize, then try again. If problems persist, see the [Docker troubleshooting guide](https://docs.docker.com/desktop/troubleshoot-and-support/troubleshoot/) for additional help.

## Step 1: Clone Drasi Server Repo and Open in Dev Container

Clone the Drasi Server repository:

```bash
git clone https://github.com/drasi-project/drasi-server.git
```

Open in the repo folder in VS Code (for example):

```bash
cd drasi-server
code .
```

When VS Code opens, you'll see a notification:
> **Folder contains a Dev Container configuration file. Reopen folder to develop in a container.**

Click **Reopen in Container**.

{{< alert title="Manually Open Dev Container" color="info" >}}
If you don't see the notification described, press `F1` and type "Dev Containers: Reopen in Container".
{{< /alert >}}

## Step 2: Wait for Setup

The container takes several minutes to build on first run. The setup script will:
1. Install PostgreSQL client
2. Build Drasi Server in release mode

Watch the terminal for: **"Drasi Server development environment is ready!"**

## Step 3: Verify the Build

Once the setup completes, verify that Drasi Server was built successfully by checking its version:

```bash
./target/release/drasi-server --version
```

You should see output showing the version number, for example:

```
drasi-server 0.1.0
```

If you see a "file not found" error, the build may not have completed. Check the terminal output for errors and try rebuilding the container.

## Step 4: Start the Tutorial Database

In the VS Code terminal run the following commands to start the PostgreSQL database using Docker Compose:

```bash
cd examples/getting-started
docker compose -f database/docker-compose.yml up -d
```

Verify it's running:

```bash
docker compose -f database/docker-compose.yml ps
```

You should see the `getting-started-postgres` container with a status of `running`:

```
NAME                       IMAGE         COMMAND                  SERVICE    CREATED          STATUS          PORTS
getting-started-postgres   postgres:16   "docker-entrypoint.s…"   postgres   10 seconds ago   Up 9 seconds    0.0.0.0:5432->5432/tcp
```

If the container shows a different status or you see errors, check the container logs with `docker compose -f database/docker-compose.yml logs`. See the [Docker Compose documentation](https://docs.docker.com/compose/how-tos/troubleshoot/) for additional troubleshooting help.

## Step 5: Return to Repository Root

Return to the repository root directory before continuing with the tutorial:

```bash
cd ../..
```

---

## ✅ Setup Complete!

You now have:
- Drasi Server built at `./target/release/drasi-server`
- Tutorial database running with sample data

<div class="card-grid">
  <a href="../#phase-1">
    <div class="unified-card unified-card--tutorials">
      <div class="unified-card-icon"><i class="fas fa-arrow-right"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Continue with the Tutorial</h3>
        <p class="unified-card-summary">Create your first change-driven solution.</p>
      </div>
    </div>
  </a>
</div>

---

## Dev Container Tips

### Port Forwarding

The Dev Container automatically forwards ports to your local machine. Check the **Ports** tab in VS Code to access:
- Port 8080 (Drasi Server API)
- Port 8081 (SSE stream)

### Rebuild the Container

If you change `.devcontainer/devcontainer.json`:
1. Press `F1`
2. Select "Dev Containers: Rebuild Container"

### Exit the Dev Container

To return to local development:
1. Press `F1`
2. Select "Dev Containers: Reopen Folder Locally"
