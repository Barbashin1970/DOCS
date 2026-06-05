# Триаж критичных задач (баг-репорты p1, инциденты): SLA на назначение исполнителя

**source_id:** `leyka-bug-triage`  
**Дата редакции регламента:** 2026-05-22  
**Домен:** ИТ-поддержка (`it-support`)

> Триаж — это первая защита от паники.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: leyka-bug-triage
---
flowchart LR
  n_sensor_age(("📋 LEYKA · возраст max untriaged (ч)"))
  n_in_p1_sla(("p1 SLA — пинг lead"))
  n_thr_p1_sla["2 [1…24]"]
  n_in_p1_crit(("p1 SLA — эскалация"))
  n_thr_p1_crit["6 [1…72]"]
  n_sensor_p1(("📋 LEYKA · untriaged p1"))
  n_thr_p1_count["1 — наличие p1"]
  n_sensor_total(("📋 LEYKA · untriaged всего"))
  n_in_pile(("Куча неразобранных"))
  n_thr_pile["10 [1…200]"]
  n_formula[/"Кому пинг и насколько срочно"/]
  n_switch{"Действие триажа"}
  n_out_0[\"🟢 Норма · P1"\]
  n_out_1[\"🟠 Пинг lead'у · P2"\]
  n_out_2[\"🔴 Эскалация менеджеру · P3"\]
  n_shacl_p1TriageSlaHours[["SHACL: p1TriageSlaHours"]]
  n_shacl_p1TriageCriticalHours[["SHACL: p1TriageCriticalHours"]]
  n_shacl_untriagedPileWarnCount[["SHACL: untriagedPileWarnCount"]]
  n_sensor_age --> n_in_p1_sla
  n_sensor_p1 --> n_thr_p1_count
  n_sensor_total --> n_in_pile
  n_in_p1_sla --> n_thr_p1_sla
  n_in_p1_crit --> n_thr_p1_crit
  n_in_pile --> n_thr_pile
  n_thr_p1_sla --> n_formula
  n_thr_p1_crit --> n_formula
  n_thr_p1_count --> n_formula
  n_thr_pile --> n_formula
  n_formula --> n_switch
  n_switch -->|ok| n_out_0
  n_switch -->|ping_lead| n_out_1
  n_switch -->|escalate| n_out_2
  n_in_p1_sla --> n_shacl_p1TriageSlaHours
  n_in_p1_crit --> n_shacl_p1TriageCriticalHours
  n_in_pile --> n_shacl_untriagedPileWarnCount

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_sensor_age,n_in_p1_sla,n_in_p1_crit,n_sensor_p1,n_sensor_total,n_in_pile sensor;
  class n_thr_p1_sla,n_thr_p1_crit,n_thr_p1_count,n_thr_pile param;
  class n_formula formula;
  class n_switch decision;
  class n_out_0,n_out_1,n_out_2 action;
  class n_shacl_p1TriageSlaHours,n_shacl_p1TriageCriticalHours,n_shacl_untriagedPileWarnCount shacl;
```

## Исходные файлы

- [leyka-bug-triage.flow.json](../../../backend/data/fixtures/leyka-bug-triage.flow.json) — визуальный граф регламента (Rule DSL).
- [leyka-bug-triage.data.ttl](../../../backend/data/fixtures/leyka-bug-triage.data.ttl) — Turtle: параметры и метаданные регламента.
- [leyka-bug-triage.shapes.ttl](../../../backend/data/fixtures/leyka-bug-triage.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
