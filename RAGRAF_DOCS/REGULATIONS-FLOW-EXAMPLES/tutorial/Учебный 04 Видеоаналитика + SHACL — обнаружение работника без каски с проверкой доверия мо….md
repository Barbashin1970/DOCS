# Учебный 04: Видеоаналитика + SHACL — обнаружение работника без каски с проверкой доверия модели

**source_id:** `tutorial-04-shacl-validation`  
**Дата редакции регламента:** 2026-05-28  
**Домен:** Учебный набор регламентов (`tutorial`)

> ЧЕТВЁРТЫЙ УЧЕБНЫЙ ШАГ.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: tutorial-04-shacl-validation
---
flowchart LR
  n_camera_sensor(("Камера на площадке"))
  n_confidence_in(("Доверие модели"))
  n_shacl_confidence[["SHACL: 0..1"]]
  n_confidence_threshold["0.8 ± 0.1"]
  n_confidence_compare{"доверие ≥ 0.8?"}
  n_foreman_output[\"Прораб · P1"\]
  n_safety_output[\"Инженер ОТ · P2"\]
  n_camera_sensor --> n_confidence_in
  n_confidence_in --> n_shacl_confidence
  n_confidence_in --> n_confidence_threshold
  n_confidence_threshold --> n_confidence_compare
  n_confidence_compare -->|true| n_foreman_output
  n_confidence_compare -->|true| n_safety_output

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_camera_sensor,n_confidence_in sensor;
  class n_shacl_confidence shacl;
  class n_confidence_threshold param;
  class n_confidence_compare decision;
  class n_foreman_output,n_safety_output action;
```

## Исходные файлы

- [tutorial-04-shacl-validation.flow.json](../../../backend/data/fixtures/tutorial-04-shacl-validation.flow.json) — визуальный граф регламента (Rule DSL).
- [tutorial-04-shacl-validation.data.ttl](../../../backend/data/fixtures/tutorial-04-shacl-validation.data.ttl) — Turtle: параметры и метаданные регламента.
- [tutorial-04-shacl-validation.shapes.ttl](../../../backend/data/fixtures/tutorial-04-shacl-validation.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
