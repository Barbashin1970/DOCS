# Регламент NSK · life indices

**source_id:** `nsk-life-indices`  
**Дата редакции регламента:** 2026-05-20  
**Домен:** Города и районы (NSK OpenData) (`city`)

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: nsk-life-indices
---
flowchart LR
  n_sensor_0(("📡 Open-Meteo · weather (current) · temperature_2m"))
  n_in_0(("🚗 Гололёд: штраф"))
  n_thr_0["5 [0…8]"]
  n_shacl_0[["SHACL: temp_black_ice_score"]]
  n_sensor_1(("📡 Open-Meteo · weather (current) · wind_speed_10m"))
  n_in_1(("🚗 Сильный ветер: порог (м/с)"))
  n_thr_1["12 [5…20]"]
  n_shacl_1[["SHACL: wind_strong_ms"]]
  n_sensor_2(("📡 Open-Meteo · weather (current) · wind_speed_10m"))
  n_in_2(("🚗 Сильный ветер: штраф"))
  n_thr_2["2 [0…5]"]
  n_shacl_2[["SHACL: wind_strong_score"]]
  n_sensor_3(("📡 Open-Meteo · air-quality · pm2_5"))
  n_in_3(("🚶 PM2.5 высокий порог"))
  n_thr_3["35 [15…60]"]
  n_shacl_3[["SHACL: pm25_high"]]
  n_sensor_4(("📡 Open-Meteo · air-quality · pm2_5"))
  n_in_4(("🚶 PM2.5 высокий штраф"))
  n_thr_4["4 [0…6]"]
  n_shacl_4[["SHACL: pm25_high_penalty"]]
  n_sensor_5(("📡 Open-Meteo · air-quality · european_aqi"))
  n_in_5(("🚶 AQI средний порог"))
  n_thr_5["40 [20…60]"]
  n_shacl_5[["SHACL: aqi_medium"]]
  n_sensor_6(("📡 Open-Meteo · weather (current) · temperature_2m"))
  n_in_6(("🚶 Прохладно: порог (°C)"))
  n_thr_6["-5 [-15…0]"]
  n_shacl_6[["SHACL: temp_chilly"]]
  n_sensor_7(("📡 Open-Meteo · weather (current) · wind_speed_10m"))
  n_in_7(("🚶 Ветер: порог (м/с)"))
  n_thr_7["10 [5…20]"]
  n_shacl_7[["SHACL: wind_threshold_ms"]]
  n_cmp_7{"превышен?"}
  n_sensor_8(("📡 Open-Meteo · weather (daily) · temperature_2m_min"))
  n_in_8(("🏗️ Критич. мороз: порог (°C)"))
  n_thr_8["-30 [-50…-20]"]
  n_shacl_8[["SHACL: temp_critical"]]
  n_cmp_8{"превышен?"}
  n_sensor_9(("📡 Open-Meteo · air-quality · pm2_5"))
  n_in_9(("🏗️ Штиль+загрязнение: штраф"))
  n_thr_9["2 [0…5]"]
  n_shacl_9[["SHACL: smog_score"]]
  n_formula[/"3 индекса: Driver / Walk / Utility"/]
  n_switch{"Классификация уровня"}
  n_out_0[\"Безопасно · P1"\]
  n_out_1[\"Осторожно · P2"\]
  n_out_2[\"Опасно · P3"\]
  n_out_3[\"Крайне опасно · P4"\]
  n_in_0 --> n_thr_0
  n_in_0 --> n_shacl_0
  n_in_1 --> n_thr_1
  n_in_1 --> n_shacl_1
  n_in_2 --> n_thr_2
  n_in_2 --> n_shacl_2
  n_in_3 --> n_thr_3
  n_in_3 --> n_shacl_3
  n_in_4 --> n_thr_4
  n_in_4 --> n_shacl_4
  n_in_5 --> n_thr_5
  n_in_5 --> n_shacl_5
  n_in_6 --> n_thr_6
  n_in_6 --> n_shacl_6
  n_in_7 --> n_thr_7
  n_in_7 --> n_shacl_7
  n_thr_7 --> n_cmp_7
  n_in_8 --> n_thr_8
  n_in_8 --> n_shacl_8
  n_thr_8 --> n_cmp_8
  n_in_9 --> n_thr_9
  n_in_9 --> n_shacl_9
  n_thr_0 --> n_formula
  n_thr_1 --> n_formula
  n_thr_2 --> n_formula
  n_thr_3 --> n_formula
  n_thr_4 --> n_formula
  n_thr_5 --> n_formula
  n_thr_6 --> n_formula
  n_cmp_7 --> n_formula
  n_cmp_8 --> n_formula
  n_thr_9 --> n_formula
  n_formula --> n_switch
  n_switch -->|Безопасно| n_out_0
  n_switch -->|Осторожно| n_out_1
  n_switch -->|Опасно| n_out_2
  n_switch -->|Крайне опасно| n_out_3

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_sensor_0,n_in_0,n_sensor_1,n_in_1,n_sensor_2,n_in_2,n_sensor_3,n_in_3,n_sensor_4,n_in_4,n_sensor_5,n_in_5,n_sensor_6,n_in_6,n_sensor_7,n_in_7,n_sensor_8,n_in_8,n_sensor_9,n_in_9 sensor;
  class n_thr_0,n_thr_1,n_thr_2,n_thr_3,n_thr_4,n_thr_5,n_thr_6,n_thr_7,n_thr_8,n_thr_9 param;
  class n_shacl_0,n_shacl_1,n_shacl_2,n_shacl_3,n_shacl_4,n_shacl_5,n_shacl_6,n_shacl_7,n_shacl_8,n_shacl_9 shacl;
  class n_cmp_7,n_cmp_8,n_switch decision;
  class n_formula formula;
  class n_out_0,n_out_1,n_out_2,n_out_3 action;
```

## Исходные файлы

- [nsk-life-indices.flow.json](../../../backend/data/fixtures/nsk-life-indices.flow.json) — визуальный граф регламента (Rule DSL).
- [nsk-life-indices.data.ttl](../../../backend/data/fixtures/nsk-life-indices.data.ttl) — Turtle: параметры и метаданные регламента.
- [nsk-life-indices.shapes.ttl](../../../backend/data/fixtures/nsk-life-indices.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
