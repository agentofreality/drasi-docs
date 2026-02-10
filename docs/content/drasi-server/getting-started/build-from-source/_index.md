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

Verify installations:

```bash
docker ps
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

## Step 2: Verify the Build

```bash
./target/release/drasi-server --version
```

## Step 3: Start the Tutorial Database

Navigate to the examples directory and start PostgreSQL:

```bash
cd examples/getting-started
docker compose -f database/docker-compose.yml up -d
```

Verify it's running:

```bash
docker compose -f database/docker-compose.yml ps
```

You should see the `getting-started-postgres` container running.

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
