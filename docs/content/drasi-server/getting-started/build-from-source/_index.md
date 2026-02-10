---
type: "docs"
title: "Setup: Build from Source"
linkTitle: "Build from Source"
weight: 40
description: "Build Drasi Server from source code"
---

Build Drasi Server from source. This approach is ideal for contributors or if you want to modify the code.

## Prerequisites

- **Docker** — Required for the tutorial database ([Install Docker Desktop](https://www.docker.com/products/docker-desktop/))
- **Git** — For cloning the repository
- **Rust 1.88+** — For building Drasi Server ([Install via rustup](https://rustup.rs/))

### Verify Docker is Running

```bash
docker ps
```

If Docker is running, you'll see a table with these headings showing running containers (even if no containers are running):

```
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

If you see an error like `Cannot connect to the Docker daemon`, Docker isn't running. Start Docker Desktop and wait for it to fully initialize, then try again. If problems persist, see the [Docker troubleshooting guide](https://docs.docker.com/desktop/troubleshoot-and-support/troubleshoot/) for additional help.

### Verify Rust prerequisites

```bash
rustc --version   # Should be 1.88.0 or later
cargo --version
```

## Step 1: Clone and Build

Clone the repository:

```bash
git clone https://github.com/drasi-project/drasi-server.git
cd drasi-server
```

Build in release mode:

```bash
cargo build --release
```

This takes several minutes on first build.

## Step 2: Create Symlink

Create a symlink so the binary is accessible from the repository root:

```bash
ln -sf ./target/release/drasi-server ./drasi-server
```

Verify the symlink works:

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

---

## Development Tips

These commands are useful when contributing to Drasi Server:

```bash
# Rebuild after code changes
cargo build --release

# Run with debug logging
RUST_LOG=debug ./target/release/drasi-server --config my-config.yaml

# Run tests
cargo test
```
