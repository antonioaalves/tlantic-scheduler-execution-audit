# Product vision & validations backlog — execution analytics platform

**Status:** living document for **algortimo-gd** ideas. Platform index: [docs/02-vision-and-validations.md](../../02-vision-and-validations.md).

## Doc overview

| Field | Value |
|-------|--------|
| Companion | [execution-analytics-platform_discovery_document.md](./execution-analytics-platform_discovery_document.md) |
| Purpose | Capture **how you want the web app to look and behave** and **which validations** to run, mapped to pipeline checkpoints |
| Audience | You + team; future you when prioritizing sprints |

**How to use:** Every new idea gets one row in the tables below (or a short bullet under a checkpoint). Link to discovery checkpoints **A–E** so nothing floats without an implementation anchor.

---

## Working title (rename when repo is created)

Pick a display name and a GitHub repo slug (see naming suggestions in chat / README).

| | |
|--|--|
| Display name | _TBD_ |
| GitHub repo | _TBD_ |
| One-liner | Central place to audit WFM process runs, data → solver → integration, across clients. |

---

## Vision (web app)

_Describe in your own words — starter prompts:_

- **Who:** Developers / support on the scheduling algorithm team (not store managers).
- **Job to be done:** “Something failed in client X — where, and with what inputs, without VPN-hopping?”
- **Success:** Open a link → see runs across tenants → drill one execution → see checkpoint timeline → see validations pass/fail → jump to WFM error text or replay plan.

### Pages / areas (sketch)

| Area | Intent | Phase |
|------|--------|-------|
| **Home / overview** | Cross-tenant KPIs: success rate, failures by checkpoint, slow runs | v1 |
| **Executions list** | Filter by tenant, date, status, algorithm, data model | v1 |
| **Execution detail** | Timeline A→E, durations, error messages, metrics | v1 |
| **Validation report** | Tab on detail: all rules run for this run | v1 |
| **Compare** | Two runs or client vs client (counts/fingerprints) | v2 |
| **Replay / sandbox** | Trigger re-transform or re-solve (controlled) | v2+ |
| **Admin** | Tenants, ingest health, manual reload | v1 |

_Add or strike rows as you refine._

### UX principles (optional)

- Default to **metadata + metrics**, expand to artefacts on demand.
- Every screen answers: **which `wfm_proc_id`, which `posto_id`, which checkpoint**.
- Validations show **severity**, **message**, and **what data tier** they need (T0/T1/T2 from discovery doc).

---

## Validations backlog

**Severity:** `blocker` (data/run untrustworthy) | `warning` (investigate) | `info` (audit only)  
**Status:** `idea` | `specced` | `implemented` | `wontfix`

### Checkpoint A — WFM process structure

| ID | Validation | Severity | Needs (data) | Status | Notes |
|----|------------|----------|--------------|--------|-------|
| A-001 | Process reached terminal status G or I with at least one child attempt | info | T0, Oracle or push | idea | |
| A-002 | No orphan P status older than N hours | warning | T0 + poll time | idea | |
| A-003 | Thread-cap retries (`errMaxThreads`) under threshold per tenant/day | warning | esc_processo_erros | idea | |
| A-004 | Parent `api_proc_id` linked to all child `wfm_proc_id` attempts | info | T0 | idea | |
| | _your ideas_ | | | idea | |

### Checkpoint B — Data loading

| ID | Validation | Severity | Needs (data) | Status | Notes |
|----|------------|----------|--------------|--------|-------|
| B-001 | All required `sql_processing_paths` entities loaded with rows > 0 | blocker | T1 entity counts | idea | Define “required” per data_model |
| B-002 | `valid_emp` count vs `df_mpd_valid_employees` within expected ratio | warning | T1 | idea | |
| B-003 | Date range on process matches loaded parameter CFG | warning | T0 + T1 | idea | |
| B-004 | Alcampo vs Salsa: expected entity set present for model | blocker | T1 + config | idea | |
| | _your ideas_ | | | idea | |

### Checkpoint C — Transforms → `medium_data`

| ID | Validation | Severity | Needs (data) | Status | Notes |
|----|------------|----------|--------------|--------|-------|
| C-001 | `medium_data` contains `df_colaborador`, `df_calendario`, `df_estimativas` | blocker | T1 keys | idea | From allocation_cycle guard |
| C-002 | Calendar covers `[start_date, end_date]` (or extended passado window) without gaps | warning | T1 or T2 | idea | |
| C-003 | Estimativas row count vs calendar days × shifts within bounds | warning | T1 | idea | |
| C-004 | `invalid_entities` empty after load_matrices / func_inicializa | blocker | T1 | idea | |
| C-005 | Special-day adjustment applied (Christmas/NY) when year requires | info | T1 flag | idea | |
| | _your ideas_ | | | idea | |

### Checkpoint D — Algorithm & solver

| ID | Validation | Severity | Needs (data) | Status | Notes |
|----|------------|----------|--------------|--------|-------|
| D-001 | Solver status success and stats present | blocker | T1 | idea | |
| D-002 | Coverage % above tenant-configured floor | warning | T1 | idea | |
| D-003 | Failed run has non-empty `allocation_error` captured | info | T1 | idea | |
| D-004 | Algorithm params match CFG defaults unless override documented | info | T1 | idea | |
| D-005 | Duration p95 anomaly vs tenant baseline | warning | T0 history | idea | v2 |
| | _your ideas_ | | | idea | |

### Checkpoint E — Format & integration

| ID | Validation | Severity | Needs (data) | Status | Notes |
|----|------------|----------|--------------|--------|-------|
| E-001 | `format_results` validation passed | blocker | T0 | idea | |
| E-002 | Insert row count > 0 when insert enabled | warning | T1 | idea | |
| E-003 | Unconverted F/D days count = 0 or all logged to WFM | warning | T1 + errors | idea | |
| E-004 | Orchestrator status G only if all postos succeeded | blocker | T0 per posto | idea | |
| E-005 | Insert duration under threshold | info | T0 | idea | |
| | _your ideas_ | | | idea | |

### Cross-cutting / web-app-only

| ID | Validation | Severity | Needs (data) | Status | Notes |
|----|------------|----------|--------------|--------|-------|
| X-001 | Same `wfm_proc_id` + `posto_id` not duplicated in index | blocker | Ingest idempotency | idea | |
| X-002 | `deployment_version` present for every run | info | T0 | idea | |
| X-003 | Schema drift: entity row count vs 7-day median | warning | T1 history | idea | v2 |
| | _your ideas_ | | | idea | |

---

## Feature ideas (not validations)

_Use for UI, ingest, ops — still tag with checkpoint if relevant._

| ID | Idea | Checkpoint | Phase | Status | Notes |
|----|------|------------|-------|--------|-------|
| F-001 | Email/Slack digest of overnight failures | A, D | v2 | idea | |
| F-002 | Export execution audit PDF for ticket | all | v2 | idea | |
| F-003 | “Copy correlation bundle” for Slack | all | v1 | idea | tenant + proc + posto + version |
| | _your ideas_ | | | idea | |

---

## New repo — suggested doc layout

Copy or recreate these in the GitHub repo root:

```text
README.md                 # one-liner, link to docs, how to run ingest (later)
docs/
  01-discovery.md         # copy from execution-analytics-platform_discovery_document.md
  02-vision-backlog.md    # this file
  03-telemetry-schema.md  # execution_audit_v1 (when ready)
  04-ingestion-strategy.md
LICENSE                   # if applicable
```

Keep **01** as stable ground truth from algortimo-gd; keep **02** as the messy, frequently edited backlog.

---

## Changelog (optional)

| Date | Change |
|------|--------|
| 2026-10-26 | Initial template |
