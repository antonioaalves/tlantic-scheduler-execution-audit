# Ingestion strategy — decision log

**Status:** partial. Ingest topology and phased scope are recorded below. Bucket layout, warehouse technology, BFF contract details, loader notifications, and full deduplication rules stay open.

Do not implement ingest, emitters, or a warehouse schema from this file alone.

## Agreed shape (read this first)

Each **client environment** runs its own **execution export agent** (working name): a small service that reads Oracle and writes **raw** objects to a **bucket**. **algortimo-gd** and **algoritmo-gh-pd** use **separate export agents** (or separate modes with separate bucket destinations)—not one mixed exporter that the central app must classify.

**This repository** (central app) owns **all transformation** from raw bucket objects into the audit store and UI. Export agents do not normalize into `execution_audit_v1`; they land queries/results as agreed per product.

Two export paths, both **terminal processes only** for the foreseeable future (status **`G` / `I`** and related WFM error rows—see [01-discovery.md](01-discovery.md)):

| Path | Trigger | Role |
|------|---------|------|
| **Daily batch** | Cron on the client side (same container as the algorithms or a dedicated one) | Bulk of history: processes that finished in the agreed window (e.g. previous calendar day). |
| **On-demand** | Central app → product **BFF** → export agent | Small volume: export specific `wfm_proc_id`(s) when needed. BFF stays thin (request export + confirmation); it does not carry bulk audit payloads. BFF access is intended so **only the central audit API** triggers these calls. Implementation of the BFF passthrough is **outside this repo**. |

**Bucket access** is restricted (e.g. allowlisted callers/addresses). Exact IAM/network layout is not chosen yet.

**Phased data scope:** start with a **minimal Oracle surface**—process-execution analytics on the order of **two tables**—then **add tables/queries incrementally** toward exporting **everything each algorithm reads and integrates** (per product inventories). v0 table names are not fixed in this log; inventories already cite `esc_processo` and `esc_processo_erros` for WFM process structure and errors.

**Execution-time fidelity:** for most inputs, the algorithms consume **process-associated loading tables** in Oracle that already act as a **snapshot at execution time**. Exports should prefer those associations keyed by `wfm_proc_id` (and product-specific keys from [01-discovery.md](01-discovery.md)). For data **not** represented there, the platform should still aim to capture state **as close as possible** to what existed at execution time (exact mechanism per entity stays product work—emitters, extra queries, or integration tables—not decided here).

**On-demand user intent (not “refresh”):** operators may need runs that **finished since local midnight** but are **not yet** in the last daily batch. On-demand export covers that gap. Re-exporting a process **already** loaded by the daily batch is expected to happen sometimes; **deduplication** in the central loader must collapse that to one audit row (rules below still open).

**Identity:** `wfm_proc_id` (`esc_processo.CODIGO`) identifies **one WFM execution** within an environment. At WFM level that process **runs once**; the audit upsert key is **`tenant_id` (or agreed environment id) + `project` + `wfm_proc_id`**—no WFM “attempt id.” `wfm_proc_id` is not globally unique without the environment key. **Environment provenance:** each client environment has its own export agent deployment; that deployment (plus central config) is how the platform knows **where** raw data came from. A stable **`tenant_id` field name and registry** are not in WFM today—the platform will **define** them (see decisions). One GH parent orchestrator invocation may **`Popen` many child processes**; each **`wfm_proc_id` is one run**—export **one bundle per `wfm_proc_id`**, not per orchestrator invocation.

**Algorithm rerun (future):** the platform may later **rerun the algorithm from ingested raw** in a controlled sandbox—not the same as re-querying live Oracle for “refresh.” **Local reruns** (process id and dates passed like the orchestrator, without a new WFM process) **do not exist at WFM level** and are **outside** the WFM export / `wfm_proc_id` idempotency rules until a separate audit identity is designed. Whether rerun against **live** loading tables adds value over rerun from **ingested** snapshot is **unset** (loading tables are already execution-time snapshots). GH non-determinism and replay limits in the GH inventory still apply when interpreting rerun outcomes.

## Decisions

| Date | Topic | Decision | Why | Rejected |
|------|-------|----------|-----|----------|
| 2026-10-26 | GH codebase | `algoritmo_GH_PD` is the only algoritmo_GH codebase. Audit slug `algoritmo-gh-pd`. | Confirmed for this platform. No second GH inventory. | A separate algoritmo_GH tree beside `_PD` |
| 2026-10-26 | Audit identity | GD and GH are detached audit processes. GH runs on GD results and both consume the same WFM process entities. Results stay separate records. | They do not mean the same thing and do not produce the same result. | One merged execution whose checkpoints mix both pipelines |
| 2026-10-29 | Ingest topology | Per-environment **execution export agent** reads Oracle, writes **raw** to a **bucket**. **Separate** export path per product (`algortimo-gd` vs `algoritmo-gh-pd`). Central app in **this repo** transforms raw → warehouse/UI. | Client Oracle stays local; no central VPN hub to every tenant. Clear product boundary at the bucket. | Single mixed exporter; central app guessing product from unstructured dumps |
| 2026-10-29 | Ingest modes | **Daily batch** (bulk) + **on-demand** via **BFF** (thin trigger). **Terminal processes only** for both. | Matches ops vs intraday debugging; keeps BFF small for the product team. | On-demand for in-flight (`P`) processes; bulk data through the BFF |
| 2026-10-29 | Export agent placement | Runs in the **algorithm container** or a **dedicated** sidecar in the same environment—either is acceptable. | Same network path to Oracle as today’s batch jobs. | — |
| 2026-10-29 | Phased Oracle scope | **Start small** (process-execution analytics, ~two tables), then **expand** exports toward full algorithm read/integration surfaces per inventories. | Deliver value before full B–E coverage; sole builder sets priority. | Big-bang export of all queries on day one |
| 2026-10-29 | Data fidelity target | Prefer **process-associated loading tables** (execution-time snapshot in Oracle); other inputs as **close to execution-time** as practical. | Aligns audit with what the run actually consumed. | Assuming live Oracle today equals execution-time state for all entities |
| 2026-10-29 | Intraday gap | On-demand covers processes **since 00:00** (client timezone TBD) **not yet** in daily batch—not a general “refresh” of arbitrary past runs. | Daily batch is the bulk path; intraday is the main on-demand motivator. | UI “refresh” as default pattern for re-pulling unchanged terminal runs |
| 2026-10-29 | Export granularity | **One export unit per `wfm_proc_id`** per product. | Matches one WFM process row per run; GH multi-`Popen` parent is not the export unit. | One object per orchestrator invocation |
| 2026-10-29 | WFM idempotency | Each WFM process **executes once** at WFM level. Ingest dedupe uses **`tenant_id` + `project` + `wfm_proc_id`**. | One `esc_processo.CODIGO` is one audit execution per environment. | `wfm_proc_id` + attempt id for WFM; assuming the same CODIGO can be rerun in WFM |
| 2026-10-29 | Local algorithm reruns | Reruns started locally (simulated orchestrator params: process id, dates, etc.) **are not WFM processes** and are not covered by WFM export idempotency. | Operational reality; distinct from ingest-backed sandbox rerun (future). | Treating local reruns as WFM `wfm_proc_id` audits |
| 2026-10-29 | MPD audit scope | **Out of scope.** GH audit covers **`ESC` only** (parent always launches ESC). | MPD is not a scheduling pipeline in the GH inventory. | Auditing MPD as a product |
| 2026-10-29 | Environment / `tenant_id` | No tenant key exists in WFM or algorithm repos today. **One export agent deployment per client environment** establishes data origin. The platform will **define** `tenant_id` (or equivalent) in export-agent and central config. | Provenance without relying on an external WFM caller id. | Waiting for a pre-existing WFM tenant key |

## Open — platform

| Topic | Options under consideration | Evidence still needed |
|-------|----------------------------|------------------------|
| **Bucket layout & object format** | Prefix per `tenant_id` and `project`; manifest per job; NDJSON vs Parquet vs CSV bundles | Volume, loader ergonomics, PII review |
| **BFF contract** | Sync vs async (`job_id`), polling, error shape | Separate design chat; BFF owned outside this repo |
| **Loader notifications** | Central poll vs bucket event → queue | Ops ownership; separate design chat |
| **Deduplication** | Upsert key **`tenant_id` + `project` + `wfm_proc_id`** (+ `posto_id` when GH stores per-posto); last export wins vs merge; content hash | Rules when daily batch and on-demand export the same WFM id (overlap only—not WFM reruns) |
| **Central warehouse** | Postgres (+ JSONB) vs bronze files + SQL vs document store for raw only | Query patterns, volume. Same store must filter by `project` |
| **Historical backfill** | How many days per tenant on first connect; legal / PII review | Local CSVs and solver logs are not reliable history in GH (inventory). Forward daily batch is the default direction |
| **PII boundary** | Employee ids in error rows, schedule samples, matrícula vs internal código | Security / DPO guidance. GH type 2 writes matrícula into `int_schedule_algorithm.EMPLOYEE_ID`. Credentials in `acessos.txt` must never be copied into exports |
| **Fail-open emitters** | Inside GD/GH batch vs export-agent-only Oracle | Needed for in-memory B–D not in loading tables; GH child is the emitter location if push is chosen later |
| **Algorithm rerun** | Sandbox rerun from **ingested raw** (future); correlation for **local non-WFM** reruns | Value vs ingested snapshot; GH CBC non-determinism; infra. Distinct from ingest re-export overlap |
| **Environment key format** | Exact `tenant_id` name, registry, and bucket prefix convention | Defined in this platform; tied to export-agent deployment config |
| **Event schema** | One envelope with product-specific payloads vs two event types | Either way, one stored audit is one product. A GH record may reference a GD audit id. `data_model` is GD-only. Numeric `process_type` is GH-only. See [01-discovery.md](01-discovery.md). **`03-telemetry-schema.md`** still follows loader/raw conventions |
| **UI stack** | Not listed as a candidate set yet | After phase 0–1 alignment |
| **Ingestion (rejected alternatives)** | — | **Central** ETL host pulling all client Oracles without per-environment agent; **push** of bulk payloads through the product BFF |

## Open — product identity

| Question | Why it matters |
|----------|----------------|
| Is GH `process_type` 2 vs 3 two products or two passes of one product? | Type 3 reads `CORE_SCHED_ALGORITHM_READ` for the same process code and updates it. The backlog must not score it as a type 2 insert. |
| Should `parent_api_proc_id` be a correlation key for GH? | It exists on the parent command line and on some parent error rows. The child logger uses `wfm_proc_id` only. |
| Which GD result does a GH run consume, and which key links the two audits? | Product fact: GH runs on GD results. The exact handoff (table, `wfm_proc_id`, another id) is not recorded yet. The link is a relationship, and the two audits stay separate rows. |
| Exact v0 Oracle table list per product | Process-execution analytics first (~two tables); names and joins come from each product inventory, not from this log. |

## Phased direction (aligned with decisions above)

| Phase | Goal |
|-------|------|
| **0** | Product inventories, this decision log, vision backlogs |
| **1** | Raw export conventions + central loader for v0 tables; agree `execution_audit_v1` (or equivalent) for the warehouse **shape** as transforms stabilize. Optional fail-open emitters later, per product |
| **2** | Export agents (per product) + daily batch + minimal UI (execution list + checkpoint timeline, project filter). On-demand via BFF when BFF contract exists |
| **3** | Expand Oracle export surface toward full read/integration coverage; validations; compare; richer dashboards (including GH day strip). Future **ingest-backed** algorithm rerun |

Forecast stays out of these phases until someone writes its discovery.
