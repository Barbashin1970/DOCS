# Kanban WIP-limit: не больше N задач в работе на исполнителя

**source_id:** `leyka-wip-limit`  
**Дата редакции регламента:** 2026-05-22  
**Домен:** ИТ-поддержка (`it-support`)

> Узкое горлышко: исполнитель набрал больше задач, чем способен закрыть параллельно.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: leyka-wip-limit
---
flowchart LR
  n_sensor_max(("📋 LEYKA · max in_progress на исполнителя"))
  n_in_normal(("Норма WIP (warn)"))
  n_thr_normal["3 [1…10]"]
  n_in_critical(("Критический WIP"))
  n_thr_critical["5 [2…20]"]
  n_sensor_overloaded(("📋 LEYKA · перегруженных исполнителей"))
  n_in_team_warn(("Когда системная проблема"))
  n_thr_team["2 [1…50]"]
  n_formula[/"Кто перегружен и насколько"/]
  n_switch{"Действие"}
  n_out_0[\"🟢 Норма · P1"\]
  n_out_1[\"🟠 Пинг исполнителю · P2"\]
  n_out_2[\"🔴 Эскалация ПМ · P3"\]
  n_shacl_wipNormalPerAssignee[["SHACL: wipNormalPerAssignee"]]
  n_shacl_wipCriticalPerAssignee[["SHACL: wipCriticalPerAssignee"]]
  n_shacl_overloadedTeamWarnCount[["SHACL: overloadedTeamWarnCount"]]
  n_sensor_max --> n_in_normal
  n_sensor_overloaded --> n_in_team_warn
  n_in_normal --> n_thr_normal
  n_in_critical --> n_thr_critical
  n_in_team_warn --> n_thr_team
  n_thr_normal --> n_formula
  n_thr_critical --> n_formula
  n_thr_team --> n_formula
  n_formula --> n_switch
  n_switch -->|ok| n_out_0
  n_switch -->|ping| n_out_1
  n_switch -->|escalate| n_out_2
  n_in_normal --> n_shacl_wipNormalPerAssignee
  n_in_critical --> n_shacl_wipCriticalPerAssignee
  n_in_team_warn --> n_shacl_overloadedTeamWarnCount

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_sensor_max,n_in_normal,n_in_critical,n_sensor_overloaded,n_in_team_warn sensor;
  class n_thr_normal,n_thr_critical,n_thr_team param;
  class n_formula formula;
  class n_switch decision;
  class n_out_0,n_out_1,n_out_2 action;
  class n_shacl_wipNormalPerAssignee,n_shacl_wipCriticalPerAssignee,n_shacl_overloadedTeamWarnCount shacl;
```

## Исходные файлы

- [leyka-wip-limit.flow.json](../../../backend/data/fixtures/leyka-wip-limit.flow.json) — визуальный граф регламента (Rule DSL).
- [leyka-wip-limit.data.ttl](../../../backend/data/fixtures/leyka-wip-limit.data.ttl) — Turtle: параметры и метаданные регламента.
- [leyka-wip-limit.shapes.ttl](../../../backend/data/fixtures/leyka-wip-limit.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
