# Vision and validations backlog — algoritmo_GH_PD

Copied from `algoritmo_GH_PD` `docs/execution-analytics-platform_vision-and-validations-backlog.md` on 26/10/2026. Platform index: [docs/02-vision-and-validations.md](../../02-vision-and-validations.md).

Template for product ideas. Rows with status `idea` that were seeded from code are guards, status letters, or error codes that already exist. They are not a committed validation suite.

Empty rows are for the product owner. Do not treat this file as an implementation plan.

Project slug: `algoritmo-gh-pd`. Checkpoint letters match `execution-analytics-platform_discovery_document.md`.

Severity vocabulary (product): `blocker` | `major` | `minor` | `info`. Status vocabulary: `idea` | `accepted` | `rejected` | `shipped`.

## A — Job structure

External process, parent/child, status, concurrency. Code: `orquestrador.py`, `main.py`, `AlgorithmConstructor.setup_env`.

| ID | Validation | Severity | Data needed | Status | Notes |
| --- | --- | --- | --- | --- | --- |
| A-01 | `process_type` is 2 or 3 before any child starts | idea | parent argv, exit path | idea | Parent prints and exits otherwise (`orquestrador.py`). |
| A-02 | Claim to status `P` succeeds before `main.py` is spawned | idea | `set_process_status` return value | idea | Return other than 1 logs `errCallSubProc` and does not spawn. |
| A-03 | Child with zero valid employees ends as parameter status `I` and does not continue | idea | `GET_PROCESS_VALID_EMPLOYEES_V2` row count, parameter status | idea | `AlgorithmConstructor.validate_process` logs `errNoColab` and `sys.exit()`. |
| A-04 | Uncaught exception in `runner.run()` ends as parameter status `I` | idea | exception flag, `SET_PROCESS_PARAMETER_STATUS` | idea | `main.py` `except` path, `type_error='E'`, `errCallSubProc`. |
| A-05 |  |  |  | idea |  |
| A-06 |  |  |  | idea |  |

## B — Data loading

Inputs. Code: `GlobalDataInitialization`, `SchedulerDataHandler` loaders, `GetGlobalData`.

| ID | Validation | Severity | Data needed | Status | Notes |
| --- | --- | --- | --- | --- | --- |
| B-01 | Exactly one `FK_UNIDADE` and one `FK_SECAO` on the valid-employee set | idea | distinct counts | idea | `extract_identifiers` raises `ValueError` otherwise. |
| B-02 | Granularity is present, positive, and divides 60 | idea | `GRANULARIDADE_ESCALA`, `secao_id` | idea | Raised in `ESCRunner.run` and again in `SchedulerMain.run`. |
| B-03 | `faixas_sec` is non-empty or the run fails before the posto loop | idea | `get_faixa_sec` row count, error row `getFaixaSec` | idea | Empty frame raises after `errFaixaSec`. |
| B-04 |  |  |  | idea |  |
| B-05 |  |  |  | idea |  |

## C — Transforms to solver inputs

Weekly template and per-day collaborator rows. Code: `ContractCalendar`, `TemplateSelector`, `preprocess_week_templates`.

| ID | Validation | Severity | Data needed | Status | Notes |
| --- | --- | --- | --- | --- | --- |
| C-01 | A week with no selected template records `AE_ERRO_TEMPLATE` on that employee’s T-days | idea | `invalid_days_by_employee_date`, `template_history` | idea | `register_invalid_week` in `SchedulerMain`. |
| C-02 | Template selection logs `AE_ERRO_TEMPLATE` or `AE_ERRO_DISPONIBILIDADE` when no candidate remains | idea | `SET_PROCESS_ERRORS` rows with `process_type=TEMPLATE_SELECTION` | idea | `TemplateSelector` uses `type_error='L'`. |
| C-03 |  |  |  | idea |  |
| C-04 |  |  |  | idea |  |

## D — Solver execution

Daily CBC and post-solve rules. Code: `SchedulerMain.create_schedule`, `SchedulerValidator`.

| ID | Validation | Severity | Data needed | Status | Notes |
| --- | --- | --- | --- | --- | --- |
| D-01 | Each posto-day records solver status and elapsed seconds | idea | `LpStatus`, solve seconds from `create_schedule` | idea | Status strings include PuLP names plus `Error` on exception. Infeasible hard mode may run a second “SOFT fallback” solve. |
| D-02 | An employee with zero valid shift configs is registered `AE_ERRO_GENERICO_HORARIOS` | idea | invalid-day registry, `process_type=SCHEDULE_GENERATION` | idea | `create_schedule` when `config_count==0`. |
| D-03 | Hard-rule hits are visible even while invalid-day registration is off | idea | validator counts by rule name; DB rows `SCHEDULE_VALIDATION` / `HARD_RULE_VIOLATION` | idea | Call site sets `is_registering_invalid_days=False`. Rules: `pause_window`, `mandatory_break`, `break_duration`, `availability_window`, `interjornada`, `closure`. |
| D-04 |  |  |  | idea |  |
| D-05 |  |  |  | idea |  |

## E — Format and integration

CSV shaping and Oracle write. Code: `create_schedule_dataframe`, `insert_schedule_batch`, type-3 `UPDATE`.

| ID | Validation | Severity | Data needed | Status | Notes |
| --- | --- | --- | --- | --- | --- |
| E-01 | Type 2 insert target is `wfm.int_schedule_algorithm` and `rows_inserted` is recorded | idea | return value of `insert_schedule_batch`, `FK_PROCESSO` | idea | Exception returns 0 and does not raise. Process status can still become `G`. |
| E-02 | Invalid-day rows leave the run as `SCHED_TYPE=Z` with `OPTION_C1` from `InvalidDayCodes` | idea | insert payload counts by `TYPE`/`OPTION_C1` | idea | Codes in code: `AE_ERRO_TEMPLATE`, `AE_ERRO_GENERICO_HORARIOS`, `AE_ERRO_DISPONIBILIDADE`, `AE_ERRO_DESC_ENTRE_JORNADAS`, `AE_ERRO_CARGA_AUS_FAIXA_HOR`. |
| E-03 | Type 3 writes clocks on `wfm.CORE_SCHED_ALGORITHM_READ` and sets `RESPONSE_IND=Y` | idea | update count, `FK_PROCESSO` | idea | Separate path from the insert. Do not score type 3 with the type 2 row count. |
| E-04 |  |  |  | idea |  |
| E-05 |  |  |  | idea |  |

## Web-app feature ideas

Mapped to checkpoints. Empty on purpose beyond a few navigation ideas that follow from the pipeline. Not a UI stack decision.

| ID | Feature | Checkpoints | Status | Notes |
| --- | --- | --- | --- | --- |
| UI-01 | Run list filtered by `project=algoritmo-gh-pd`, `wfm_proc_id`, `process_type`, final status `G`/`I` | A, E | idea | Same list component as algortimo-gd if `project` is a column. |
| UI-02 | Posto timeline: A→E for one `(wfm_proc_id, posto_id)` | A–E | idea | This project has no stage registry; the timeline is the checkpoint array. |
| UI-03 | Day strip for checkpoint D: solver status, seconds, fallback flag, violation rule counts | D | idea | One CBC solve per day. Do not reuse GD’s single `allocation_cycle` chart without a day axis. |
| UI-04 | Outcome panel: `rows_inserted` vs `Z` counts by `OPTION_C1`, or type-3 `rows_updated` | E | idea | Employee identifiers on the wire should follow the PII decision (código vs matrícula). |
| UI-05 |  |  | idea |  |
| UI-06 |  |  | idea |  |

## Reminder

Validations and UI rows are product ideas. Discovery only seeded checks that the current code already enforces or already logs. Accept, reject, or rewrite them before any emitter is built.
