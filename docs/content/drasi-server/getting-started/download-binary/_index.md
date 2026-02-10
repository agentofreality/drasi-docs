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

### Verify Docker is Running

```bash
docker ps
```

If Docker is running, you'll see a table with these headings showing running containers (even if no containers are running):

```
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

If you see an error like `Cannot connect to the Docker daemon`, Docker isn't running. Start Docker Desktop and wait for it to fully initialize, then try again. If problems persist, see the [Docker troubleshooting guide](https://docs.docker.com/desktop/troubleshoot-and-support/troubleshoot/) for additional help.

## Step 1: Clone the Repository

Clone the Drasi Server repository to get the tutorial files:

```bash
git clone https://github.com/drasi-project/drasi-server.git
cd drasi-server
```

## Step 2: Download Drasi Server

Download the binary for your platform into the repository root:

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

---

## ✅ Setup Complete!

You now have Drasi Server accessible at `./drasi-server` from the tutorial folder.

<div class="card-grid">
  <a href="../#database">
    <div class="unified-card unified-card--tutorials">
      <div class="unified-card-icon"><i class="fas fa-arrow-right"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Continue with the Tutorial</h3>
        <p class="unified-card-summary">Create your first change-driven solution.</p>
      </div>
    </div>
  </a>
</div>
