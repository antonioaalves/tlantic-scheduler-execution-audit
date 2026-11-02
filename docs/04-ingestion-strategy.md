# Ingestion strategy — decision log

**Status:** partial. Ingest topology, provenance, re-export retention, raw file format, and export run manifest (with per-process completion markers) are recorded below. Warehouse technology, BFF contract details, and loader notifications stay open.

Do not implement ingest, emitters, or a warehouse schema from this file alone.

## Agreed shape (read this first)

Each **client environment** runs its own **execution export agent** (working name): a small service that reads Oracle and writes **raw** objects to a **bucket**. **algoritmo-gd** and **algoritmo-gh** use **separate export agents** (or separate modes with separate bucket destinations)—not one mixed exporter that the central app must classify.

**This repository** (central app) owns **all transformation** from raw bucket objects into the audit store and UI. Export agents do not normalize into `execution_audit_v1`; they land queries/results as agreed per product.

Two export paths, both **terminal processes only** for the foreseeable future (status **`G` / `I`** and related WFM error rows—see [01-discovery.md](01-discovery.md)):

| Path | Trigger | Role |
|------|---------|------|
| **Daily batch** | Cron on the client side (same container as the algorithms or a dedicated one) | Bulk of history: processes that finished in the agreed window (e.g. previous calendar day). |
| **On-demand** | Central app → product **BFF** → export agent | Small volume: export specific `wfm_proc_id`(s) when needed. BFF stays thin (request export + confirmation); it does not carry bulk audit payloads. BFF access is intended so **only the central audit API** triggers these calls. Implementation of the BFF passthrough is **outside this repo**. |

**Bucket access** is restricted (e.g. allowlisted callers/addresses). Exact IAM/network layout is not chosen yet.

**Phased data scope:** the first slice is **two queries**, the same for both products. One reads `wfm.esc_processo`. The other reads `wfm.esc_processo_erros` joined to `wfm.esc_colaborador`, so employee fields travel in that result. There is no separate `esc_colaborador` file. SQL text is written outside this log. Later phases add queries. Integration targets already named: `wfm.int_pre_schedule_algorithm` for **algoritmo-gd**, `wfm.int_schedule_algorithm` for **algoritmo-gh**. Those are not in this slice.

**Execution-time fidelity:** for most inputs, the algorithms consume **process-associated loading tables** in Oracle that already act as a **snapshot at execution time**. Exports should prefer those associations keyed by `wfm_proc_id` (and product-specific keys from [01-discovery.md](01-discovery.md)). For data **not** represented there, the platform should still aim to capture state **as close as possible** to what existed at execution time (exact mechanism per entity stays product work—emitters, extra queries, or integration tables—not decided here).

**On-demand user intent (not “refresh”):** operators may need runs that **finished since local midnight** but are **not yet** in the last daily batch. On-demand export covers that gap. Re-exporting a process **already** loaded by the daily batch is expected to happen sometimes. The central loader collapses those exports to **one audit row**, using the earliest extraction (see decisions).

**Identity:** `wfm_proc_id` (`esc_processo.CODIGO`) identifies **one WFM execution** inside one client environment. At WFM level that process **runs once**. There is no WFM attempt id. `wfm_proc_id` is unique only inside that environment, so the same `CODIGO` in `prd` and in `tst` is two runs.

Provenance is three fields, set in export-agent config. They are not read from WFM, from `FK_EMPRESA`, or from the Oracle user.

| Field | Meaning | Values |
|-------|---------|--------|
| `tenant_id` | Client name | One name per client |
| `env` | Which instance of that client | `prd`, `tst`, `dev` |
| `project` | Which product exported the run | `algoritmo-gd`, `algoritmo-gh` |

A GD agent and a GH agent in the same client environment share `tenant_id` and `env`. They differ by `project`. Continent is not an identity field: one client instance belongs to one client.

The audit upsert key is **`tenant_id` + `env` + `project` + `wfm_proc_id`**.

One GH parent orchestrator invocation may **`Popen` many child processes**; each **`wfm_proc_id` is one run**—export **one bundle per `wfm_proc_id`**, not per orchestrator invocation.

**Re-export:** the agent does not number attempts. Each export records `extracted_at`, the time that extraction ran. The object key is `{tenant_id}/{env}/{project}/{wfm_proc_id}/{extracted_at}/`, and the same timestamp is stored inside the object. A later export does not replace an earlier object. For one audit key, the warehouse payload is the export with the **earliest** `extracted_at`. That copy is the closest to execution: loading tables already hold the execution snapshot, and core tables can change afterwards. Exports are terminal processes only (`G` / `I`), so the earliest extraction is the one taken soonest after the run.

**Raw files:** one JSON document per table inside that folder. The file name is the table (or query) name. A later phase adds a file; older folders stay valid without it. The value is a list of row objects, column names as returned by Oracle. A single table may move to NDJSON (same objects, one per line) when loading the whole list hurts. Parquet is a further option for a large table when volume justifies the library. Other tables stay JSON.

**Export runs (daily batch and on-demand):** Each trigger starts **one export run** for that `tenant_id` + `env` + `project`. The daily cron is **not** two parallel dailies; a **second run** happens only when the first run **failed** and ops or automation retries (a new run, new `run_id`). On-demand is its own run with the same manifest rules.

For each `wfm_proc_id` in the run, the agent writes the process folder under `{tenant_id}/{env}/{project}/{wfm_proc_id}/{extracted_at}/`, then a **per-process completion marker** in that folder after all JSON files for the current export slice are present. The central loader **must not** read a process folder until that marker exists.

When the run finishes (success or partial failure), the agent writes **one run manifest** last, at `{tenant_id}/{env}/{project}/runs/{run_id}/manifest.json`. The manifest records `run_id`, `mode` (`daily` or `on-demand`), `started_at`, `finished_at`, provenance (`tenant_id`, `env`, `project`), and each in-scope process: `wfm_proc_id`, `extracted_at`, and status (`exported` or `failed`). Failed processes have no completion marker and no process folder (or an explicit failed row only in the manifest—agent policy TBD in implementation). **`finished_at` on the manifest** is when export for that run is done; the central app may start or finalize transforms for that run after the manifest is present. **Load progress** (how many manifest rows are in the warehouse) is tracked by the **central loader** in the audit store, not by the export agent writing to a separate central database.

**Algorithm rerun (future):** the platform may later **rerun the algorithm from ingested raw** in a controlled sandbox—not the same as re-querying live Oracle for “refresh.” **Local reruns** (process id and dates passed like the orchestrator, without a new WFM process) **do not exist at WFM level** and are **outside** the WFM export / `wfm_proc_id` idempotency rules until a separate audit identity is designed. Whether rerun against **live** loading tables adds value over rerun from **ingested** snapshot is **unset** (loading tables are already execution-time snapshots). GH non-determinism and replay limits in the GH inventory still apply when interpreting rerun outcomes.

## Decisions

| Date | Topic | Decision | Why | Rejected |
|------|-------|----------|-----|----------|
| 2026-10-26 | GH codebase | `algoritmo-gh` is the only GH product. No second GH inventory. | Confirmed for this platform. | A second GH codebase |
| 2026-10-26 | Audit identity | GD and GH are detached audit processes. GH runs on GD results and both consume the same WFM process entities. Results stay separate records. | They do not mean the same thing and do not produce the same result. | One merged execution whose checkpoints mix both pipelines |
| 2026-10-29 | Ingest topology | Per-environment **execution export agent** reads Oracle, writes **raw** to a **bucket**. **Separate** export path per product (`algoritmo-gd` vs `algoritmo-gh`). Central app in **this repo** transforms raw → warehouse/UI. | Client Oracle stays local; no central VPN hub to every tenant. Clear product boundary at the bucket. | Single mixed exporter; central app guessing product from unstructured dumps |
| 2026-10-29 | Ingest modes | **Daily batch** (bulk) + **on-demand** via **BFF** (thin trigger). **Terminal processes only** for both. | Matches ops vs intraday debugging; keeps BFF small for the product team. | On-demand for in-flight (`P`) processes; bulk data through the BFF |
| 2026-10-29 | Export agent placement | Runs in the **algorithm container** or a **dedicated** sidecar in the same environment—either is acceptable. | Same network path to Oracle as today’s batch jobs. | — |
| 2026-10-29 | Phased Oracle scope | **Start small** (process-execution analytics, ~two tables), then **expand** exports toward full algorithm read/integration surfaces per inventories. | Deliver value before full B–E coverage; sole builder sets priority. | Big-bang export of all queries on day one |
| 2026-10-29 | Data fidelity target | Prefer **process-associated loading tables** (execution-time snapshot in Oracle); other inputs as **close to execution-time** as practical. | Aligns audit with what the run actually consumed. | Assuming live Oracle today equals execution-time state for all entities |
| 2026-10-29 | Intraday gap | On-demand covers processes **since 00:00** (client timezone TBD) **not yet** in daily batch—not a general “refresh” of arbitrary past runs. | Daily batch is the bulk path; intraday is the main on-demand motivator. | UI “refresh” as default pattern for re-pulling unchanged terminal runs |
| 2026-10-29 | Export granularity | **One export unit per `wfm_proc_id`** per product. | Matches one WFM process row per run; GH multi-`Popen` parent is not the export unit. | One object per orchestrator invocation |
| 2026-10-29 | WFM idempotency | Each WFM process **executes once** at WFM level. Key composition superseded on 2026-11-02 (`env` added; `tenant_id` is the client name). | One `esc_processo.CODIGO` is one audit execution per environment. | `wfm_proc_id` + attempt id for WFM; assuming the same CODIGO can be rerun in WFM |
| 2026-10-29 | Local algorithm reruns | Reruns started locally (simulated orchestrator params: process id, dates, etc.) **are not WFM processes** and are not covered by WFM export idempotency. | Operational reality; distinct from ingest-backed sandbox rerun (future). | Treating local reruns as WFM `wfm_proc_id` audits |
| 2026-10-29 | MPD audit scope | **Out of scope.** GH audit covers **`ESC` only** (parent always launches ESC). | MPD is not a scheduling pipeline in the GH inventory. | Auditing MPD as a product |
| 2026-10-29 | Environment / `tenant_id` | No tenant key exists in WFM or algorithm repos today. **One export agent deployment per client environment** establishes data origin. Field meaning superseded on 2026-11-02. | Provenance without relying on an external WFM caller id. | Waiting for a pre-existing WFM tenant key |
| 2026-11-02 | Provenance fields | `tenant_id` is the **client name**. `env` is `prd`, `tst`, or `dev`. `project` is `algoritmo-gd` or `algoritmo-gh`. Both products in one client environment share `tenant_id` and `env`. Continent is not an identity field. | One client instance belongs to one client. `env` separates production, test, and development, where the same `CODIGO` can exist. | Continent in the key; one id per export agent; `tenant_id` meaning the whole environment |
| 2026-11-02 | Audit key | **`tenant_id` + `env` + `project` + `wfm_proc_id`**. | Same client and same `CODIGO` in `prd` and `tst` are different runs. | A key that omits `env` once `tenant_id` is only the client name |
| 2026-11-02 | Re-export retention | Each export is its own object, keyed by `{tenant_id}/{env}/{project}/{wfm_proc_id}/{extracted_at}/`. The agent stores `extracted_at` and does not assign an attempt number. A later export leaves the earlier object in place. | The earlier extraction is closer to execution. Replacing the object would drop that copy. Daily batch and on-demand do not share an attempt counter. | One stable path per `wfm_proc_id` that the next export overwrites; an attempt counter on the agent |
| 2026-11-02 | Which export wins | The warehouse row for an audit key uses the export with the **earliest** `extracted_at`. Later objects stay in the bucket. | Loading tables are the execution snapshot. Core-table reads drift after the run, so the soonest terminal extraction is the closer copy. | Last export wins; field-level merge across exports |
| 2026-11-02 | Raw file format | **One JSON document per table**, file named for that table or query. Adding a table in a later phase adds a file; older exports simply lack it. One table may switch to **NDJSON** when that list is too large to load whole. **Parquet** only if volume justifies the library, and only for that table. | The set of tables grows by phase. JSON matches a list of row objects and needs no extra library on the agent. | CSV; Parquet or NDJSON for every table from the start |
| 2026-11-02 | First export slice | **Two queries, both products.** (1) `wfm.esc_processo`. (2) `wfm.esc_processo_erros` joined to `wfm.esc_colaborador` in the agent SQL, one JSON file, no `esc_colaborador` file of its own. The screen shows matrícula and name from that join, and does not show the employee code. The code is still stored. Query text is not specified here. | Same WFM process and error tables for GD and GH. The join carries the employee fields the screen needs. | A third file for `esc_colaborador`; guessing the GH error table name; integration tables in this slice |
| 2026-11-02 | Export run coordination | **Per-process completion marker** in each `{wfm_proc_id}/{extracted_at}/` folder after all slice JSON files are written, plus **one run manifest per daily or on-demand export** at `{tenant_id}/{env}/{project}/runs/{run_id}/manifest.json`, written **last**. Manifest lists each `wfm_proc_id` / `extracted_at` / status and `finished_at`. One daily schedule per product environment; **retry only after a failed run** (new `run_id`). Load progress lives in the central audit store, not in a DB written by the agent. | Batch boundary and export finish time for the loader; safe incremental load while a long daily run is still writing; no half-written process folders. | Folder-only discovery; manifest without per-folder markers; export agent pushing run state to a central DB outside the bucket |

## Open — platform

| Topic | Options under consideration | Evidence still needed |
|-------|----------------------------|------------------------|
| **Run manifest details** | Marker file name; manifest JSON schema; whether `run_id` matches future BFF `job_id` on on-demand | Implementation in export agent and Phase 1 loader fixtures |
| **BFF contract** | Sync vs async (`job_id`), polling, error shape | Separate design chat; BFF owned outside this repo |
| **Loader notifications** | Central poll vs bucket event → queue | Ops ownership; separate design chat |
| **Per-posto grain** | `posto_id` only if a later warehouse row is one posto. v0 audit row stays one per `wfm_proc_id` | Needed when posto-level events are stored |
| **Central warehouse** | Postgres (+ JSONB) vs bronze files + SQL vs document store for raw only | Query patterns, volume. Same store must filter by `project` |
| **Historical backfill** | How many days per tenant on first connect; legal / PII review | Local CSVs and solver logs are not reliable history in GH (inventory). Forward daily batch is the default direction |
| **PII boundary** | Employee code is stored. The error query also brings matrícula and name; the screen shows those and does not show the employee code. Still open: schedule samples. Credentials in `acessos.txt` must never be copied into exports | Security / DPO guidance for anything beyond this display rule |
| **Fail-open emitters** | Inside GD/GH batch vs export-agent-only Oracle | Needed for in-memory B–D not in loading tables; GH child is the emitter location if push is chosen later |
| **Algorithm rerun** | Sandbox rerun from **ingested raw** (future); correlation for **local non-WFM** reruns | Value vs ingested snapshot; GH CBC non-determinism; infra. Distinct from ingest re-export overlap |
| **Event schema** | One envelope with product-specific payloads vs two event types | Either way, one stored audit is one product. A GH record may reference a GD audit id. `data_model` is GD-only. Numeric `process_type` is GH-only. See [01-discovery.md](01-discovery.md). **`03-telemetry-schema.md`** still follows loader/raw conventions |
| **UI stack** | Not listed as a candidate set yet | After phase 0–1 alignment |
| **Ingestion (rejected alternatives)** | — | **Central** ETL host pulling all client Oracles without per-environment agent; **push** of bulk payloads through the product BFF |

## Open — product identity

| Question | Why it matters |
|----------|----------------|
| Is GH `process_type` 2 vs 3 two products or two passes of one product? | Type 3 reads `CORE_SCHED_ALGORITHM_READ` for the same process code and updates it. The backlog must not score it as a type 2 insert. |
| Should `parent_api_proc_id` be a correlation key for GH? | It exists on the parent command line and on some parent error rows. The child logger uses `wfm_proc_id` only. |
| Which GD result does a GH run consume, and which key links the two audits? | Product fact: GH runs on GD results. The exact handoff (table, `wfm_proc_id`, another id) is not recorded yet. The link is a relationship, and the two audits stay separate rows. |

## Phased direction (aligned with decisions above)

| Phase | Goal |
|-------|------|
| **0** | Product inventories, this decision log, vision backlogs |
| **1** | Raw export conventions + central loader for v0 tables; agree `execution_audit_v1` (or equivalent) for the warehouse **shape** as transforms stabilize. Optional fail-open emitters later, per product |
| **2** | Export agents (per product) + daily batch + minimal UI (execution list + checkpoint timeline, project filter). On-demand via BFF when BFF contract exists |
| **3** | Expand Oracle export surface toward full read/integration coverage; validations; compare; richer dashboards (including GH day strip). Future **ingest-backed** algorithm rerun |

Forecast stays out of these phases until someone writes its discovery.
