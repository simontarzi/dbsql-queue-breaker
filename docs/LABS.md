# Databricks Labs submission plan — DBSQL Queue Breaker

Notes and a checklist for proposing this project to **[Databricks Labs](https://github.com/databrickslabs)** (`go/labs`). Internal‑facing planning doc.

## One‑liner
A Databricks Labs tool that scans a customer's own SQL warehouses and tells them **which lever to pull** — *add clusters* (queries queuing at capacity) or *size up* (queries spilling to disk) — turning two silent, unflagged performance problems into concrete, reversible actions. Runs entirely on the customer's own `system.*` tables.

## Why it's a Labs fit
- Increases customers' velocity‑to‑value (faster dashboards) and, as a side effect, grows DBSQL consumption.
- Self‑service, read‑only, no data leaves the workspace, no internal Databricks data.
- Natural **graduation path**: fold the recommendation into the native SQL Warehouse monitoring UI.

## What a Labs submission requires (from `go/labs`)
There is an **approval process** — many projects don't get approved. Requirements:
1. **A Product Manager sponsor** (gating) — here, a DBSQL / SQL Warehouses PM.
2. **Quality + security review** — no internal data/secrets (this project already qualifies: only `system.*` + the workspace's own data).
3. Meets the Labs engineering bar (tests, CI, docs, semantic versioning, PyPI packaging if Python), and an **active maintainer**.

## Submission steps
1. Socialize in **#databricks-labs** (Slack) and line up a **PM sponsor** — do this before investing.
2. Create a folder in the **Labs Projects Drive folder** and clone the proposal template **`go/labs/proposal`** into it.
3. Fill it in (customer evidence, GTM, markitecture, graduation strategy, project tracking) and share with **labs@databricks.com**.
4. Add a **SUBMITTED** row on the `go/labs` page with a link to the proposal.
5. On approval, the sponsor requests `labs-oss@databricks.com` to create the repo under `github.com/databrickslabs` (via the **GitHub EMU "New Repo Request"** flow) + a `LABS_PYPI_TOKEN`, with main‑branch protection.

Full pre‑filled proposal draft (Google Doc): *[Labs Proposal] DBSQL Queue Breaker — Warehouse Autoscaling Advisor* (internal).

## Architecture (customer‑safe)
| Need | Source |
|---|---|
| Queue signal | `system.query.history.waiting_at_capacity_duration_ms` |
| Scale‑up‑latency (excluded) | `system.query.history.waiting_for_compute_duration_ms` |
| Spill / size‑up signal | `system.query.history.spilled_local_bytes` |
| Warehouse name / size / type / min‑max clusters | `system.compute.warehouses` |
| Workspace name | `system.access.workspaces_latest` |

Recommendation logic: **add clusters** = `ceil(p95 concurrent‑queue‑depth / 10)`; **size up** = spill present (≥10% of queries) → next t‑shirt size.

## Packaging options
- Ship the **AI/BI dashboard** (this `.lvdash.json`) as a **Databricks Asset Bundle (DAB)** so it installs via `databricks labs install`.
- Optionally a `databricks labs queue-breaker` **CLI** that emits the ranked list for scripting.
- Telemetry per Labs guidance: unique REST **user‑agent** + a unique Python **package prefix** registered in the Labs ETL.

## Open TODOs before submitting
- [ ] Secure a **DBSQL PM sponsor**.
- [ ] **PyPI name** check (candidates: `dbsql-warehouse-advisor`, `dbsql-scaling-advisor`, `dbsql-queue-breaker`).
- [ ] Confirm `system.query.history` queue/spill granularity across clouds/regions.
- [ ] Decide DAB vs CLS packaging; add tests + CI.

*Provided as‑is; not an official Databricks product.*
