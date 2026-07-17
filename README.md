# 📊 DBSQL Queue Breaker

An importable **Databricks AI/BI dashboard** that finds SQL warehouses in *your own* workspace that need more headroom — and tells you **which lever to pull**:

- **Add clusters** (scale *out*) — queries are **queuing at capacity** (the warehouse keeps hitting its max‑cluster ceiling and makes queries wait).
- **Size up** (scale *up*) — queries are **spilling to disk** (they run out of memory, so a bigger t‑shirt size, not more clusters, is the fix).

Both are silent problems today — nothing in the product flags them — and both fixes are cheap and reversible. Everything runs on **your own system tables**: read‑only, per‑workspace, **no data leaves your workspace, no internal Databricks data involved.**

---

## What you get (single page)

- **Percentile selector** — a dashboard‑wide **Percentile (recommendation basis)** dropdown (**Max · p95 · p90 · p75 · p50**, default p95). Drives clusters‑to‑add, the KPI, wait buckets, scatter, and both tables. *Max* sizes for the worst spike (aggressive); *p50* for the typical moment (conservative).
- **Size‑up trigger selector** — a dashboard‑wide **Size‑up trigger** dropdown that changes *when a warehouse is flagged "Size up"*:
  - **Frequency** (default) — ≥10% of queries spilled (count‑based).
  - **Intensity** — total spill ≥50% of bytes read (flags *heavy* spill even if only a few queries spill; node‑capacity‑independent).
  - **Peak volume** — any single query spilled ≥5 GB (one‑off memory‑bound queries).
- **KPI + knob** — "Warehouses over wait threshold" with an adjustable **`Queue-wait threshold (sec)`** parameter (compares each warehouse's queue wait **at the selected percentile**).
- **Queued queries per day** trend, **wait‑severity** and **recommended‑action** bar charts, and a **Where to act** scatter (queue wait at the selected percentile vs. clusters‑to‑add).
- **Table 1 — scale‑up (add clusters):** Workspace · Warehouse · Size/type · Recommended action · Min · Max now · Autoscaling · Scale up to · +Clusters · queued‑at‑once (selected pct **and** max) · queue metrics.
- **Table 2 — disk spill (size‑up candidates):** Workspace · Warehouse · Current size/type · **Size up?** (per the selected trigger) · Scale up to · Spilling queries · % spill · **Avg / Max / Total spill (GB)** · **Spill:read ratio**.
- **Workspace filter**, **percentile selector**, and **size‑up trigger** all apply across the whole dashboard.

## How the recommendations work
- **Queue signal:** `waiting_at_capacity_duration_ms` (queued because the warehouse was at capacity). Deliberately **excludes** `waiting_for_compute_duration_ms` (transient scale‑up latency) so you only see genuine under‑provisioning.
- **Add clusters:** a concurrent‑queue‑depth sweep → **`clusters_to_add = ceil(depth_at_selected_pct / 10)`** (~10 queries per cluster; percentile is the dashboard‑wide selector, default p95), added to the configured `max_clusters`.
- **Size up:** based on **`spilled_local_bytes`** — the signal Databricks' own warehouse‑sizing docs use ("excessive spill indicates the warehouse size is too small"). The **Size‑up trigger** dropdown chooses the rule:
  - *Frequency* — `spill_queries ≥ 10% of total_queries`.
  - *Intensity* — `SUM(spilled_local_bytes) ≥ 50% of SUM(read_bytes)` (magnitude relative to data processed — catches severe spill regardless of how many queries spill).
  - *Peak volume* — `MAX(spilled_local_bytes) ≥ 5 GB`.
  - Flagged warehouses get the next t‑shirt size suggested.
- **Window:** trailing 30 days.

> **Note on "% of node memory":** a literal "spill > X% of node storage" isn't computable from system tables — they don't expose per‑node local‑disk/memory capacity. The **Intensity** mode (spill vs. bytes read) is the node‑capacity‑independent way to capture "heavy spill by magnitude."

## Requirements
- Unity Catalog workspace with **system tables enabled**, and the viewer needs `SELECT` (or `USE SCHEMA`) on:
  - **`system.query`** (`system.query.history`) — queue + spill signals (`waiting_at_capacity_duration_ms`, `spilled_local_bytes`, `read_bytes`).
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
- The `ceil(depth / 10)` (clusters) and the spill‑trigger thresholds (`≥10%` frequency, `≥50%` intensity, `≥5 GB` peak) are **heuristics** — starting points. Use the two selectors to sanity‑check how sensitive a recommendation is (e.g., a warehouse that only needs clusters at *Max* but not *p95* is a rare‑spike case, not sustained under‑provisioning; one flagged only under *Peak volume* has occasional huge queries rather than pervasive spill).
- Sizing up helps memory‑bound / parallelizable work, not tiny or I/O‑bound queries. Treat every recommendation as a **candidate to confirm** in the SQL warehouse monitoring page before changing settings.
- Provided **as‑is**, no warranty or official support. **Not an official Databricks product.**
