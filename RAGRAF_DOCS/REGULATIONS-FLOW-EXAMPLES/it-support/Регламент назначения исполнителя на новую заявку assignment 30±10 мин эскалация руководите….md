# Регламент назначения исполнителя на новую заявку (assignment 30±10 мин, эскалация руководителю 60±15 мин, повторные уведомления каждые 15±5 мин)

**source_id:** `support-assignment-overdue`  
**Дата редакции регламента:** 2026-05-26  
**Домен:** ИТ-поддержка (`it-support`)

> Регламент применяется к каждому тикету в статусе 'new' старше assignmentSlaMinutes.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: support-assignment-overdue
---
flowchart LR
  n_sensor_status_age(("⏱️ service-desk · возраст в статусе 'new'"))
  n_sensor_team_wip(("👥 LEYKA · макс. задач у инженера"))
  n_sensor_active(("✅ LEYKA · активных исполнителей"))
  n_sensor_untriaged(("📥 LEYKA · untriaged-задачи в inbox"))
  n_in_status_age(("Возраст в 'new', мин"))
  n_in_max_wip(("Макс задач у инженера"))
  n_in_active(("Активных исполнителей"))
  n_in_untriaged(("Untriaged в inbox"))
  n_in_assignment_sla(("SLA назначения, мин"))
  n_shacl_assignment[["SHACL: assignmentSlaMinutes"]]
  n_in_escalation_sla(("SLA эскалации, мин"))
  n_shacl_escalation[["SHACL: escalationSlaMinutes"]]
  n_in_reminder(("Интервал пинга, мин"))
  n_shacl_reminder[["SHACL: reminderIntervalMinutes"]]
  n_formula_overdue[/"🧮 overdueMin = max(0, statusAge − assignmentSla)"/]
  n_formula_reminder_count[/"🧮 remindersDue = overdue / reminderInterval"/]
  n_formula_capacity[/"🧮 freeSlots = activeAssignees × 5 − maxPerAssignee × activeAssignees"/]
  n_formula_assign_score[/"📈 assignScore = freeSlots / max(1, untriagedTotal)"/]
  n_thr_assignment["30 [5…240]"]
  n_thr_escalation["60 [15…480]"]
  n_switch_band{"Действие по тикету"}
  n_out_escalate[\"🚨 LEYKA-задача руководителю ТП + SMS · P3"\]
  n_out_rebalance[\"⚡ Перебалансировка команды (нет capacity) · P2"\]
  n_out_remind[\"🔔 Повторный пинг дежурному · P2"\]
  n_out_ok[\"✅ В норме — ожидание назначения · P1"\]
  n_sensor_status_age --> n_in_status_age
  n_sensor_team_wip --> n_in_max_wip
  n_sensor_active --> n_in_active
  n_sensor_untriaged --> n_in_untriaged
  n_in_status_age --> n_formula_overdue
  n_in_assignment_sla --> n_formula_overdue
  n_in_assignment_sla --> n_shacl_assignment
  n_in_assignment_sla --> n_thr_assignment
  n_formula_overdue --> n_thr_assignment
  n_in_escalation_sla --> n_shacl_escalation
  n_in_escalation_sla --> n_thr_escalation
  n_in_status_age --> n_thr_escalation
  n_in_reminder --> n_shacl_reminder
  n_in_reminder --> n_formula_reminder_count
  n_formula_overdue --> n_formula_reminder_count
  n_in_active --> n_formula_capacity
  n_in_max_wip --> n_formula_capacity
  n_formula_capacity --> n_formula_assign_score
  n_in_untriaged --> n_formula_assign_score
  n_thr_assignment --> n_switch_band
  n_thr_escalation --> n_switch_band
  n_formula_assign_score --> n_out_rebalance
  n_switch_band -->|escalate| n_out_escalate
  n_switch_band -->|rebalance| n_out_rebalance
  n_switch_band -->|remind| n_out_remind
  n_switch_band -->|ok| n_out_ok

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_sensor_status_age,n_sensor_team_wip,n_sensor_active,n_sensor_untriaged,n_in_status_age,n_in_max_wip,n_in_active,n_in_untriaged,n_in_assignment_sla,n_in_escalation_sla,n_in_reminder sensor;
  class n_shacl_assignment,n_shacl_escalation,n_shacl_reminder shacl;
  class n_formula_overdue,n_formula_reminder_count,n_formula_capacity,n_formula_assign_score formula;
  class n_thr_assignment,n_thr_escalation param;
  class n_switch_band decision;
  class n_out_escalate,n_out_rebalance,n_out_remind,n_out_ok action;
```

## Исходные файлы

- [support-assignment-overdue.flow.json](../../../backend/data/fixtures/support-assignment-overdue.flow.json) — визуальный граф регламента (Rule DSL).
- [support-assignment-overdue.data.ttl](../../../backend/data/fixtures/support-assignment-overdue.data.ttl) — Turtle: параметры и метаданные регламента.
- [support-assignment-overdue.shapes.ttl](../../../backend/data/fixtures/support-assignment-overdue.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
