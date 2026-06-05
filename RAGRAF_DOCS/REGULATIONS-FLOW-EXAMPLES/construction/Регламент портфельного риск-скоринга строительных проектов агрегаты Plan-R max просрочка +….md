# Регламент портфельного риск-скоринга строительных проектов (агрегаты Plan-R: max просрочка + critical-count × стоимость договора, эскалация инвестору и совету директоров)

**source_id:** `construction-portfolio-risk`  
**Дата редакции регламента:** 2026-05-25  
**Домен:** Строительство (`construction`)

> Регламент срабатывает каждые 5 минут после pull_planr из Plan-R Public API.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: construction-portfolio-risk
---
flowchart LR
  n_sensor_max_overdue(("📋 Plan-R · максимум overdue_days по портфелю"))
  n_in_warn(("Warn-порог просрочки"))
  n_thr_warn["5 [1…30]"]
  n_shacl_portfolioWarnOverdueDays[["SHACL: portfolioWarnOverdueDays"]]
  n_in_critical(("Critical-порог (совет директоров)"))
  n_thr_critical["14 [1…90]"]
  n_shacl_portfolioCriticalOverdueDays[["SHACL: portfolioCriticalOverdueDays"]]
  n_sensor_critical_count(("📋 Plan-R · число critical-работ в портфеле"))
  n_in_critical_count(("Порог числа critical-работ"))
  n_thr_critical_count["2 [1…20]"]
  n_shacl_portfolioCriticalCountThreshold[["SHACL: portfolioCriticalCountThreshold"]]
  n_sensor_contract_value(("📑 Plan-R · стоимость договора (1С)"))
  n_in_contract_weight(("Вес стоимости договора"))
  n_thr_contract_weight["0.5 [0.1…5]"]
  n_shacl_portfolioContractValueWeight[["SHACL: portfolioContractValueWeight"]]
  n_sensor_clock_hour(("🕐 RAGRAF clock · текущий час UTC"))
  n_in_digest_hour(("Час дайджеста, UTC"))
  n_thr_digest_hour["7 [0…23]"]
  n_shacl_dailyDigestHourUtc[["SHACL: dailyDigestHourUtc"]]
  n_formula_risk_score[/"risk_score = max_overdue × critical_count × contract_value × weight / 1000"/]
  n_formula_severity_band[/"max(warn, critical)"/]
  n_switch_severity{"Уровень портфельного риска"}
  n_out_digest[\"📰 Дневной digest директору · P1"\]
  n_out_gip_plan[\"📋 Задача ГИП: план компенсирующих мер за 48ч · P2"\]
  n_out_board_escalation[\"🚨 Совет директоров: совещание за 24ч · P3"\]
  n_sensor_max_overdue --> n_in_warn
  n_in_warn --> n_thr_warn
  n_in_warn --> n_shacl_portfolioWarnOverdueDays
  n_in_critical --> n_thr_critical
  n_in_critical --> n_shacl_portfolioCriticalOverdueDays
  n_sensor_critical_count --> n_in_critical_count
  n_in_critical_count --> n_thr_critical_count
  n_in_critical_count --> n_shacl_portfolioCriticalCountThreshold
  n_sensor_contract_value --> n_in_contract_weight
  n_in_contract_weight --> n_thr_contract_weight
  n_in_contract_weight --> n_shacl_portfolioContractValueWeight
  n_sensor_clock_hour --> n_in_digest_hour
  n_in_digest_hour --> n_thr_digest_hour
  n_in_digest_hour --> n_shacl_dailyDigestHourUtc
  n_in_warn --> n_formula_severity_band
  n_in_critical --> n_formula_severity_band
  n_in_warn --> n_formula_risk_score
  n_in_critical_count --> n_formula_risk_score
  n_in_contract_weight --> n_formula_risk_score
  n_thr_warn --> n_switch_severity
  n_thr_critical --> n_switch_severity
  n_thr_critical_count --> n_switch_severity
  n_formula_risk_score --> n_switch_severity
  n_thr_digest_hour --> n_out_digest
  n_switch_severity -->|ok| n_out_digest
  n_switch_severity -->|warning| n_out_gip_plan
  n_switch_severity -->|board_escalate| n_out_board_escalation

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_sensor_max_overdue,n_in_warn,n_in_critical,n_sensor_critical_count,n_in_critical_count,n_sensor_contract_value,n_in_contract_weight,n_sensor_clock_hour,n_in_digest_hour sensor;
  class n_thr_warn,n_thr_critical,n_thr_critical_count,n_thr_contract_weight,n_thr_digest_hour param;
  class n_shacl_portfolioWarnOverdueDays,n_shacl_portfolioCriticalOverdueDays,n_shacl_portfolioCriticalCountThreshold,n_shacl_portfolioContractValueWeight,n_shacl_dailyDigestHourUtc shacl;
  class n_formula_risk_score,n_formula_severity_band formula;
  class n_switch_severity decision;
  class n_out_digest,n_out_gip_plan,n_out_board_escalation action;
```

## Исходные файлы

- [construction-portfolio-risk.flow.json](../../../backend/data/fixtures/construction-portfolio-risk.flow.json) — визуальный граф регламента (Rule DSL).
- [construction-portfolio-risk.data.ttl](../../../backend/data/fixtures/construction-portfolio-risk.data.ttl) — Turtle: параметры и метаданные регламента.
- [construction-portfolio-risk.shapes.ttl](../../../backend/data/fixtures/construction-portfolio-risk.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
