# Vision and validations index

**Status:** living index. Product ideas stay in the product files. This page only says how to add them.

| Product | Backlog |
|---------|---------|
| **algortimo-gd** | [vision and validations](products/algortimo-gd/execution-analytics-platform_vision-and-validations-backlog.md) |
| **algoritmo_GH_PD** | [vision and validations](products/algoritmo-gh-pd/execution-analytics-platform_vision-and-validations-backlog.md) |

Forecast ideas do not have a backlog yet.

## Why the tables stay separate

The two backlogs were seeded from different code and they do not share an id scheme or a severity vocabulary.

| | algortimo-gd | algoritmo-gh-pd |
|--|----------------|-----------------|
| Ids | `A-001` | `A-01` |
| Severity | `blocker`, `warning`, `info` | `blocker`, `major`, `minor`, `info` |
| Status | `idea`, `specced`, `implemented`, `wontfix` | `idea`, `accepted`, `rejected`, `shipped` |
| Seeded checks | GD stage names, `medium_data` frames, `allocation_cycle` | GH guards already in code: `process_type` 2/3, granularity, template errors, daily CBC status, type 2 insert vs type 3 update |

A GD validation such as “`medium_data` contains `df_colaborador`” applies to the GD audit. A GH validation such as “type 3 sets `RESPONSE_IND=Y`” applies to the GH audit. A list can show both products. The detail of one row is one audit process. A later “consumed GD result” link is a relationship between two records.

## How to add an idea

1. Put it in the product backlog, not in this index.
2. Give it the next id for that product and checkpoint.
3. Name the checkpoint letter (A–E) and the data tier (T0–T3) it needs.
4. Leave status at `idea` until the team accepts it.
5. If an idea is truly shared (ingest health, tenant filter, copy correlation bundle), add it to **both** backlogs or note it under cross-cutting in the GD file and add a GH counterpart. Do not invent a third id space here.

Web-app areas already sketched for GD (home, execution list, detail timeline, validation report, compare, replay, admin) are a starting shell. GH adds a day strip on checkpoint D, because that product solves once per day, and an outcome panel that must not score type 3 with the type 2 insert count.

Compare and digests stay ideas until export scope and PII are sufficient for them. Ingest topology is in [04-ingestion-strategy.md](04-ingestion-strategy.md). **Algorithm rerun** from ingested raw is a future platform idea (see that log); GH replay constraints in the GH backlog still apply.
