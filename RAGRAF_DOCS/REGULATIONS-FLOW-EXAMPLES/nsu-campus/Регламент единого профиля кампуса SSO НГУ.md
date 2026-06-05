# Регламент единого профиля кампуса (SSO) НГУ

**source_id:** `nsu-campus-digital-identity`  
**Дата редакции регламента:** 2024-02-28  
**Домен:** Кампус НГУ (`nsu-campus`)

> Регламент единого профиля кампуса НГУ (разд.

## Граф регламента (Rule DSL Flow)

```mermaid
---
title: nsu-campus-digital-identity
---
flowchart LR
  n_output[\"Рекомендация · P2"\]
  n_in_ssoCoverage(("ssoCoverage"))
  n_thr_ssoCoverage["95.0 ± 5.0 %"]
  n_cmp_ssoCoverage{"вне диапазона?"}
  n_in_firstAuthLatency(("firstAuthLatency"))
  n_thr_firstAuthLatency["3.0 ± 2.0 сек"]
  n_cmp_firstAuthLatency{"вне диапазона?"}
  n_in_activeProfiles(("activeProfiles"))
  n_thr_activeProfiles["92.0 ± 5.0 %"]
  n_cmp_activeProfiles{"вне диапазона?"}
  n_in_accountCompromiseAlerts(("accountCompromiseAlerts"))
  n_thr_accountCompromiseAlerts["0.0 ± 0.0 шт/мес"]
  n_cmp_accountCompromiseAlerts{"вне диапазона?"}
  n_in_deprovisioningLatency(("deprovisioningLatency"))
  n_thr_deprovisioningLatency["8.0 ± 8.0 часы"]
  n_cmp_deprovisioningLatency{"вне диапазона?"}
  n_in_mfaPenetration(("mfaPenetration"))
  n_thr_mfaPenetration["80.0 ± 10.0 %"]
  n_cmp_mfaPenetration{"вне диапазона?"}
  n_formula_demo[/"max(ssoCoverage, firstAuthLatency…)"/]
  n_shacl_ssoCoverage[["SHACL: ssoCoverage"]]
  n_shacl_firstAuthLatency[["SHACL: firstAuthLatency"]]
  n_shacl_activeProfiles[["SHACL: activeProfiles"]]
  n_shacl_accountCompromiseAlerts[["SHACL: accountCompromiseAlerts"]]
  n_shacl_deprovisioningLatency[["SHACL: deprovisioningLatency"]]
  n_shacl_mfaPenetration[["SHACL: mfaPenetration"]]
  n_in_ssoCoverage --> n_thr_ssoCoverage
  n_thr_ssoCoverage --> n_cmp_ssoCoverage
  n_cmp_ssoCoverage -->|outside| n_output
  n_in_firstAuthLatency --> n_thr_firstAuthLatency
  n_thr_firstAuthLatency --> n_cmp_firstAuthLatency
  n_cmp_firstAuthLatency -->|outside| n_output
  n_in_activeProfiles --> n_thr_activeProfiles
  n_thr_activeProfiles --> n_cmp_activeProfiles
  n_cmp_activeProfiles -->|outside| n_output
  n_in_accountCompromiseAlerts --> n_thr_accountCompromiseAlerts
  n_thr_accountCompromiseAlerts --> n_cmp_accountCompromiseAlerts
  n_cmp_accountCompromiseAlerts -->|outside| n_output
  n_in_deprovisioningLatency --> n_thr_deprovisioningLatency
  n_thr_deprovisioningLatency --> n_cmp_deprovisioningLatency
  n_cmp_deprovisioningLatency -->|outside| n_output
  n_in_mfaPenetration --> n_thr_mfaPenetration
  n_thr_mfaPenetration --> n_cmp_mfaPenetration
  n_cmp_mfaPenetration -->|outside| n_output
  n_in_ssoCoverage --> n_formula_demo
  n_in_firstAuthLatency --> n_formula_demo
  n_in_activeProfiles --> n_formula_demo
  n_in_accountCompromiseAlerts --> n_formula_demo
  n_in_ssoCoverage --> n_shacl_ssoCoverage
  n_in_firstAuthLatency --> n_shacl_firstAuthLatency
  n_in_activeProfiles --> n_shacl_activeProfiles
  n_in_accountCompromiseAlerts --> n_shacl_accountCompromiseAlerts
  n_in_deprovisioningLatency --> n_shacl_deprovisioningLatency
  n_in_mfaPenetration --> n_shacl_mfaPenetration

  classDef sensor   fill:#dbeafe,stroke:#2563eb,color:#1e3a8a;
  classDef param    fill:#f1f5f9,stroke:#475569,color:#0f172a;
  classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f;
  classDef formula  fill:#ede9fe,stroke:#7c3aed,color:#4c1d95;
  classDef shacl    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
  classDef action   fill:#dcfce7,stroke:#16a34a,color:#14532d;
  class n_output action;
  class n_in_ssoCoverage,n_in_firstAuthLatency,n_in_activeProfiles,n_in_accountCompromiseAlerts,n_in_deprovisioningLatency,n_in_mfaPenetration sensor;
  class n_thr_ssoCoverage,n_thr_firstAuthLatency,n_thr_activeProfiles,n_thr_accountCompromiseAlerts,n_thr_deprovisioningLatency,n_thr_mfaPenetration param;
  class n_cmp_ssoCoverage,n_cmp_firstAuthLatency,n_cmp_activeProfiles,n_cmp_accountCompromiseAlerts,n_cmp_deprovisioningLatency,n_cmp_mfaPenetration decision;
  class n_formula_demo formula;
  class n_shacl_ssoCoverage,n_shacl_firstAuthLatency,n_shacl_activeProfiles,n_shacl_accountCompromiseAlerts,n_shacl_deprovisioningLatency,n_shacl_mfaPenetration shacl;
```

## Исходные файлы

- [nsu-campus-digital-identity.flow.json](../../../backend/data/fixtures/nsu-campus-digital-identity.flow.json) — визуальный граф регламента (Rule DSL).
- [nsu-campus-digital-identity.data.ttl](../../../backend/data/fixtures/nsu-campus-digital-identity.data.ttl) — Turtle: параметры и метаданные регламента.
- [nsu-campus-digital-identity.shapes.ttl](../../../backend/data/fixtures/nsu-campus-digital-identity.shapes.ttl) — SHACL-ограничения.

---

Эта страница сгенерирована автоматически из `*.flow.json` скриптом 
[`scripts/export_flows_to_mermaid.py`](../../../scripts/export_flows_to_mermaid.py). 
Регенерируется при изменении регламента. Возврат к индексу — [`../README.md`](../README.md).
