# Discovery index — scheduler execution audit

**Status:** draft. This file is the cross-product map. It does not replace the product inventories. Ingest topology and phased export scope are in [04-ingestion-strategy.md](04-ingestion-strategy.md). Storage, UI, and telemetry schema are not fully chosen here.

| Product | Audit slug | Inventory |
|---------|------------|-----------|
| **algortimo-gd** | `algortimo-gd` | [discovery](products/algortimo-gd/execution-analytics-platform_discovery_document.md) |
| **algoritmo_GH_PD** | `algoritmo-gh-pd` | [discovery](products/algoritmo-gh-pd/execution-analytics-platform_discovery_document.md) |

Forecast is a possible later product. It has no checkpoint map in this repo yet.

The slug `algortimo-gd` keeps the spelling already used in the GD inventory. `algoritmo_GH_PD` is the only algoritmo_GH codebase. Its audit slug is `algoritmo-gh-pd`.

## Two audit processes

These are detached processing items. Each execution audit belongs to one product.

| Fact | What it means for the platform |
|------|--------------------------------|
| **Separate audits** | A GD run and a GH run are different audit processes. Each has its own timeline, validations, and result. |
| **GH runs on GD results** | algoritmo_GH consumes algortimo-gd output. A GH audit may later point at the GD result it consumed. That link is a relationship between two audits. |
| **Different results** | The two pipelines do not mean the same thing and do not produce the same result. A GH outcome is not a GD outcome under another name. |
| **Same process entities** | Both consume the same WFM process entities (`esc_processo`, status procedures, process errors, `CORE_ALG_PARAMETERS`). Shared entity shape is how a process is stored. It is not a shared execution id. |

`project` stays on every audit record. A screen can list both products. Opening one record shows that product’s checkpoints only.

## Shared audit axis

Both products talk to WFM Oracle (`esc_processo`, `S_PROCESSO` status and error procedures, `CORE_ALG_PARAMETERS`) and do their real work **per posto inside one `wfm_proc_id`**. The letters A–E are the shared axis:

| ID | Meaning | What stays shared |
|----|---------|-------------------|
| **A** | Job / WFM process structure | Poll, claim, status letters, concurrency, terminal `G` / `I` |
| **B** | Data loading | Entities and counts loaded before transforms |
| **C** | Transforms into solver inputs | In-memory structures the solver actually receives |
| **D** | Algorithm and solver | Outcome, timings, failure reasons |
| **E** | Format and integration | What was written back to WFM |

Status letters in both inventories: `N` polled, `P` claimed, `G` done, `I` invalid or failed. `E` is an error-row type, not a process status.

Keys both inventories already have: `wfm_proc_id`, `posto_id`, dates, a user string, and some form of algorithm identity.

Keys the platform adds: `deployment_version` (not in code today). **`tenant_id`** (or equivalent environment id) is **not in WFM or the algorithm repos**; it will be **defined in this platform** and set per **export-agent deployment** ([04-ingestion-strategy.md](04-ingestion-strategy.md)). Do not map it from `FK_EMPRESA` or the Oracle user without an explicit decision.

Telemetry tiers T0–T3 are defined in each product discovery. Default direction in both: **T0 + T1** first; full artefacts (T3) stay off until explicitly approved.

## Do not flatten these differences

| Topic | algortimo-gd | algoritmo-gh-pd |
|-------|----------------|-----------------|
| Child execution | In-process `run_batch_process`; parent sets `G` / `I` | Parent `Popen`s `main.py` and sends itself `SIGTERM`; the child sets parameter status `G` / `I` |
| Stage names | Registry: `data_loading`, then `processing` substages `treat_params`, `load_matrices`, `func_inicializa`, `allocation_cycle`, `format_results`, `insert_results` | No stage registry. A–E map to functions (`ESCRunner`, `SchedulerMain`, daily CBC). Do not emit GD substage names for GH |
| Algorithm identity | `algorithm_name` plus `data_model` (Salsa vs Alcampo) | Audit scope: **`ESC` only** for GH (`algorithm_name=ESC`; parent always launches ESC). The GH factory also lists `MPD` in code—that branch is **not in platform audit scope** ([04-ingestion-strategy.md](04-ingestion-strategy.md)). Extra discriminator is numeric `process_type` **2** (insert) vs **3** (update a prior read-model). Do not store that number in `data_model` |
| Solver grain | One allocation cycle per posto | Weekly template selection, then one PuLP/CBC solve **per day** |
| Integration | `insert_results` | Type 2 inserts `wfm.int_schedule_algorithm` (employee id rewritten to matrícula). Type 3 updates `wfm.CORE_SCHED_ALGORITHM_READ` |
| `api_proc_id` | Passed into the child call dict and used on status updates | Parent argv only. Child status uses WFM `CODIGO` (`wfm_proc_id`) |

UI label that stays honest (from the GH inventory):

```text
project: algortimo-gd     | stages: data_loading > processing/{treat_params…insert_results} | keys: tenant_id, wfm_proc_id, posto_id, data_model, algorithm_name, deployment_version
project: algoritmo-gh-pd  | checkpoints: A job > B load > C template/day rows > D daily CBC > E int_schedule_algorithm | CORE_SCHED_ALGORITHM_READ | keys: tenant_id (new), wfm_proc_id, posto_id, process_type, algorithm_name=ESC, deployment_version (new)
```

`process_type` is required on GH events and is not required on GD events. `data_model` is required on GD events and is not required on GH events.

## Event sketches (not a schema)

Each product discovery contains its own `execution_audit_v1` sketch. They are not the same document:

- GD sketch is one message with `data_model`, `algorithm_name`, and checkpoint ids such as `wfm` and `data_loading`.
- GH sketch is one event per `(project, wfm_proc_id, posto_id)` with `project`, numeric `process_type`, and checkpoint letters A–E. A parent-only event may have `posto_id` null.

A storage shape that can hold both products is a later decision. It belongs in `03-telemetry-schema.md` after the open questions in `04-ingestion-strategy.md` are answered. Each stored audit remains one product’s execution. The sketches are not accepted.

## What this index does not decide

Warehouse technology, PII boundary, backfill depth, bucket object layout, BFF contract, loader notifications, deduplication rules, and UI stack stay open. Ingest **shape** (per-environment export agent → raw bucket → central transform in this repo; separate GD/GH exporters; daily batch + on-demand via BFF; terminal runs only) is in [04-ingestion-strategy.md](04-ingestion-strategy.md). Future **algorithm rerun from ingested raw** is noted there; it is not the same as intraday on-demand **export**.
