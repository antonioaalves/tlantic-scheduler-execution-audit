# Discovery document: execution analytics & audit platform

**Status:** draft — product inventory for **algortimo-gd** only. Shared platform scope is in [docs/01-discovery.md](../../01-discovery.md).

## Doc overview

| Field | Value |
|-------|--------|
| Jira | — |
| Title | What to carry from algortimo-gd into a centralized execution audit web app |
| Date | 26/10/2026 |
| Status | Draft — for team review before a new repository is created |
| Related | Prior chat on ingestion (push vs pull, bronze/silver/gold); dev-guide-first workflow for later implementation stories |

---

## Summary

This document inventories **how algortimo-gd runs today**, defines **audit checkpoints** aligned with the real code path, and lists **what information exists (or can be reproduced)** at each checkpoint. It is the first artefact for a separate analytics/audit project—not a commitment to ingest everything on day one.

**Out of scope (for this doc):** choosing the analytics database, building the web UI, or implementing telemetry hooks in algortimo-gd.

**Recommended workflow (answer to “ideas first vs code first?”):**

1. **Use this document first** — ground the new project in observable facts from algortimo-gd (pipeline, keys, persisted vs ephemeral data).
2. **Keep a short “ideas backlog”** (separate file or section below) — capture floating thoughts **mapped to a checkpoint**, not as a unstructured wish list.
3. **Create the new repository** once the checkpoint map and v1 telemetry schema are agreed — the repo README can link back here.
4. **Implement ingestion + app** in the new repo; changes to algortimo-gd should be small, optional hooks (fail-open) in a later story.

Writing ideas only before the inventory tends to reinvent checkpoints that already exist in code (e.g. substages in `project_structure.py`). Doing inventory first makes every idea testable: “which checkpoint?”, “already in Oracle?”, “replay from inputs?”.

---

## Problem (today)

| Area | Current behavior |
|------|------------------|
| WFM orchestration | Parent `orquestrador.py` polls Oracle (`S_PROCESSO.GET_PROCESS_BY_STATUS`), sets status `P`, calls `run_batch_process`, then `G` or `I` via param status. |
| Batch pipeline | `batch_process.py` → `AlgoritmoGDService`: stages `data_loading`, then `processing`. |
| Processing | Six substages per **posto** (`posto_id_list`): treat_params → load_matrices → func_inicializa → allocation_cycle → format_results → insert_results. |
| Persistence | WFM process row + `esc_processo_erros` messages; optional CSV debug under `data/output/`; **no centralized store** of intermediate matrices or solver I/O. |
| Multi-tenant | Same codebase; client-specific Oracle + config; schema “almost” identical across clients. |
| Developer visibility | VPN to client DBs; logs on server; no cross-client dashboard. |

---

## Target behavior (analytics platform — vision)

Developers open one web app (no VPN) to:

1. **Audit WFM process structure** — parent/child process, status timeline, errors, concurrency limits.
2. **Audit data flow to the solver** — which entities loaded, validation outcomes, shapes/counts at key transforms.
3. **Audit algorithm → solver mapping** — inputs (`medium_data`), parameters, factory choice (Salsa vs Alcampo).
4. **Audit solver results** — status, stats, validation, failure reasons.
5. **Audit integration** — formatted insert payload, insert success, WFM error rows on failure.

Support **daily batch** refresh plus **on-demand** re-export for one execution (same ingestion path).

---

## End-to-end pipeline (as implemented)

```text
[WFM Oracle]
  esc_processo (+ packages S_PROCESSO.*)
  esc_processo_erros (via set_process_errors)
        │
        ▼
orquestrador.py
  • PROC_COD = AlgoritmoHorariosPython_Pai
  • Poll MPD / status N → start child
  • external_call_dict: wfm_proc_id, dates, child_number, user, …
  • set_process_status(P) → run_batch_process → set_process_param_status(G|I)
        │
        ▼
batch_process.py → AlgoritmoGDService
        │
        ├─ Stage 1: data_loading
        │     DataModelFactory (alcampo_data_model | salsa_data_model)
        │     load_process_data(sql_processing_paths)
        │
        └─ Stage 2: processing (per posto_id)
              treat_params → load_matrices → func_inicializa
              → allocation_cycle (AlgorithmFactory → algorithm.run(medium_data))
              → format_results → insert_results (optional)
```

Reference configuration: `src/settings/project_structure.py` (substages and sequences).

---

## Correlation keys (use everywhere in the new project)

| Key | Source | Notes |
|-----|--------|--------|
| `tenant_id` / `environment_id` | **New** — not in code today | Must be configured per deployment (client + continent + env name). |
| `wfm_proc_id` | `external_call_dict['current_process_id']` | Primary business key for one WFM run. |
| `api_proc_id` | Orchestrator argv / parent process | Parent WFM process. |
| `child_number` | Orchestrator loop index | Distinguishes parallel children under same parent. |
| `posto_id` | `auxiliary_data['current_posto_id']` | One processing loop iteration; many errors are per posto. |
| `start_date`, `end_date` | `esc_processo` / external_call_dict | Scheduling horizon. |
| `data_model` | `GD_algorithmName` → alcampo vs salsa | Drives different SQL paths and treatment. |
| `algorithm_name` | treat_params / parameter defaults | e.g. SALSA vs Alcampo algorithm module. |
| `process_type` | WFM messages | e.g. `AlgoritmoHorariosPython_Pai`, `data_loading`, `processing_stage`, substage names in logs. |
| `deployment_version` | **New** | Git tag / build id of algortimo-gd on that server. |

All telemetry events should include: `{ tenant_id, wfm_proc_id, posto_id?, checkpoint, status, started_at, ended_at, duration_ms, deployment_version }`.

---

## Audit checkpoints — what exists and what to bring

### Checkpoint A — WFM process structure

**What it is:** How work enters the Python stack and how status is reported back.

**Code / data today:**

| Signal | Where | Persisted? |
|--------|--------|------------|
| Process queue | `get_process_by_status.sql` → `S_PROCESSO.GET_PROCESS_BY_STATUS` | Oracle |
| Process detail | `get_process_by_id.sql` → `esc_processo` | Oracle |
| Status transitions | `set_process_status`, `set_process_param_status` | Oracle |
| Concurrency / retries | `get_all_params` (GLOBAL_MAX_CONCURRENT_PROCESSES, etc.) | Oracle params |
| Parent wait / thread cap | `set_process_errors` (iniProc, errMaxThreads, …) | `esc_processo_erros` |
| Child invocation | `external_call_dict` in orchestrator | Ephemeral unless logged |

**Bring to analytics (v1 — metadata):**

- Snapshot of `esc_processo` fields needed for list/detail views (codigo, dates, user, status, FK_COLABORADOR, section/unit ids if present on row).
- Timeline: status changes + error rows ordered by time.
- Orchestrator params: max concurrent, retries (for explaining “why waiting”).

**Bring (v2 — optional):**

- Pull-based daily sync from Oracle where network allows; else push a compact “process_finished” event from orchestrator.

**Ideas parking lot (examples — map here, not loose):**

- “Show queue depth per client” → Checkpoint A + orchestrator poll metrics.

---

### Checkpoint B — Data loading (raw → data model)

**What it is:** Stage `data_loading`: select model, load SQL entities into the data model container.

**Code / data today:**

| Signal | Where | Persisted? |
|--------|--------|------------|
| Entity list | `src/settings/sql_filepaths.json` → `sql_processing_paths` | Config |
| Load success/fail | `data_model.load_process_data` | WFM errors on fail (`data_loading`) |
| Model choice | `GD_algorithmName` → alcampo vs salsa | Logs + derived at runtime |
| Valid employees / MPD | queries: `valid_emp`, `df_mpd_valid_employees`, … | Oracle → in-memory DataFrames |

**sql_processing_paths (initial load entities):**

- `valid_emp`, `df_mpd_valid_employees`, `df_fk_colaborador_matricula`
- `params_lq`, `df_feriados`, `df_closed_days`, `params_df`, `parameters_cfg`
- (insert SQL paths used later: `insert_results_df`, `insert_compensatory_results_df`)

**sql_auxiliary_paths (loaded in later substages — still part of “data flow”):**

- Calendar / structure: `df_calendario_passado`, `df_estrutura_wfm`, `df_estimativas` inputs (`df_orcamento`, `df_granularidade`, `df_faixa_horario`, …)
- Rules & HR: `df_process_rules`, `df_disponibilidade`, absences, contracts, cycles, etc.
- Alcampo-specific: `alcampo_sql_raw_paths`, `alcampo_sql_auxiliary_paths`

**Bring to analytics (v1 — summaries, not full tables):**

- Per entity: `name`, `row_count`, `column_count`, `load_duration_ms`, `success`.
- `data_model` name, `message_lang`, `posto_id_list` length.
- On failure: `error_code` / message key from load stage (already routed to WFM).

**Bring (v2 — audit / replay):**

- Hashed fingerprints of inputs (e.g. hash of sorted employee ids + date range) to compare runs without storing PII-wide dumps.
- Sample rows or schema JSON for drift detection between clients.

**Reproducibility note:** The modular pipeline allows re-running transforms from the same DB inputs; analytics does not need to store every matrix if the team can trigger replay in a controlled sandbox — but that is a **product decision** (replay job vs store bronze).

---

### Checkpoint C — Transformations (`func_inicializa` / matrices → `medium_data`)

**What it is:** Substages `load_matrices` and `func_inicializa` build processed structures, culminating in **`medium_data`** for the algorithm.

**Code / data today:**

| Signal | Where | Persisted? |
|--------|--------|------------|
| Matrix load validation | `_execute_load_matrices_substage` | WFM error keys e.g. `invalidLoadMatrices` |
| Calendar / estimativas pipeline | `load_estimativas_transformations` in `base.py` | In-memory; logs shapes |
| Required solver inputs | `allocation_cycle` checks | Logs: `df_colaborador`, `df_calendario`, `df_estimativas` |
| Invalid entities | validation helpers in data models | Logs; sometimes lists in messages |

**Bring to analytics (v1):**

- Substage timings: load_matrices, func_inicializa (per posto).
- Validation outcome: pass/fail + `invalid_entities` list when available.
- **Medium_data summary:** key names present, row counts for the three required frames, date range coverage, count of workers/days (from logs or lightweight counters).

**Bring (v2):**

- Dimension snapshots for debugging failed runs only (feature flag): e.g. estimativas min/max/mean per day, calendar hole detection counts.

**Ideas parking lot:**

- “Visualize estimativas vs calendar alignment” → Checkpoint C metrics, not solver.

---

### Checkpoint D — Algorithm & solver (`allocation_cycle`)

**What it is:** `AlgorithmFactory` → `algorithm.run(data=medium_data, algorithm_treatment_params=…)`.

**Code / data today:**

| Signal | Where | Persisted? |
|--------|--------|------------|
| Algorithm name & params | treat_params + config defaults | Logs; `algorithm_treatment_params` |
| Run outcome | `results` dict from algorithm | Partially in `rare_data`; stats in algorithm modules |
| Failure | `results.status == 'failed'`, `allocation_error` | Logs; WFM on earlier/later stages |
| Success stats | e.g. `_calculate_comprehensive_stats`, `_create_metadata` in Alcampo/Salsa paths | Returned in memory; export metadata in results object |
| Stage handler | `complete_substage(..., result_data=…)` | Process manager / logs if tracking enabled |

**Bring to analytics (v1):**

- `algorithm_name`, normalized `algorithm_params` (keys + non-secret values).
- Solver status, duration, high-level stats: workers, days, assignments, coverage %, objective/feasibility flags (exact fields depend on algorithm — define a **union schema** with optional fields).
- On failure: error message / exception class (truncated), no full LP dump initially.

**Bring (v2):**

- Solver attribute subset (iterations, gap, model size) if exposed in `solver_attributes` today.
- Link to optional debug exports (xlsx/csv) if still written under `data/output/`.

---

### Checkpoint E — Results formatting & WFM integration

**What it is:** `format_results` → `validate_format_results` → `insert_results` → `validate_insert_results`.

**Code / data today:**

| Signal | Where | Persisted? |
|--------|--------|------------|
| Formatted payload | `data_model.formatted_data` | Debug CSV: `df_insert_results-{process_id}-{posto_id}.csv` |
| Unconverted F/D days | `unconverted_fd_day_events` | Logged to `esc_processo_erros` when DB available |
| Insert | `insert_results.sql` via data manager | Oracle WFM tables |
| Insert failure | WFM `errInsertWFM` | `esc_processo_erros` |
| Final orchestrator status | `G` / `I` | Oracle |

**Bring to analytics (v1):**

- Row count scheduled for insert, insert duration, success/fail.
- Count of unconverted F/D warnings (already logged to DB — can be pulled or echoed in telemetry).
- Whether insert was skipped (`insert_results` decision false).

**Bring (v2):**

- Aggregates of insert payload (shift type counts per day), not full schedule unless compliance approves.

---

## What is already persisted without new instrumentation

Useful for **pull-based** or **hybrid** analytics even before code changes:

| Store | Content | Access pattern |
|-------|---------|----------------|
| `esc_processo` | Process header, dates, status | SQL / WFM getters |
| `esc_processo_erros` | Rich audit trail: info + error messages, many with posto/employee/day | `set_process_errors` |
| Server log files | Stage/substage progress, shapes, DEBUG | File ship or grep (noisy) |
| `data/output/` CSV/XLSX | Results and insert previews on some deployments | Not guaranteed in prod |

The message catalog (`df_messages.csv`) explains **error codes** — the analytics app should load a copy per language/version for human-readable audit.

---

## Proposed telemetry tiers (for the new project)

| Tier | Contents | Storage | When |
|------|-----------|---------|------|
| **T0 — Execution index** | Keys, status, durations, checkpoint pass/fail | SQL/document “gold” | Every run |
| **T1 — Metrics** | Row counts, stats, validation lists | JSON documents or JSONB | Every run |
| **T2 — Fingerprints** | Hashes, schema versions, param snapshot | Bronze | Failures or sample % |
| **T3 — Artefacts** | CSV/Parquet samples, solver logs | Object storage | Explicit opt-in / failed runs |

Default v1: **T0 + T1** pushed at end of run (or pulled from Oracle for T0 partial).

---

## Suggested event schema (sketch for new repo)

Single message `execution_audit_v1` (array of checkpoint records):

```json
{
  "schema_version": 1,
  "tenant_id": "client-eu-prod",
  "deployment_version": "algortimo-gd@abc123",
  "wfm_proc_id": 12345,
  "api_proc_id": 999,
  "child_number": 1,
  "posto_id": 121,
  "data_model": "salsa_data_model",
  "algorithm_name": "…",
  "horizon": { "start_date": "2026-01-01", "end_date": "2026-01-31" },
  "overall_status": "success",
  "checkpoints": [
    {
      "id": "wfm",
      "status": "success",
      "duration_ms": 0,
      "metrics": {}
    },
    {
      "id": "data_loading",
      "status": "success",
      "duration_ms": 1200,
      "metrics": { "entities": [{ "name": "valid_emp", "rows": 42 }] }
    }
  ]
}
```

Refine field names when the first emitter is implemented in algortimo-gd.

---

## Components to reference when building the new repo (not copy wholesale)

| algortimo-gd area | Reuse intent |
|-------------------|--------------|
| `orquestrador.py` | Document parent lifecycle; optional hook after `run_batch_process`. |
| `src/services/algoritmo_gd.py` | Substage boundaries = emitter insertion points. |
| `src/settings/project_structure.py` | Canonical checkpoint names. |
| `src/settings/sql_filepaths.json` | Entity catalog for “what was loaded”. |
| `src/data_models/base.py` | `medium_data` keys, allocation/format/insert contracts. |
| `data/Queries/WFM_Process/*` | Oracle pull queries for hybrid ingestion. |
| `src/orquestrador_functions/Logs/process_wfm_errors.py` | Error row semantics. |

The analytics repo should **not** depend on Oracle at runtime for the UI; it depends on its **warehouse** filled by ingest.

---

## Open questions (resolve before v1 ingest)

1. **Push vs pull per tenant:** Is outbound HTTPS from app servers allowed?
2. **PII boundary:** Are employee ids in `esc_processo_erros` acceptable in the central store?
3. **Replay:** Will the team trigger sandbox reruns from the app, or only view telemetry?
4. **Alcampo vs Salsa:** One dashboard with `data_model` dimension, or separate KPI definitions?
5. **Historical backfill:** How many days of `esc_processo` / errors per client for launch?

---

## Ideas backlog (template)

Add rows as you think of features:

| Idea | Checkpoint | v1/v2 | Notes |
|------|------------|-------|-------|
| Cross-client failure rate dashboard | A, D | v1 | Needs T0 only |
| Compare estimativas row counts client vs client | B, C | v2 | Drift detection |
| One-click re-run transform without solver | C | v2 | Sandbox + VPN job |
| … | | | |

---

## Next steps

1. **Review this doc** — confirm checkpoints match how you explain the system to the team.
2. **Answer open questions** — especially PII and push vs pull.
3. **Create new repository** with README linking here; define `execution_audit_v1` JSON schema as code (Pydantic/JSON Schema).
4. **Story in algortimo-gd:** optional fail-open telemetry emitter at substage boundaries (separate development guide).
5. **Story in analytics repo:** ingest API + daily job + minimal UI (process list + checkpoint drill-down).

---

## Acceptance criteria (for this discovery phase)

- [ ] Team agrees the five checkpoints (A–E) match the audit scope.
- [ ] v1 telemetry tier (T0+T1) is accepted as first milestone.
- [ ] Correlation keys and `tenant_id` strategy defined.
- [ ] Ideas backlog uses checkpoint column for new entries.
- [ ] Decision recorded: new repo created vs monorepo folder (recommend **separate repo** for ETL/app lifecycle).

---

## Suggested commit message (when adding this doc to algortimo-gd)

```text
docs: execution analytics platform discovery from algortimo-gd pipeline

Inventory WFM orchestration, substages, and data entities to seed a centralized audit app.
```
