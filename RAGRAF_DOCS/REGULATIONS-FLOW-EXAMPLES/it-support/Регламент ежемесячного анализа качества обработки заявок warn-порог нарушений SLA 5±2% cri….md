# Регламент ежемесячного анализа качества обработки заявок (warn-порог нарушений SLA 5±2%, critical-порог 10±3%, warn по переоткрытиям 8±3%, отчёт руководителю 1±0 числа)

**source_id:** `support-monthly-quality-review`  
**Дата редакции регламента:** 2026-05-26  
**Домен:** ИТ-поддержка (`it-support`)

> Применяется на каждом snapshot support-sla-violation с period='month' (формируется в полночь 1-го числа).

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: support-monthly-quality-review
---
flowchart LR
  n_sensor_breach_ratio(("📊 service-desk · доля нарушений SLA, %"))
  n_sensor_reopened(("🔁 service-desk · переоткрытых тикетов"))
  n_sensor_total_resolved(("📦 service-desk · всего закрытых"))
  n_sensor_throughput_7d(("🚀 LEYKA · закрыто за 7 дней"))
  n_sensor_in_time_ratio(("⏲️ LEYKA · доля «в срок», 0..1"))
  n_in_breach_ratio(("Доля нарушений, %"))
  n_in_reopened(("Переоткрыто"))
  n_in_total_resolved(("Всего закрыто"))
  n_in_throughput_7d(("Throughput за 7 дней"))
  n_in_in_time_ratio(("Доля «в срок»"))
  n_in_warn_breach(("Warn-порог нарушений, %"))
  n_shacl_warn[["SHACL: breachRatioWarnPercent"]]
  n_in_critical_breach(("Critical-порог, %"))
  n_shacl_critical[["SHACL: breachRatioCriticalPercent"]]
  n_in_reopened_warn(("Warn по переоткрытиям, %"))
  n_shacl_reopened[["SHACL: reopenedRatioWarnPercent"]]
  n_in_report_day(("День отчёта"))
  n_shacl_report_day[["SHACL: reportDeliveryDayOfMonth"]]
  n_formula_reopen_ratio[/"🧮 reopenRatio = reopenedCount / totalResolved × 100"/]
  n_formula_health_score[/"📈 healthScore = 100 − breach − reopenRatio/2 + (inTime × 20)"/]
  n_formula_productivity[/"🚀 productivity = throughput7d × inTimeRatio"/]
  n_thr_warn["5 [1…30]"]
  n_thr_critical["10 [5…50]"]
  n_thr_health["Health ≥ 80"]
  n_switch_status{"Уровень качества за период"}
  n_out_critical_plan[\"🚨 План улучшений + обучение команды · P3"\]
  n_out_warning_report[\"📋 Warning: разбор топ-5 нарушений · P2"\]
  n_out_prod_drop[\"📉 Productivity drop: throughput упал, healthScore < 80 · P2"\]
  n_out_ok_report[\"✅ Отчёт «всё в норме» + benchmarking · P1"\]
  n_out_reopen_audit[\"🔍 Аудит качества решений (parallel-branch) · P2"\]
  n_sensor_breach_ratio --> n_in_breach_ratio
  n_sensor_reopened --> n_in_reopened
  n_sensor_total_resolved --> n_in_total_resolved
  n_sensor_throughput_7d --> n_in_throughput_7d
  n_sensor_in_time_ratio --> n_in_in_time_ratio
  n_in_reopened --> n_formula_reopen_ratio
  n_in_total_resolved --> n_formula_reopen_ratio
  n_in_breach_ratio --> n_formula_health_score
  n_in_reopened --> n_formula_health_score
  n_in_total_resolved --> n_formula_health_score
  n_in_in_time_ratio --> n_formula_health_score
  n_in_throughput_7d --> n_formula_productivity
  n_in_in_time_ratio --> n_formula_productivity
  n_in_warn_breach --> n_shacl_warn
  n_in_warn_breach --> n_thr_warn
  n_in_breach_ratio --> n_thr_warn
  n_in_critical_breach --> n_shacl_critical
  n_in_critical_breach --> n_thr_critical
  n_in_breach_ratio --> n_thr_critical
  n_in_reopened_warn --> n_shacl_reopened
  n_in_reopened_warn --> n_out_reopen_audit
  n_formula_reopen_ratio --> n_out_reopen_audit
  n_formula_health_score --> n_thr_health
  n_in_report_day --> n_shacl_report_day
  n_thr_warn --> n_switch_status
  n_thr_critical --> n_switch_status
  n_thr_health --> n_switch_status
  n_switch_status -->|critical| n_out_critical_plan
  n_switch_status -->|warning| n_out_warning_report
  n_switch_status -->|prod_drop| n_out_prod_drop
  n_switch_status -->|ok| n_out_ok_report

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_sensor_breach_ratio,n_sensor_reopened,n_sensor_total_resolved,n_sensor_throughput_7d,n_sensor_in_time_ratio,n_in_breach_ratio,n_in_reopened,n_in_total_resolved,n_in_throughput_7d,n_in_in_time_ratio,n_in_warn_breach,n_in_critical_breach,n_in_reopened_warn,n_in_report_day sensor;
  class n_shacl_warn,n_shacl_critical,n_shacl_reopened,n_shacl_report_day shacl;
  class n_formula_reopen_ratio,n_formula_health_score,n_formula_productivity formula;
  class n_thr_warn,n_thr_critical,n_thr_health param;
  class n_switch_status decision;
  class n_out_critical_plan,n_out_warning_report,n_out_prod_drop,n_out_ok_report,n_out_reopen_audit action;
```

## Исходные файлы

- [support-monthly-quality-review.flow.json](../../../backend/data/fixtures/support-monthly-quality-review.flow.json) — визуальный граф регламента (Rule DSL).
- [support-monthly-quality-review.data.ttl](../../../backend/data/fixtures/support-monthly-quality-review.data.ttl) — Turtle: параметры и метаданные регламента.
- [support-monthly-quality-review.shapes.ttl](../../../backend/data/fixtures/support-monthly-quality-review.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
