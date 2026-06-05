# Регламент NSK · mobile index

**source_id:** `nsk-mobile-index`  
**Дата редакции регламента:** 2026-05-20  
**Домен:** Города и районы (NSK OpenData) (`city`)

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: nsk-mobile-index
---
flowchart LR
  n_sensor_0(("📡 2-ТП Воздух (статика) · emissions_total_t"))
  n_in_0(("Вес стационарных (I_стац)"))
  n_thr_0["0.6 [0…1]"]
  n_shacl_0[["SHACL: w_stationary"]]
  n_sensor_1(("📡 Производный (traffic_index) · I_mobile"))
  n_in_1(("Вес транспорта (I_транспорт)"))
  n_thr_1["0.4 [0…1]"]
  n_shacl_1[["SHACL: w_mobile"]]
  n_in_2(("🚘 Вес плотности автопарка"))
  n_thr_2["0.4 [0…1]"]
  n_shacl_2[["SHACL: alpha_cars"]]
  n_sensor_3(("📡 Yandex Traffic · level"))
  n_in_3(("🚦 Вес пробок"))
  n_thr_3["0.6 [0…1]"]
  n_shacl_3[["SHACL: alpha_jam"]]
  n_in_4(("🚇 Компенсация метро"))
  n_thr_4["0.5 [0…1]"]
  n_shacl_4[["SHACL: beta_metro"]]
  n_in_5(("🚊 Компенсация трамвай"))
  n_thr_5["0.3 [0…1]"]
  n_shacl_5[["SHACL: beta_tram"]]
  n_in_6(("🚎 Компенсация троллейбус"))
  n_thr_6["0.2 [0…1]"]
  n_shacl_6[["SHACL: beta_trolley"]]
  n_in_7(("📊 Макс. компенсация ОТ"))
  n_thr_7["0.4 [0…1]"]
  n_shacl_7[["SHACL: k_PT"]]
  n_sensor_8(("📡 Open-Meteo · weather (current) · wind_speed_10m"))
  n_in_8(("💨 Рассеивание ветром"))
  n_thr_8["0.3 [0…1]"]
  n_shacl_8[["SHACL: gamma_wind"]]
  n_sensor_9(("📡 Open-Meteo · weather (current) · wind_speed_10m"))
  n_in_9(("🌫 Накопление при штиле"))
  n_thr_9["0.4 [0…1]"]
  n_shacl_9[["SHACL: gamma_calm"]]
  n_in_10(("⛰ Надбавка за котловину"))
  n_thr_10["0.4 [0…1]"]
  n_shacl_10[["SHACL: delta_basin"]]
  n_formula[/"I_total = w_s * I_stationary + w_m * I_mobile"/]
  n_switch{"Классификация уровня"}
  n_out_0[\"🟢 Чистый транспорт · P1"\]
  n_out_1[\"🟡 Умеренная нагрузка · P2"\]
  n_out_2[\"🟠 Высокая нагрузка · P3"\]
  n_out_3[\"🔴 Критическая нагрузка · P4"\]
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
  n_in_8 --> n_thr_8
  n_in_8 --> n_shacl_8
  n_in_9 --> n_thr_9
  n_in_9 --> n_shacl_9
  n_in_10 --> n_thr_10
  n_in_10 --> n_shacl_10
  n_thr_0 --> n_formula
  n_thr_1 --> n_formula
  n_thr_2 --> n_formula
  n_thr_3 --> n_formula
  n_thr_4 --> n_formula
  n_thr_5 --> n_formula
  n_thr_6 --> n_formula
  n_thr_7 --> n_formula
  n_thr_8 --> n_formula
  n_thr_9 --> n_formula
  n_thr_10 --> n_formula
  n_formula --> n_switch
  n_switch -->|Чистый транспорт| n_out_0
  n_switch -->|Умеренная нагрузка| n_out_1
  n_switch -->|Высокая нагрузка| n_out_2
  n_switch -->|Критическая нагрузка| n_out_3

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_sensor_0,n_in_0,n_sensor_1,n_in_1,n_in_2,n_sensor_3,n_in_3,n_in_4,n_in_5,n_in_6,n_in_7,n_sensor_8,n_in_8,n_sensor_9,n_in_9,n_in_10 sensor;
  class n_thr_0,n_thr_1,n_thr_2,n_thr_3,n_thr_4,n_thr_5,n_thr_6,n_thr_7,n_thr_8,n_thr_9,n_thr_10 param;
  class n_shacl_0,n_shacl_1,n_shacl_2,n_shacl_3,n_shacl_4,n_shacl_5,n_shacl_6,n_shacl_7,n_shacl_8,n_shacl_9,n_shacl_10 shacl;
  class n_formula formula;
  class n_switch decision;
  class n_out_0,n_out_1,n_out_2,n_out_3 action;
```

## Исходные файлы

- [nsk-mobile-index.flow.json](../../../backend/data/fixtures/nsk-mobile-index.flow.json) — визуальный граф регламента (Rule DSL).
- [nsk-mobile-index.data.ttl](../../../backend/data/fixtures/nsk-mobile-index.data.ttl) — Turtle: параметры и метаданные регламента.
- [nsk-mobile-index.shapes.ttl](../../../backend/data/fixtures/nsk-mobile-index.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
