# Учебный 06: Kleene UNKNOWN — три исхода вместо двух при возможном offline-датчике

**source_id:** `tutorial-06-kleene-unknown`  
**Дата редакции регламента:** 2026-05-28  
**Домен:** Учебный набор регламентов (`tutorial`)

> ШЕСТОЙ УЧЕБНЫЙ ШАГ.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: tutorial-06-kleene-unknown
---
flowchart LR
  n_smoke_in(("Уровень задымления"))
  n_known_check[/"значение известно?"/]
  n_alarm_check[/"превышение?"/]
  n_classify_switch{"Состояние датчика и уровня"}
  n_normal_output[\"Норма · P3"\]
  n_alarm_output[\"Тревога: пожар · P1"\]
  n_offline_output[\"Датчик неисправен · P2"\]
  n_smoke_in --> n_known_check
  n_smoke_in --> n_alarm_check
  n_known_check --> n_classify_switch
  n_alarm_check --> n_classify_switch
  n_classify_switch -->|known_normal| n_normal_output
  n_classify_switch -->|known_alarm| n_alarm_output
  n_classify_switch -->|unknown| n_offline_output

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_smoke_in sensor;
  class n_known_check,n_alarm_check formula;
  class n_classify_switch decision;
  class n_normal_output,n_alarm_output,n_offline_output action;
```

## Исходные файлы

- [tutorial-06-kleene-unknown.flow.json](../../../backend/data/fixtures/tutorial-06-kleene-unknown.flow.json) — визуальный граф регламента (Rule DSL).
- [tutorial-06-kleene-unknown.data.ttl](../../../backend/data/fixtures/tutorial-06-kleene-unknown.data.ttl) — Turtle: параметры и метаданные регламента.
- [tutorial-06-kleene-unknown.shapes.ttl](../../../backend/data/fixtures/tutorial-06-kleene-unknown.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
