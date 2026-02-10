---
type: "docs"
title: "Setup: Build from Source"
linkTitle: "Build from Source"
weight: 40
description: "Build Drasi Server from source code"
---

Build Drasi Server from source. This approach is ideal for contributors or if you want to modify the code.

## Prerequisites

- **Git** — [Install Git](https://git-scm.com/downloads)
- **Docker** — Required for the tutorial database ([Install Docker Desktop](https://www.docker.com/products/docker-desktop/))
- **Rust 1.88+** — For building Drasi Server ([Install via rustup](https://rustup.rs/))

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

### Verify Rust is Installed

```bash
rustc --version   # Should be 1.88.0 or later
cargo --version
```

## Step 1: Clone and Build

Clone the repository and build Drasi Server:

```bash
git clone https://github.com/drasi-project/drasi-server.git
cd drasi-server
cargo install --path . --root . --locked
```
This takes several minutes on first build.

The `--root .` flag tells Cargo to install the Drasi Server binary to `./bin/drasi-server` in the current directory, which is where the tutorial expects it.

## Step 2: Verify the Build

Verify the binary works:

```bash
./bin/drasi-server --version
```

You should see output showing the version number, for example:

```
drasi-server 0.1.0
```

---

## ✅ Environment Setup Complete!

You now have Drasi Server accessible at `./bin/drasi-server` from the repository root.

<p><a href="../#database" class="btn btn-success btn-lg">Continue with the Tutorial <i class="fas fa-arrow-right ms-2"></i></a></p>