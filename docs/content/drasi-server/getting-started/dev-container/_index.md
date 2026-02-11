---
type: "docs"
title: "Setup: Dev Container"
linkTitle: "Dev Container"
weight: 20
description: "Run Drasi Server in a VS Code Dev Container"
---

Use VS Code Dev Containers for a consistent development environment with all dependencies pre-installed.

## Prerequisites

- **Git** — [Install Git](https://git-scm.com/downloads)
- **Docker Desktop** — [Install Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker Engine on Linux)
  - Recommended resources: 4+ CPU cores, 8+ GB memory
- **VS Code** — [Install Visual Studio Code](https://code.visualstudio.com/)
- **Dev Containers extension** — [Install from VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

### Verify Git is Installed

```bash
git --version
```

You should see output like `git version 2.x.x`. If you see "command not found", install Git from the link above.

### Verify Docker is Running

```bash
docker ps
```

If Docker is running, you'll see a table with these headings showing running containers (even if no containers are running):

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
If you don't see the notification described, press `F1` and type "Dev Containers: Reopen in Container". Then press enter to run the command.
{{< /alert >}}

## Step 2: Wait for Setup

The container takes several minutes to build on first run. The setup script will:
1. Install PostgreSQL client
2. Build and install Drasi Server to `./bin/drasi-server`

Watch the terminal for: **"Drasi Server Getting Started tutorial environment is ready!"**

## Step 3: Verify the Build

Verify that Drasi Server is accessible running the following command in the terminal:

```bash
./bin/drasi-server --version
```

You should see output showing the version number, for example:

```
drasi-server 0.1.0
```

If you see a "file not found" error, the build may not have completed. Check the terminal output for errors and try rebuilding the container.

---

## ✅ Environment Setup Complete!

You now have Drasi Server accessible at `./bin/drasi-server` from the repository root.

<p><a href="../#database" class="btn btn-success btn-lg">Continue with the Tutorial <i class="fas fa-arrow-right ms-2"></i></a></p>

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
