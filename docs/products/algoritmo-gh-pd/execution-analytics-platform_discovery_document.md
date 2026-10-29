# Execution analytics discovery — algoritmo_GH_PD

Copied from `algoritmo_GH_PD` `docs/execution-analytics-platform_discovery_document.md` on 26/10/2026. That repository remains the code ground truth. Shared platform scope is in [docs/01-discovery.md](../../01-discovery.md).

Discovery only. This document inventories what the code does today. It does not choose ingestion (push/pull), the audit database, or the UI stack.

Workspace: `algoritmo_GH_PD` (folder `algoritmo_GH_PD`). Suggested audit `project` slug: `algoritmo-gh-pd`. WFM process codes in code: `AlgoritmoHorariosPython_Pai` (parent) and `AlgoritmoHorariosPython_Filho` (child). A second string, `AlgoritmoHorarios_Filho_PYTHON`, is set on `AlgorithmConstructor.proc_code` and is not passed to the status writers.

Sister project used for alignment: `algoritmo_GD` (audit slug in the prior pass: `algortimo-gd`). Comparison is limited to what is in that repo’s `orquestrador.py`, `src/services/algoritmo_gd.py`, and `src/settings/project_structure.py`, plus the checkpoint keys already agreed for that project.

Existing narrative in `algoritmo_GH_PD`: `architecture.md`. Where it disagrees with the files below, the code wins.

## Summary

`algoritmo_GH_PD` is a Python workforce **horários** generator. An external caller starts `orquestrador.py`. That parent reads pending rows from WFM Oracle (`esc_processo` via `S_PROCESSO.GET_PROCESS_BY_STATUS`), marks one process `P`, and starts `main.py` as a subprocess. The child loads employees and section context, then for each posto runs a weekly template selector and a **per-day PuLP/CBC** shift model, and writes schedules back to Oracle.

There is no stage registry. Stages exist only as the call order inside `ESCRunner` → `SchedulerMain`. The A–E checkpoint frame still fits. Algortimo-gd’s named substages (`treat_params`, `load_matrices`, `func_inicializa`, `allocation_cycle`, `format_results`, `insert_results`) are **not** present here and should not be copied as this project’s checkpoint names.

### Recommended workflow

1. **Inventory first** (this document): entrypoints, checkpoints, keys, what is already stored.
2. **Vision / validations backlog**: product ideas per checkpoint (`execution-analytics-platform_vision-and-validations-backlog.md`). Seed rows below are only guards and error codes that already exist in code.
3. **New repo docs**: copy these files into `tlantic-scheduler-execution-audit` and keep `project = algoritmo-gh-pd` beside `algortimo-gd`.
4. **Implement later**: emitters, storage, and UI. Do not lock those choices in this pass.

Audit principles carried over: metadata first, fail-open emitters, no hard platform decisions without a separate decision.

## Entrypoints

| Role | File | How it starts | What it does |
| --- | --- | --- | --- |
| Parent orchestrator | `orquestrador.py` | Script execution (no `if __name__` guard). Comment: “GET ARG BY API”. | Oracle connect, concurrency params, poll processes, `Popen` child, status/error writes. |
| Child batch | `main.py` | `subprocess.Popen([sys.executable, "main.py", ...])` from the parent, or a direct debug launch. | Parse argv, connect, build runner, `runner.run()`, set parameter status `G` or `I`. |
| Runner factory | `Functions/Algorithm_Functions/get_algorithm_runner.py` | Called by `main.py`. | `ESC` → `ESCRunner`. `MPD` → `MPDRunner`. Anything else raises `ValueError`. |
| ESC pipeline | `Classes/AlgorithmBaseClasses/ESCRunner.py` | `runner.run()`. | Valid employees → global init → `SchedulerMain.run` per `posto_id`. |
| MPD | `Classes/AlgorithmBaseClasses/MPDRunner.py` | Same factory. | `setup_env()` then a print. Not a scheduling pipeline. The factory call omits `process_type` and `connection_handler`, which `AlgorithmRunner.__init__` requires, so selecting `MPD` does not match the constructor. |
| Child argument object | `Classes/AlgorithmPrepClasses/Orquestrador.py` | `main.py` after connect. | `product` → `Argument_Handlers/argument_loader.py`. `debug` → `Argument_Handlers/manual_config.py` (`get_manual_arguments`). That module is gitignored (`*manual_config.py`) and is not in the tree. |

No HTTP route, queue consumer, or scheduler library lives in this repo. The external trigger is whatever process executes `orquestrador.py` and passes arguments.

### Parent arguments (`orquestrador.py`)

When `len(sys.argv) > 2`:

| Index | Name in code | Use |
| --- | --- | --- |
| `sys.argv[1]` | `api_proc_id` | `fk_process` on parent info/error rows (`iniProc`, `errMaxThreads`). |
| `sys.argv[2]` | `api_user` | User string on those parent rows. |
| `sys.argv[3]` | `process_type` | Must be `2` or `3`. Otherwise the script prints and `sys.exit()`. |

If `len(sys.argv) <= 2`, the script sets `api_proc_id = 999` and `api_user = 'WFM'` and does **not** assign `process_type` before the `process_type` check.

### Child arguments (parent `Popen` list)

`orquestrador.py` launches:

```text
python main.py
  [1] api_proc_id          parent argv id
  [2] wfm_proc_id          sec_to_proc.CODIGO
  [3] wfm_user             sec_to_proc.USER_CRIACAO
  [4] data_ini             DATA_INI as dd-mm-YYYY
  [5] data_fim             DATA_FIM as dd-mm-YYYY
  [6] wfm_proc_colab       FK_COLABORADOR or the string "None"
  [7] child index          i+1
  [8] generation_type      literal "product"
  [9] algorithm_to_run     literal "ESC"
  [10] process_type        "2" or "3"
```

`main.py` does **not** use that numbering for its own names:

| `main.py` variable | Actual argv | Effect |
| --- | --- | --- |
| `api_proc_id` | `sys.argv[2]` | WFM `CODIGO`, used as `fk_process` / `process_id` on child status and fatal-error writes. |
| `api_user` | `sys.argv[3]` | WFM `USER_CRIACAO`. |
| `generation_type` | `sys.argv[8]` | `product` (or `debug` on the fallback path). |
| `algorithm_to_run` | `sys.argv[9]` | `ESC` from the parent. |
| `process_type` | `sys.argv[10]` | `2` or `3`. |

`Argument_Handlers/argument_loader.py` reads `sys.argv[1]` as `api_proc_id` and `sys.argv[2]` as `wfm_proc_id`. `Classes/AlgorithmPrepClasses/Orquestrador.get_process()` returns `wfm_proc_id`, user, dates, collaborator, and child number. It does **not** return the parent `api_proc_id`. The child logger process id is `wfm_proc_id`.

If `main.py` cannot parse argv, it falls back to `algorithm_to_run='ESC'`, `generation_type='debug'`, `api_proc_id='999'`, `api_user='PYTHON_DEBUG_CLIENT'`, `process_type=2`, then still connects to Oracle.

Accepted algorithms in `main.py`: `ESC`, `MPD`. The parent always passes `ESC`.

## End-to-end pipeline

```text
External caller
  (process that executes orquestrador.py; not defined in this repo)
        |
        v
[A] orquestrador.py
  ConnectionHandler -> Connection/connect.py
    acessos.txt, cx_Oracle, ALTER SESSION CURRENT_SCHEMA=WFM
  CORE_ALG_PARAMETERS: GLOBAL_MAX_CONCURRENT_PROCESSES,
    LOCAL_MAX_CONCURRENT_PROCESSES, MAX_RETRIES, RETRY_WAIT_TIME
  S_PROCESSO.GET_PROCESS_BY_STATUS
    type 2: user WFM, process_type ESC, event 2, status N
    type 3: user WFM, process_type ESC, event 3, status P
  S_PROCESSO.get_total_process_by_status('ESC','P')
  if a slot exists:
    S_PROCESSO.SET_PROCESS_STATUS(..., 'P') on esc process CODIGO
    subprocess.Popen(main.py, ...)
    os.kill(parent pid, SIGTERM)   # inside the spawn loop, after the first successful Popen
        |
        v
[A] main.py
  reject algorithm not in {ESC, MPD}
  connect + Logger (messages lang ES)
  Orquestrador.get_process()  -> wfm_proc_id, dates, user, colab, child_number
  get_algorithm_runner(...)
        |
        +-- MPDRunner.run: setup_env + print. Stop. (constructor arity does not match the factory)
        |
        v
[A/B] ESCRunner.run
  AlgorithmRunner.setup_env
    AlgorithmConstructor.initialize (TZ=GMT, random.seed(123))
    log iniSubproc
    get_process_valid_emp -> S_PROCESSO.GET_PROCESS_VALID_EMPLOYEES_V2
    empty frame: log E errNoColab, SET_PROCESS_PARAMETER_STATUS 'I', sys.exit()
  GlobalDataInitialization.initialize
    unit_id, secao_id, posto_id list, employee_roles from the employee frame
    get_all_params -> CORE_ALG_PARAMETERS
    get_gran_equi -> s_pck_core_parameter.getnumberattr('GRANULARIDADE_ESCALA','S', secao)
    get_faixa_sec -> Data/Queries/GlobalData/get_faixas_sec.sql
  if granularity missing or "Error": raise ValueError
  SchedulerMain.run(postos, ...)
        |
        v
[B/C/D/E per posto] SchedulerMain.run
  reject granularity <= 0 or not a divisor of 60
  SchedulerDataHandler (loads Templates_horarios.csv)
  TemplateSelector + SchedulerValidator(is_registering_invalid_days=False)
  is_closure_enabled_for_unit(unit_id)
  temp dir cbc_solver_{wfm_proc_id}_
  for each posto_id:
        |
        v
    [B] run_scheduler_for_post — loads
      colab ids for posto, date sequence, faixas_posto from faixas_sec
      esc_tmp_pdv_ideal (min/ideal), esc_feriado tipo 3, absences,
      horas certas, max_workers_on_break, CORE_PRO_EMP_HORARIO_DET (pré-definidos)
      type 2: core_pro_emp_contract + CORE_PRE_SCHEDULE_ALGORITHM,
              core_pro_emp_availability, CORE_PRO_EMP_MOV (FC),
              CORE_PRO_CONTRACT_EXCEPTION
      type 3: core_sched_algorithm_read + core_contract_labor
      both: esc_horario_colaborador extended history
      empty type-2 contract frame or empty type-3 generated frame: return 0
        (caller ignores the return; main can still set status G)
        |
        v
    [C] transforms into solver inputs
      ContractCalendar day types
      preprocess_week_templates -> TemplateSelector.select_best_pattern_for_week
      fixed-shift map, FC credits, interval exceptions
      per date: collaborator rows (hours, break, window, interjornada)
      type 2: build_interjornada_lookahead caps time_frame_end
      demand pattern from matriz_min_ideal (+ closure windows)
        |
        v
    [D] create_schedule
      PuLP LpProblem Employee_Scheduling_Day_{n}
      PULP_CBC_CMD(msg=True, timeLimit=20, gapRel=0.01, keepFiles=False)
      infeasible + role slacks: second solve (SOFT fallback)
      SchedulerValidator.validate_day + report
        |
        v
    [E] after the day loop, still inside the posto
      assign_unused_template_hours_to_ta
      create_schedule_dataframe -> employee_schedules.csv
        rows T, T/A, R, and Z (OPTION_C1 = invalid-day code)
      type 2: map CODIGO -> MATRICULA, merge calendar TYPE/SUBTYPE,
              shape OPTION_*, write employee_schedules_insert.csv,
              INSERT wfm.int_schedule_algorithm
      type 3: UPDATE wfm.CORE_SCHED_ALGORITHM_READ
              (HORA_INI/FIM_*, RESPONSE_IND='Y') per row
  after all postos: delete CBC temp dir
        |
        v
[A/E] main.py if runner.run() returns
  SET_PROCESS_ERRORS type I, message endSubproc
  SET_PROCESS_PARAMETER_STATUS 'G'
  disconnect
  kill every local process whose name contains cbc.exe
  sys.exit() with no status (exit code 0)

[A/E] main.py if runner.run() raises Exception
  SET_PROCESS_ERRORS type E, message errCallSubProc
  SET_PROCESS_PARAMETER_STATUS 'I'
  disconnect, kill cbc.exe, sys.exit(1)
```

`AlgorithmConstructor` empty-employee path calls `sys.exit()` with no argument **after** status `I`. `SystemExit` is not an `Exception`, so `main.py` does not overwrite that with `G`.

## Audit checkpoints

A–E matches this pipeline. Use them as the shared axis with algortimo-gd. Under each letter, this project’s real substages are functions, not a config registry.

### A — External process / job structure

| Substage | Module | What is observable |
| --- | --- | --- |
| A1 parent start | `orquestrador.py` | Connects, reads concurrency parameters, rejects `process_type` other than 2 or 3. |
| A2 poll | `Functions/WFM_Process/Getters.py` `get_process_by_status`, `get_total_process_by_status` | SQL files under `Data/Queries/WFM_Process/Getters/`. |
| A3 concurrency wait | `orquestrador.py` loop | If count of `ESC` + status `P` ≥ global max, log `iniProc` (`type_error='I'`), sleep `RETRY_WAIT_TIME`, increment retries. The pending dataframe is **not** re-queried inside the loop. |
| A4 claim | `set_process_status(..., status='P')` | Only if the setter returns 1 does the child start. Failure logs `errCallSubProc` with `type_error='E'`. |
| A5 spawn | `subprocess.Popen` then `os.kill(os.getpid(), signal.SIGTERM)` | Signal is sent on the first successful child, still inside the `for` loop. `local_processes` is never incremented. |
| A6 child boot | `main.py`, `get_algorithm_runner` | Algorithm gate. Logger bound to `wfm_proc_id`. |
| A7 child env | `AlgorithmRunner.setup_env` → `AlgorithmConstructor` | Timezone `GMT`, seed `123`, info log `iniSubproc`, employee gate. |
| A8 terminal status | `main.py` | Success: parameter status `G` plus info row `endSubproc`. Fatal exception: parameter status `I` plus `type_error='E'`. Empty employees: parameter status `I` then `sys.exit()`. |

Status letters used in this repo:

| Letter | Writer | Where |
| --- | --- | --- |
| `N` | Read only | Parent type 2 poll (`get_process_by_status` status `N`). |
| `P` | `S_PROCESSO.SET_PROCESS_STATUS` | Parent before spawn. Also the type 3 poll filter. |
| `G` | `S_PROCESSO.SET_PROCESS_PARAMETER_STATUS` | Child after `runner.run()` returns. |
| `I` | Parameter status and, on spawn failure, `SET_PROCESS_STATUS` | Invalid/interrupted. |
| `E` | Not a process status | `type_error` on `SET_PROCESS_ERRORS` (error row, not the process state letter). |

`type_error` values written by this code: `I` (info), `E` (error), `L` (template / keyholder logs), `W` (validator and closure).

Process-type strings stored on error rows (not the same as numeric `process_type` 2/3):

- Parent: `AlgoritmoHorariosPython_Pai`
- Child fatal/success rows in `main.py`: `AlgoritmoHorariosPython_Filho`
- Employee gate: `getProcessValidEmp`
- Init info: `Initialization`
- Later domain rows: `get_all_params`, `getFaixaSec`, `get_gran_equi`, `TEMPLATE_SELECTION`, `SCHEDULE_GENERATION`, `SCHEDULE_VALIDATION`, `CLOSURE_VALIDATION`, `KEYHOLDERS_RULE_VALIDATION`, `FIXED_SHIFT_VALIDATION`, `DATA_VALIDATION`, `TA_ASSIGNMENT`, `DAY_LOOP_FALLBACK`

Concurrency parameters read from `CORE_ALG_PARAMETERS` (`SYS_P_NAME`): `GLOBAL_MAX_CONCURRENT_PROCESSES`, `LOCAL_MAX_CONCURRENT_PROCESSES`, `MAX_RETRIES`, `RETRY_WAIT_TIME`. If a value is missing, defaults are 3, 1, 1, and 0. The `max(...)` calls in `orquestrador.py` do not assign their result, so they do not change the stored number.

`get_process_by_id` is imported in `orquestrador.py` and implemented in `Getters.py` (`select * from esc_processo where codigo = :i`). Nothing in the parent calls it.

### B — Data loading / inputs

| Substage | Function | Source |
| --- | --- | --- |
| B1 valid employees | `AlgorithmConstructor.validate_process` → `get_process_valid_emp` | `S_PROCESSO.GET_PROCESS_VALID_EMPLOYEES_V2(:process_id)` |
| B2 identifiers | `GlobalDataInitialization.extract_identifiers` | Columns on that frame: `FK_UNIDADE`, `FK_SECAO`, `FK_TIPO_POSTO`, `FK_COLABORADOR`, `PRIORIDADE_FOLGAS`. Exactly one distinct unit and one distinct section, or `ValueError`. Postos are the distinct `FK_TIPO_POSTO` list. |
| B3 roles | same | `PRIORIDADE_FOLGAS` 1 → `manager`, 2 → `keyholder`, else `normal`. |
| B4 parameters | `get_algorithm_parameters` → `GetGlobalData.get_all_params` | `Data/Queries/GlobalData/get_all_parameters.sql` on `CORE_ALG_PARAMETERS`. Filtered in Python by unit, section, posto. |
| B5 granularity | `getGranularity` → `get_gran_equi` | `s_pck_core_parameter.getnumberattr('GRANULARIDADE_ESCALA','S', :secao)`. |
| B6 section windows | `validate_faixas_sec` → `get_faixa_sec` | `get_faixas_sec.sql` (`S_SECAO.getFaixa`, `S_FERIADO.verificaFeriado`). Empty frame raises. |
| B7 posto slice | `SchedulerDataHandler.get_colaborator_ids`, `initialize_date_sequence`, `fetch_schedule_data` | In-memory filter and slot expansion. Dates in argv are `dd-mm-YYYY`. |
| B8 demand, holidays, absences, rules | `load_matriz_min_ideal`, `load_holidays_tipo_3`, `get_absences_break`, `load_horas_certas_param`, `load_max_workers_on_break`, `load_horarios_pre_definidos_df`, `load_extended_schedule` | See catalog. |
| B9 type 2 labor | `load_colab_contract_info`, `load_time_frame`, `load_folga_compensatoria_mov_df`, `load_interval_exceptions` | See catalog. Empty contract frame returns 0 from the posto function. |
| B10 type 3 prior schedules | `load_generated_schedules`, `load_contract_info` | `wfm.core_sched_algorithm_read` joined to `wfm.core_pro_emp_availability`, then `core_contract_labor`. Empty schedule frame returns 0. |
| B11 templates file | `SchedulerDataHandler._load_weekly_templates` | `Data/Queries/Templates_Schedule/Templates_horarios.csv` (`;`, columns `weekly_hours`, `rest_days`, `pattern`, `working_days`, `id`). |
| B12 messages | `message_loader.get_messages` | `Data/Translations/Traducoes_algoritmo_horarios_trads.csv`, language hardcoded `ES` in `main.py` and `orquestrador.py`. |

`ESCRunner.get_all_params` also asks `GlobalDataInitialization.get_param` for `geracaoType`, `sundayInterjornadas`, and `saturdayInterjornadas`. `ESCRunner.run` only forwards `granularity` into `SchedulerMain`. The other three values are not used after that call.

Present in `SchedulerDataHandler` but **not called** from `SchedulerMain` or `ESCRunner`: `load_calendar_matrix`, `update_calendar_with_preloaded`, `load_collaborator_matrix`. Matching SQL files exist at `Data/Queries/PostData/matriz_calendario.sql` and `matriz_colaborador.sql`. The live type-2 calendar path is `load_colab_contract_info` (inline SQL), not those files.

`SchedulerDataHandler.log_process_error` only contains a commented print. Load failures there do not reach `SET_PROCESS_ERRORS` unless a caller raises.

### C — Transforms → model / solver inputs

| Substage | Function | Output kept in memory |
| --- | --- | --- |
| C1 day types | `ContractCalendar` | Views: `work_days` (`TYPE='T'` and `SUBTYPE!='A'`), `ta_days` (`TYPE='T'` and `SUBTYPE='A'`), `folga_compensatoria_days`, `r_only_days`, `n_only_days`. |
| C2 fixed shifts | `build_fixed_shift_by_employee_date` | Map `(employee_id, schedule_date)` → pinned shift. Off-grid rows log `FIXED_SHIFT_VALIDATION`. |
| C3 week template | `preprocess_week_templates` → `TemplateSelector.select_best_pattern_for_week` | `template_history[(employee, monday)]` with `daily_assignment`. Failure registers invalid week `AE_ERRO_TEMPLATE` via `register_invalid_week` (every T-day of that week). |
| C4 FC / locks / intervals | `resolve_folga_compensatoria_credits_for_week`, locked-hour filters, `resolve_interval_for_employee_day` | Adjusted hours and break bounds for the day row. |
| C5 day rows (type 2) | loop in `run_scheduler_for_post` | DataFrame columns include `FK_COLABORADOR`, `OPTION_N1` (contract id), `hours`, break min/max, pause min/max, `interjornada`, `bandahoraria`, `break_bool`, `time_frame_start`, `time_frame_end`. |
| C6 day rows (type 3) | same loop, other branch | Built from already generated `TIPO='T'` rows and `core_contract_labor` fields (`MINIMUMMEALPERIOD`, `MAXIMUMMEALPERIOD`, `MINIMUMCONTINUOUSWORK`, `MAXIMUMCONTINUOUSWORK`, `HOURBETWEENWORKDAY`, `BANDWIDTHHOUR`). `OPTION_C4='Y'` marks not-to-change and subtracts one slot from end times before the solve. |
| C7 lookahead | `build_interjornada_lookahead`, `apply_lookahead_caps_to_df` | Type 2 only. Tightens `time_frame_end`. Skipped when `template_history` says `is_full_week_pre_definido`. |
| C8 demand vector | `create_demand_pattern_from_matriz` | Per-slot demand for that date. Closure windows attached when the unit flag is on. |
| C9 demand ranking | `process_scheduling_with_weekly_analysis` | `highest_demand_dates` passed into template selection. |

Type 2 is “choose weekly hours, then place them with CBC”. Type 3 is “read a prior algorithm schedule for this `FK_PROCESSO`, then place those hours”. Both still enter `create_schedule`.

### D — Algorithm / solver execution

| Substage | Function | Notes |
| --- | --- | --- |
| D1 model build | `SchedulerMain.create_schedule` | `LpProblem(..., LpMinimize)`. Binary configs `(employee, start_slot, break_slots, work_slots_before_break)`. Employees with zero valid configs are not forced to a shift; `register_invalid_day(..., AE_ERRO_GENERICO_HORARIOS, process_type='SCHEDULE_GENERATION')`. |
| D2 closure filter | `ClosureRules.build_closure_slots` and helpers | Used while generating configs. All-rejected case logs `CLOSURE_VALIDATION` / `CLOSURE_NO_FIT` (`type_error='W'`). |
| D3 role constraints | same method | Managers/keyholders opening and closing. Infeasible hard mode releases slacks and solves again. Missing open and close ability logs `KEYHOLDERS_RULE_VALIDATION` / `SOLVER` (`type_error='L'`). |
| D4 solve | `PULP_CBC_CMD` | `timeLimit=20`, `gapRel=0.01`, `keepFiles=False`, `msg=True`. Stdout captured by `capture_solver_output_to_logger` into the Python logger (`[SOLVER]`). Progress lines at INFO, other lines at DEBUG. |
| D5 status | `LpStatus[prob.status]` | Logged with elapsed seconds. Exception sets status string `Error`. |
| D6 post-solve checks | break monotonicity block, role summary | File/console logger. Role summary is counts of opens/closes, not a DB write. |
| D7 hard rules | `SchedulerValidator.validate_day` then `report` | Rules: `pause_window`, `mandatory_break`, `break_duration`, `availability_window`, `interjornada`, `closure`. File warning plus `SET_PROCESS_ERRORS` (`type_error='W'`, `process_type='SCHEDULE_VALIDATION'`, `error_code='HARD_RULE_VIOLATION'`, employee and schedule day). |

`SchedulerMain.run` constructs the validator with `is_registering_invalid_days=False`. Violations are logged. They do **not** become `SCHED_TYPE='Z'` rows unless that flag is changed. The class default is `True`; the call site overrides it.

Pré-definido employees (`skipped_employee_ids` / `fixed_schedule`) are not judged by the validator.

Solver working files: directory from `tempfile.mkdtemp(prefix=f"cbc_solver_{proc_code}_")`, assigned to `TMP`/`TEMP`/`TMPDIR`. `keepFiles=False`. The directory is removed after the posto loop with `shutil.rmtree`. Removal is not in a `finally`; an exception skips it. `main.py` then kills every process whose name contains `cbc.exe`, not only this run’s solver.

Large days (`total_slots > 20 * slots_per_hour`) use `start_step=2` and `break_position_step=2`.

### E — Format results / integration / side effects

| Substage | Function | Side effect |
| --- | --- | --- |
| E1 leftover TA hours | `TemplateSelector.assign_unused_template_hours_to_ta` | Fills `ta_hours_by_date` / related maps before the dataframe build. |
| E2 schedule frame | `create_schedule_dataframe` | Columns `EMPLOYEE_ID`, `SCHEDULE_DT`, `SCHED_TYPE`, `SCHED_SUBTYPE`, start/end 1 and 2, `MEAL_TYPE`. Worked solver columns become `SCHED_TYPE='T'`. TA map becomes `T`/`A`. `r_hours_by_date` becomes `R`. Invalid-day registry becomes `Z` with `OPTION_C1` = error code (`InvalidDayCodes.INVALID_DAY_SCHED_TYPE`). |
| E3 local CSV | same, default `employee_schedules.csv` | Overwritten per posto (fixed filename). |
| E4 type 2 shape | block `process_type == 2` in `run_scheduler_for_post` | `load_matricula_for_ids` maps `esc_colaborador.CODIGO` → `MATRICULA` and **replaces** `EMPLOYEE_ID` before insert. Left-merge onto contract `TYPE`/`SUBTYPE`. Split shifts: `OPTION_TYPE='P'`; `OPTION_N1` is the gap in minutes; `OPTION_C1` becomes `P` if gap ≥ 60 else `C` if gap &gt; 0. Invalid rows force `TYPE='Z'` and keep `OPTION_C1`. |
| E5 type 2 insert file | `result.to_csv("employee_schedules_insert.csv")` | Same overwrite behaviour. Gitignored. |
| E6 type 2 DB insert | `insert_schedule_batch` | `INSERT INTO wfm.int_schedule_algorithm` (`FK_PROCESSO`, `DT_CREATE=sysdate`, `EMPLOYEE_ID`, `SCHEDULE_DT`, `SCHED_TYPE`, `SCHED_SUBTYPE`, four clock columns, `OPTION_TYPE`, `OPTION_C1`, `OPTION_N1`, `MEAL_TYPE`). Commit. On exception returns 0 and prints; the posto function still returns 1. |
| E7 type 3 update | inline `UPDATE` in `run_scheduler_for_post` | `wfm.CORE_SCHED_ALGORITHM_READ` set `HORA_INI_1/FIM_1/INI_2/FIM_2`, `RESPONSE_IND='Y'` where `FK_COLABORADOR`, `DATA`, `FK_PROCESSO`. One commit per row. |
| E8 child close | `main.py` | Info error-row `endSubproc`, parameter status `G`, disconnect, `cbc.exe` kill. |

`Data/Queries/Insert_read_schedules/insert_schedules.sql` is a comment-style draft and is **not** what `insert_schedule_batch` executes. The live statement is the string in `Functions/Insert_read_schedules/Insert_schedules.py`. The draft omits `MEAL_TYPE`, which the Python insert includes.

Success status `G` is independent of `rows_inserted`. A zero-row insert, a posto that returned 0, or a validator in log-only mode can still finish as `G` if nothing raises.

`create_schedule` can call `visualize_daily_staffing`, which writes `day_{n}_{date}_staffing_visualization.png` and calls `plt.show()`. The call in the day loop is commented out, so PNGs are not produced on the current path.

## Correlation keys

### Present on a normal parent→child run

| Key | Where it exists | Scope |
| --- | --- | --- |
| `wfm_proc_id` | `esc_processo.CODIGO` via `GET_PROCESS_BY_STATUS`; child `process_id`; insert `FK_PROCESSO`; type 3 `FK_PROCESSO` | One WFM horários process. Best join key to WFM and to `int_schedule_algorithm`. |
| `api_proc_id` | Parent `sys.argv[1]` only | Used as `fk_process` on some **parent** log rows. Not stored on the child logger and not returned by `get_process()`. Do not assume it equals `wfm_proc_id`. |
| `process_type` | Parent argv and child argv `10` | `2` generate-from-contract, `3` place an already generated read-model. Not a data-model name. |
| `algorithm_name` | Child argv `9`, factory | Today always `ESC` from the parent. `MPD` is a second factory branch. |
| `generation_type` | argv `8` | `product` or `debug`. |
| `child_number` | argv `7` (`i+1`) | Index in the parent’s spawn loop. Not a stable job id across retries. |
| `wfm_user` | `USER_CRIACAO` | Passed into status procedures as the user argument. |
| `wfm_proc_colab` | `FK_COLABORADOR` on the process row | Optional single-employee filter. Held as `colab_matricula` when it is not the string `NA`. Naming in code says matricula; the column read is `FK_COLABORADOR`. |
| `unit_id` | `FK_UNIDADE` | One per child run (enforced). |
| `secao_id` | `FK_SECAO` | One per child run (enforced). |
| `posto_id` | `FK_TIPO_POSTO` | **List** on the child. The posto loop is the natural sub-run. One process → many postos, sequential, same Oracle session. |
| `start_date`, `end_date` | `DATA_INI`, `DATA_FIM` | Generation window. Parent formats `dd-mm-YYYY`. |
| `EMPLOYEE_ID` | Solver uses `FK_COLABORADOR` / `esc_colaborador.CODIGO`. Type 2 insert replaces it with `MATRICULA`. | Employee grain inside a posto. Two identifiers. |
| `schedule_day` | Contract, solver, error rows, insert `SCHEDULE_DT` | Day grain. Error procedure expects `yyyy-mm-dd`. |
| `contract_id` | `OPTION_N1` / `CONTRACT_ID` during the solve | Not the same field as insert `OPTION_N1` after type-2 shaping (that becomes a break-gap or a constant). |

`FK_EMPRESA` is read only inside `is_closure_enabled_for_unit` (`esc_unidade`). It is not attached to the run. It is not a tenant id.

No column or variable in this repo is `tenant_id`, `deployment_version`, `audit_run_id`, or `data_model`.

### Keys the audit platform must add

| Key | Why |
| --- | --- |
| `project` | Constant `algoritmo-gh-pd` so one UI can filter beside `algortimo-gd`. |
| `tenant_id` | Not in this codebase. Required for cross-tenant analytics. Do not invent it from `FK_EMPRESA` or the Oracle user in `acessos.txt` until that mapping is confirmed. |
| `deployment_version` | Not in git metadata inside the running process (no version file read at startup). |
| `audit_run_id` | `wfm_proc_id` + `posto_id` is the business grain, but a rerun of the same WFM process would collide if emitters append. |
| `parent_api_proc_id` | Only if the external caller’s id must be kept. Today it dies in the parent except on parent-scoped error rows. |
| `data_model` | Algortimo-gd uses model names (`salsa`, `alcampo`, …). Here the closest existing discriminator is `process_type` (`2` or `3`) plus `algorithm_name=ESC`. Do not overload `data_model` with `process_type` until the product names them. |

Suggested grain for a v1 event: one event per `(project, wfm_proc_id, posto_id)` with checkpoint array A–E. A parent-level event (checkpoint A only) is a separate row keyed by `wfm_proc_id` and, when present, `parent_api_proc_id`.

## What is already persisted vs ephemeral

### Persisted (Oracle, schema session `WFM`)

| Target | How this repo writes or reads it | Grain |
| --- | --- | --- |
| Process status via `S_PROCESSO.SET_PROCESS_STATUS` | `set_process_status.sql` | User, process id, status `P` or `I`. |
| Parameter status via `S_PROCESSO.SET_PROCESS_PARAMETER_STATUS` | `set_process_parameter_status.sql` | User, process id, `G` or `I`. |
| Error/info rows via `S_PROCESSO.SET_PROCESS_ERRORS` | `set_process_errors.sql` | User, `fk_process`, `type_error`, `process_type` string, `error_code`, `description` (bound as `VARCHAR2(242)` in the anonymous block), `employee_id`, `schedule_day`. |
| `esc_processo` | Read by `get_process_by_id.sql` (`select *`). Poll uses the pipelined function, which the parent reads as columns `CODIGO`, `USER_CRIACAO`, `DATA_INI`, `DATA_FIM`, `FK_COLABORADOR`. | Process. |
| `wfm.int_schedule_algorithm` | Insert, type 2 | One row per shaped schedule line. `EMPLOYEE_ID` is matrícula at insert time. |
| `wfm.CORE_SCHED_ALGORITHM_READ` | Read (type 3) and update clocks + `RESPONSE_IND` | Prior schedule for `FK_PROCESSO`. |
| `CORE_ALG_PARAMETERS` | Read | Parameters. Also `max_workers_on_break` and horas-certas lookups in `core_parameter` / `core_parameter_cfg`. |
| Labor and calendar tables | Read only | Listed in the catalog. This repo does not create them. |

`SchedulerValidator` comments name `esc.processos_erros` as the error table. This repo only calls `S_PROCESSO.SET_PROCESS_ERRORS`. The package body is not in the tree, so the physical table name is **not** confirmed here.

`InvalidDayCodes` documents that `OPTION_C1` codes must already exist in `ESC_DICIONARIO`. This repo does not query that table.

### Local files (not an audit store)

| Path | When | Lifetime |
| --- | --- | --- |
| `logs/scheduler_YYYYMMDD_HHMMSS.log` | `SchedulerMain.setup_logger` when `debug_mode=True` (the `SchedulerMain(...)` call in `ESCRunner` uses that default). | One file per `SchedulerMain` instance. Gitignored (`*.log`). Overlaps postos of one process. |
| `log.txt` | `Connection/connect.py` on connect/disconnect failure. Relative path, not under `logs/`. | Append. Gitignored. |
| `employee_schedules.csv` | Every posto, `create_schedule_dataframe`. | Last posto wins. Gitignored. |
| `employee_schedules_insert.csv` | Type 2, before insert. | Last posto wins. Gitignored. |
| `cbc_solver_{wfm_proc_id}_*` temp dir | During solves. | Deleted after the posto loop if no exception. CBC `keepFiles=False`. |
| `day_*_staffing_visualization.png` | Only if `visualize_daily_staffing` is called. | Not on the current day-loop path. Gitignored (`*.png`). |
| `Connection/acessos.txt` | Credentials. | Gitignored. Not an audit artefact. |

Stdout from `main.py` and `orquestrador.py` is printed to the console. The redirect to `os.devnull` is commented out. There is no structured log shipper.

### Ephemeral in-memory

Held on `SchedulerMain` / `TemplateSelector` / `GlobalDataInitialization` and dropped when the process exits:

- `valid_employees`, `parameter_dataset`, `faixas_sec`, `employee_roles`
- per posto: `matriz_min_ideal`, `contract_and_labor_data`, `contract_time_frames`, `interval_exceptions`, `folga_compensatoria_mov_df`, `horarios_pre_definidos_df`, `absences_by_day`
- `template_history`, `pattern_usage_tracker`, `invalid_days_by_employee_date`, `ta_hours_by_date`, `r_hours_by_date`, `accumulated_hours_by_employee_date`
- `banda_horaria_tracking`, `role_history` (`open_count`, `close_count`)
- daily `all_results` (slot grids) and `all_shifts` / `shift_details`
- PuLP problem and CBC status, except the lines copied into the scheduler log
- `SchedulerMain.results` / `get_results`: `ESCRunner.run` returns 1 **before** the commented `get_results()` call, and `run_scheduler_for_post` does not fill `self.results`

Message catalogue keys such as `iniSubprocPosto`, `okInsertWFM`, `errInsertWFM`, `noMinIdeal` exist in the translation CSV and are **not** referenced by current Python. They look like leftovers from the older R flow. Do not treat them as live checkpoints.

## Entity / data catalog

### Config-driven queries (files)

| File | Executed by | Object |
| --- | --- | --- |
| `Data/Queries/WFM_Process/Getters/get_process_by_status.sql` | `get_process_by_status` | `table(S_PROCESSO.GET_PROCESS_BY_STATUS(:user,:process_type,:event_type,:status))` |
| `Data/Queries/WFM_Process/Getters/get_total_process_by_status.sql` | `get_total_process_by_status` | `S_PROCESSO.get_total_process_by_status` → column `TOTAL_P`. Call site forces `ESC` and `P`. |
| `Data/Queries/WFM_Process/Getters/get_process_by_id.sql` | `get_process_by_id` (unused by parent) | `esc_processo` |
| `Data/Queries/WFM_Process/Getters/get_process_valid_employess.sql` | `get_process_valid_emp` | `table(S_PROCESSO.GET_PROCESS_VALID_EMPLOYEES_V2(:process_id))` |
| `Data/Queries/WFM_Process/Setters/set_process_status.sql` | `set_process_status` | `S_PROCESSO.SET_PROCESS_STATUS` |
| `Data/Queries/WFM_Process/Setters/set_process_parameter_status.sql` | `set_process_param_status` | `S_PROCESSO.SET_PROCESS_PARAMETER_STATUS` |
| `Data/Queries/WFM_Process/Setters/set_process_errors.sql` | `set_process_errors` | `S_PROCESSO.SET_PROCESS_ERRORS` |
| `Data/Queries/GlobalData/get_all_parameters.sql` | `get_all_params` | `CORE_ALG_PARAMETERS` columns `FK_UNIDADE`, `FK_SECAO`, `FK_GRUPO`, `FK_TIPO_POSTO`, `SYS_P_NAME`, `CHARVALUE`, `NUMBERVALUE`, `DATEVALUE` |
| `Data/Queries/GlobalData/get_faixas_sec.sql` | `get_faixa_sec` | Section open/close per day (`INI`, `FIM`, `DATA`, `DIA_SEMANA`, `fer_next_day`) |
| `Data/Queries/PostData/matriz_calendario.sql` | Not called. Duplicate logic exists inside `load_calendar_matrix`, also uncalled. | `WFM.CORE_PRE_SCHEDULE_ALGORITHM` ⋈ `wfm.esc_colaborador` |
| `Data/Queries/PostData/matriz_colaborador.sql` | Not called. Inline twin is `load_collaborator_matrix`, also uncalled. | `wfm.core_algorithm_variables` ⋈ `WFM.ESC_COLABORADOR` (includes `nome`, contract hour limits) |
| `Data/Queries/Insert_read_schedules/insert_schedules.sql` | Not executed | Draft `insert into wfm.int_schedule_algorithm` |

Path casing: `Getters.py` opens `Data/Queries/WFM_PROCESS/...` while the directory on disk is `WFM_Process`. That is the same path on Windows.

### Inline SQL that is on the live path (`SchedulerDataHandler` / `SchedulerMain` / `GetGlobalData`)

| Function | Objects | Notable columns |
| --- | --- | --- |
| `get_gran_equi` | `s_pck_core_parameter.getnumberattr` | Attribute `GRANULARIDADE_ESCALA`, level `S`, section id |
| `load_matriz_min_ideal` | `wfm.esc_tmp_pdv_ideal`, `esc_tipo_posto`, `esc_secao`, `esc_unidade` | `CALCULADOS`→`ESTIMADO`, `OBRIGATORIOS`→`MINIMO`, `QTDE_PDVS`→`IDEAL`, `horario`→`HORA_INI` |
| `load_holidays_tipo_3` | `wfm.esc_feriado` | `DATA` where `TIPO=3` and `FK_UNIDADE` |
| `get_absences_break` | `S_PROCESSO.GET_ABSENCE_AP` | `fk_colaborador`, `data`, `fk_motivo_ausencia`, period bounds, `horas_ausencia` |
| `load_horas_certas_param` | `wfm.core_parameter_cfg`, `wfm.core_parameter` | Resolved in code to a boolean-like flag (see function). Failure logs `DATA_VALIDATION` through `wfm_logger`. |
| `load_max_workers_on_break` | `WFM.CORE_ALG_PARAMETERS` | `SYS_P_NAME='max_workers_on_break'`. Missing or 0 disables the constraint (returns 0). |
| `load_horarios_pre_definidos_df` | `wfm.CORE_PRO_EMP_HORARIO_DET` ⋈ `esc_colaborador` | `LOCK_IND`, `HORARIO_IND`, clock columns, `WORKLOAD_TEMPLATE`, `MATRICULA` |
| `load_colab_contract_info` | `wfm.CORE_PRE_SCHEDULE_ALGORITHM` ⋈ `wfm.core_pro_emp_contract` | `TYPE`, `SUBTYPE`, `CPEC.*`, filter `PROCESS_ID`, `EXCLUSION_DATE is NULL` |
| `load_time_frame` | `wfm.core_pro_emp_availability` ⋈ `wfm.core_pro_emp_horario_det` | Availability plus `DESCANSO` |
| `load_folga_compensatoria_mov_df` | `WFM.CORE_PRO_EMP_MOV` | `VALUE_OPT1='D'`, `VALUE`, `SCHEDULE_DAY`, `SCHEDULE_DAY_REF` |
| `load_interval_exceptions` | `wfm.CORE_PRO_CONTRACT_EXCEPTION` | `code='INTERVAL_RANGE_EXCEPTION'`, `type_interval='I'` |
| `load_extended_schedule` | `wfm.esc_horario_colaborador` | History around the window for earliest entrance / latest exit |
| `load_generated_schedules` | `wfm.core_sched_algorithm_read` ⋈ `wfm.core_pro_emp_availability` | `csar.*`, `cpea.*`, `FINAL_HORA_INICIO`, `FINAL_HORA_FIM` |
| `load_contract_info` | `core_contract_labor` | `select *` for ids in `OPTION_N1` |
| `load_matricula_for_ids` | `wfm.esc_colaborador` | `MATRICULA`, `CODIGO` |
| `is_closure_enabled_for_unit` | `wfm.esc_unidade` | `FK_EMPRESA` where `CODIGO=:unit_id` and `NOME` like `%LEAL CENTER%`. Compared to `CLOSURE_FK_EMPRESA_CODES = ('1')` (Python parses `('1')` as the string `'1'`, not a one-tuple). Windows themselves are hardcoded in `closure_windows_for_date`: Mon–Sat 12:00–13:30, Sunday 12:00–16:00. |
| `insert_schedule_batch` | `wfm.int_schedule_algorithm` | See checkpoint E. |
| type 3 update | `wfm.CORE_SCHED_ALGORITHM_READ` | Clock columns and `RESPONSE_IND`. |

### Main in-memory objects by checkpoint

| Checkpoint | Object | Built in |
| --- | --- | --- |
| A | `sec_to_proc` DataFrame | `orquestrador.py` |
| A | `process_available` dict | `Orquestrador.get_process` |
| B | `valid_emp` | `AlgorithmConstructor` |
| B | extraction dict `unit_id`, `secao_id`, `posto_id`, `colab_matricula`, `employee_roles` | `GlobalDataInitialization.get_extraction_results` |
| B | `parameter_dataset`, `faixas_sec`, `granularity` | `GlobalDataInitialization` |
| B | `templates` dict keyed by `(weekly_hours, rest_days)` | `SchedulerDataHandler` |
| C | `cal` (`ContractCalendar`) | type 2 branch |
| C | `template_history`, `fixed_shift_by_employee_date`, `interval_exceptions` | `SchedulerMain` / `TemplateSelector` |
| C | `all_collaborators`, `demand_pattern_df`, `lookahead_constraints` | day loop |
| D | `prob`, `valid_configs`, `shift_details`, `LpStatus` | `create_schedule` |
| D | `RuleViolation` list | `SchedulerValidator` |
| E | `schedule_df`, `dfTratado`, `insert_data` | posto tail |

### Connection

`Connection/connect.py`: `configparser` on `Connection/acessos.txt` section `Config` keys `user`, `pwd`, `url`, `port`, `service_name`. DSN via `cx_Oracle.makedsn`. Encoding `ISO-8859-1`. Instant client path prefix `C:\oracle\instantclient_23_6`. `ALTER SESSION SET CURRENT_SCHEMA=WFM`. `ensure_connection` pings and reconnects. Failures append `log.txt`.

## Telemetry tiers

This process emits none of these as audit events. Tiers are what a v1 emitter should send. Fail-open: a failed emit must not change WFM status or the insert.

| Tier | Meaning | v1 for algoritmo-gh-pd |
| --- | --- | --- |
| T0 index | Identity and outcome so a run can be found | **Emit.** `project`, `wfm_proc_id`, `posto_id` (null on parent-only events), `process_type`, `algorithm_name`, `generation_type`, `unit_id`, `secao_id`, `start_date`, `end_date`, `child_number`, checkpoint name, status (`started` / `ok` / `failed` / `skipped`), timestamps. `tenant_id` and `deployment_version` as null until a source exists. |
| T1 metrics | Counts and timings, no row payloads | **Emit.** Employee count, posto count, day count, granularity minutes, template-history size, invalid-day count by `error_code`, solver status and seconds per day (summarised: counts of Optimal / Infeasible / Error, max seconds, fallback count), validator violation count by rule name, `rows_inserted` or type-3 update count, final parameter status `G`/`I`. |
| T2 fingerprints | Stable digests so two runs can be compared without storing the grids | **Emit counts and hashes, not frames.** Per input object: row count + hash of sorted ids (employee ids, contract ids, template ids chosen). Hash of insert payload column tuple `(SCHEDULE_DT, SCHED_TYPE, SCHED_SUBTYPE, clocks, OPTION_*)` after matrícula mapping. Do not hash raw Oracle rows that include names. |
| T3 artefacts | Full matrices, LP files, CSVs, shift grids | **Do not emit in v1.** Demand matrices, contract frames, CBC files, and `employee_schedules*.csv` contain operational schedules. Keep a pointer design open (object storage later) but do not turn today’s CSV side effect into the audit store. |

Per-day solver lines in `logs/scheduler_*.log` are a local T1/T3 mix. V1 should summarise them (T1) rather than copy the file.

## Sketch: one `execution_audit_v1` event

Field names below exist in this project unless marked `new`. One event per posto. Parent poll/spawn can be a second event with `posto_id: null` and only checkpoint A.

```json
{
  "event_type": "execution_audit_v1",
  "project": "algoritmo-gh-pd",
  "tenant_id": null,
  "deployment_version": null,
  "audit_run_id": null,
  "wfm_proc_id": 123456,
  "parent_api_proc_id": null,
  "process_type": 2,
  "algorithm_name": "ESC",
  "generation_type": "product",
  "proc_code_parent": "AlgoritmoHorariosPython_Pai",
  "proc_code_child": "AlgoritmoHorariosPython_Filho",
  "wfm_user": "WFM_USER",
  "child_number": 1,
  "wfm_proc_colab": null,
  "unit_id": "10",
  "secao_id": "20",
  "posto_id": "164",
  "start_date": "2026-10-01",
  "end_date": "2026-10-31",
  "granularity_minutes": 15,
  "final_param_status": "G",
  "checkpoints": [
    {
      "checkpoint": "A",
      "name": "job_structure",
      "status": "ok",
      "modules": ["orquestrador.py", "main.py", "ESCRunner.run", "AlgorithmConstructor.setup_env"],
      "metrics": {
        "polled_status": "N",
        "set_process_status": "P",
        "valid_employee_rows": 40,
        "global_max_concurrent_processes": 3,
        "processing_esc_count": 1
      }
    },
    {
      "checkpoint": "B",
      "name": "data_loading",
      "status": "ok",
      "modules": ["GlobalDataInitialization.initialize", "SchedulerDataHandler.run inputs"],
      "metrics": {
        "parameter_rows": 12,
        "faixas_sec_rows": 31,
        "colab_ids": 18,
        "matriz_min_ideal_rows": 2000,
        "contract_rows": 400,
        "pre_definidos_rows": 10,
        "fc_mov_rows": 3,
        "absence_rows": 5,
        "templates_loaded": true
      }
    },
    {
      "checkpoint": "C",
      "name": "transforms_to_solver_inputs",
      "status": "ok",
      "modules": ["ContractCalendar", "preprocess_week_templates", "TemplateSelector.select_best_pattern_for_week"],
      "metrics": {
        "template_history_entries": 80,
        "invalid_weeks": 1,
        "invalid_day_codes": {"AE_ERRO_TEMPLATE": 5}
      }
    },
    {
      "checkpoint": "D",
      "name": "solver_execution",
      "status": "ok",
      "modules": ["SchedulerMain.create_schedule", "SchedulerValidator.validate_day"],
      "solver": "PULP_CBC_CMD",
      "solver_params": {"timeLimit": 20, "gapRel": 0.01, "keepFiles": false},
      "metrics": {
        "days": 31,
        "status_counts": {"Optimal": 30, "Infeasible": 1},
        "fallback_solves": 1,
        "max_solve_seconds": 20.1,
        "violations_by_rule": {"interjornada": 2},
        "validator_registers_invalid_days": false
      }
    },
    {
      "checkpoint": "E",
      "name": "format_and_integrate",
      "status": "ok",
      "modules": ["create_schedule_dataframe", "insert_schedule_batch"],
      "target": "wfm.int_schedule_algorithm",
      "metrics": {
        "schedule_rows": 500,
        "z_rows": 5,
        "rows_inserted": 500,
        "employee_id_space": "MATRICULA"
      }
    }
  ]
}
```

Type 3 would set `"target": "wfm.CORE_SCHED_ALGORITHM_READ"` and `"metrics.rows_updated"` instead of `rows_inserted`. Employee id space during D is `CODIGO`; the sketch’s E metric records which space was written.

## Open questions

Network

- Where an emitter would run (same Windows host as the batch vs a side process) and whether it may open any socket. Today the only network client is `cx_Oracle` to the DSN in `acessos.txt`.
- Whether parent and child share a host. The parent does not wait on the child (`Popen` plus `SIGTERM`). An audit of “the job” has to be emitted inside the child for checkpoints B–E.
- `main.py` kills every `cbc.exe` on the machine at exit. Multi-process hosts would interfere. Not an audit decision; it affects how parallel runs can be observed.

PII

- Type 2 writes `MATRICULA` into `int_schedule_algorithm.EMPLOYEE_ID`. Solver logs use internal `EMPLOYEE_ID` / `FK_COLABORADOR` (`CODIGO`).
- `load_collaborator_matrix` selects `cav.nome` but is not on the live path. Do not start calling it for audit.
- Scheduler logs include shift clocks, contract ids, and employee ids. Translation messages include the user string.
- T3 payloads and raw CSV copies are schedule data. V1 should stay on T0–T2.
- `acessos.txt` holds database credentials. Emitters must not copy that file or the DSN password into events.

Identity

- Is `esc_processo.CODIGO` unique per attempt, or can the same code be rerun (status back to `N`)? That decides whether `wfm_proc_id` alone is an idempotency key.
- Should `parent_api_proc_id` (`sys.argv[1]` of the parent) be part of the correlation model? It is not on child status rows today.
- Is numeric `process_type` 2 vs 3 two products or two passes of one product? Type 3 reads `core_sched_algorithm_read` for the same `proc_code`.
- Is there a tenant or company key the WFM caller already has outside this repo?

Backfill

- Historical process outcome is whatever `SET_PROCESS_STATUS` / `SET_PROCESS_PARAMETER_STATUS` / `SET_PROCESS_ERRORS` stored, plus `int_schedule_algorithm` and type-3 updates. Descriptions are truncated by the PL/SQL `VARCHAR2(242)` bind.
- Local `logs/scheduler_*.log` and CSVs are not a reliable history (overwritten CSV names, gitignored, temp CBC dirs deleted).
- No in-repo watermark or “already audited” flag.

Replay

- There is no stage checkpoint table. A failed child does not resume at posto N; a new process would be a new WFM row or the same row set back to a pollable status by something outside this repo.
- `random.seed(123)` is set, but CBC time limits and `gapRel` mean a replay is not guaranteed to match.
- `keepFiles=False` removes the practical solver artefact.
- Debug path depends on gitignored `manual_config.py`.

Behaviour that validations may want to treat as product questions (already true in code)

- Posto early `return 0` (no contracts, no type-3 schedules) does not fail the process.
- Insert errors return 0 and still allow status `G`.
- Hard-rule violations do not flip the day to `Z` while `is_registering_invalid_days=False`.
- Parent `SIGTERM` after the first `Popen`, and `local_processes` never incremented, so configured local concurrency is not what the loop executes.
- Several translation keys describe posto-level WFM messages the Python code never sends.

## Components to reference (do not copy wholesale)

Use these as the map of where an emitter would sit later. They are not a library.

| Concern | Reference |
| --- | --- |
| Parent poll, status `P`, spawn | `orquestrador.py` |
| Child outcome `G`/`I` | `main.py` |
| Process SQL | `Functions/WFM_Process/Getters.py`, `Setters.py`, `Data/Queries/WFM_Process/` |
| DB log wrapper | `Classes/AlgorithmPrepClasses/Logger.py` |
| Employee gate and ids | `Classes/AlgorithmIniClasses/AlgorithmConstructor.py`, `GlobalDataInitialization.py` |
| Live input SQL | `Classes/SchedulerAlgorithmClasses/SchedulerDataHandler.py`, `Functions/Data_Handlers/GetGlobalData.py` |
| Week pattern vs day solve split | `SchedulerMain.preprocess_week_templates`, `SchedulerMain.create_schedule` |
| Invalid-day codes | `Classes/SchedulerAlgorithmClasses/InvalidDayCodes.py` |
| Post-solve rules | `Classes/SchedulerAlgorithmClasses/SchedulerValidator.py` (`DEFAULT_HARD_RULES`) |
| Integration | `Functions/Insert_read_schedules/Insert_schedules.py` and the type-3 `UPDATE` block in `SchedulerMain.run_scheduler_for_post` |
| Oracle boundary | `Connection/connect.py`, `Classes/AlgorithmPrepClasses/ConnectionHandler.py` |

`architecture.md` sections 15–17 are a proposed refactor (services, repositories). They are not the running pipeline.

## Cross-project notes (algortimo-gd vs algoritmo-gh-pd)

Shared, safe to use as one audit axis:

| Shared fact | Both projects |
| --- | --- |
| System of record | WFM Oracle, session/schema `WFM`, `cx_Oracle`, `acessos.txt`. |
| Process table | `esc_processo` (`CODIGO`, dates, `FK_COLABORADOR`, `USER_CRIACAO`). |
| Poll | `S_PROCESSO.GET_PROCESS_BY_STATUS` and `get_total_process_by_status` for `ESC` + `P`. |
| Status letters | `N` polled, `P` claimed, `G` done, `I` invalid/failed. |
| Error channel | `S_PROCESSO.SET_PROCESS_ERRORS` with `type_error`, description, optional employee and day. |
| Parameters | `CORE_ALG_PARAMETERS` plus `get_all_params`. |
| Loop grain | Work is per `posto_id` inside one `wfm_proc_id`. |
| Checkpoint letters | A job, B load, C transform, D solve, E format/integrate. |
| Keys to add | `tenant_id`, `deployment_version`. `wfm_proc_id` and `posto_id` already exist. |

Divergences the UI must not flatten:

| Topic | algortimo-gd | algoritmo-gh-pd |
| --- | --- | --- |
| How the child runs | `orquestrador.py` calls `batch_process.run_batch_process` in-process and then sets status `G`/`I` itself. | Parent `Popen`s `main.py` and sends itself `SIGTERM`. The child sets parameter status `G`/`I`. |
| Stage model | `ProcessStageHandler` and `project_structure`: stages `data_loading` and `processing`, substages `treat_params`, `load_matrices`, `func_inicializa`, `allocation_cycle`, `format_results`, `insert_results`. | No stage handler. A–E map onto functions listed above. Do not emit GD substage names for this project. |
| Service | `src/services/algoritmo_gd.py` (`_execute` data_loading / processing). | `ESCRunner` + `SchedulerMain`. |
| Algorithm identity | Factory of named models (`salsa_algorithm` and siblings under `src/algorithms/`). Audit key `algorithm_name` plus `data_model`. | Factory of `ESC` or `MPD`. Parent always launches `ESC`. Extra discriminator is numeric `process_type` 2 vs 3, not a model class name. |
| Solver shape | Allocation cycle inside the GD model code (one processing stage per posto). | Two layers per posto: weekly `TemplateSelector`, then one CBC solve **per day** (`timeLimit=20`, `gapRel=0.01`). |
| Output | GD `insert_results` substage (insert flag in stage config). | Type 2 insert `wfm.int_schedule_algorithm`. Type 3 update `wfm.CORE_SCHED_ALGORITHM_READ`. Employee id rewritten to `MATRICULA` on type 2. |
| Invalid days | GD validations live under `src/data_models/validations/`. | `InvalidDayCodes` (`AE_ERRO_*`) become `SCHED_TYPE='Z'` / `OPTION_C1`. Separate post-solve `SchedulerValidator` rules, currently log-only for Z. |
| Concurrency control | GD parent does not `Popen` and does not kill `cbc.exe`. | This parent polls global `P` count, then self-terminates; child kills all `cbc.exe`. |
| `api_proc_id` vs `wfm_proc_id` | GD passes `wfm_proc_id` into `external_call_dict` and uses it on status updates. | Both ids exist on the command line. Child status uses WFM `CODIGO` only. |

UI label shape that stays honest:

```text
project: algortimo-gd     | stages: data_loading > processing/{treat_params…insert_results} | keys: tenant_id, wfm_proc_id, posto_id, data_model, algorithm_name, deployment_version
project: algoritmo-gh-pd  | checkpoints: A job > B load > C template/day rows > D daily CBC > E int_schedule_algorithm|CORE_SCHED_ALGORITHM_READ | keys: tenant_id (new), wfm_proc_id, posto_id, process_type, algorithm_name=ESC, deployment_version (new)
```

`process_type` is project-specific. Do not require `data_model` on algoritmo-gh-pd events until a product name is chosen. Do not require `process_type` on algortimo-gd events.

## Out of scope for this pass

Ingestion transport, audit database, UI stack, emitter implementation, backfill jobs, and any change to WFM status or schedule writes.
