# HOWTO: подключение РАГРАФ к Plan-R по API-токену

**Что:** как подключить РАГРАФ к боевому Plan-R (строительный PMS) по API-токену.
**Зачем:** Plan-R — pull-источник: РАГРАФ тянет стройобъекты и просроченные работы
→ датчики `pms-contract-meta` / `pms-stage-overdue` → Flow DSL.
**Дата:** 2026-06-30. **Применимо к:** Plan-R релиз 604+.

> Внизу — раздел **«Обобщение на другие проекты»**: тот же приём для любого
> upstream с авторизацией по токену.

---

## 0. Карта решения (что вообще нужно)

Код клиента уже готов — [planr_client.py](../../backend/app/services/etl/planr_client.py).
Подключение к боевому инстансу = **только три переменные окружения**:

| Переменная | Что | Пример |
|---|---|---|
| `PLANR_BASE_URL` | адрес API, **без конечного слеша** | `https://planr-ngu.bim-info.ru` |
| `PLANR_AUTH_TOKEN` | **чистый** токен из админки | `8f1e99ba-c30a-...` |
| `PLANR_TENANT_ID` | ID пространства (шлётся как `x-tenant-id`) | `6ccba28e-ccf6-...` |
| `PLANR_AUTH_SCHEME` | схема заголовка: `raw`/`apikey`/`bearer` (**дефолт `raw`**) | `raw` |

Авторизация Plan-R = **два заголовка**:
```
Authorization: <ТОКЕН>      ⚠️ ГОЛЫЙ токен (дефолт PLANR_AUTH_SCHEME=raw)
x-tenant-id:   <ID пространства>
```
> ⚠️ **Внимание (проверено curl 2026-06-30):** руководство админа §4.2 декларирует
> `Authorization: apikey <ТОКЕН>`, но **боевой инстанс НГУ отвечает 401 на `apikey`
> и 200 на голый токен** (совпадает со swagger v602). Поэтому дефолт схемы — `raw`.
> Если попадётся инстанс строго по руководству — поставь `PLANR_AUTH_SCHEME=apikey`.
Логин/пароль в env класть **не нужно** — они только для входа в браузер, чтобы
один раз сгенерировать токен. РАГРАФ работает по токену.

---

## 1. Предусловие на стороне Plan-R (один раз)

В руководстве §4.2: на сервере Plan-R должна быть включена переменная
`API_KEY_AUTH_ENABLED=true`. Это env **их** инстанса (НГУ / bim-info), не наш.
Если токен валиден, но всё равно прилетает `401` — попроси их подтвердить, что
авторизация по API-ключам включена.

---

## 2. Создать токен (админ-консоль Plan-R, §4.2)

1. Зайти в админку: `https://planr-ngu-admin.bim-info.ru/` (свой админ-логин;
   у нового инстанса дефолт — `superadmin` / `superadmin`).
2. Раздел **«Пользователи»** → найти пользователя, **от имени которого** будет
   ходить РАГРАФ (например `superuser`). ⚠️ Токен наследует **права и доступ к
   пространствам именно этого пользователя** — РАГРАФ сможет ровно то, что он.
3. **«Детали»** на пользователе.
4. Блок **«Список IP токенов»** → **«Добавить»**.
5. В окне:
   - **Название** — напр. `RAGRAF integration`;
   - **«Сгенерировать»** → появится токен (скопировать сразу);
   - **Срок действия** — поставить подальше (напр. +1 год), иначе молча протухнет;
   - **«Проверка IP» / список IP** — ⚠️ **оставить ВЫКЛЮЧЕННОЙ** (или список пустым).
     Railway ходит с динамических egress-IP; включённый белый список = все запросы
     РАГРАФа отбиваются. Заодно проверить, что в карточке пользователя поле
     **«Белый лист IPv4»** не заполнено ограничительно.
6. **«Создать»** → скопировать токен.

## 3. Взять ID пространства (для `x-tenant-id`, §3.1)

Админка → **«Пространства»** → нужное пространство → **«Детали»** → вкладка
**«Общее»** → поле **«ID пространства»** (вид `46fd19d8-2043-4fed-...`).
Это `PLANR_TENANT_ID`.

---

## 4. Прописать переменные в Railway

Railway → проект РАГРАФ → **сервис бэкенда** → вкладка **Variables** → добавить:

```
PLANR_BASE_URL  = https://planr-ngu.bim-info.ru
PLANR_AUTH_TOKEN = <чистый токен из шага 2>
PLANR_TENANT_ID  = <ID пространства из шага 3>
# PLANR_AUTH_SCHEME=raw — дефолт, для НГУ задавать не нужно; поставь apikey
# только если конкретный инстанс строго по руководству релиза 604.
```

После сохранения Railway передеплоит — клиент переключится с встроенного мока на
боевой Plan-R (при пустом `PLANR_BASE_URL` он идёт на `127.0.0.1` → mock).

Локально — те же строки в `backend/.env` (он в `.gitignore`). **Токен не коммитить.**

---

## 5. Проверить, что взлетело

Открыть `GET /api/live/planr/health` (эндпоинт в [live.py](../../backend/app/api/live.py)).
Токен он не раскрывает, отдаёт диагноз:

```json
{"configured": true, "mode": "live", "base_url": "https://planr-ngu.bim-info.ru",
 "tenant_id": "...", "auth_scheme": "raw", "reachable": true, "http_status": 200,
 "spaces_sample": 1}
```

Как читать:

| Признак | Что значит | Что делать |
|---|---|---|
| `mode:"live"` + `reachable:true` + `http_status:200` | ✅ работает | — |
| `mode:"mock"` | `PLANR_BASE_URL` пуст → ходим в демо-мок | задать боевой URL |
| `reachable:false` + `"HTTP 401"` | токен/схема/`API_KEY_AUTH_ENABLED` | проверить токен и §1 |
| `reachable:false` + `"HTTP 403"` | неверный `x-tenant-id` | проверить ID пространства |
| `reachable:false` + `"HTTP 404"` | другой префикс пути API (не `/public-api`) | см. §6 |
| `configured:false` | `PLANR_AUTH_TOKEN` пуст | задать токен |

Проверить вручную из терминала (без РАГРАФа):
```bash
curl -s https://planr-ngu.bim-info.ru/public-api/spaces \
  -H "Authorization: apikey <ТОКЕН>" \
  -H "x-tenant-id: <ID пространства>" | head
```

---

## 6. Два подводных камня

1. **Схема авторизации (raw vs apikey) — главная ловушка.** Руководство релиза 604
   декларирует `Authorization: apikey <token>`, но **боевой инстанс НГУ на `apikey`
   отвечает 401, а на голый токен — 200** (проверено curl 2026-06-30; совпадает со
   swagger v602). Поэтому дефолт `PLANR_AUTH_SCHEME=raw` — шлём голый токен. Схема
   задаётся свойством `planr_auth_header` в [config.py](../../backend/app/config.py)
   (`raw`/`apikey`/`bearer`). Если токен уже содержит префикс схемы — он не
   дублируется. **Диагностика схемы — curl-ом напрямую в Plan-R** обоими вариантами:
   ```bash
   TOKEN='...'; TENANT='...'; BASE='https://planr-ngu.bim-info.ru/public-api/spaces'
   curl -s -o /dev/null -w "apikey: %{http_code}\n" "$BASE" -H "Authorization: apikey $TOKEN" -H "x-tenant-id: $TENANT"
   curl -s -o /dev/null -w "raw:    %{http_code}\n" "$BASE" -H "Authorization: $TOKEN"        -H "x-tenant-id: $TENANT"
   ```
   Какой даёт 200 — ту схему и ставь. ⚠️ Подставляй **реальный** токен, не плейсхолдер.
2. **Префикс пути.** Клиент зовёт `<base>/public-api/<path>`
   ([planr_client.py](../../backend/app/services/etl/planr_client.py)). На plan-r.tech и
   на инстансе bim-info путь обычно `/public-api`. Если health даёт `404` — у
   инстанса другой префикс (напр. `/api`); поправить в `PlanrClient.get()` одной
   строкой.

---

## 7. Как это работает под капотом (кратко)

```
scheduler (раз в 15 мин) / POST /api/live/pull
  → planr_client.fetch_aggregates()        # читает PLANR_* из settings
      → PlanrClient(GET /public-api/eps, /versions, /wbs)
      → facilities (стройобъекты) + works (просроченные работы)
  → live_data_store.append_snapshot()      # pms-contract-meta / pms-stage-overdue
  → applier → flow_executor                # регламенты срабатывают на просрочку
```
Пока боевой Plan-R не подключён, всё это крутится на встроенном моке
[_planr_mock.py](../../backend/app/api/_planr_mock.py) (синтетический НСК-стройпул) —
демо работает без внешних зависимостей.

---

## 8. Обобщение на другие проекты (рецепт «подключение по токену»)

Чтобы так же подключить РАГРАФ к любому upstream с авторизацией по токену
(другой PMS, таск-трекер, API города), повтори паттерн:

1. **Поля в [config.py](../../backend/app/config.py)** рядом с `LEYKA`/`PLANR`:
   ```python
   xxx_base_url: str = ""
   xxx_auth_token: str = Field(default="", repr=False)   # repr=False — секрет
   xxx_tenant_id: str = ""                               # если нужен tenant/space
   xxx_timeout: float = 15.0

   xxx_auth_scheme: str = "raw"                          # raw|apikey|bearer

   @property
   def xxx_auth_header(self) -> str:
       token = self.xxx_auth_token.strip()
       if not token:
           return ""
       low = token.lower()
       if low.startswith(("apikey ", "bearer ")):
           return token
       scheme = self.xxx_auth_scheme.strip().lower()
       if scheme == "apikey":
           return f"apikey {token}"
       if scheme == "bearer":
           return f"Bearer {token}"
       return token   # raw
   ```
   Имена ENV пишутся в UPPER_SNAKE: `XXX_BASE_URL`, `XXX_AUTH_TOKEN`, …
   ⚠️ **Урок Plan-R: не доверяй документации по схеме авторизации — проверь
   curl-ом** (`raw` vs `apikey` vs `bearer`) до интеграции. Руководство Plan-R
   говорило `apikey`, а боевой инстанс принимал только голый токен. Поэтому схема
   вынесена в env (`XXX_AUTH_SCHEME`), а не захардкожена.

2. **Клиент** по образцу [planr_client.py](../../backend/app/services/etl/planr_client.py):
   `PlanrConfig.from_env()` читает `settings` (не `os.getenv` напрямую — иначе
   синглтон `settings` не подхватит), `__aenter__` ставит заголовки, `get()`
   добавляет общий префикс пути. Возвращай `None` при пустом токене (интеграция
   выключена).

3. **Health-эндпоинт** по образцу `GET /api/live/planr/health` — лёгкий GET к
   «корню» upstream, токен не раскрывать, отдавать `configured/mode/reachable/
   http_status/error`. Это экономит часы на отладке боевого подключения.

4. **Мок** ([_planr_mock.py](../../backend/app/api/_planr_mock.py)) — чтобы демо и тесты
   работали без внешней зависимости. `_check_auth` должен принимать и голый токен,
   и `apikey/bearer <token>` (снимай схему перед сравнением).

5. **Тесты** ([tests/test_planr_mock.py](../../backend/tests/test_planr_mock.py)):
   - юнит на `xxx_auth_header` (голый → `apikey …`, префиксованный → без дубля);
   - юнит на детекцию live/mock;
   - мок принимает `apikey`-схему;
   - в puller-тесте патчить **атрибуты `settings`** (`monkeypatch.setattr(client.settings, …)`),
     а НЕ `monkeypatch.setenv` — `from_env` читает синглтон.

6. **ENV в Railway** — `XXX_BASE_URL` (без слеша), `XXX_AUTH_TOKEN` (чистый),
   `XXX_TENANT_ID` (если нужен). Логин/пароль upstream в env не класть.

**Чек-лист (TL;DR):** поля в config → клиент `from_env(settings)` → health-эндпоинт
→ мок с tolerant-auth → тесты на settings (не env) → 3 переменные в Railway →
открыть `/health` → читать диагноз по таблице §5.

---

## 9. Эксплуатация и мощность (боевой Plan-R)

Замер на боевом Айбим (2026-06-30): полный pull = **~3000 WBS** из РФ-сервера,
~70–80 HTTP-вызовов (eps + versions + атрибуты + 12 версий × WBS-страницы по 100),
несколько минут с учётом латентности РФ↔облако. Отсюда рекомендации:

1. **Интервал pull — главная ручка.** Источник `planr-pms` сид-дефолт
   `interval_seconds=300` (5 мин) — для ~3000 WBS это слишком часто. Поднять до
   **3600–14400** (1–4 ч): графики не меняются поминутно.
   - UI: Пульт ETL → `planr-pms` → интервал.
   - API: `POST /api/etl-control/sources/planr-pms` `{"interval_seconds": 14400}`
     (минимум 30с).
2. **Шедулер последовательный** (`tick()` обходит источники по очереди,
   `sequential by intent`). Значит долгий Plan-R-pull **задерживает** остальные
   источники (погода/LEYKA/часы/ANPR) в том же тике. Большой интервал planr
   делает эту блокировку редкой. (На будущее — вынести тяжёлый pull в отдельный
   конкурентный таск.)
3. **Таймаут/ретраи/конкурентность** (`config.py` / `planr_client.py`):
   `PLANR_TIMEOUT` (дефолт 30с) для медленного РФ-сервера можно 45–60с; ретраи
   3×; семафор 4 одновременных WBS-запроса — **не повышать** (не нагружать их
   сервер). Per-version таймаут → этап пропускается (graceful), pull не падает.
4. **Объём снапшотов и ретеншн (gap).** За pull пишется ≤ **207** строк: top-200
   просроченных этапов (`_STAGE_CAP` в puller) + 2 агрегата + 5 contract-meta.
   - При 4ч-интервале ≈ 1,2k строк/день; при 5-мин ≈ 60k/день.
   - ⚠️ `etl_snapshots` сейчас **не чистится** → том Railway растёт. Нужен purge
     (удалять `pms-*` старше 30–90 дней) — бэклог P1. Временная мера — редкий
     интервал из п.1.
5. **Кап top-200.** Если нужно больше критичных этапов — поднять `_STAGE_CAP`
   или писать только critical (≥14 дней). Сейчас лог сообщает, сколько усечено.
6. **Сеть/регион.** РФ↔Railway латентность умножается на ~75 вызовов/pull. Если
   Railway в US/EU — рассмотреть регион ближе к РФ; иначе принять медленный pull
   при редком интервале.
7. **Инкрементальность (будущее).** Тянуть WBS только активных/изменившихся
   версий (по `updatedAt`) вместо всех 3000 — снизит и время, и нагрузку на
   сервер заказчика.
8. **Мониторинг.** `GET /api/live/planr/health` (reachable/http_status);
   длительность и статус pull — в `etl_runs` и `GET /api/live/health`
   (last_status + freshness источника `planr-pms`). Алерт на повторные таймауты.

**Рекомендуемый старт для пилота:** `interval_seconds=14400` (4 ч),
`PLANR_TIMEOUT=45`, `_STAGE_CAP=200` (как есть) + завести задачу на purge снапшотов.

---

## 10. Сидирование демо-данных (ЗАПИСЬ в Plan-R)

Plan-R Public API умеет не только читать. Разведано OPTIONS/curl на боевом
инстансе (2026-06-30) — что можно записать:

| Операция | Эндпоинт | Примечание |
|---|---|---|
| Создать узел EPS | `POST /eps` | `{name,type,parentId,sortOrder}`; type = facility(группа)/project/schedule/**version** |
| Проставить атрибут | `PUT /eps-attribute-values` | `{epsId,attributeId,value}` |
| Создать этап WBS | `POST /wbs` | ⚠️ версия передаётся в **ТЕЛЕ** `{"EpsId":<version>, "parentId":<version>, …}` — НЕ query-param |
| Изменить/удалить узел | `PUT`/`DELETE /eps/{id}` | откат демо-данных |
| Создать пространство | — | нельзя (`/spaces` GET-only) → завести в админ-консоли |

**Что НЕ пишется через API:** даты этапа (старт/финиш/целевой) и `slip` —
их считает движок планирования Plan-R (импорт графика / UI), публичный CRUD их
игнорирует (PUT даёт 200, но GET возвращает `null`). На WBS «прилипают» только
`progress` и `cost_plan`.

**Следствие для демо.** Структуру (группа + объекты + договоры + суммы) сеем
живьём, а **просрочку выводим из статус-прокси**: сидер пишет реальный атрибут
«Статус проекта» (Критический/Под контролем/В графике), а puller для засеянных
данных переводит статус → дни просрочки (18/5/0). Применяется СТРОГО к версиям
со `status='version'` (наш сид) — боевые `actual`/`target` не трогаются
(`_status_to_overdue` в [planr_client.py](../../backend/app/services/etl/planr_client.py)).

**Свежее демо-пространство отдаёт пустой `/eps-attributes`** (нет label-
определений). Поэтому puller имеет фолбэк `KNOWN_ATTR_KEYS` (стабильные
attributeId → каноничный ключ) — иначе значения «теряют» ключ.

**Сидер** — [planr_seeder.py](../../backend/app/services/etl/planr_seeder.py)
(данные из `data/fixtures/demo-arhi-day.json`). Пишет в ОТДЕЛЬНОЕ демо-
пространство (`--tenant`), не в боевой портфель. Без `--yes` — сухой прогон:
```bash
cd backend
python -m app.services.etl.planr_seeder inspect --tenant <DEMO_ID>          # дерево (read-only)
python -m app.services.etl.planr_seeder spike   --tenant <DEMO_ID> --yes    # 1 объект + проверка цепочки
python -m app.services.etl.planr_seeder full    --tenant <DEMO_ID> --yes    # 4 объекта АРХИ → группа «Мета»
python -m app.services.etl.planr_seeder cleanup --tenant <DEMO_ID> --yes    # удалить группу «Мета»
```
Защита: запись в `tenant == PLANR_TENANT_ID` (боевой) требует `--force-prod`.

**Селектор пространств на /planr (PLANR_SPACES).** Чтобы на странице переключаться
между боевым Айбим и демо-«Метой» без правки env каждый раз:
- `PLANR_TENANT_ID` = **primary**-пространство (на нём срабатывают регламенты —
  пишутся датчики `pms-stage-overdue`/`pms-contract-meta`). Для демо это «Мета».
- `PLANR_SPACES` = CSV доп. пространств для селектора (кешируется только портфель,
  без датчиков). Пример (Мета primary + Айбим как вид):
  ```
  PLANR_TENANT_ID = 6a8cc014-1440-4f58-86be-caa86faf75da   # «Мета» (демо, регламенты)
  PLANR_SPACES    = 6ccba28e-ccf6-4b63-a69f-ffbbc89bf1e9   # «Айбим-ПрО» (боевой портфель)
  ```
puller тянет каждое пространство (изолированно try/except) и кеширует портфель
`pms-portfolio` с `region=space_id`. Эндпоинты: `GET /api/live/planr/spaces`
(список для дропдауна) и `GET /api/live/planr/portfolio?space=<id>` (по умолчанию
primary). На странице — дропдаун в шапке (★ = primary). ⚠️ Пул Айбим тяжёлый
(~3000 WBS, минуты) → держи интервал `planr-pms` 1–4 ч (см. §9).

---

## 11. Уведомления в мессенджер (Telegram / MAX)

РАГРАФ шлёт вердикты (просрочка/портфельный риск) ответственному в мессенджер — не
только задачей в LEYKA. Канал **авто**: Telegram если настроен, иначе MAX, иначе
«сухой» режим (лог). Получатель — из справочника контактов или из дефолта (Plan-R
контакты через API не отдаёт). Кнопка на `/planr` — «Уведомить в мессенджер».
Эндпоинт: `POST /api/live/planr/notify?space=<id>&channel=telegram|max|auto`.

### 11.1 Telegram (по умолчанию — проще, без юрлица)

1. Создать бота у `@BotFather` → токен.
2. Узнать `chat_id`: написать боту в личку ИЛИ добавить бота в группу и написать в
   неё → открыть `GET /api/live/telegram/updates` → взять `chat.id` (для группы
   отрицательный, напр. `-5212780865`).
3. Env (Railway + локальный `.env`):
   ```
   TELEGRAM_BOT_TOKEN         = <токен @BotFather>
   TELEGRAM_DEFAULT_RECIPIENT = -5212780865   # chat_id группы или личка
   ```
Эндпоинты: `GET /api/live/telegram/health`, `POST /api/live/telegram/test`,
`GET /api/live/telegram/updates`. Код: [telegram_notifier.py](../../backend/app/services/telegram_notifier.py).
Схема: `POST {base}/bot<token>/sendMessage` `{chat_id,text,parse_mode:"Markdown"}`;
при 400 на Markdown — авто-повтор без разметки.

### 11.2 MAX (альтернатива — требует верифицированное юрлицо РФ)

> ⚠️ С августа 2025 боты MAX доступны только **верифицированным юрлицам РФ** (ООО/АО);
> ИП, самозанятые, физлица — не допускаются. Поэтому дефолтный канал демо — Telegram.
> MAX подключать через юрлицо (напр. Айбим), когда понадобится именно он.

**Как включить:**
1. Создать бота у `@MasterBot` в MAX → получить токен (MAX → Чат-боты → Расширенные
   настройки → Настроить). В настройках бота **включить «добавление в группы»**.
2. **Добавить бота в группу** (⚠️ НЕ по ссылке-приглашении — она для людей; бота
   добавляет админ): открыть группу → Участники → «Добавить» → найти бота по имени →
   добавить → выдать право **«Отправка сообщений»** (участник/админ). Без членства и
   этого права бот в чат постить не сможет.
3. Узнать получателя: `chat_id` группы (из ссылки `web.max.ru/<id>` или через
   `GET /api/live/max/updates` после сообщения в группе) или свой `user_id` (написать
   боту в личку). Формат для env: `chat:<id>` (группа) или `user:<id>` (личка).
4. Прописать env (Railway + локальный `.env`):
   ```
   MAX_BOT_TOKEN        = <токен из @MasterBot>
   MAX_DEFAULT_RECIPIENT = user:123456   # или chat:789  (демо: личка/чат основателя)
   # MAX_API_BASE=https://platform-api.max.ru — дефолт; переопредели, если инстанс иной
   ```
Без `MAX_BOT_TOKEN` канал в **«сухом» режиме** — логирует, что отправил бы (демо/тесты
работают без внешней зависимости).

**Эндпоинты** ([live.py](../../backend/app/api/live.py)):
- `GET /api/live/max/health` — диагностика (токен не раскрывает; mode live/dry-run).
- `POST /api/live/max/test?recipient=&text=` — тест-отправка (проверить канал).
- `POST /api/live/planr/notify?space=<id>` — дайджест портфеля пространства в MAX
  (получатель: arg `recipient` → куратор из справочника → `MAX_DEFAULT_RECIPIENT`).
На странице `/planr` — кнопка **«Уведомить в Макс»** (шлёт `notify` по текущему пространству).

**Справочник контактов** — [planr-contacts.json](../../backend/data/fixtures/planr-contacts.json):
`{"by_name": {"ФИО": "user:.."}, "by_role": {"Куратор": "chat:.."}}`. Резолв: ФИО → роль
→ `MAX_DEFAULT_RECIPIENT`. Для демо можно оставить пустым — всё уйдёт на дефолт.
Код: [max_notifier.py](../../backend/app/services/max_notifier.py), [planr_contacts.py](../../backend/app/services/planr_contacts.py).
Схема Bot API: `POST {base}/messages?user_id=|chat_id=`, заголовок `Authorization: <token>`,
тело `{"text","format":"markdown","notify":true}` (≤4000 симв).

---

## Связанные документы

- [REQ-PLANR-INTEGRATION-EXPANSION.md](REQ-PLANR-INTEGRATION-EXPANSION.md) — расширение дата-контура (регламенты, EVM, маршрутизация, демо-сценарии).
- [REQ-SIGMA-INTEGRATION.md](../REQ-SIGMA-INTEGRATION.md) — стыковка с СИГМОЙ (другой upstream).
- [REQ-ZET-INTEGRATION.md](../REQ-ZET-INTEGRATION.md) — мост РАГРАФ ↔ ZET.
- Код: [config.py](../../backend/app/config.py), [planr_client.py](../../backend/app/services/etl/planr_client.py), [_planr_mock.py](../../backend/app/api/_planr_mock.py), [live.py](../../backend/app/api/live.py).
- Первоисточник: «Руководство администратора PLAN-R, релиз 604» (§3.1 ID пространства, §4.2 API-токены).
