# Регламент пинга задач LEYKA (просроченные / зависшие / без разметки)

**source_id:** `leyka-overdue-ping`  
**Дата редакции регламента:** 2026-05-22  
**Домен:** ИТ-поддержка (`it-support`)

> Бот РАГРАФ отправляет пинг исполнителю в LEYKA.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: leyka-overdue-ping
---
flowchart LR
  n_sensor_overdue_count(("📋 LEYKA · просроченных задач (count)"))
  n_in_overdue_count(("Просроченных — порог warn"))
  n_thr_overdue_count_warn["5 [0…50]"]
  n_in_overdue_crit(("Просроченных — порог critical"))
  n_thr_overdue_count_crit["15 [0…100]"]
  n_sensor_overdue_max(("📋 LEYKA · максимум days_overdue"))
  n_in_overdue_max_warn(("Глубина просрочки — warn (дней)"))
  n_thr_overdue_max_warn["7 [0…60]"]
  n_in_overdue_max_crit(("Глубина просрочки — critical (дней)"))
  n_thr_overdue_max_crit["30 [0…180]"]
  n_sensor_stale_inp(("📋 LEYKA · зависших в работе (count)"))
  n_in_stale_inp(("Зависших в in_progress — порог warn"))
  n_thr_stale_inp["3 [0…30]"]
  n_sensor_stale_inbox(("📋 LEYKA · в inbox без разметки (count)"))
  n_in_stale_inbox(("В inbox без разметки — порог warn"))
  n_thr_stale_inbox["5 [0…50]"]
  n_formula[/"Классификация TTL-сигнала"/]
  n_switch{"Действие пинга"}
  n_out_0[\"🟢 Норма · P1"\]
  n_out_1[\"🟡 Внимание · P2"\]
  n_out_2[\"🟠 Пинг исполнителям · P3"\]
  n_out_3[\"🔴 Эскалация менеджеру · P4"\]
  n_shacl_overdueWarnCount[["SHACL: overdueWarnCount"]]
  n_shacl_overdueCriticalCount[["SHACL: overdueCriticalCount"]]
  n_shacl_overdueMaxDaysWarn[["SHACL: overdueMaxDaysWarn"]]
  n_shacl_overdueMaxDaysCritical[["SHACL: overdueMaxDaysCritical"]]
  n_shacl_staleInProgressWarnCount[["SHACL: staleInProgressWarnCount"]]
  n_shacl_staleInboxWarnCount[["SHACL: staleInboxWarnCount"]]
  n_sensor_overdue_count --> n_in_overdue_count
  n_sensor_overdue_max --> n_in_overdue_max_warn
  n_sensor_stale_inp --> n_in_stale_inp
  n_sensor_stale_inbox --> n_in_stale_inbox
  n_in_overdue_count --> n_thr_overdue_count_warn
  n_in_overdue_crit --> n_thr_overdue_count_crit
  n_in_overdue_max_warn --> n_thr_overdue_max_warn
  n_in_overdue_max_crit --> n_thr_overdue_max_crit
  n_in_stale_inp --> n_thr_stale_inp
  n_in_stale_inbox --> n_thr_stale_inbox
  n_thr_overdue_count_warn --> n_formula
  n_thr_overdue_count_crit --> n_formula
  n_thr_overdue_max_warn --> n_formula
  n_thr_overdue_max_crit --> n_formula
  n_thr_stale_inp --> n_formula
  n_thr_stale_inbox --> n_formula
  n_formula --> n_switch
  n_switch -->|ok| n_out_0
  n_switch -->|attention| n_out_1
  n_switch -->|ping| n_out_2
  n_switch -->|escalate| n_out_3
  n_in_overdue_count --> n_shacl_overdueWarnCount
  n_in_overdue_crit --> n_shacl_overdueCriticalCount
  n_in_overdue_max_warn --> n_shacl_overdueMaxDaysWarn
  n_in_overdue_max_crit --> n_shacl_overdueMaxDaysCritical
  n_in_stale_inp --> n_shacl_staleInProgressWarnCount
  n_in_stale_inbox --> n_shacl_staleInboxWarnCount

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_sensor_overdue_count,n_in_overdue_count,n_in_overdue_crit,n_sensor_overdue_max,n_in_overdue_max_warn,n_in_overdue_max_crit,n_sensor_stale_inp,n_in_stale_inp,n_sensor_stale_inbox,n_in_stale_inbox sensor;
  class n_thr_overdue_count_warn,n_thr_overdue_count_crit,n_thr_overdue_max_warn,n_thr_overdue_max_crit,n_thr_stale_inp,n_thr_stale_inbox param;
  class n_formula formula;
  class n_switch decision;
  class n_out_0,n_out_1,n_out_2,n_out_3 action;
  class n_shacl_overdueWarnCount,n_shacl_overdueCriticalCount,n_shacl_overdueMaxDaysWarn,n_shacl_overdueMaxDaysCritical,n_shacl_staleInProgressWarnCount,n_shacl_staleInboxWarnCount shacl;
```

## Исходные файлы

- [leyka-overdue-ping.flow.json](../../../backend/data/fixtures/leyka-overdue-ping.flow.json) — визуальный граф регламента (Rule DSL).
- [leyka-overdue-ping.data.ttl](../../../backend/data/fixtures/leyka-overdue-ping.data.ttl) — Turtle: параметры и метаданные регламента.
- [leyka-overdue-ping.shapes.ttl](../../../backend/data/fixtures/leyka-overdue-ping.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
