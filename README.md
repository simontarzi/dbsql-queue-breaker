# 📊 DBSQL Queue Breaker

An importable **Databricks AI/BI dashboard** that finds SQL warehouses in *your own* workspace that need more headroom — and tells you **which lever to pull**:

- **Add clusters** (scale *out*) — queries are **queuing at capacity** (the warehouse keeps hitting its max‑cluster ceiling and makes queries wait).
- **Size up** (scale *up*) — queries are **spilling to disk** (they run out of memory, so a bigger t‑shirt size, not more clusters, is the fix).

Both are silent problems today — nothing in the product flags them — and both fixes are cheap and reversible. Everything runs on **your own system tables**: read‑only, per‑workspace, **no data leaves your workspace, no internal Databricks data involved.**

---

## What you get (single page)

- **KPI + knob:** "Warehouses over p95 threshold" with an adjustable **`p95 wait threshold (sec)`** parameter — set it to 60, 300, 30… and the count updates.
- **Queued queries per day** — daily total trend.
- **Warehouses by wait severity (p95)** and **Warehouses by recommended action** bar charts.
- **Where to act** scatter — p95 wait vs. clusters‑to‑add, colored by recommended action.
- **Table 1 — "All warehouses to scale up (add clusters …)":** Workspace · Warehouse · Size/type · **Recommended action** · Min · Max now · Autoscaling · **Scale up to** (clusters) · **+Clusters** · queue metrics.
- **Table 2 — "Warehouses spilling to disk (size‑up candidates)":** Workspace · Warehouse · Current size/type · **Scale up to** (next t‑shirt size) · Spilling queries · % spill · Avg spill (GB).
- **Workspace filter** across the whole dashboard.

## How the recommendations work
- **Queue signal:** `waiting_at_capacity_duration_ms` (queued because the warehouse was at capacity). Deliberately **excludes** `waiting_for_compute_duration_ms` (transient scale‑up latency) so you only see genuine under‑provisioning.
- **Add clusters:** a concurrent‑queue‑depth sweep → **`clusters_to_add = ceil(p95_concurrent_queued / 10)`** (~10 queries per cluster for classic routing), added to the configured `max_clusters`.
- **Size up:** **`spilled_local_bytes > 0`** — the signal Databricks' own warehouse‑sizing docs use ("excessive spill indicates the warehouse size is too small"). A warehouse where ≥10% of queries spill is flagged **Size up**, with the next t‑shirt size suggested.
- **Window:** trailing 30 days.

## Requirements
- Unity Catalog workspace with **system tables enabled**, and the viewer needs `SELECT` (or `USE SCHEMA`) on:
  - **`system.query`** (`system.query.history`) — queue + spill signals.
  - **`system.compute`** (`system.compute.warehouses`) — warehouse name, size, type, min/max clusters.
  - **`system.access`** (`system.access.workspaces_latest`) — workspace names.
- A SQL warehouse to run the dashboard on.

> If any of those schemas isn't granted, the dashboard's queries will error — ask an admin to `GRANT USE SCHEMA` on the relevant `system.*` schema.

## Install
**Option A — Import in the UI (recommended)**
1. Download `dbsql-queue-breaker.lvdash.json`.
2. In Databricks: **Dashboards → Create ▾ → Import dashboard from file**, pick the file.
3. Attach a SQL warehouse and **Publish**.

**Option B — CLI**
```bash
databricks lakeview create \
  --json '{"display_name":"DBSQL Queue Breaker","parent_path":"/Users/<you>",
           "warehouse_id":"<YOUR_WAREHOUSE_ID>",
           "serialized_dashboard": <contents of the .lvdash.json as a string>}'
```

## Notes & disclaimer
- The `ceil(depth / 10)` (clusters) and `≥10% spill` (size‑up) thresholds are **heuristics** — a starting point. Sizing up helps memory‑bound / parallelizable work, not tiny or I/O‑bound queries. Treat every recommendation as a **candidate to confirm** in the SQL warehouse monitoring page before changing settings.
- Provided **as‑is**, no warranty or official support. **Not an official Databricks product.**
