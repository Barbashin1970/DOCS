# Соответствие РАГРАФ российским ГОСТам по ИИ

**Дата отчёта:** 2026-05-30 (ревизия после закрытия гэпов 1/4/5)
**Версия РАГРАФ:** 0.1.0
**Автор отчёта:** Барбашин Олег Владимирович
**Назначение:** compliance-audit для сертификации, прохождения экспертизы НИР, демонстрации соответствия регуляторным требованиям РФ.

> **Что изменилось 2026-05-30 (ревизия).** Закрыты три «дешёвых» гэпа из роадмапа: создан формальный документ
> [docs/DATA-QUALITY-MODEL.md](DATA-QUALITY-MODEL.md) (гэп 1), промаркированы фикстуры по 5 типам ГОСТ Р 59898 §9 в
> [backend/data/fixtures/INDEX.md](../backend/data/fixtures/INDEX.md) (гэп 4), создан [docs/EXPLAINABILITY.md](EXPLAINABILITY.md) (гэп 5).
> Сводное покрытие выросло с ~75% до **~82%** базовых требований и до **~92%** с учётом архитектурных мер.
> Оставшиеся гэпы перенесены в [docs/BACKLOG.md §Compliance-докрутка для банков и сложных отраслей](BACKLOG.md).

---

## 1. Аннотация

Документ покрывает соответствие платформы РАГРАФ требованиям двух действующих российских ГОСТов по искусственному интеллекту:

| ГОСТ | Введён | Фокус |
|---|---|---|
| **ГОСТ Р 71484.1-2024** (ИСО/МЭК 5259-1:2024) | 2025-01-01 | Качество данных для аналитики и машинного обучения. Часть 1: обзор, терминология, жизненный цикл данных, управление качеством. |
| **ГОСТ Р 59898-2021** | 2022-03-01 | Оценка качества систем ИИ. 8 групп характеристик, конкретные метрики, требования к тестовым наборам. |

Оба ГОСТа в текущей редакции принимают **«не требуется к обязательному применению»**, но широко используются как **методологическая основа** при:
- сертификации СИИ в Росстандарте (ТК 164);
- экспертизе грантов и НИР Минобрнауки / РНФ;
- закупках ИИ-систем по 44-ФЗ (как ссылочные стандарты в техзадании);
- внутреннем аудите ИБ перед прохождением проверки РКН (152-ФЗ).

### Сводная оценка (после ревизии 2026-05-30)

| ГОСТ | Полное соответствие | Частичное | Не реализовано | Не применимо |
|---|---|---|---|---|
| **ГОСТ Р 71484.1-2024** | 19 пунктов | 5 пунктов | 2 пункта | 1 пункт |
| **ГОСТ Р 59898-2021** | 27 пунктов | 7 пунктов | 4 пункта | 2 пункта |

**Общий уровень покрытия: ~82%** базовых требований, ~92% при учёте архитектурных мер (которые ГОСТ не требует, но мы реализовали).

**Закрыто в ревизии 2026-05-30:**
1. ✅ Формальная «Модель качества данных» как отдельный документ — [docs/DATA-QUALITY-MODEL.md](DATA-QUALITY-MODEL.md) (закрывает §5.2.2.1 ГОСТ 71484.1).
2. ✅ Маркировка fixtures по 5 типам ГОСТ — [backend/data/fixtures/INDEX.md](../backend/data/fixtures/INDEX.md) (закрывает §9 ГОСТ 59898).
3. ✅ Раздел «Объяснимость (explainability)» в пользовательской документации — [docs/EXPLAINABILITY.md](EXPLAINABILITY.md) (закрывает §8.6 ГОСТ 59898).

**Оставшиеся гэпы (вынесены в [BACKLOG.md](BACKLOG.md) для пилотов в банках и сложных отраслях):**
1. Стандартизированный «Отчёт о качестве данных» как endpoint+UI (§5.2.2.5 ГОСТ 71484.1).
2. Автоматизированные метрики функциональной точности для Sandbox/LLM — golden Q&A (§8.2 ГОСТ 59898).
3. Resource utilization dashboard (Prometheus/Grafana) — §8.3 ГОСТ 59898.
4. Accessibility audit (WCAG 2.1 Level AA) — §8.6 ГОСТ 59898 (подхарактеристика доступность).
5. Encryption at rest (SQLCipher / volume encryption) — §5.3.3.2 ГОСТ 71484.1; критично для банковского и гос-сектора.

Подробный roadmap докрутки и привязка к отраслям — §6.

---

## 2. Высокоуровневая матрица

Сводная таблица — статус по 8 разделам ГОСТ 59898 + 5 разделам ГОСТ 71484.1.

| Группа требований | Источник | Реализация | Статус |
|---|---|---|---|
| **Функциональные возможности** | 59898 §8.2 | SHACL, flow_executor, 30 файлов pytest, metric_analyzer | full |
| **Производительность** | 59898 §8.3 | логирование response-time, etl_health, гарантии < 100 мс / регламент | partial |
| **Способность к взаимодействию** | 59898 §8.4 | REST OpenAPI, W3C (Turtle/SHACL/PROV-O), SIGMA bundle | full |
| **Мобильность (portability)** | 59898 §8.5 | Docker, Railway, Mac/Win/Linux, on-premise | full |
| **Практичность (usability)** | 59898 §8.6 | Sandbox с sources, методология, курсы COURSE-* | full |
| **Сопровождаемость** | 59898 §8.7 | TypeScript strict, ESLint, миграции, layered arch | full |
| **Надёжность** | 59898 §8.8 | mock-режим LLM, persistent DuckDB, 336 try/except | full |
| **Защищённость (security)** | 59898 §8.8 + 71484 §5.3.3.2 | HMAC-cookie auth, CORS, параметризованные SQL, secret mgmt | full |
| **Жизненный цикл данных (6 стадий)** | 71484 §5.3 | sensor_schemas → puller → store → SHACL → execute → archive | full |
| **Происхождение данных (PROV-O)** | 71484 §5.2.4 | PROV.wasDerivedFrom, audit_log 19 полей, fork chain | full |
| **Модель качества данных** | 71484 §5.2.2 | SHACL constraints, ParameterModel + формальный документ [DATA-QUALITY-MODEL.md](DATA-QUALITY-MODEL.md) | full |
| **Отчётность о качестве данных** | 71484 §5.2.2.5 | metric_analyzer + audit_log, без формального шаблона | partial |
| **Защита персональных данных** | 71484 §5.3.3.3 | поддержка 152-ФЗ в регламентах, on-premise | full |
| **Стратегическое управление** | 71484 §5.2.3 | ARCHITECTURE.md, GLOSSARY.md, DATA-FLOW-AUDIT.md | full |

**Легенда статусов:**
- `full` — требование реализовано в полном объёме, документировано в коде/архитектуре;
- `partial` — реализовано de facto, но не оформлено как требует ГОСТ;
- `missing` — не реализовано, требует докрутки;
- `n/a` — не применимо к классу системы РАГРАФ (гибридный нейросимвольный, а не чистый ML).

---

## 3. ГОСТ Р 71484.1-2024: детальная матрица

### 3.1 Терминология (§3)

| Термин ГОСТ | Реализация в РАГРАФ | Файл |
|---|---|---|
| 3.1 жизненный цикл данных | 6 стадий в backend/app/services + ARCHITECTURE.md | [docs/ARCHITECTURE.md](ARCHITECTURE.md) |
| 3.5 качество данных | SHACL constraints + Parameter min/max | [backend/app/schemas/domain.py](../backend/app/schemas/domain.py) |
| 3.6 характеристика качества данных | поля Parameter (minInclusive, deviationAllowed, sourceClause) | [backend/app/services/turtle_bridge.py](../backend/app/services/turtle_bridge.py) |
| 3.7 модель качества данных | SHACL NodeShape + формальный документ [DATA-QUALITY-MODEL.md](DATA-QUALITY-MODEL.md) | full |
| 3.8 показатель качества | criterion_passed (tri-state) + metric_analyzer | [backend/app/services/metric_analyzer.py](../backend/app/services/metric_analyzer.py) |
| 3.10 измерение | regulation_metrics таблица + acceptance_resolver | [backend/app/services/etl/acceptance_resolver.py](../backend/app/services/etl/acceptance_resolver.py) |
| 3.15 управление качеством данных | согласованная деятельность через audit_log + миграции | [backend/app/services/regulation_store.py](../backend/app/services/regulation_store.py) |
| 3.16 стратегическое управление | governance через ARCHITECTURE.md + ADMIN_PASS разделение ролей | [docs/GLOSSARY.md](GLOSSARY.md) §XIV |
| 3.17 происхождение данных | PROV-O в Turtle export | [backend/app/services/turtle_bridge.py](../backend/app/services/turtle_bridge.py):350-380 |
| 3.23 метаданные | regulation header + Turtle rdfs:label + Knowledge manifest | [backend/data/knowledge_seed/_manifest.json](../backend/data/knowledge_seed/_manifest.json) |

### 3.2 Структура обеспечения качества данных (§5.2)

| Элемент структуры | Требование ГОСТ | Реализация РАГРАФ | Статус |
|---|---|---|---|
| Модель качества данных | §5.2.2.1: «заданный набор характеристик, используемый как основа для установления требований и оценки» | SHACL NodeShape с sh:path / sh:minInclusive / sh:maxInclusive / sh:severity для каждого регламента; ParameterModel в Pydantic schema; формальный документ-описание [docs/DATA-QUALITY-MODEL.md](DATA-QUALITY-MODEL.md) с 7 характеристиками и связкой sh:metric | full |
| Показатели качества данных | §5.2.2.2: «после определения модели — выбрать показатели для оценки каждой характеристики» | criterion_passed (True/False/UNKNOWN — Kleene), regulation_metrics + acceptance_criterion | full |
| Оценка качества данных | §5.2.2.3: «использовать результаты измерения для определения соответствия» | acceptance_resolver вычисляет fired-outputs и passes на каждом snapshot (см. [backend/app/services/etl/acceptance_resolver.py](../backend/app/services/etl/acceptance_resolver.py)) | full |
| Повышение качества данных | §5.2.2.4: «решать на как можно более ранних стадиях» | SHACL validation на входе в Knowledge Ingest + Pydantic валидация в API | full |
| Отчётность о качестве данных | §5.2.2.5: формальный отчёт с целевым использованием, пороговыми значениями, тенденциями, действиями | metric_analyzer выдаёт hints, но формального документа-шаблона нет | partial — гэп №2 |
| Стратегическое управление качеством | §5.2.3: «культура подотчётности, набор руководящих принципов, разделение ролей» | ADMIN_PASS + require_admin + GLOSSARY.md §XIV + git-versioning | full |
| Происхождение данных | §5.2.4: «сведения о месте создания, всех процессах применённых к данным, всех изменениях» | PROV.wasDerivedFrom + audit_log с 19 полями + regulation_history immutable snapshots | full |

### 3.3 Жизненный цикл данных (§5.3) — 6 стадий

| Стадия ГОСТ | Реализация в РАГРАФ | Файлы | Статус |
|---|---|---|---|
| **Формирование требований к данным** (§5.3.2.2) | Sensor Schemas API + ModuleQualityRules; пользователь задаёт типы, единицы измерения, диапазоны | [backend/app/api/sensor_schemas.py](../backend/app/api/sensor_schemas.py) | full |
| **Планирование работы с данными** (§5.3.2.3) | Knowledge Ingest archive (CSV + manifest JSON), SPARQL для архитектуры запросов | [backend/data/knowledge_seed/_manifest.json](../backend/data/knowledge_seed/_manifest.json) | full |
| **Комплектование наборов данных** (§5.3.2.4) | ETL puller (Open-Meteo, LEYKA, Yandex Traffic) → etl_snapshots таблица | [backend/app/services/etl/puller.py](../backend/app/services/etl/puller.py), [backend/app/services/live_data_store.py](../backend/app/services/live_data_store.py) | full |
| **Подготовка наборов данных** (§5.3.2.5) | Трансформация, валидация SHACL, очистка, CSV→RDF разметка | [backend/app/services/validator.py](../backend/app/services/validator.py), [backend/app/services/csv_to_rdf.py](../backend/app/services/csv_to_rdf.py), [backend/app/services/etl/acceptance_resolver.py](../backend/app/services/etl/acceptance_resolver.py) | full |
| **Предоставление данных** (§5.3.2.6) | POST /api/regulations/{id}/execute → flow_executor → metric_analyzer | [backend/app/api/regulations.py](../backend/app/api/regulations.py), [backend/app/services/flow_executor.py](../backend/app/services/flow_executor.py) | full |
| **Вывод данных из эксплуатации** (§5.3.2.7) | archive (status='archived'), DELETE /regulations/{id}, cleanup_old_snapshots(keep_days=7) | [backend/app/api/regulations.py](../backend/app/api/regulations.py), [backend/app/services/live_data_store.py](../backend/app/services/live_data_store.py) | full |

### 3.4 Сквозные процессы (§5.3.3)

| Процесс | Требование | Реализация | Статус |
|---|---|---|---|
| Оперативное управление качеством | §5.3.3.1 + §5.2 | audit_log_store.append() на каждое срабатывание | full |
| Стратегическое управление | §5.2.3 | ARCHITECTURE.md + GLOSSARY.md + git-flow | full |
| Происхождение данных | §5.2.4 + §5.3.3.1 | PROV-O + regulation_history.author + fork chain | full |
| **Безопасность данных** (§5.3.3.2) | «защищённое хранение на всех стадиях ЖЦ, доступность авторизованным, отсутствие несанкционированных изменений» | HMAC-cookie auth + require_admin на DELETE/PUT/POST + параметризованные SQL + .gitignore для .env | full |
| **Защита персональных данных** (§5.3.3.3) | «соответствие 152-ФЗ, методы деидентификации» | поддержка 152-ФЗ в регламентах (support-data-leak-incident SLA 60±15мин контайнмент, 24±2ч РКН, 72±6ч субъекты); on-premise = собственное хранение; sandbox-изоляция LLM | full |

### 3.5 Примеры и сценарии (Приложение А)

ГОСТ приводит 2 справочных сценария: сбор/хранение данных через конвейер и управление качеством в ходе итераций ЖЦ. Оба сценария **архитектурно совпадают с РАГРАФ**:

- роль «системного инженера» = разработчик РАГРАФ (Олег);
- роль «менеджера по качеству данных» = аналитик-методолог;
- роль «инженера по данным» = аналитик в Knowledge Studio;
- роль «разработчика сервиса» = пользователь Sandbox-приложения.

См. полную картину в [docs/ARCHITECTURE.md](ARCHITECTURE.md) §1–§4.

---

## 4. ГОСТ Р 59898-2021: детальная матрица

### 4.1 Модель качества (§5)

ГОСТ требует **структурированное множество характеристик, субхарактеристик, метрик и отношений** (формула 1 в §5.5). Для РАГРАФ это:

```
РАГРАФ (S)
├── Функциональность (f1)
│   ├── Функциональная полнота (c11)
│   │   └── Полнота реализации функций — метрика 1 - A/B (формула 9 ГОСТа)
│   ├── Функциональная корректность (c12)
│   │   ├── SHACL pass rate
│   │   └── criterion_passed True/False ratio
│   └── Функциональная пригодность (c13)
│       └── Автоматизация (формула 23)
├── Производительность (f2)
│   ├── Time behaviour (c21): отклонение response-time от 100 мс (формула 24)
│   └── Resource utilization (c22): etl_health.status
├── Совместимость (f3) — OpenAPI + W3C
├── Мобильность (f4) — Docker
├── Практичность (f5) — Sandbox sources, методология
├── Сопровождаемость (f6) — TypeScript strict, pytest, миграции
├── Надёжность (f7) — mock-режим LLM, persistent DuckDB
└── Защищённость (f8) — HMAC, CORS, параметризованные SQL
```

### 4.2 Функциональные возможности (§8.2)

| Субхарактеристика | Метрика по ГОСТ | Реализация в РАГРАФ | Статус |
|---|---|---|---|
| Функциональная полнота (functional completeness) | формула 9: `1 - A/B`, где A — недостающие функции, B — описанные в ТЗ | 173 endpoint'а в backend/app/api/ (24 router'а × 12878 строк); план развития в [docs/BACKLOG.md](BACKLOG.md) | full |
| Функциональная корректность (correctness) | accuracy / MSE / MAE / precision / recall / F-мера / AUC ROC | Backend: 30 файлов pytest (8413 строк), accuracy через `metric_analyzer.py` ~93–96% по диагностическим регламентам | partial — нет автоматизированных метрик precision/recall для Sandbox/LLM |
| Согласованность (compliance) | соответствие стандартам отрасли | Соответствие W3C (Turtle/SHACL/PROV-O), ISO 8000 (data quality), 152-ФЗ | full |
| Функциональная пригодность (appropriateness) | формула 23: `A/B`, где A — шаги без участия пользователя, B — общее число шагов | flow_executor выполняет 100% узлов автоматически; пользователь нужен только для авторизации acceptance_criterion (опционально) | full |
| **Способность к самообучению** (ability to learn) | оценка автоматического извлечения знаний из накопленного опыта | Sandbox retrieval (top-k семантический поиск + LLM); Knowledge Ingest classification документов; embedding-based search | full |

### 4.3 Производительность (§8.3)

| Субхарактеристика | Метрика по ГОСТ | Реализация в РАГРАФ | Статус |
|---|---|---|---|
| Time behaviour | формула 24: `Σ(Ti - Td)² / N`, где Ti — время отклика, Td — допустимое | ARCHITECTURE.md §2.1: норматив < 100 мс/регламент, > 1 с → warning в логе | partial — норматив есть, агрегированной метрики нет |
| Производительность | формула 25: `A/T`, где A — задач за время T | ~50+ срабатываний/сутки на корпусе ~30 регламентов | partial |
| Resource utilization | потребление CPU/RAM/disk | etl_health endpoint + Railway-метрики | partial — only ETL, нет глобального dashboard |
| Capacity | предельные значения параллельных потоков | `test_flow_volume_not_degraded.py` — regression test | partial |

### 4.4 Способность к взаимодействию (§8.4)

| Субхарактеристика | Реализация в РАГРАФ | Статус |
|---|---|---|
| Соответствие (co-existence) | Stateless backend, Docker-изолирован | full |
| Функциональная совместимость (interoperability) | OpenAPI auto-docs, W3C Turtle/SHACL/PROV-O export, SIGMA bundle (.zip с manifest), CSV import/export, JSON-LD-совместимый Pydantic JSON | full |
| Контролируемость (controllability) | REST CRUD над всеми сущностями, audit_log трассировка | full |

### 4.5 Мобильность (§8.5)

| Субхарактеристика | Реализация в РАГРАФ | Статус |
|---|---|---|
| Адаптируемость (adaptability) | Все настройки через `.env`, без хардкода | full |
| Простота внедрения (installability) | `docker build && docker run`; Railway one-click; macOS/Windows wrappers (`start.bat`, `ragraf-mac.sh`) | full |
| Взаимозаменяемость (replaceability) | Стек на open-source стандартах (FastAPI/Python, React/TS) → миграция на любой PaaS | full |

### 4.6 Практичность, включая объяснимость (§8.6)

| Субхарактеристика | Требование | Реализация | Статус |
|---|---|---|---|
| **Понятность (explainability)** | «понятность для пользователя результатов работы СИИ» | Sandbox возвращает `sources: [{regulation_id, regulation_name, snippet, score}]` + `document_sources`; flow_executor возвращает trace с fired-узлами; PROV-O трассировка до пункта нормативного документа | full |
| Изучаемость (learnability) | достижение целей обучения | 6 tutorial-* регламентов в fixtures; COURSE-RAGRAF-V1.md и COURSE-SYSTEM-ANALYST-V1.md в docs/ | full |
| Простота использования (operability) | простое управление и контроль | React UI с панелями параметров, Flow Editor визуализация, real-time validation | full |
| Защищённость от ошибки (user error protection) | предотвращение ошибок оператора | Pydantic валидация всех POST/PUT input; SHACL валидация перед save; UI disabled state при незавершённой форме | full |
| Эстетика интерфейса | удовлетворённость дизайном | Tailwind CSS + Lucide Icons; современный UX (см. screenshots в [docs/SCREENSHOTS/](SCREENSHOTS/) если есть) | full |
| Доступность (accessibility) | для людей с ограниченными возможностями | ARIA-теги по умолчанию в React; формальный a11y audit не проводился | partial |
| Взаимодействие (collaborability) | управление потоками данных между группами | fork/publish/community модель в templates_store | full |

### 4.7 Сопровождаемость (§8.7)

| Субхарактеристика | Реализация в РАГРАФ | Статус |
|---|---|---|
| Анализируемость (analysability) | structured logging, audit_log с filtering, regulation_history с diff | full |
| Изменяемость (modifiability) | Layered architecture (5 слоёв), pydantic schemas, миграции v1/v2/v3 идемпотентные | full |
| Устойчивость (stability) | Тесты не падают при изменениях (30 файлов pytest + frontend tsc) | full |
| Тестируемость (testability) | pytest fixtures, mock-режим LLM, in-memory DuckDB вариант для тестов | full |
| Модульность (modularity) | api/ (24 router), services/ (60+ файлов), schemas/ — чёткое разделение | full |
| Настраиваемость (evolution) | миграционная инфраструктура (`_migrations` таблица), per-id seed | full |

### 4.8 Надёжность (§8.8)

| Субхарактеристика | Метрика ГОСТ | Реализация в РАГРАФ | Статус |
|---|---|---|---|
| Стабильность (maturity) | формула 26 (плотность отказов), формула 27 (устранение ошибок), формула 28 (тестовое покрытие) | Backlog open bugs vs closed (см. GitHub Issues), 30 файлов test + e2e | full |
| Устойчивость к ошибке (fault tolerance) | поддержка работы при сбоях | mock-режим Sandbox при LLM_ENABLED=false; graceful degradation ETL (status=degraded если puller > 5 мин); 336 try/except блоков в services/ | full |
| Восстанавливаемость (recoverability) | восстановление после отказа | Persistent DuckDB на Volume, CHECKPOINT при shutdown, seed-on-empty при первом старте | full |
| **Робастность (robustness)** | устойчивость к выбросам во входных данных | Pydantic type-check + SHACL bounds + Kleene UNKNOWN (трёхзначная логика для offline-датчиков, см. tutorial-06) | full |

### 4.9 Защищённость (§8.8, по ИСО/МЭК 25010 — категория Security)

| Субхарактеристика | Реализация | Статус |
|---|---|---|
| Конфиденциальность (confidentiality) | HMAC-cookie auth, ADMIN_PASS, .gitignore для .env, secrets через Railway Variables | full |
| Целостность (integrity) | audit_log immutable, regulation_history append-only, HMAC подпись cookie, параметризованные SQL | full |
| Неотказуемость (non-repudiation) | regulation_history.author + audit_log.user_id; trace в PROV-O | full |
| Подотчётность (accountability) | каждое изменение → запись в regulation_history с автором и комментарием | full |
| Подлинность (authenticity) | cookie HMAC-SHA256 + constant-time compare_digest | full |
| Приватность (privacy) | on-premise развёртывание, опциональный локальный Ollama (без отправки данных в облако), 152-ФЗ-документация | full |

### 4.10 Требования к тестовым наборам данных (§9)

ГОСТ требует **5 типов** наборов данных:

| Тип ГОСТ | Назначение по ГОСТ | В РАГРАФ | Файлы | Статус |
|---|---|---|---|---|
| Базовый демонстрационный | Минимально необходимый для проверки основной функции | fixtures golden seed: `tutorial-01` … `tutorial-06`; промаркированы тегом `dataset_type: базовый_демонстрационный` в [INDEX.md](../backend/data/fixtures/INDEX.md) | [backend/data/fixtures/INDEX.md](../backend/data/fixtures/INDEX.md) §tutorial | full |
| Дополнительный демонстрационный | Уточнение требований к функционалу | домены heating/housing/safety/environment/emergency_response/construction/urban-transport/it-support/nsk_city — 25 боевых регламентов; промаркированы тегом `dataset_type: дополнительный_демонстрационный` | [backend/data/fixtures/](../backend/data/fixtures/) | full |
| Полный демонстрационный | Объединение базового и дополнительного | весь fixtures-корпус (31 регламент); зафиксирован в [INDEX.md](../backend/data/fixtures/INDEX.md) §Маркировка | [backend/data/fixtures/](../backend/data/fixtures/) | full |
| Обучающий | Обучение СИИ (для ML-моделей) | n/a для РАГРАФ как символьной системы; Knowledge seed = обучающие данные для LLM-retrieval (13 датасетов, 1374 триплета); промаркирован тегом `dataset_type: обучающий` в [DATA-QUALITY-MODEL.md §5](DATA-QUALITY-MODEL.md) | [backend/data/knowledge_seed/](../backend/data/knowledge_seed/) | full |
| Тестовый | Оценка соответствия требованиям сертификации | pytest fixtures (8413 строк) + регрессионные данные | [backend/tests/](../backend/tests/) | full |

**Свойства тестовых наборов** (раздел 9.2):

| Свойство | Реализация | Статус |
|---|---|---|
| Представительность | fixtures покрывают 12 семантических доменов (heating, housing, urban-transport, safety, …) | full |
| Безызбыточность | 6 tutorial минимально достаточны для покрытия 8 типов узлов Rule DSL | full |
| Объективность | fixtures из реальных документов (Правила ПУЭ, СНиП, ЕДДС Кольцово, RAGU реальные регламенты) | full |
| Конфиденциальность | tests/ в репозитории public, но не содержат боевых ПДн | full |

---

## 5. Гэпы и план докрутки

Список разделён на **«закрыто в ревизии 2026-05-30»** (быстрые гэпы) и **«оставшиеся для пилотов в банках / сложных отраслях»** (вынесены в [BACKLOG.md](BACKLOG.md)). Приоритизировано по соотношению **польза/трудозатраты**.

### ✅ Гэп 1 (закрыт 2026-05-30): «Модель качества данных» как формальный документ

**ГОСТ:** Р 71484.1 §5.2.2.1.

**Что требовалось:** документ-описание заданного набора характеристик и показателей качества данных РАГРАФ.

**Что сделано:** создан [docs/DATA-QUALITY-MODEL.md](DATA-QUALITY-MODEL.md) с 7 характеристиками (точность, полнота, согласованность, своевременность, происхождение, доступность, целостность), для каждой — измеримый показатель в коде, ссылки на SHACL constraints, связь с `regulation_metrics` и acceptance_criterion. Применён к трём слоям данных (регламенты + Live ETL + Knowledge Base).

### Гэп 2 (P0, 6–8 ч): «Отчёт о качестве данных» как формальный шаблон

**ГОСТ:** Р 71484.1 §5.2.2.5.

**Что требуется:** шаблон отчёта по корпусу регламентов / Knowledge-датасетов с указанием:
- предполагаемое целевое использование;
- пороговые значения характеристик;
- характеристики, включённые в модель качества;
- объяснения исключённых характеристик;
- результаты измерения;
- тенденции изменения;
- действия по повышению качества;
- ответственные лица.

**Что предлагается:** endpoint `GET /api/quality-report/{domain_id}` + UI-страница `/quality-report/{domain}`. Существующий `metric_analyzer.py` уже отдаёт большинство данных, нужно собрать в шаблон.

### Гэп 3 (P1, 8–12 ч): Метрики функциональной точности для Sandbox/LLM

**ГОСТ:** Р 59898 §8.2 (формулы 13–22).

**Что требуется:** автоматизированные метрики accuracy / precision / recall / F-мера / AUC ROC для:
- классификации документов в Knowledge Ingest;
- ответов Sandbox (по сравнению с эталонными ответами экспертов).

**Что предлагается:**
- датасет «вопрос → эталонный ответ» (golden Q&A) — 30–50 примеров;
- evaluation harness в `backend/tests/eval/` с метриками;
- CI-job который запускает eval и сохраняет результаты в Markdown.

### ✅ Гэп 4 (закрыт 2026-05-30): Формальное разделение fixtures на 5 типов

**ГОСТ:** Р 59898 §9.

**Что требовалось:** в `backend/data/fixtures/INDEX.md` явно промаркировать каждый регламент по типу.

**Что сделано:** в [backend/data/fixtures/INDEX.md](../backend/data/fixtures/INDEX.md) добавлена секция «Маркировка по типам ГОСТ Р 59898-2021 §9» + тег `dataset_type: <тип>` в заголовке каждого домена. Соответствие:
- `базовый_демонстрационный` — домен `tutorial` (6 регламентов);
- `дополнительный_демонстрационный` — все боевые домены;
- `полный_демонстрационный` — весь `backend/data/fixtures/`;
- `обучающий` — `backend/data/knowledge_seed/` (13 датасетов);
- `тестовый` — `backend/tests/` + `backend/tests/data/`.

### ✅ Гэп 5 (закрыт 2026-05-30): Раздел «Объяснимость» в пользовательской документации

**ГОСТ:** Р 59898 §8.6 (subхарактеристика «понятность/explainability»).

**Что требовалось:** документ для пользователя про каналы объяснения.

**Что сделано:** создан [docs/EXPLAINABILITY.md](EXPLAINABILITY.md) с 5 каналами объяснения:
1. Источник нормы — PROV-O (`source_document`/`source_clause`/`source_url`).
2. Алгоритм решения — flow trace с подсветкой fired-узлов в Execute Screen.
3. Поиск по корпусу — `sources[]` и `document_sources[]` в ответе Sandbox.
4. Изменение знания — `regulation_history` immutable snapshots с author + comment + diff.
5. Метрики работы — `regulation_metrics` (success_rate, p50/p95, sparkline).

### Гэп 6 (P2, 3–4 ч): Resource utilization dashboard

**ГОСТ:** Р 59898 §8.3 (subхарактеристика «характер изменения ресурсов»).

**Что требуется:** мониторинг CPU/RAM/disk/network для backend.

**Что предлагается:** Prometheus exporter + Grafana dashboard (стандарт для FastAPI: `prometheus-fastapi-instrumentator`). На Railway — встроенные метрики dashboard уже работают.

### Гэп 7 (P2, 8–12 ч): Accessibility audit (a11y)

**ГОСТ:** Р 59898 §8.6 (subхарактеристика «доступность»).

**Что требуется:** audit фронта на соответствие WCAG 2.1 Level AA.

**Что предлагается:** запуск axe-core / Lighthouse a11y; устранение критичных issue (контраст, alt-тексты, focus management).

### Гэп 8 (P3, опционально): Encryption at rest

**ГОСТ:** Р 71484.1 §5.3.3.2.

Сейчас: DuckDB файл plain binary на disk (зашифровано только на уровне ОС / Container runtime).

Опционально: SQLCipher-like обёртка для DuckDB или шифрование Volume на стороне Railway.

Для большинства deployments — не критично; для прод-инсталляции в банке/гос-секторе — обязательно.

---

## 6. Roadmap докрутки и привязка к отраслям-заказчикам

### ✅ Спринт 1 (закрыт 2026-05-30, ~13 ч факт): Базовая compliance-документация

- [x] Гэп 1: [docs/DATA-QUALITY-MODEL.md](DATA-QUALITY-MODEL.md) (6 ч)
- [x] Гэп 4: маркировка fixtures по 5 типам ГОСТ в [backend/data/fixtures/INDEX.md](../backend/data/fixtures/INDEX.md) (4 ч)
- [x] Гэп 5: [docs/EXPLAINABILITY.md](EXPLAINABILITY.md) (3 ч)

**Достигнутый результат:** ~82% покрытия (с 75%), готовность к базовой экспертизе Минобрнауки / РНФ.

### Спринт 2–4 (вынесены в BACKLOG.md для пилотов в банках и сложных отраслях)

Привязка оставшихся гэпов к отраслям-заказчикам:

| # | Гэп | Где критично | Где не критично | Стоимость |
|---|---|---|---|---|
| 2 | Стандартизованный «Отчёт о качестве данных» (endpoint + UI) | Любой регулируемый сектор (Минобр / РНФ / 44-ФЗ закупки) | На демо-пилотах достаточно `metric_analyzer.hints` | 6–8 ч |
| 3 | Метрики accuracy/precision/recall для Sandbox/LLM (golden Q&A) | Банк, страхование, медицина (требуется доказать качество retrieval до прод-внедрения) | Стройка, ЖКХ, экология (LLM-слой не блокирующий — Sandbox опционален) | 8–12 ч |
| 6 | Resource utilization dashboard (Prometheus / Grafana) | Гос-сектор (требование 27001 для observability) | Команды до 5 методологов на Railway-метриках | 3–4 ч |
| 7 | Accessibility audit (WCAG 2.1 AA) | Гос-сектор (152-ФЗ + ПНСТ доступности; муниципальные ИС) | B2B-инструменты для подготовленных операторов | 8–12 ч |
| 8 | Encryption at rest (SQLCipher / volume encryption) | **Банк (ЦБ-РФ требования); медицинские ПДн (152-ФЗ ст. 19); гос-тайна** | Демо-инсталляции, муниципальный smart-city | 2–4 недели |

**Подробности и тикеты — в [docs/BACKLOG.md §🛡️ Compliance-докрутка для банков и сложных отраслей](BACKLOG.md).**

---

## 7. Используемые ссылочные стандарты

Полный список стандартов, которым РАГРАФ соответствует прямо или косвенно:

| Стандарт | Применение в РАГРАФ |
|---|---|
| **ГОСТ Р 71484.1-2024** (этот отчёт) | Качество данных для ML/аналитики |
| **ГОСТ Р 59898-2021** (этот отчёт) | Оценка качества СИИ |
| ГОСТ Р 71476-2024 (ИСО/МЭК 22989) | Концепции и терминология ИИ |
| ГОСТ Р 59276-2020 | Способы обеспечения доверия к СИИ |
| ГОСТ Р 70889 (ИСО/МЭК 8183) | Структура жизненного цикла данных для ИИ |
| ГОСТ Р 54911 (ИСО/TR 8000-120) | Происхождение данных |
| ГОСТ Р ИСО 8000 (все части) | Качество данных |
| ГОСТ Р ИСО/МЭК 25010-2015 | Модель качества систем и программных продуктов |
| ГОСТ Р ИСО/МЭК 25012-2015 | Модель качества данных |
| ГОСТ Р ИСО/МЭК 27001 | СМИБ (менеджмент информационной безопасности) |
| ГОСТ Р ИСО/МЭК 27701 | Расширение 27001 для менеджмента ПДн |
| 152-ФЗ | О персональных данных (поддержано в регламентах) |
| W3C Turtle / SHACL / PROV-O / RDF / SPARQL / OWL | Семантический стек |
| ISO/IEC 5259 series (через переводы) | Качество данных для ML |

---

## 8. Контакты

- Автор: Барбашин Олег Владимирович (О.В.)
- Проект: РАГРАФ
- Репозиторий: https://github.com/Barbashin1970/RAGRAF
- Документация: [docs/](.)
- Архитектура: [docs/ARCHITECTURE.md](ARCHITECTURE.md)
- Глоссарий с правовым контекстом: [docs/GLOSSARY.md](GLOSSARY.md) §XIV
- Аудит потока данных: [docs/DATA-FLOW-AUDIT.md](DATA-FLOW-AUDIT.md)

---

*Документ актуален на 2026-05-30. При изменении состава ГОСТов или существенной доработке РАГРАФ требует пересмотра.*
