---
type: "docs"
title: "Setup: Download Binary"
linkTitle: "Download Binary"
weight: 10
description: "Download a prebuilt Drasi Server binary for macOS or Linux"
---

This is the fastest way to get started with Drasi Server. Download a prebuilt binary and start the tutorial database.

## Prerequisites

- **Docker** — Required for the tutorial database ([Install Docker Desktop](https://www.docker.com/products/docker-desktop/))
- **curl** and **tar** — For downloading and extracting the binary
- **Git** — For cloning the tutorial files

Verify Docker is running:

```bash
docker ps
```

## Step 1: Download Drasi Server

Download the binary for your platform:

{{< tabpane persist="header" >}}
{{< tab header="macOS (Apple Silicon)" lang="bash" >}}
curl -sL https://github.com/drasi-project/drasi-server/releases/latest/download/drasi-server-darwin-arm64.tar.gz | tar xz
chmod +x drasi-server
{{< /tab >}}
{{< tab header="macOS (Intel)" lang="bash" >}}
curl -sL https://github.com/drasi-project/drasi-server/releases/latest/download/drasi-server-darwin-amd64.tar.gz | tar xz
chmod +x drasi-server
{{< /tab >}}
{{< tab header="Linux (x64)" lang="bash" >}}
curl -sL https://github.com/drasi-project/drasi-server/releases/latest/download/drasi-server-linux-amd64.tar.gz | tar xz
chmod +x drasi-server
{{< /tab >}}
{{< tab header="Linux (ARM64)" lang="bash" >}}
curl -sL https://github.com/drasi-project/drasi-server/releases/latest/download/drasi-server-linux-arm64.tar.gz | tar xz
chmod +x drasi-server
{{< /tab >}}
{{< /tabpane >}}

Verify the download:

```bash
./drasi-server --version
```

## Step 2: Get the Tutorial Files

Clone the Drasi Server repository to get the tutorial database configuration:

```bash
git clone https://github.com/drasi-project/drasi-server.git
cd drasi-server/examples/getting-started
```

## Step 3: Start the Tutorial Database

Start PostgreSQL with Docker Compose:

```bash
docker compose -f database/docker-compose.yml up -d
```

Wait a few seconds, then verify it's running:

```bash
docker compose -f database/docker-compose.yml ps
```

You should see the `getting-started-postgres` container running.

## Step 4: Return to Repository Root

Return to the repository root directory before continuing with the tutorial:

```bash
cd ../..
```

## Step 5: Verify Setup

Test that Drasi Server can run by referencing the path where you downloaded the binary:

```bash
# Adjust the path based on where you downloaded it
~/drasi-server --version
```

---

## ✅ Setup Complete!

You now have:
- Drasi Server binary ready to run
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
