# Регламент при прорыве теплового ввода (smart-valve, обходчик, оповещение медблока)

**source_id:** `heat-inlet-breach`  
**Дата редакции регламента:** 2024-09-10  
**Домен:** Теплоснабжение (`heating`)

> При подтверждении утечки в тепловом узле: 1) Уведомить инженерную службу объекта и городские теплосети.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: heat-inlet-breach
---
flowchart LR
  n_pressure_in(("Давление узла"))
  n_pressure_thr["4.0 ± 0.5 атм"]
  n_pressure_cmp{"давление вне нормы?"}
  n_fall_in(("Скорость падения P"))
  n_fall_thr["0.0 ± 0.2 атм/мин"]
  n_fall_cmp{"резкое падение?"}
  n_temp_in(("Температура подачи"))
  n_temp_thr["70 ± 10 °C"]
  n_temp_cmp{"температура вне нормы?"}
  n_breach_check[/"Подтверждённый прорыв"/]
  n_walker_confirm[\"Запрос обходчика · P2"\]
  n_smart_valve[\"Перекрыть smart-valve · P1"\]
  n_emergency_brigade[\"Аварийная бригада · P1"\]
  n_shacl_pressure[["SHACL: inletPressure"]]
  n_shacl_pressureFallRate[["SHACL: pressureFallRate"]]
  n_shacl_inletTemperature[["SHACL: inletTemperature"]]
  n_pressure_in --> n_pressure_thr
  n_pressure_thr --> n_pressure_cmp
  n_pressure_cmp -->|pressure_out| n_breach_check
  n_fall_in --> n_fall_thr
  n_fall_thr --> n_fall_cmp
  n_fall_cmp -->|fall| n_breach_check
  n_temp_in --> n_temp_thr
  n_temp_thr --> n_temp_cmp
  n_temp_cmp -->|temp_out| n_smart_valve
  n_breach_check -->|true| n_walker_confirm
  n_breach_check -->|after_confirm| n_smart_valve
  n_breach_check -->|confirmed_breach| n_emergency_brigade
  n_pressure_cmp --> n_shacl_pressure
  n_fall_in --> n_shacl_pressureFallRate
  n_temp_in --> n_shacl_inletTemperature

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_pressure_in,n_fall_in,n_temp_in sensor;
  class n_pressure_thr,n_fall_thr,n_temp_thr param;
  class n_pressure_cmp,n_fall_cmp,n_temp_cmp decision;
  class n_breach_check formula;
  class n_walker_confirm,n_smart_valve,n_emergency_brigade action;
  class n_shacl_pressure,n_shacl_pressureFallRate,n_shacl_inletTemperature shacl;
```

## Исходные файлы

- [heat-inlet-breach.flow.json](../../../backend/data/fixtures/heat-inlet-breach.flow.json) — визуальный граф регламента (Rule DSL).
- [heat-inlet-breach.data.ttl](../../../backend/data/fixtures/heat-inlet-breach.data.ttl) — Turtle: параметры и метаданные регламента.
- [heat-inlet-breach.shapes.ttl](../../../backend/data/fixtures/heat-inlet-breach.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
