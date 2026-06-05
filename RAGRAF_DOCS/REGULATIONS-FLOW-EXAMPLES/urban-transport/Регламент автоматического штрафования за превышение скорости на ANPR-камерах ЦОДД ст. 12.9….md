# Регламент автоматического штрафования за превышение скорости на ANPR-камерах ЦОДД (ст. 12.9 КоАП РФ, ступенчатая шкала 500₽/5000₽ с прогрессивной надбавкой, постановление через ГИБДД-API, LEYKA-задача инспектору)

**source_id:** `traffic-speed-fine`  
**Дата редакции регламента:** 2026-05-26  
**Домен:** Городской транспорт (`urban-transport`)

> Регламент запускается на каждое событие от двух физически независимых детекторов одной ANPR-камеры ЦОДД: vd-anpr (распознавание ГРЗ) + vd-vehicle-speed (радар/laser измеритель скорости), связанных общим track_id.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: traffic-speed-fine
---
flowchart LR
  n_sensor_plate(("🎥 vd-anpr · распознавание ГРЗ"))
  n_sensor_speed(("📡 vd-vehicle-speed · радар скорости"))
  n_sensor_zone_limit(("🚧 vd-traffic-zone-limit · лимит зоны камеры (ЦОДД)"))
  n_in_plate(("ГРЗ авто (для постановления)"))
  n_in_speed(("Измеренная скорость, км/ч"))
  n_in_speed_limit(("Лимит зоны камеры, км/ч (60 / 40 / 20)"))
  n_shacl_speed_limit[["SHACL: speedLimitKmh"]]
  n_formula_excess[/"🧮 excess = speed − speedLimit"/]
  n_in_warn_excess(("Warn-порог excess (КоАП ч.2)"))
  n_thr_warn["20 [10…40]"]
  n_shacl_warn[["SHACL: warnExcessKmh"]]
  n_in_critical_excess(("Critical-порог excess (КоАП ч.4)"))
  n_thr_critical["40 [30…80]"]
  n_shacl_critical[["SHACL: criticalExcessKmh"]]
  n_in_warn_fine(("Базовая warn-штраф, ₽"))
  n_shacl_warn_fine[["SHACL: warnFineRub"]]
  n_in_critical_fine(("Минимальный critical-штраф, ₽"))
  n_shacl_critical_fine[["SHACL: criticalFineRub"]]
  n_in_penalty_per_kmh(("Надбавка, ₽/км/ч (прогрессив)"))
  n_shacl_penalty[["SHACL: penaltyRubPerKmh"]]
  n_in_sla(("SLA инспектора, мин"))
  n_shacl_sla[["SHACL: inspectorSlaMinutes"]]
  n_formula_fine_warn[/"💵 fineWarn = warnFineRub + (excess−warn) × penalty"/]
  n_formula_fine_critical[/"🚨 fineCritical = max(criticalFineRub, excess × penalty)"/]
  n_switch_band{"Тяжесть нарушения · строгий первый"}
  n_out_critical_fine[\"🚨 Постановление critical (формула × ГРЗ) + LEYKA инспектору · P3"\]
  n_out_warn_fine[\"💵 Постановление warn (формула × ГРЗ) — авто · P2"\]
  n_out_ok[\"✅ Норма — ничего не делаем · P1"\]
  n_sensor_plate --> n_in_plate
  n_sensor_speed --> n_in_speed
  n_sensor_zone_limit --> n_in_speed_limit
  n_in_speed --> n_formula_excess
  n_in_speed_limit --> n_formula_excess
  n_in_speed_limit --> n_shacl_speed_limit
  n_formula_excess --> n_thr_warn
  n_formula_excess --> n_thr_critical
  n_in_warn_excess --> n_thr_warn
  n_in_warn_excess --> n_shacl_warn
  n_in_critical_excess --> n_thr_critical
  n_in_critical_excess --> n_shacl_critical
  n_in_warn_fine --> n_shacl_warn_fine
  n_in_critical_fine --> n_shacl_critical_fine
  n_in_penalty_per_kmh --> n_shacl_penalty
  n_in_sla --> n_shacl_sla
  n_in_warn_fine --> n_formula_fine_warn
  n_in_warn_excess --> n_formula_fine_warn
  n_in_penalty_per_kmh --> n_formula_fine_warn
  n_in_speed --> n_formula_fine_warn
  n_in_speed_limit --> n_formula_fine_warn
  n_in_critical_fine --> n_formula_fine_critical
  n_in_penalty_per_kmh --> n_formula_fine_critical
  n_in_speed --> n_formula_fine_critical
  n_in_speed_limit --> n_formula_fine_critical
  n_thr_warn --> n_switch_band
  n_thr_critical --> n_switch_band
  n_formula_fine_critical --> n_out_critical_fine
  n_formula_fine_warn --> n_out_warn_fine
  n_in_plate --> n_out_critical_fine
  n_in_plate --> n_out_warn_fine
  n_switch_band -->|critical| n_out_critical_fine
  n_switch_band -->|warn| n_out_warn_fine
  n_switch_band -->|ok| n_out_ok

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_sensor_plate,n_sensor_speed,n_sensor_zone_limit,n_in_plate,n_in_speed,n_in_speed_limit,n_in_warn_excess,n_in_critical_excess,n_in_warn_fine,n_in_critical_fine,n_in_penalty_per_kmh,n_in_sla sensor;
  class n_shacl_speed_limit,n_shacl_warn,n_shacl_critical,n_shacl_warn_fine,n_shacl_critical_fine,n_shacl_penalty,n_shacl_sla shacl;
  class n_formula_excess,n_formula_fine_warn,n_formula_fine_critical formula;
  class n_thr_warn,n_thr_critical param;
  class n_switch_band decision;
  class n_out_critical_fine,n_out_warn_fine,n_out_ok action;
```

## Исходные файлы

- [traffic-speed-fine.flow.json](../../../backend/data/fixtures/traffic-speed-fine.flow.json) — визуальный граф регламента (Rule DSL).
- [traffic-speed-fine.data.ttl](../../../backend/data/fixtures/traffic-speed-fine.data.ttl) — Turtle: параметры и метаданные регламента.
- [traffic-speed-fine.shapes.ttl](../../../backend/data/fixtures/traffic-speed-fine.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
