---
type: "docs"
title: "Getting Started"
linkTitle: "Getting Started"
weight: 5
no_list: true
hide_readingtime: true
description: "Build your first change-driven solution with Drasi Server"
---

This tutorial will help you get Drasi Server up and running and show you how to create {{< term "Source" "Sources" >}}, {{< term "Continuous Query" "Continuous Queries" >}}, and {{< term "Reaction" "Reactions" >}}.

After completing this tutorial (~20-30 minutes), you will have built a complete change-driven solution that captures changes from a PostgreSQL database and reacts to them in real-time.

## What You'll Build

You'll create a message tracking system that monitors a PostgreSQL database in real-time. The solution includes:

- A **PostgreSQL Source** that captures database changes via CDC (Change Data Capture)
- Three **Continuous Queries** that filter, aggregate, and detect time-based patterns
- **Reactions** that stream results to your browser and console

As you add, update, and delete messages in the database, you'll see the query results update automatically in real-time.

## Choose How You Want to Get Started With Drasi Server

There are multiple ways to set up Drasi Server. Each approach will guide you through the same hands-on tutorial, but you can choose the method that best fits your preferences and environment.

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
  <a href="build-from-source/">
    <div class="unified-card unified-card--tutorials">
      <div class="unified-card-icon"><i class="fas fa-hammer"></i></div>
      <div class="unified-card-content">
        <h3 class="unified-card-title">Build from Source</h3>
        <p class="unified-card-summary">Clone and build Drasi Server yourself. Ideal for contributors.</p>
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
</div>