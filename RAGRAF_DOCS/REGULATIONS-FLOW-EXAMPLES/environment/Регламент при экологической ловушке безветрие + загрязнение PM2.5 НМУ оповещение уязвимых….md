# Регламент при экологической ловушке: безветрие + загрязнение PM2.5 (НМУ, оповещение уязвимых групп, предписания предприятиям)

**source_id:** `air-quality-smog-trap`  
**Дата редакции регламента:** 2024-11-30  
**Домен:** Городская экология (`environment`)

> При безветрии (ветер < 1.5 м/с) и росте PM2.5 формируется экологическая ловушка.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: air-quality-smog-trap
---
flowchart LR
  n_wind_in(("Ветер"))
  n_wind_thr["3.0 ± 1.5 м/с"]
  n_wind_cmp{"штиль?"}
  n_pm25_in(("PM2.5"))
  n_pm25_thr["10 ± 10 мкг/м³"]
  n_pm25_cmp{"загрязнение > 20?"}
  n_pdk_in(("Часы выше ПДК"))
  n_pdk_thr["0 ± 4 ч/сутки"]
  n_pdk_cmp{"ПДК ВОЗ > 35?"}
  n_trap_check[/"Экологическая ловушка"/]
  n_severity{"Уровень критичности"}
  n_citizen_alert[\"Совет жителям · P3"\]
  n_vulnerable_sms[\"SMS уязвимым группам · P2"\]
  n_nmu_regime[\"Режим НМУ · P1"\]
  n_emergency_broadcast[\"Экстренное оповещение · P1"\]
  n_shacl_pm25[["SHACL: pm25"]]
  n_shacl_windSpeed[["SHACL: windSpeed"]]
  n_shacl_pdkExceedanceHours[["SHACL: pdkExceedanceHours"]]
  n_wind_in --> n_wind_thr
  n_wind_thr --> n_wind_cmp
  n_wind_cmp -->|calm| n_trap_check
  n_pm25_in --> n_pm25_thr
  n_pm25_thr --> n_pm25_cmp
  n_pm25_cmp -->|elevated| n_trap_check
  n_pm25_cmp -->|level| n_severity
  n_pdk_in --> n_pdk_thr
  n_pdk_thr --> n_pdk_cmp
  n_pdk_cmp -->|pdk_exceeded| n_severity
  n_trap_check -->|trap| n_severity
  n_severity -->|warning| n_citizen_alert
  n_severity -->|elevated| n_vulnerable_sms
  n_severity -->|critical| n_nmu_regime
  n_severity -->|critical| n_emergency_broadcast
  n_pm25_cmp --> n_shacl_pm25
  n_wind_in --> n_shacl_windSpeed
  n_pdk_in --> n_shacl_pdkExceedanceHours

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_wind_in,n_pm25_in,n_pdk_in sensor;
  class n_wind_thr,n_pm25_thr,n_pdk_thr param;
  class n_wind_cmp,n_pm25_cmp,n_pdk_cmp,n_severity decision;
  class n_trap_check formula;
  class n_citizen_alert,n_vulnerable_sms,n_nmu_regime,n_emergency_broadcast action;
  class n_shacl_pm25,n_shacl_windSpeed,n_shacl_pdkExceedanceHours shacl;
```

## Исходные файлы

- [air-quality-smog-trap.flow.json](../../../backend/data/fixtures/air-quality-smog-trap.flow.json) — визуальный граф регламента (Rule DSL).
- [air-quality-smog-trap.data.ttl](../../../backend/data/fixtures/air-quality-smog-trap.data.ttl) — Turtle: параметры и метаданные регламента.
- [air-quality-smog-trap.shapes.ttl](../../../backend/data/fixtures/air-quality-smog-trap.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
