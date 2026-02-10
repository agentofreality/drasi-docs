---
type: "docs"
title: "Setup: GitHub Codespace"
linkTitle: "GitHub Codespace"
weight: 20
description: "Run Drasi Server in a cloud-based GitHub Codespace"
---

Run everything in the cloud with GitHub Codespaces. No local installation required — just a browser and a GitHub account.

## Prerequisites

- A **GitHub account** (free tier works fine)
- A **web browser**

That's it! Docker, Rust, and all dependencies are pre-installed in the Codespace.

## Step 1: Launch the Codespace

Click the button below to create a new Codespace:

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/drasi-project/drasi-server)

Or manually:
1. Go to [github.com/drasi-project/drasi-server](https://github.com/drasi-project/drasi-server)
2. Click the green **Code** button
3. Select the **Codespaces** tab
4. Click **Create codespace on main**

## Step 2: Wait for Setup

The Codespace takes a few minutes to initialize. The setup script will:
1. Install PostgreSQL client
2. Build Drasi Server in release mode

Watch the terminal for: **"Drasi Server development environment is ready!"**

{{< alert title="Build time" color="info" >}}
The first build takes several minutes. Subsequent Codespace sessions are faster if you don't delete the Codespace.
{{< /alert >}}

## Step 3: Verify the Build

```bash
./target/release/drasi-server --version
```

## Step 4: Start the Tutorial Database

```bash
cd examples/getting-started
docker compose -f database/docker-compose.yml up -d
```

Verify it's running:

```bash
docker compose -f database/docker-compose.yml ps
```

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

## Codespace Tips

### Port Forwarding

The Codespace automatically forwards ports. Check the **Ports** tab to access:
- Port 8080 (Drasi Server API)
- Port 8081 (SSE stream)

If you can't connect, right-click the port and select **Port Visibility → Public**.

### Stop the Codespace

To save your free hours:
1. Click **Codespaces** in the bottom-left corner
2. Select **Stop Current Codespace**

Or manage all Codespaces at [github.com/codespaces](https://github.com/codespaces).

### Delete When Done

To free storage quota, delete the Codespace from [github.com/codespaces](https://github.com/codespaces).
