# 📊 DBSQL Queue Breaker

An importable **Databricks AI/BI dashboard** that finds SQL warehouses in *your own* workspace whose queries are **queuing at capacity** — i.e. the warehouse keeps hitting its max‑cluster ceiling and makes queries wait before they even start — and tells you **how much to scale each one up**.

Queuing is avoidable latency your users feel on every dashboard and query, and nothing in the product flags it today. The fix is usually a ~2‑minute, fully reversible change: raise the warehouse's **max clusters** (autoscaling limit), or size it up.

> Runs entirely on **your own `system.query.history`** — read‑only, no data leaves your workspace, no internal Databricks data involved.

---

## What you get

**Page 1 — Overview (current UX)**
- KPIs: warehouses queuing · avg queue wait · queries queued (30d) · worst wait · warehouses with p95 ≥ 5 min · total clusters to add
- Queued queries per day & average queue wait per day (trend)
- Warehouses by wait severity, and a "where to act" scatter (p95 wait vs. clusters to add)

**Page 2 — Warehouses to scale up**
- A ranked table of every queuing warehouse with the concrete recommendation: **Add clusters**, plus avg / p95 / max wait, % queued, and queue days.

## How the recommendation works
- **Queue signal:** `waiting_at_capacity_duration_ms` from `system.query.history` — time a query spent queued because the warehouse was at capacity (all clusters busy). This deliberately **excludes** `waiting_for_compute_duration_ms` (transient scale‑up latency), so you only see *genuine* under‑provisioning.
- **How much to add:** a concurrent‑queue‑depth sweep over the queued intervals → **`clusters_to_add = ceil(p95_concurrent_queued / 10)`** (~10 queries per cluster for classic routing). A tiny warehouse with a large number is a **size‑up** conversation (bump the t‑shirt size), not just more clusters.
- **Window:** trailing 30 days. Warehouse is identified by **ID** — find it under *SQL Warehouses* in the workspace.

## Requirements
- Unity Catalog workspace with **system tables enabled**, specifically the `system.query` schema (query history).
- The dashboard viewer needs `SELECT` on `system.query.history` (or `USE SCHEMA` on `system.query`).
- A SQL warehouse to run the dashboard.

## Install
**Option A — Import in the UI**
1. Download `dbsql-queue-breaker.lvdash.json`.
2. In Databricks: **Dashboards → Create → Import dashboard from file** and pick the file.
3. Attach a SQL warehouse and **Publish**.

**Option B — CLI**
```bash
databricks lakeview create \
  --display-name "DBSQL Queue Breaker" \
  --warehouse-id <YOUR_WAREHOUSE_ID> \
  --json '{"parent_path":"/Users/<you>","serialized_dashboard": <contents of the .lvdash.json as a string>}'
```

## Notes & disclaimer
- The `ceil(depth / 10)` heuristic is a starting point (classic warehouse routing ≈ 10 queries/cluster); confirm the target with your team and the warehouse monitoring page before changing settings.
- Provided **as‑is**, no warranty or official support. Not an official Databricks product.
