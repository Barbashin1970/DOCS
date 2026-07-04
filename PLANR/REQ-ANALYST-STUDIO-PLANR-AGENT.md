# REQ: Диалоговый аналитик по проектам Plan-R в Студии аналитика

**Дата:** 2026-07-02. **Статус:** дизайн готов, P0 к реализации.
**Идея (Олег):** в Студии аналитика (`/sandbox`) спросить «какие проекты требуют
внимания по Айбим-ПрО?» → агент **уточняет вопросами**, что именно волнует, держит в
контексте **регламенты + данные проекта** и через LLM выдаёт обоснованный ответ — в
духе **ZET-инжиниринга** (задачный подход: уточнение → декомпозиция → детерминированная
проверка).
**Связано:** [REQ-ZET-INTEGRATION.md](../REQ-ZET-INTEGRATION.md),
[REQ-PLANR-CONFIG-BINDINGS.md](REQ-PLANR-CONFIG-BINDINGS.md),
[REQ-PLANR-INTEGRATION-EXPANSION.md](REQ-PLANR-INTEGRATION-EXPANSION.md).

> Чек-листы = источник правды по % реализации. `[x]` сделано, `[ ]` осталось.

---

## 1. Ценность

Управленец получает **проверяемый** ранжированный список рисков со ссылками на конкретные
проекты/этапы/регламенты — вместо ручного обхода дашбордов. Числа (`overdue_days`,
`critical_count`) **считает код, LLM их только формулирует** → нет галлюцинации цифр.

## 2. UX-сценарий (пример диалога)

```
Пользователь: Какие проекты требуют внимания по Айбим-ПрО?

Агент (уточнение, панель кнопок как pickDomain):
  1. Что волнует?  [Сроки/просрочка] [Бюджет] [Разрешения/ГГЭ] [Подрядчик] [Всё]
  2. Горизонт?     [Сейчас (факт)] [Прогноз риска]
  3. Порог?        [Только critical (≥14 дн)] [Все просрочки (>0)]

Пользователь: [Сроки] · [Сейчас] · [Только critical]

Агент (числа из кеша, обоснование от LLM):
  По «ООО Айбим-ПрО» (снапшот 02.07): критичных этапов 51, макс. просрочка 70 дн.
  🔴 1. {объект}, этап «…» — 70 дн, факт 42%, подрядчик … → [construction-overdue-stage] critical
        /planr?space=… · /regulations/construction-overdue-stage
  🟠 …
  Портфельный риск: [construction-portfolio-risk] critical.
  Разрешения/ГГЭ не оценивались — этих полей нет в кеше портфеля (нужен целевой pull).
```

## 3. Архитектура (переиспользуем чат Студии, без нового LLM-клиента)

- **Контекст (дёшево, из DuckDB-кеша, без сети):** новый `_build_planr_context(space_id,
  focus, threshold, horizon)` в `sandbox.py` (по образцу `_build_regulation_context` /
  `_live_verdict_block`). Источники: `live_data_store.latest_snapshot("pms-portfolio",
  "json", region=space_id)` (фолбэк `_global`) → summary+projects+top-8..15 overdue_stages;
  `planr_bindings_store.list_for_space` (enabled) → применимые регламенты; `regulation_store.get`
  → тела регламентов через существующий `_build_regulation_context`.
- **LLM:** та же `sandbox.chat()` → `client.chat.completions.create` (провайдер из
  `settings.llm_provider`, ретраи/timeout/mock-fallback как есть). Добавляем ветку retrieval
  (Plan-R-режим вместо корпусного поиска) + ветку правил цитирования.
- **Ссылки только из реестра:** `/planr?space={space_id}` и `/regulations/{regulation_id}`
  (id из bindings — валиден) → LLM не выдумывает ссылки. Заземление в `ChatResponse.sources`.
- **НЕ трогаем** тяжёлый `fetch_aggregates` (~3000 WBS) — только кеш.

## 4. ZET-стиль (проверяемость)

- **Уточнение = достройка компонент задачи** (Цель/focus, Ограничение/порог, Горизонт)
  детерминированным оркестратором (`planr-clarify`, аналог `route-domain`, трёхзначная
  развилка Клини), а не LLM в ходе генерации.
- **Числа считает Python** (`overdue_days≥14`=critical, сортировка, разметка 🔴/🟠,
  агрегаты по подрядчику); LLM запрещено пересчитывать/выдумывать.
- **Честная граница:** отсутствующие в кеше поля (permit/ГГЭ) → «нужен целевой pull», не выдумка.
  Прогноз — только под forecast-гейтом (≥10 прогонов).

## 5. Реализация фазами

### P0 — вопрос → контекст из кеша → ответ со ссылками  🟡 (бэкенд готов 2026-07-02)
Реализовано ИЗОЛИРОВАННЫМ путём (решение: не трогать fragile `chat()`, отдельный эндпоинт):
- [x] `sandbox.py`: `_build_planr_context(space_id)` (портфель из кеша `latest_snapshot` +
      применимые регламенты из bindings + их имена), `_planr_deterministic_answer` (числа
      считает код), `planr_ask(space_id, question)` — LLM только формулирует; без LLM →
      детерминированный список.
- [x] `api/sandbox.py`: `POST /api/sandbox/planr-ask {space_id, question}`.
- [x] Анти-галлюцинация: числа — Python, LLM не пересчитывает; ссылки только из реестра
      (`space_id`, `bindings.regulation_id`); нет поля (permit/ГГЭ) → честный отказ.
- [x] Mock-fallback (LLM off → тот же детерминированный список).
- [x] Тест `tests/test_sandbox_planr.py` (числа, ссылки, регламенты из bindings) — зелёный.
- [x] **Frontend (2026-07-03):** пресет «Проекты Plan-R» (`systemPromptPresets.ts`,
      effect `planr_mode`) → режим с выбором пространства (`/api/live/planr/spaces`) над
      инпутом; `lib/api.ts` `sandbox.planrAsk`; в `SandboxScreen.send()` при planrMode
      зовём `planr-ask`. **+14 типовых бизнес-вопросов** (Task 4) кликабельными чипами.
- [x] **Триплеты (Task 3):** `data/knowledge_seed/planr-construction-analytics.csv`
      (иерархия EPS + показатели + атрибуты) → домен `planr` в `domain_knowledge`;
      инжектятся в контекст `planr_ask` (терминология/связи для рассуждения LLM).
- [x] **Словарь (Task 2):** раздел «Часть XXI. Строительная аналитика Plan-R» в
      `docs/GLOSSARY.md` + категория «Строительство · Plan-R» в `/glossary`.

> **P0 = 100%.** Дальше — P1 (уточняющий диалог кнопками) и P2 (декомпозиция+сверка).

### P1 — уточняющий диалог  ⬜
- [ ] `api/sandbox.py`: `POST /api/sandbox/planr-clarify` (недостающие слоты + кандидаты).
- [ ] `sandbox.py`: `_planr_clarify_slots(query)` + поля `planr_focus/threshold/horizon` в chat().
- [ ] `SandboxScreen.tsx`: при Plan-R-интенте сперва `planr-clarify`; панель кнопок
      (переиспользовать `clarify`/`pickDomain`); индикатор пространства в `ContextPanel.tsx`.

### P2 — декомпозиция + детерминированная сверка  ⬜
- [ ] Декомпозиция «внимания» на детекторы (просрочка / бюджет-прокси / подрядчик-агрегат),
      каждый со своим уровнем → единый ранжированный список.
- [ ] Интеграция `regulation_analysis.py` (непротиворечивость применимых регламентов).
- [ ] `ChatResponse.sources` = структура заземления {project, stage, regulation_id, field_value};
      forecast-гейт для горизонта «прогноз».

## 6. Риски / ограничения

- **LLM на демо:** на Railway `LLM_ENABLED=false`→mock, `EMBEDDINGS_ENABLED=false`→keyword.
  Митигация: Plan-R-режим НЕ зависит от эмбеддингов (контекст детерминированный из кеша);
  в mock — тот же ранжированный список шаблоном. Живая формулировка — облачный провайдер.
- **Размер контекста:** только summary+projects+top-8..15 stages (бюджет по образцу
  `DOC_FULL_BUDGET_CHARS`); хвост — по доп-уточнению.
- **Тяжёлый pull запрещён** в hot-path — только `latest_snapshot`.
- **Галлюцинации:** метрики — Python; ссылки — из реестра; нет поля → честный отказ.
- **Свежесть:** помечать `pulled_at`; нет снапшота → «запустите ETL-tick», не пустой ответ.

## Файлы
Правки: `backend/app/services/sandbox.py`, `backend/app/api/sandbox.py`,
`frontend/src/components/sandbox/{SandboxScreen,ContextPanel}.tsx`,
`frontend/src/components/sandbox/systemPromptPresets.ts`, `frontend/src/lib/api.ts`.
Переиспуемые источники: `live_data_store.latest_snapshot`, `planr_bindings_store`,
`regulation_store`, `regulation_analysis.py`.
