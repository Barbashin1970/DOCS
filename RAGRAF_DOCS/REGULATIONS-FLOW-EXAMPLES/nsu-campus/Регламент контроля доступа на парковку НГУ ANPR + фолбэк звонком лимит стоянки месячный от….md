# Регламент контроля доступа на парковку НГУ (ANPR + фолбэк звонком, лимит стоянки, месячный отчёт)

**source_id:** `nsu-parking-anpr`  
**Дата редакции регламента:** 2024-09-01  
**Домен:** Кампус НГУ (`nsu-campus`)

> ВЪЕЗД.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: nsu-parking-anpr
---
flowchart LR
  n_anpr_in_sensor(("Камера-ANPR (въезд)"))
  n_call_sensor(("Входящий звонок"))
  n_anpr_out_sensor(("Камера-ANPR (выезд)"))
  n_plate_in(("ГРЗ (въезд)"))
  n_conf_in(("Confidence ANPR"))
  n_phone_in(("MSISDN звонящего"))
  n_plate_out(("ГРЗ (выезд)"))
  n_conf_thr["0.85 ± 0.05"]
  n_conf_cmp{"conf ≥ 0.85?"}
  n_plate_in_allowlist[/"ГРЗ в allowlist?"/]
  n_phone_in_allowlist[/"Телефон в allowlist?"/]
  n_access_decision{"Разрешение въезда"}
  n_barrier_open[\"Открыть шлагбаум · P3"\]
  n_operator_fallback[\"Запрос оператора · P2"\]
  n_access_denied[\"Отказ во въезде · P2"\]
  n_duration_calc[/"duration = exit − entry"/]
  n_max_hours_thr["12 ± 0 ч"]
  n_overtime_cmp{"duration > 12 ч + grace?"}
  n_owner_notify[\"SMS владельцу · P2"\]
  n_exit_log[\"Лог выезда · P3"\]
  n_exit_without_entry[\"ALERT: выезд без въезда · P1"\]
  n_monthly_report[\"Месячный отчёт · P3"\]
  n_shacl_plate[["SHACL: формат ГРЗ"]]
  n_shacl_recognitionConfidence[["SHACL: recognitionConfidence"]]
  n_anpr_in_sensor --> n_plate_in
  n_anpr_in_sensor --> n_conf_in
  n_call_sensor --> n_phone_in
  n_anpr_out_sensor --> n_plate_out
  n_plate_in --> n_plate_in_allowlist
  n_plate_in --> n_shacl_plate
  n_conf_in --> n_conf_thr
  n_conf_thr --> n_conf_cmp
  n_phone_in --> n_phone_in_allowlist
  n_plate_in_allowlist -->|plate_ok| n_access_decision
  n_conf_cmp -->|conf_ok| n_access_decision
  n_phone_in_allowlist -->|phone_ok| n_access_decision
  n_access_decision -->|auto_open| n_barrier_open
  n_access_decision -->|operator_open| n_operator_fallback
  n_access_decision -->|denied| n_access_denied
  n_plate_out --> n_duration_calc
  n_duration_calc --> n_max_hours_thr
  n_max_hours_thr --> n_overtime_cmp
  n_overtime_cmp -->|over| n_owner_notify
  n_overtime_cmp -->|ok| n_exit_log
  n_plate_out -->|no_entry_record| n_exit_without_entry
  n_exit_log --> n_monthly_report
  n_conf_in --> n_shacl_recognitionConfidence

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_anpr_in_sensor,n_call_sensor,n_anpr_out_sensor,n_plate_in,n_conf_in,n_phone_in,n_plate_out sensor;
  class n_conf_thr,n_max_hours_thr param;
  class n_conf_cmp,n_access_decision,n_overtime_cmp decision;
  class n_plate_in_allowlist,n_phone_in_allowlist,n_duration_calc formula;
  class n_barrier_open,n_operator_fallback,n_access_denied,n_owner_notify,n_exit_log,n_exit_without_entry,n_monthly_report action;
  class n_shacl_plate,n_shacl_recognitionConfidence shacl;
```

## Исходные файлы

- [nsu-parking-anpr.flow.json](../../../backend/data/fixtures/nsu-parking-anpr.flow.json) — визуальный граф регламента (Rule DSL).
- [nsu-parking-anpr.data.ttl](../../../backend/data/fixtures/nsu-parking-anpr.data.ttl) — Turtle: параметры и метаданные регламента.
- [nsu-parking-anpr.shapes.ttl](../../../backend/data/fixtures/nsu-parking-anpr.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
