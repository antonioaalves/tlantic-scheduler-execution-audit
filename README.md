# tlantic-scheduler-execution-audit

Central **execution audit and analytics** for two detached scheduling processes, **algortimo-gd** and **algoritmo_GH_PD**. GH runs on GD results. They consume the same WFM process entities and produce different results, so each run is its own audit. Track those runs across client environments, drill into pipeline checkpoints, and run validations—without requiring every developer to VPN into every Oracle instance.

> **Status:** Discovery and documentation phase. **Ingest topology** (per-environment export agent → bucket → central transform) is recorded in [docs/04-ingestion-strategy.md](docs/04-ingestion-strategy.md). Bucket layout, warehouse, BFF contract, and dedupe rules are still open; see [Decisions (open)](#decisions-open) below.

Forecast may be added later. It is not in the checkpoint map yet.

---

## Why this exists

The algorithms run on many servers with **similar but not identical** Oracle schemas. Operations today are visible through **per-client VPN**, logs, and WFM tables (`esc_processo`, process errors via `S_PROCESSO.SET_PROCESS_ERRORS`). There is **no cross-tenant, cross-product view** of:

- WFM process lifecycle and errors
- Data loading and transformation before the solver
- Solver inputs, results, and integration back into WFM

This repository will hold the **web app**, **transform/load from the bucket**, and **warehouse** for that view. **Raw Oracle exports** are produced by per-environment **execution export agents** (one product per agent); see [docs/04-ingestion-strategy.md](docs/04-ingestion-strategy.md). It is a companion to the algorithm repositories—not a fork of either solver.

---

## Products in scope

| Product | Audit slug | Role |
|---------|------------|------|
| **algortimo-gd** | `algortimo-gd` | Own audit. Batch orchestrator, data-model factory (Salsa / Alcampo), allocation cycle, WFM insert |
| **algoritmo_GH_PD** | `algoritmo-gh-pd` | Own audit. The only algoritmo_GH codebase. Horários generator that runs on GD results: weekly templates, daily PuLP/CBC per posto, type 2 insert or type 3 update |
| **Forecast** | — | Possible later. No inventory yet |

GD and GH are separate audit processes. They share WFM process entities (`esc_processo` and the status/error procedures). A GH record may later reference the GD result it consumed. The two results stay separate rows.

`algoritmo_GH_PD` is the WFM horários generator. A parent process polls Oracle `esc_processo`, a child (`ESC`) builds weekly templates and solves a daily PuLP/CBC model per posto, then inserts `wfm.int_schedule_algorithm` (process type 2) or updates `wfm.CORE_SCHED_ALGORITHM_READ` (process type 3). It uses the same A–E axis as algortimo-gd, with its own substages and keys (`process_type`, `algorithm_name=ESC`).

- **A — job structure:** `orquestrador.py`, `main.py`, `ESCRunner`, `AlgorithmConstructor`
- **B — data loading:** `GlobalDataInitialization`, `GetGlobalData`, `SchedulerDataHandler`
- **C — transforms:** `ContractCalendar`, `TemplateSelector`, `preprocess_week_templates`
- **D — solver:** `SchedulerMain.create_schedule` (CBC), `SchedulerValidator`
- **E — integration:** `create_schedule_dataframe`; type 2 `insert_schedule_batch`; type 3 `UPDATE` of `CORE_SCHED_ALGORITHM_READ`

Repository links: fill in when the remotes are wired.

---

## Scope (audit checkpoints)

Each audit uses the same five checkpoint letters. The substages inside those letters belong to that product. A GD record shows GD stages. A GH record shows the GH day solver and GH integration. The map is in [docs/01-discovery.md](docs/01-discovery.md).

| ID | Checkpoint | Covers |
|----|------------|--------|
| **A** | WFM process structure | Orchestrator queue, status, concurrency, process errors |
| **B** | Data loading | SQL entities and inputs for that product |
| **C** | Transforms → solver inputs | GD matrices / `medium_data`; GH templates and day rows |
| **D** | Algorithm & solver | GD allocation cycle; GH daily CBC |
| **E** | Format & integration | GD insert; GH type 2 insert or type 3 update |

Correlation keys that both products can share: `project`, `tenant_id` (new, per deployment), `wfm_proc_id`, `posto_id`, dates, `algorithm_name`, `deployment_version` (new).

Product-specific keys: `data_model` on algortimo-gd; numeric `process_type` (`2` or `3`) on algoritmo-gh-pd. Do not overload one field with the other.

---

## Repository layout

```text
README.md
docs/
  01-discovery.md                 # Cross-product map (not a merged inventory)
  02-vision-and-validations.md    # How to file ideas; tables stay per product
  03-telemetry-schema.md          # execution_audit_v1 — not written yet
  04-ingestion-strategy.md        # Ingest decision log (partial)
  open-ideas.md                   # Scratch list
  products/
    algortimo-gd/                 # GD discovery + validations backlog
    algoritmo-gh-pd/              # GH_PD discovery + validations backlog + README section
```

---

## Principles

1. **Ground truth before build** — Checkpoint maps come from each algorithm repo’s code, not from assumptions. Where a narrative doc and the code disagree, the code wins.
2. **Two audits, one letter axis** — A–E is a shared vocabulary. Each record is one product’s execution. GH may reference the GD result it consumed; that link does not merge the two results.
3. **Ideas map to a product and a checkpoint** — Add rows in that product’s backlog.
4. **Telemetry tiers** — Prefer metadata + metrics (T0/T1) first; full artefacts (T3) only when needed and approved.
5. **Fail-open emitters** — Future hooks in either algorithm repo must not change WFM status or schedule writes if this platform is down.
6. **No hard decisions under uncertainty** — See open decisions; implement only what is agreed per phase.

---

## Decisions (open)

**Recorded** ingest and scope decisions live in [docs/04-ingestion-strategy.md](docs/04-ingestion-strategy.md). Do **not** treat the rows below as settled until they appear there:

| Topic | Options under consideration | Blockers / evidence needed |
|-------|----------------------------|----------------------------|
| **Bucket layout & loader** | Prefixes, manifest, format; poll vs event-driven load | PII, volume; separate design chats for BFF and notifications |
| **Deduplication** | Upsert on `tenant_id` + `project` + `wfm_proc_id`; daily vs on-demand overlap | Overlap handling only—WFM runs once per `CODIGO` ([04-ingestion-strategy.md](docs/04-ingestion-strategy.md)) |
| **Historical backfill** | Depth on first connect vs forward-only daily batch | How many days per tenant; legal/PII review |
| **Warehouse** | Postgres (+ JSONB) vs bronze files + SQL vs document store for raw only | Query patterns, volume. Must filter by `project` |
| **PII boundary** | Employee ids, matrícula vs código, schedule samples | Security / DPO guidance |
| **Algorithm rerun** | Future sandbox rerun from **ingested raw** | Distinct from intraday on-demand **export**; GH CBC may not match original run |
| **Event schema** | One `execution_audit_v1` with optional product fields vs two event types | GD and GH sketches differ (`data_model` vs `process_type`, posto event vs stage names) |
| **Export agents vs algorithm repos** | Oracle-only export agents for v0; optional fail-open emitters later | In-memory checkpoints not in loading tables |

---

## Phased direction (soft — not committed)

| Phase | Goal |
|-------|------|
| **0** | Docs: both discoveries, vision backlogs, open decisions |
| **1** | Agree `execution_audit_v1` for both products; optional fail-open emitters later |
| **2** | Export agents + daily batch + minimal UI (execution list + checkpoint timeline, filter by project). On-demand export via BFF when available |
| **3** | Expand raw export surface; validations; compare; richer dashboards (GH day strip). Future ingest-backed algorithm rerun |

Exact stack (FastAPI, React, Streamlit, Prefect, etc.) **TBD** after phase 0–1 alignment.

---

## Related projects

| Project | Role |
|---------|------|
| **algortimo-gd** | GD audit source. Batch, solver, WFM integration |
| **algoritmo_GH_PD** | GH audit source. The only GH codebase. Runs on GD results, then its own horários solve and WFM write |
| **tlantic-scheduler-execution-audit** (this repo) | Ingest, store, validate, present runs across tenants and products |

---

## Contributing (team)

1. Add validation or feature ideas to the product backlog under `docs/products/`, with checkpoint and status `idea`.
2. Do not implement ingest or schema changes without a decision row in `docs/04-ingestion-strategy.md`.
3. Prefer small, reviewable PRs tied to a phase.

---

## License

_TBD — align with company open-source / internal policy._
