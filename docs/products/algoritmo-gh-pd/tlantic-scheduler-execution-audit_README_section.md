# README section — algoritmo_GH_PD

Copied from `algoritmo_GH_PD` `docs/tlantic-scheduler-execution-audit_README_section.md` on 26/10/2026. The live summary is in the repository [README](../../../README.md).

## algoritmo_GH_PD

`algoritmo_GH_PD` (audit slug `algoritmo-gh-pd`) is the WFM horários generator: a parent process polls Oracle `esc_processo`, a child (`ESC`) builds weekly templates and solves a daily PuLP/CBC model per posto, then inserts `wfm.int_schedule_algorithm` (process type 2) or updates `wfm.CORE_SCHED_ALGORITHM_READ` (process type 3). It contributes job, load, template, solver, and integration checkpoints to the shared audit platform under the same A–E axis as algortimo-gd, with project-specific substages and keys (`process_type`, `algorithm_name=ESC`).

- **A — job structure:** `orquestrador.py` (poll, status `P`, `Popen`), `main.py` (status `G`/`I`), `Classes/AlgorithmBaseClasses/ESCRunner.py`, `Classes/AlgorithmIniClasses/AlgorithmConstructor.py`
- **B — data loading:** `Classes/AlgorithmIniClasses/GlobalDataInitialization.py`, `Functions/Data_Handlers/GetGlobalData.py`, `Classes/SchedulerAlgorithmClasses/SchedulerDataHandler.py`
- **C — transforms:** `Classes/SchedulerAlgorithmClasses/ContractCalendar.py`, `Classes/SchedulerAlgorithmClasses/TemplateSelector.py`, `SchedulerMain.preprocess_week_templates`
- **D — solver:** `Classes/SchedulerAlgorithmClasses/SchedulerMain.py` (`create_schedule`, CBC), `Classes/SchedulerAlgorithmClasses/SchedulerValidator.py`
- **E — integration:** `SchedulerMain.create_schedule_dataframe`, `Functions/Insert_read_schedules/Insert_schedules.py` (type 2), inline `UPDATE` of `CORE_SCHED_ALGORITHM_READ` (type 3)

Repository: `[link to algoritmo_GH_PD repo]`

Discovery write-up (copy from that repo): `docs/execution-analytics-platform_discovery_document.md` and `docs/execution-analytics-platform_vision-and-validations-backlog.md`.
