# REQ: Настраиваемые контуры интеграции Plan-R (проект ↔ регламент ↔ канал)

**Дата:** 2026-07-02. **Статус:** P0 в работе.
**Цель:** убрать хардкод связей и дать админу РАГРАФ настраивать прямо в UI, какой
**проект Plan-R** каким **регламентом** обрабатывается и в какой **бот/группу**
уходят уведомления — без правок кода и деплоя.
**Связано:** [REQ-PLANR-INTEGRATION-EXPANSION.md](REQ-PLANR-INTEGRATION-EXPANSION.md),
[HOWTO-PLANR-API-TOKEN.md](HOWTO-PLANR-API-TOKEN.md) (§10 сидер, §11 мессенджеры).

> Чек-листы ниже — источник правды по проценту реализации. `[x]` = сделано,
> `[ ]` = осталось. Обновлять по мере готовности.

---

## 1. Проблема (где хардкод сейчас)

Логика регламентов УЖЕ не в коде — они в `regulation_store` (DuckDB), редактируются
и дублируются в РАГРАФ. Хардкод — в **связях**:

1. **Список регламентов на /planr** — массив `REGS` (2 шт) зашит во фронте, одинаков
   для всех пространств; ссылки «редактировать» для Меты ведут на те же регламенты,
   что и для Айбима.
2. **Датчики `pms-*`** пишутся `region="_global"` и только для primary-пространства →
   регламенты фаерят на данных одного проекта; остальные не оцениваются отдельно.
3. **Доставка** — единый recipient по умолчанию; нет связи «проект → своя группа/бот».

## 2. Модель: развязать 3 сущности через binding

- **Регламент** (логика) — есть, редактируемый (`regulation_store`).
- **Проект Plan-R** (источник) — пространство (Айбим/Мета) [+ опц. facility внутри].
- **Канал доставки** — Telegram-бот/группа, MAX, LEYKA.

**Binding** — строка, связывающая их (много-ко-многим):
```
{ id, space_id, project_id?, regulation_id, channel, recipient, enabled, note, created_at, updated_at }
```
- один регламент → на N проектов (переиспользование без копии);
- разные пороги на проект → дубль регламента в редакторе + binding на копию;
- разные получатели → в binding свой канал/группа.

---

## 3. P0 — снять хардкод списка и доставки  ✅ (сделано 2026-07-02)

**Бэкенд**
- [x] Таблица `planr_bindings` (DuckDB) + store-модуль `planr_bindings_store.py`
      (create/list/list_for_space/get/update/delete, RLock общей БД live_data_store).
- [x] Лениво-сид дефолтов `ensure_seeded(space_ids)`: для пространств без привязок —
      по 2 дефолтных construction-регламента (overdue-stage, portfolio-risk),
      channel=auto, recipient="" (= default). Сохраняет текущее поведение как данные.
- [x] CRUD API `app/api/planr_bindings.py`: `GET /api/planr-bindings[?space=]`,
      `POST`, `PUT /{id}`, `DELETE /{id}`, `POST /{id}/test` (тест-отправка по binding).
- [x] Регистрация роутера в `main.py`.
- [x] `notify` (live.py) шлёт дайджест пространства по **получателям из bindings**
      этого пространства (distinct channel+recipient), фолбэк — default recipient.

**Фронтенд**
- [x] `lib/planrBindings.ts` — типы + list/create/update/delete/test.
- [x] Экран **«Контуры интеграции Plan-R»** (`/planr-bindings`), engineer/executive-гейт:
      таблица bindings + строка добавления (дропдаун *Пространство* из `/spaces`,
      *Регламент* из `/api/datasets`, *Канал* + *Получатель*, toggle enabled,
      inline-правка канала/получателя, кнопки «Тест», «Удалить»).
- [x] Пункт меню в `navSections.ts` (roles: engineer/executive).
- [x] `/planr`: список регламентов и ссылки — **из bindings выбранного пространства**
      (замена хардкод-`REGS`); фолбэк на 2 дефолта, если привязок нет.

**Приёмка P0**
- [x] На /planr при переключении Айбим↔Мета список регламентов берётся из bindings.
- [x] В «Контурах интеграции» можно создать binding {Мета → регламент → группа-2} и
      «Тест» шлёт по каналу/получателю binding (проверено dry-run; live — с токеном).
- [x] notify для пространства уходит получателям его bindings (distinct, дедуп).
- [x] Тесты: store CRUD + seed + dedup получателей (33 зелёных). Фронт `tsc` чист.

> **Статус: P0 = 100%.** Дальше — P1 (регламенты фаерят на каждом проекте) и P2.

## 4. P1 — регламенты/уведомления по каждому проекту

### 4.A — авто-уведомление при ухудшении  ✅ (сделано 2026-07-02, Вариант A)
- [x] После pull по каждому пространству: если появились НОВЫЕ критические этапы
      (overdue≥14) vs прошлый pull — авто-дайджест по bindings пространства.
- [x] Дедуп состояния (`planr_notify_state`: множество ключей крит. этапов) — без спама;
      флаг `PLANR_AUTONOTIFY` (дефолт on). Частота = интервал pull.
- [x] Маршрут по каналам bindings (Telegram/MAX); фолбэк — default recipient.
- [x] Тесты: дедуп состояния + e2e dry-run (новые критические ловятся, повтор молчит).

### 4.B — «настоящий» flow-applier per-space  ⬜ (отложено)
- [ ] Датчики `pms-*` per-space (`region=space_id`), не только primary `_global`.
- [ ] `apply_one` с override региона + binding-driven прогон Flow DSL на данных space.
- [ ] Метрики/журнал прогонов с указанием space; независимые срабатывания Айбим/Мета.
> Открытие: сейчас construction-регламенты оцениваются через портфельный дайджест
> /planr, а не через flow-applier (region-mismatch: pms пишутся в `_global`, applier
> читает под город; доставки уведомления в applier нет). B = отдельная проводка —
> делать, когда понадобится реальный прогон Flow DSL по каждому проекту.

## 5. P2 — гранулярность и авто-маршрутизация  ⬜

- [ ] Binding до уровня **facility** (конкретный объект внутри пространства).
- [ ] Пороги-оверрайды прямо в binding (без дубля регламента).
- [ ] Получатель по **EPS-роли** (Куратор/ГИП) через `planr_contacts` — авто-выбор
      группы/лички по ответственному проекта (чтение EPS-ролей — из REQ-EXPANSION).
- [ ] Импорт/экспорт набора bindings (перенос конфигурации между стендами).

---

## 6. Данные / API / UI (P0, детально)

**Таблица `planr_bindings`:** `id TEXT PK, space_id TEXT, project_id TEXT NULL,
regulation_id TEXT, channel TEXT('auto'|'telegram'|'max'|'leyka'), recipient TEXT,
enabled BOOLEAN, note TEXT, created_at TIMESTAMP, updated_at TIMESTAMP`.

**API** (`/api/planr-bindings`): list[?space], create, update, delete, `POST /{id}/test`.

**UI** («Контуры интеграции Plan-R»): таблица + форма из 3 дропдаунов
(Пространство · Регламент · Канал/Получатель) + enabled + «Тест»/«Удалить». Паттерн —
как admin-матрица ролей (`useSyncExternalStore` + админ-гейт).

## 7. Ограничения / решения
- Дайджест /planr — портфельный (не по одному регламенту), поэтому в P0 `notify`
  рассылает по **distinct получателям** bindings пространства; пер-регламентная
  доставка при срабатывании — в P1 (когда регламент фаерит на своём space).
- Каналы: Telegram (основной), MAX (под юрлицо), LEYKA (задача) — из notifier'ов.
- Регламенты остаются в `regulation_store` (binding хранит только `regulation_id`).

## Файлы (P0)
- Новые: `backend/app/services/planr_bindings_store.py`, `backend/app/api/planr_bindings.py`,
  `frontend/src/lib/planrBindings.ts`, `frontend/src/components/planr/PlanrBindingsScreen.tsx`,
  `backend/tests/test_planr_bindings.py`.
- Правки: `backend/app/main.py` (роутер), `backend/app/api/live.py` (notify по bindings),
  `frontend/src/App.tsx` (роут), `frontend/src/lib/navSections.ts` (меню),
  `frontend/src/components/planr/PlanrPortfolioScreen.tsx` (регламенты из bindings).
