# Регламент мониторинга автономных дымовых пожарных извещателей АДПИ GSM (ЕДДС р.п. Кольцово)

**source_id:** `koltsovo-edds-adpi-monitoring`  
**Дата редакции регламента:** 2020-12-04  
**Домен:** Ситуационный центр / ЕДДС (`emergency_response`)

> Мониторинг АДПИ GSM в ЕДДС р.п.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: koltsovo-edds-adpi-monitoring
---
flowchart LR
  n_delivery_in(("Доставка тревоги"))
  n_delivery_thr["15 ± 5 с"]
  n_delivery_cmp{"канал GSM ок?"}
  n_confirm_in(("Окно подтверждения"))
  n_confirm_thr["5 ± 2 мин"]
  n_confirm_cmp{"подтвердили / нет?"}
  n_false_in(("Ложные срабатывания, %"))
  n_false_thr["5 ± 2 %"]
  n_false_cmp{"качество в норме?"}
  n_decision{"Решение оператора"}
  n_dispatch_fire[\"В пожарную ДДС «01» · P1"\]
  n_low_battery[\"Низкий заряд → УК / собственнику · P3"\]
  n_quality_review[\"Превышение ложных → анализ · P3"\]
  n_mass_escalate[\"Массовое срабатывание → ЦУКС · P1"\]
  n_shacl_delivery[["SHACL: alarmDeliverySeconds"]]
  n_formula_demo[/"max(alarmDeliverySeconds, operatorConfirmMinutes…)"/]
  n_shacl_operatorConfirmMinutes[["SHACL: operatorConfirmMinutes"]]
  n_shacl_falseAlarmRatePercent[["SHACL: falseAlarmRatePercent"]]
  n_delivery_in --> n_delivery_thr
  n_delivery_thr --> n_delivery_cmp
  n_delivery_cmp -->|delivered| n_decision
  n_confirm_in --> n_confirm_thr
  n_confirm_thr --> n_confirm_cmp
  n_confirm_cmp -->|confirm_window| n_decision
  n_false_in --> n_false_thr
  n_false_thr --> n_false_cmp
  n_false_cmp -->|above_5pct| n_quality_review
  n_decision -->|confirmed_fire| n_dispatch_fire
  n_decision -->|no_contact| n_dispatch_fire
  n_decision -->|low_battery| n_low_battery
  n_decision -->|mass_event| n_mass_escalate
  n_delivery_cmp --> n_shacl_delivery
  n_delivery_in --> n_formula_demo
  n_confirm_in --> n_formula_demo
  n_false_in --> n_formula_demo
  n_confirm_in --> n_shacl_operatorConfirmMinutes
  n_false_in --> n_shacl_falseAlarmRatePercent

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_delivery_in,n_confirm_in,n_false_in sensor;
  class n_delivery_thr,n_confirm_thr,n_false_thr param;
  class n_delivery_cmp,n_confirm_cmp,n_false_cmp,n_decision decision;
  class n_dispatch_fire,n_low_battery,n_quality_review,n_mass_escalate action;
  class n_shacl_delivery,n_shacl_operatorConfirmMinutes,n_shacl_falseAlarmRatePercent shacl;
  class n_formula_demo formula;
```

## Исходные файлы

- [koltsovo-edds-adpi-monitoring.flow.json](../../../backend/data/fixtures/koltsovo-edds-adpi-monitoring.flow.json) — визуальный граф регламента (Rule DSL).
- [koltsovo-edds-adpi-monitoring.data.ttl](../../../backend/data/fixtures/koltsovo-edds-adpi-monitoring.data.ttl) — Turtle: параметры и метаданные регламента.
- [koltsovo-edds-adpi-monitoring.shapes.ttl](../../../backend/data/fixtures/koltsovo-edds-adpi-monitoring.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
