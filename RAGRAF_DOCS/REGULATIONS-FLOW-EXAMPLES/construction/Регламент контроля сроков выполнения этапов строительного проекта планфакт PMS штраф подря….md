# Регламент контроля сроков выполнения этапов строительного проекта (план/факт PMS, штраф подрядчику, эскалация инвестору)

**source_id:** `construction-overdue-stage`  
**Дата редакции регламента:** 2026-05-25  
**Домен:** Строительство (`construction`)

> При обнаружении просрочки этапа в PMS: 1) Зафиксировать разрыв (план vs факт).

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: construction-overdue-stage
---
flowchart LR
  n_sensor_stage_overdue(("📋 PMS · просрочка этапа (days)"))
  n_in_allowed(("Допустимая просрочка (warn)"))
  n_thr_allowed["3 [0…30]"]
  n_in_critical(("Критическая просрочка (эскалация)"))
  n_thr_critical["14 [1…180]"]
  n_in_penalty(("Штраф за день, ₽"))
  n_formula_penalty[/"penalty_total = overdue_days * penaltyRubPerDay"/]
  n_switch_severity{"Уровень просрочки"}
  n_out_gip_notification[\"📣 SMS ГИП + расчёт штрафа"\]
  n_out_investor_escalation[\"🚨 Эскалация инвестору, рассмотрение расторжения"\]
  n_shacl_allowedOverdueDays[["SHACL: allowedOverdueDays"]]
  n_shacl_criticalOverdueDays[["SHACL: criticalOverdueDays"]]
  n_shacl_penaltyRubPerDay[["SHACL: penaltyRubPerDay"]]
  n_sensor_contract_meta(("📑 1С · метаданные договора (млн ₽)"))
  n_in_contract_value(("Сумма договора, млн ₽"))
  n_shacl_contractValueMillionRub[["SHACL: contractValueMillionRub"]]
  n_formula_risk_score[/"risk_score = overdue × contract / 1000"/]
  n_sensor_stage_overdue --> n_in_allowed
  n_in_allowed --> n_thr_allowed
  n_in_critical --> n_thr_critical
  n_in_penalty --> n_formula_penalty
  n_in_allowed --> n_formula_penalty
  n_thr_allowed --> n_switch_severity
  n_thr_critical --> n_switch_severity
  n_formula_penalty --> n_out_gip_notification
  n_switch_severity -->|Просрочка ≥ allowedOverdueDays| n_out_gip_notification
  n_switch_severity -->|Просрочка ≥ criticalOverdueDays| n_out_investor_escalation
  n_in_allowed --> n_shacl_allowedOverdueDays
  n_in_critical --> n_shacl_criticalOverdueDays
  n_in_penalty --> n_shacl_penaltyRubPerDay
  n_sensor_contract_meta --> n_in_contract_value
  n_in_contract_value --> n_shacl_contractValueMillionRub
  n_in_contract_value --> n_formula_risk_score
  n_in_allowed --> n_formula_risk_score
  n_formula_risk_score --> n_out_investor_escalation

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_sensor_stage_overdue,n_in_allowed,n_in_critical,n_in_penalty,n_sensor_contract_meta,n_in_contract_value sensor;
  class n_thr_allowed,n_thr_critical param;
  class n_formula_penalty,n_formula_risk_score formula;
  class n_switch_severity decision;
  class n_out_gip_notification,n_out_investor_escalation action;
  class n_shacl_allowedOverdueDays,n_shacl_criticalOverdueDays,n_shacl_penaltyRubPerDay,n_shacl_contractValueMillionRub shacl;
```

## Исходные файлы

- [construction-overdue-stage.flow.json](../../../backend/data/fixtures/construction-overdue-stage.flow.json) — визуальный граф регламента (Rule DSL).
- [construction-overdue-stage.data.ttl](../../../backend/data/fixtures/construction-overdue-stage.data.ttl) — Turtle: параметры и метаданные регламента.
- [construction-overdue-stage.shapes.ttl](../../../backend/data/fixtures/construction-overdue-stage.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
