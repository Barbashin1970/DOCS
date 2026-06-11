# Требование к интеграции RAGRAF ↔ СИГМА

**Адресат:** команда RAGRAF (для имплементации) + архитекторы СИГМЫ (для согласования).
**Автор:** RAGRAF.
**Статус:** v0.2 — переписано после анализа [external/ETL-pdf/](../external/ETL-pdf/) (reference-имплементация СИГМЫ от НГУ, 2025).
**Дата:** 2026-05-19.

> **Заменяет** ранний черновик `REQ-SIGMA-ETL.md`. Тот строился на гипотезе push-маршрутизации
> «событие → routing.json → POST /execute». Анализ исходников ЦД Теплосетей показал, что
> архитектура СИГМЫ **pull-based**, а регламенты исполняет сам ЦД через SPARQL-запросы к FUSEKI.

> **Важно про статус СИГМЫ.** Reference-имплементация — это **версия 2.0, всё ещё в развитии**.
> Команда СИГМЫ продолжает дорабатывать архитектуру (например, цепочки регламентов / wiring
> были учтены ими **после** того как мы их реализовали в РАГРАФе — сейчас взято в работу
> на их стороне). Этот документ:
>   - Описывает **текущее состояние СИГМЫ** как факт.
>   - Где у СИГМЫ не хватает функциональности (cross-domain wiring, runtime-добавление адаптеров,
>     versioning, unified Process entity) — формулирует это как **feature requests команде СИГМЫ**,
>     а не как ограничения, под которые РАГРАФ должен подстраиваться.
>   - Архитектурные решения РАГРАФа (Twin с wiring, multi-domain Process, UI-добавление sensor'ов)
>     **легитимны и сохраняются**. Интеграция строится так, чтобы при дальнейшей зрелости СИГМЫ
>     мы не теряли свои фичи, а получали их полноценную поддержку.

---

## 0. TL;DR

| Что | Куда | Канал |
|---|---|---|
| **1. Регламенты** | RAGRAF → ЦД-домен → FUSEKI | `POST /api/v1/regulations/data` (text/plain Turtle) |
| **2. SHACL-shapes** | RAGRAF → ЦД-домен → FUSEKI | `POST /api/v1/regulations/shapes` (text/plain Turtle) |
| **3. Каталог датчиков** | ETL → RAGRAF (read-only sync) | `GET /api/v1/adapters/`, `/sensors/` |
| **4. Process (двойник)** | RAGRAF → N ЦД-инстансов | Те же endpoint'ы что и (1)-(2), раскладка по доменам |
| **5. Bundle офлайн** | RAGRAF ZIP → ручной import в ЦД | data.ttl + shapes.ttl + sensors.json + manifest.json |
| **Auth** | RAGRAF (как Application) → AUTH сервис | `POST /api/v1/apps/token` (`app_id + secret_key`) |

**Самое важное изменение в Turtle-формате:** регламенты в реальной СИГМЕ — это **плоские scalar-свойства**
(`:diameter 150 ; :pressureDeviation 0.15`), а не глубокая OWL-структура с arrays.
Подробнее в §7.

---

## 1. Реальная архитектура СИГМЫ — что мы нашли

Источник: [external/ETL-pdf/Приложение 2. Описание сервисов.docx.pdf](../external/ETL-pdf/Приложение%202.%20Описание%20сервисов%20.docx.pdf) +
листинги в Приложениях 3.

### 1.1. Четыре сервиса + FUSEKI

```
                ┌─────────────┐
External  ────▶ │  Gateway    │ ────▶ ┌─────────┐
clients         │   API       │       │  ETL    │ ◀── adapters (Python plugins)
                │ Bearer-auth │       └─────────┘    └── каждый адаптер регает
                └──────┬──────┘             │         сенсоры на старте
                       │                    │
                       │                    │ pull
                       ▼                    ▼
                 ┌─────────┐         ┌─────────┐
                 │  AUTH   │         │ EVENTS  │ ◀── POST /sources/{}/events
                 │ JWT     │         │         │    (IP-whitelist auth)
                 │ app_id  │         └─────────┘
                 │secret_kp│
                 └─────────┘

      ╔═══════════════════════════════════════════════╗
      ║          ЦД (Цифровой Двойник) — отдельный    ║
      ║          FastAPI сервис на КАЖДЫЙ ДОМЕН        ║
      ║                                               ║
      ║   ┌──────────────┐  pulls   ┌──────────────┐  ║
      ║   │ /networks    │ ───────▶ │  ETL         │  ║
      ║   │ /topology    │          │ /sensors/data│  ║
      ║   │ /logs        │          └──────────────┘  ║
      ║   │ /deviations  │  SPARQL                    ║
      ║   │ /regulations │ ───────▶ ┌──────────────┐  ║
      ║   │   /data      │ ◀──────  │ FUSEKI       │  ║
      ║   │   /shapes    │          │ (Apache Jena)│  ║
      ║   │ /tg_bot      │          └──────────────┘  ║
      ║   └──────────────┘                            ║
      ╚═══════════════════════════════════════════════╝
```

### 1.2. ETL — read-only pull API

Источник: [external/ETL-pdf/Приложение 1.pdf](../external/ETL-pdf/Приложение%201.%20Перечень%20методов%20API%20и%20json-схемы%20сервисов%20.docx.pdf) (стр. 4-11) + [Приложение 3. Листинг. Сервис ETL.pdf](../external/ETL-pdf/Приложение%203.%20Листинг.%20Сервис%20ETL.docx.pdf) (стр. 2-5).

```
GET  /api/v1/adapters/                                   → [AdapterShortInfoScheme]
GET  /api/v1/adapters/{adapter_id}                       → AdapterScheme
GET  /api/v1/adapters/{adapter_id}/sensors/              → [SensorShortInfoScheme]
GET  /api/v1/adapters/{adapter_id}/sensors/{sensor_id}   → SensorScheme
GET  /api/v1/adapters/{adapter_id}/sensors/{sensor_id}/data  → DataScheme
```

**Схемы:**
- `AdapterScheme`: `{adapter_id: "heating_network", name: "Теплосети", description: ...}`
- `SensorScheme`: `{sensor_id: "topology", name: "Топология", type: "...", description: ...}`
- `DataScheme`: `{timestamp: "...", value: <any>}` — payload произвольный, СИГМА не диктует структуру.

Адаптеры — это **код-плагины**: в `dependencies/providers.py` ЦД делает
`adapter_manager.register_adapter(heating_network)`. Каждый адаптер регистрирует
свои сенсоры в `__init__`. Список доступных модулей определяется тем, что физически
импортировано в ETL-инстанс на старте.

Кэширование — через Redis, конфигурируется per-sensor.

### 1.3. EVENTS — async push для внешних источников

`/api/v1/sources/*` и `/api/v1/sources/{id}/events/`. См. [external/ETL-pdf/Приложение 1.pdf](../external/ETL-pdf/Приложение%201.%20Перечень%20методов%20API%20и%20json-схемы%20сервисов%20.docx.pdf) (стр. 32-43).

```
POST   /api/v1/sources/                              SourceCreateRequest  → source_id
GET    /api/v1/sources/                              → [SourceShortResponse]
PUT    /api/v1/sources/{source_id}                   SourceUpdateRequest
DELETE /api/v1/sources/{source_id}
POST   /api/v1/sources/{source_id}/events/           EventCreateRequest {msg: any}
GET    /api/v1/sources/{source_id}/events/?date_from&date_to&limit&skip  → [EventResponse]
DELETE /api/v1/sources/{source_id}/events/{event_id}
POST   /api/v1/schema/                               (multipart file)  → generated schema
```

**Source**: `{name, description, ip: "0.0.0.0/0", active, data_schema: object}`. Auth по IP-whitelist
(или маске подсети). `data_schema` — произвольный JSON-Schema для валидации входящих событий.

Это **парная функциональность к ETL adapters**: ETL это для случаев когда нужен опрос внешнего HTTP API,
EVENTS — когда внешний источник сам пушит свои события.

### 1.4. AUTH — JWT для пользователей и приложений

```
POST /api/v1/users/token              → user JWT
POST /api/v1/users/token/refresh
POST /api/v1/apps/token               → application JWT (app_id + secret_key)
POST /api/v1/apps/token/refresh
POST /api/v1/users/{user_id}/apps/    RegisterApplicationRequest → {app_id, secret_key}
GET  /api/v1/users/{user_id}/apps/
DELETE /api/v1/users/{user_id}/apps/{app_id}
```

**Для РАГРАФа подходит app-auth**: админ СИГМЫ выдаёт нам `app_id + secret_key`,
мы храним их в SIGMA_APP_ID / SIGMA_SECRET_KEY env-переменных, получаем JWT
и обновляем по refresh-токену.

### 1.5. Gateway — единая точка входа

```
ANY /api-gateway/{service_name}/{path}    (HTTPBearer security)
```

Маршрутизирует на `etl`, `auth`, `events` и т.д. РАГРАФ ходит **только** через
Gateway, никогда напрямую на внутренние сервисы.

### 1.6. ЦД — отдельный сервис на каждый домен

Источник: [external/ETL-pdf/Приложение 3. Листинг. ЦД Теплосетей.docx.pdf](../external/ETL-pdf/Приложение%203.%20Листинг.%20ЦД%20Теплосетей.docx.pdf).

Listing 7 (`backend/app/routers/api/v1/regulations.py`):

```python
@router.get("/data",  response_class=PlainTextResponse)  # Turtle
@router.post("/data", status_code=201)  # body: text/plain Turtle
@router.put("/data",  status_code=204)
@router.delete("/data")

@router.get("/shapes",  response_class=PlainTextResponse)
@router.post("/shapes", status_code=201)
@router.put("/shapes",  status_code=204)
@router.delete("/shapes")
```

Listing 35 (`backend/regulations/services.py`): `RegulationService.get_deviation_level()`
делает SPARQL-запрос против `self.graph` (FUSEKI):

```python
self.graph.first(
    ['name', 'pressureDeviation', 'recommendation'],
    """
        ?regulation a :Regulation ;
            :name ?name ;
            :diameter ?diameter ;
            :pressureDeviation ?pressureDeviation .
        OPTIONAL { ?regulation :recommendation ?recommendation . }
        FILTER (?diameter < {edge_topology.diameter})
    """,
    "DESC(?diameter)",
)
```

Главный runtime-cycle — Listing 5 (`/admin/deviations/update`):

```python
for network in networks:
    log, deviations = await d_use_cases.update_deviations(
        network, topology_service, deviation_service, regulation_service
    )
    if log:
        background_tasks.add_task(
            s_use_cases.send_deviation_messages,
            bot, network, deviations, log.timestamp,
            email_service, email_subscription_service, tg_subscription_service,
        )
```

То есть ЦД — это **планировщик-аналитик**, который периодически:
1. PULL'ит данные из ETL (`deviation_service.get_deviations_from_etl(workspace_id)`)
2. Для каждого отклонения SPARQL'ит FUSEKI на «какой регламент применим к этому edge?»
3. Сравнивает `deviation_value > regulation.pressureDeviation` → CRIT/WARN/INFO
4. Пишет лог в свою БД, шлёт telegram + email подписчикам

---

## 2. Где в этой архитектуре место РАГРАФа

### 2.1. Что есть у РАГРАФа

- **Редактор регламентов** с SHACL-валидацией, версионированием, audit-trail.
- **Каталог датчиков** с UI-добавлением (`SensorSubtype` + `SensorField`).
- **Редактор двойников (Process)** — группа регламентов **с wiring между ними** (выход регламента A → вход регламента B), может быть **мульти-доменной**.
- **Поиск + RAG** над регламентами для аналитиков.

### 2.2. Текущая раскладка ответственности (СИГМА v2.0)

В текущей версии СИГМЫ:
- ЦД-сервисы организованы по доменам (один FastAPI-app на «теплосети», другой на «воздух» и т.д.).
- ЦД сам тянет данные из ETL и SPARQL'ит регламенты из своего FUSEKI.
- Cross-CD коммуникации между регламентами **нет**.

Поэтому в v1 нашей интеграции РАГРАФ выступает как:
- **Писатель регламентов в FUSEKI** (через `POST /regulations/data` на нужном ЦД).
- **Читатель каталога адаптеров** из ETL.
- **Координатор multi-domain export'а** Process'а — раскладывает регламенты по соответствующим ЦД.

### 2.3. Gap'ы СИГМЫ как feature requests

То, что наш Process в текущей СИГМЕ приходится **раскладывать по N ЦД** — это
**workaround под текущую сегментацию СИГМЫ по доменам**, а не правильное архитектурное
решение. По мере зрелости СИГМЫ ожидаем:

| Наша фича | Текущее ограничение СИГМЫ | Что нужно от их команды |
|---|---|---|
| **Wiring между регламентами** | В v2.0 учтено (взято в работу после того как реализовали мы) | Подтвердить срок появления + контракт |
| **Cross-domain wiring** (выход reg в heating → вход reg в environment) | ЦД изолированы по доменам, общей шины regulations нет | Либо event-routing через EVENTS, либо unified ЦД-процессор |
| **Unified Process entity в СИГМЕ** | Process существует только в РАГРАФе; в СИГМЕ — отдельные графы регламентов | Завести в СИГМЕ first-class сущность «процесс/двойник» с членами + wiring |
| **Runtime-добавление адаптеров через API** | Только Python-плагины при старте ETL | Endpoint для регистрации новых sensor-типов без редеплоя |
| **Версии регламентов в FUSEKI** | Только последняя | Хранение версий + endpoint для отката |

Эти gap'ы формализованы как вопросы команде СИГМЫ — см. §9 и [external/ETL-draft-code/open-questions.md](../external/ETL-draft-code/open-questions.md).

---

## 3. Вектор 1 — запись регламентов в FUSEKI через ЦД

### 3.1. Контракт ЦД

```
POST /api/v1/regulations/data
Authorization: Bearer <app-jwt>
Content-Type: text/plain
Body: <Turtle-текст всего графа регламентов>

Response: 201 Created
```

`POST /shapes` аналогично для SHACL. **Внимание:** это replace-семантика — POST'нутые регламенты создают граф **с нуля** (если граф не существует) или **ОШИБКА** если граф уже есть. Для замены — PUT.

```
PUT /api/v1/regulations/data       (для апдейта существующего графа)
PUT /api/v1/regulations/shapes
```

### 3.2. Что РАГРАФ должен сделать

| Файл | Что |
|---|---|
| `backend/app/services/sigma_client.py` | Новый. HTTP-клиент на httpx с auth-flow + retry. |
| `backend/app/services/sigma_publish.py` | Новый. `async def publish_regulations(regulations: list[Regulation], target_cd_url: str)` |
| `backend/app/api/processes.py` | Добавить `POST /processes/{id}/publish-to-sigma` — UI-кнопка в TwinDesignerScreen |
| `backend/app/config.py` | Новые env: `SIGMA_GATEWAY_URL`, `SIGMA_APP_ID`, `SIGMA_SECRET_KEY` |

### 3.3. Алгоритм publish

```python
async def publish_to_sigma(process_id: str):
    # 1. Получить app-JWT (см. §3.4)
    token = await sigma_client.get_app_token()

    # 2. Собрать регламенты Process'а по доменам
    proc = process_store.get(process_id)
    by_domain = group_by(proc.regulations, lambda r: r.domain)

    # 3. Для каждого домена — найти URL ЦД, сгенерить flat Turtle, POST или PUT
    for domain, regs in by_domain.items():
        cd_url = SIGMA_CD_URLS[domain]  # из конфига маппинга
        data_ttl = turtle_for_sigma(regs)
        shapes_ttl = shacl_for_sigma(regs)

        await sigma_client.put_or_post(
            f"{cd_url}/api/v1/regulations/data",
            data=data_ttl,
            token=token,
        )
        await sigma_client.put_or_post(
            f"{cd_url}/api/v1/regulations/shapes",
            data=shapes_ttl,
            token=token,
        )

    # 4. Записать в audit
    audit.record("sigma_publish", process_id=process_id, ...)
```

### 3.4. Auth flow

```python
async def get_app_token() -> str:
    # Кэшируем JWT в Redis/memory с TTL=expires_in
    cached = cache.get("sigma_app_token")
    if cached: return cached

    response = await httpx.post(
        f"{SIGMA_GATEWAY_URL}/api-gateway/auth/api/v1/apps/token",
        json={"app_id": SIGMA_APP_ID, "secret_key": SIGMA_SECRET_KEY},
    )
    data = response.json()
    cache.set("sigma_app_token", data["access_token"], ttl=data["expires_in"] - 60)
    return data["access_token"]
```

---

## 4. Вектор 2 — чтение каталога адаптеров/датчиков из ETL

### 4.1. Зачем

Сейчас в РАГРАФе sensor library — наша внутренняя (см. [ref_real_data](../../.claude/projects/-Users-olegbarbashin-RAGRAF/memory/ref_real_data.md):
`SensorSubtype` + `SensorField`). Аналитик добавляет сенсоры через UI.

В СИГМЕ каталог адаптеров/сенсоров **диктуется кодом ETL** (Python-плагины).
Когда РАГРАФ пишет регламент с триггером `sensor_subtype="industrial-pressure"`,
а в ETL такого `adapter_id` нет — ЦД не сможет привязать данные к параметру.

**Решение:** при создании sensor-нод в UI РАГРАФа предлагать выбор из реального
каталога ETL'а той СИГМЫ, к которой подключён РАГРАФ.

### 4.2. Что РАГРАФ должен сделать

| Файл | Что |
|---|---|
| `backend/app/services/sigma_catalog.py` | Новый. `async def fetch_adapters() -> list[Adapter]`, `async def fetch_sensors(adapter_id)` |
| `backend/app/api/sensors.py` | Расширить: `GET /api/sensors/sigma-catalog` — отдаёт список адаптеров+сенсоров из СИГМЫ |
| Frontend `SensorLibraryScreen.tsx` | Кнопка «Синхронизировать с СИГМОЙ» + таблица каталога |
| `backend/app/services/sensor_store.py` | `mapping[our_subtype_id → (adapter_id, sensor_id)]` — позволить аналитику связать наш ID с ETL'овским |

### 4.3. Стратегия sync

Не «зеркало». РАГРАФ хранит **свой каталог** (для офлайн-работы + для случаев когда
СИГМА недоступна), но **может маркировать** какие наши `subtype_id` мапятся на
реальные `(adapter_id, sensor_id)` СИГМЫ.

При публикации регламента (§3) если триггер ссылается на `sensor_subtype` без
маппинга — выдать предупреждение «датчик не привязан к реальному адаптеру СИГМЫ,
ЦД не сможет автоматически подтянуть данные».

---

## 5. Вектор 3 — экспорт Process в N ЦД одновременно

### 5.1. Сценарий

Пользователь собрал в РАГРАФе Process «Контроль воздуха + теплосети + парковка»:
4 регламента, 2 wiring-связи, разные домены (`environment`, `heating`, `safety`).

В **текущей** СИГМЕ это значит **3 разных ЦД-инстанса** (по одному на домен), каждый —
отдельное FastAPI-приложение со своим `/api/v1/regulations/data` endpoint. Multi-CD
раскладка — это **workaround** под per-domain сегментацию СИГМЫ.

В **зрелой** СИГМЕ ожидаем появления **first-class Process entity** (см. §2.3, gap-table):
один endpoint типа `POST /api/v1/processes/{id}` принимает наш Twin целиком — с регламентами,
wiring, sensor-attachments — и СИГМА сама раскладывает его по нужным ЦД с поддержкой
cross-CD eventing. Тогда мы заменим раскладку по доменам на один publish.

### 5.2. Что РАГРАФ должен сделать

**Конфигурация** `SIGMA_CD_URLS`:
```yaml
heating: "https://cd-heating.sigma.local"
environment: "https://cd-air.sigma.local"
safety: "https://cd-anpr.sigma.local"
```

**При publish**:
1. Группируем регламенты по `regulation.domain`.
2. Для каждой группы — POST/PUT на соответствующий ЦД.
3. Если какой-то ЦД недоступен — продолжаем остальные, в финальный отчёт пишем partial-success.
4. **Не делаем rollback** (это требует distributed transaction которого у СИГМЫ нет).
   Вместо этого — UI показывает «опубликовано 2/3, ЦД-environment не отвечает,
   повторите publish позже».

### 5.3. Wiring между регламентами (включая cross-domain)

`Process.wiring` в РАГРАФе — это **легитимная функциональность** двойника: связь
«вердикт регламента A → вход регламента B». Внутри одного домена это уже работает в
текущей СИГМЕ (после того как команда взяла композицию в работу). **Cross-domain wiring**
(producer в heating, consumer в environment) — функция, которой СИГМА пока не предоставляет,
но которая является **естественным расширением** композиции регламентов.

**План реализации:**

**v1 — текущее состояние СИГМЫ.**
- Внутри одного домена: wiring передаётся в ЦД как metadata регламента, ЦД использует
  её для построения SPARQL-запросов «дай мне регламент-потребителя X»  (детали — после
  ответа команды СИГМЫ, см. §9 q.6).
- Между доменами: при publish РАГРАФ **сохраняет cross-domain wiring как unresolved
  metadata** + явное предупреждение в UI «эта связь не выполнима в текущей СИГМЕ,
  ожидаем поддержку cross-CD events». Не теряем данные, не делаем вид что фичи нет.

**v2 — feature request к команде СИГМЫ.**
- Добавить cross-CD event routing через EVENTS-сервис: ЦД-A POST'ит свой вердикт
  как event, ЦД-B подписывается. РАГРАФ при publish автоматически создаёт нужные
  source-регистрации в EVENTS.
- Или: ввести в СИГМУ first-class сущность «Process / Twin» (см. таблицу gap'ов в §2.3),
  которая исполняется единым процессором с доступом ко всем ЦД-FUSEKI. Это более
  фундаментальная переработка с их стороны.

**Что НЕ делаем:**
- Не подменяем функционал СИГМЫ middleware'ом в РАГРАФе («wiring-relay сервис»).
  Это бы означало, что мы постоянно крутим прод-сервис под нагрузкой и отвечаем за
  uptime data-pipeline'а — не наш скоуп.
- Не выпиливаем cross-domain wiring из РАГРАФа как «несовместимое». Сохраняем как
  фичу с маркировкой «требует поддержки СИГМЫ для исполнения».

---

## 6. Вектор 4 — офлайн-bundle ZIP

Текущий `/api/regulations/{id}/export-bundle` и `/api/processes/{id}/bundle.zip`
**остаются как есть** — это бэкап-канал для случаев когда нет прямого коннекта
к СИГМЕ. ЦД-команда импортирует ZIP'ы вручную через `POST /api/v1/regulations/data`.

**Добавить в ZIP:**
- `sensors.json` — embedded payload-схемы (см. предыдущий черновик)
- `etl_catalog_mapping.json` (опционально) — наша таблица `(our_subtype_id → adapter_id, sensor_id)` если она есть

Bump `format_version: "1.0"` → `"2.0"`.

---

## 7. Turtle-схема: что РАГРАФ должен генерить

### 7.1. Реальная схема СИГМЫ — плоская

Из Listing 38 ЦД Теплосетей (SPARQL-запросы) выводим shape регламента:

```turtle
@prefix : <http://sigma.local/heating#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .

:reg-pressure-150 a :Regulation ;
    :name "Регламент 150мм" ;
    :diameter 150 ;
    :pressureDeviation 0.15 ;
    :diameterDeviation 0.10 ;
    :recommendation "Проверить участок А12" .
```

**Все свойства — скалярные.** Никаких `:hasParameter [ a :Parameter ; :name "p" ; :value 12 ]` —
ЦД не умеет такое парсить через `graph.first(cols, sparql, order)`.

### 7.2. Mapping из нашей domain-модели

Наш `Regulation.parameters: list[Parameter]` → плоские scalar-свойства:

```python
# RAGRAF model
reg = Regulation(
    id="reg-pressure-150",
    name="Регламент 150мм",
    parameters=[
        Parameter(name="diameter", value=150),
        Parameter(name="pressureDeviation", value=0.15),
        Parameter(name="diameterDeviation", value=0.10),
    ],
    recommendations=[
        Recommendation(text="Проверить участок А12"),
    ],
)

# SIGMA Turtle
"""
:reg-pressure-150 a :Regulation ;
    :name "Регламент 150мм" ;
    :diameter 150 ;
    :pressureDeviation 0.15 ;
    :diameterDeviation 0.10 ;
    :recommendation "Проверить участок А12" .
"""
```

Имена predicate'ов — **берутся из `parameter.name` напрямую**, без префиксов/преобразований.
Это требует от РАГРАФа дисциплины: имена параметров должны совпадать с тем что SPARQL
в ЦД ожидает (`:pressureDeviation`, `:diameter`).

### 7.3. SHACL — отдельный граф

`POST /api/v1/regulations/shapes` отдельно от `/data`. SHACL ограничения тоже должны
быть в flat-формате. Пример:

```turtle
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix : <http://sigma.local/heating#> .

:RegulationShape a sh:NodeShape ;
    sh:targetClass :Regulation ;
    sh:property [
        sh:path :pressureDeviation ;
        sh:datatype xsd:decimal ;
        sh:minInclusive 0.0 ;
        sh:maxInclusive 1.0 ;
    ] .
```

---

## 8. Что нужно реализовать в РАГРАФе — backlog

| Приоритет | Тикет | Объём |
|---|---|---|
| **P0** | `backend/app/services/sigma_client.py` — httpx-клиент с app-JWT auth | ~150 LoC + тесты |
| **P0** | `backend/app/services/sigma_turtle.py` — flat-scalar Turtle generator | ~100 LoC + тесты |
| **P0** | `backend/app/api/processes.py::publish_to_sigma` endpoint + `Process` model field `sigma_publish_log` | ~80 LoC |
| **P0** | Config: `SIGMA_GATEWAY_URL`, `SIGMA_APP_ID`, `SIGMA_SECRET_KEY`, `SIGMA_CD_URLS` (dict per domain) | в `.env.example` |
| **P0** | UI: кнопка «Опубликовать в СИГМУ» в TwinDesignerScreen с отчётом per-domain | ~3 React-компонента |
| **P1** | `backend/app/services/sigma_catalog.py` — ETL catalog fetcher + cache | ~120 LoC |
| **P1** | UI: таблица «Адаптеры СИГМЫ» с маппингом наших subtype'ов | ~2 компонента |
| **P1** | Расширение `export-bundle` секцией `sensors.json` + bump format_version | ~80 LoC |
| **P2** | Frontend: индикация «не привязан к адаптеру СИГМЫ» на sensor-узлах flow | small |
| **P2** | Cron-job `sync_sigma_catalog_daily` — раз в сутки пересинхронизировать | ~50 LoC |

---

## 9. Что нужно от команды СИГМЫ

Разделено на **факт-вопросы** (информация, чтобы начать писать клиент) и **feature requests**
(функциональность, которой СИГМЕ нужно догнать чтобы РАГРАФ мог нормально работать).

### 9.1. Факт-вопросы (блокируют v1)

1. **Где живёт SIGMA Gateway URL для нашего стенда?** Нужен боевой или dev-URL для подключения.

2. **`app_id + secret_key` для RAGRAF.** Кто создаст application-запись в AUTH сервисе?
   Получить пару → положить в наш `.env`.

3. **Маппинг domain → URL ЦД.** Какие ЦД сейчас работают (теплосети, шум, ANPR, воздух...)
   и по каким Gateway-путям доступны их `/api/v1/regulations/data`?

4. **Семантика повторного POST.** Что делает endpoint когда граф регламентов уже существует?
   PUT-replace всю целиком или DELETE+POST? Идемпотентность — на чьей стороне?

5. **Predicate namespace.** В Listing 38 видим `:diameter`, `:pressureDeviation` без явного
   prefix declaration. Какой реальный URI используется в FUSEKI?

### 9.2. Feature requests (РАГРАФ ждёт от СИГМЫ для зрелой интеграции)

Это не «вопросы что нам делать», это **требования к зрелости СИГМЫ**, которые мы фиксируем
сейчас и просим команду включить в их roadmap.

6. **Подтверждение wiring между регламентами.** Композиция регламентов взята вами в работу.
   Нужен срок появления и контракт: как именно передаётся `(producer_reg, source_output) →
   (consumer_reg, target_param)` в FUSEKI? РАГРАФ уже умеет это создавать в своей модели.

7. **Cross-domain wiring.** Когда планируется поддержка событий между ЦД разных доменов?
   Архитектурный вариант предпочтительный — через EVENTS-сервис или через unified Process
   entity (см. п. 9). РАГРАФ может ждать и хранить cross-domain wiring как pending.

8. **Версионирование регламентов в FUSEKI.** Сейчас вы храните только последнее значение.
   Просим добавить версионирование + endpoint для отката. У РАГРАФа есть свой `versions/`
   store — после реализации с вашей стороны мы синхронизируемся.

9. **First-class Process entity в СИГМЕ.** РАГРАФ оперирует понятием `Process` (Twin) —
   набор регламентов разных доменов + wiring между ними. Сейчас этого в СИГМЕ нет, и
   нам приходится раскладывать по N ЦД при publish. Просим завести единую сущность,
   принимающую Process целиком (см. §5.1, «зрелая СИГМА»).

10. **Runtime-добавление адаптеров через API.** Сейчас новый адаптер ETL = новый Python-плагин
    + редеплой ETL. РАГРАФ позволяет аналитику добавлять sensor-типы через UI в любой момент.
    Просим endpoint `POST /api/v1/adapters/` для регистрации сенсоров без редеплоя ETL.

11. **EVENTS-source vs ETL-adapter — правило выбора.** В каких случаях источник
    регистрируется как EVENTS-source, а в каких — как ETL-adapter? Это влияет на то,
    как РАГРАФ ссылается на источники в регламентах (через `sensor_subtype` или `source_id`).

### 9.3. Бонус-вопрос (не блокирующий)

12. **TG-bot и subscription-mgmt.** Per-ЦД (как в reference) или есть централизованный сервис?
    Если per-ЦД — РАГРАФ не управляет подписками, только регламентами. Если централизованный —
    есть ли API для интеграции (например, кнопка «подписаться на отклонения по этому регламенту»
    прямо из UI РАГРАФа).

---

## 10. Связанные документы

- [external/ETL-pdf/](../external/ETL-pdf/) — первоисточник: 9 PDF с описанием сервисов
  и листингами reference-имплементации (НГУ 2025).
- [ARC-SIGMA.md](ARC-SIGMA.md) — позиционирование РАГРАФа в СИГМЕ (внутренний контекст).
- [ARC.md](ARC.md) — общая архитектура РАГРАФа.
- [backend/app/schemas/domain.py](../backend/app/schemas/domain.py) — текущие
  pydantic-модели Regulation/Process/SensorSubtype.
- [backend/app/services/sigma_export.py](../backend/app/services/sigma_export.py) —
  текущий bundle builder (продолжает обслуживать офлайн-кейс).
- [backend/app/services/turtle_bridge.py](../backend/app/services/turtle_bridge.py) —
  текущий Turtle-генератор (надо адаптировать под flat-scalar формат СИГМЫ).
- Memory: [ref_real_data](../../.claude/projects/-Users-olegbarbashin-RAGRAF/memory/ref_real_data.md) —
  заметка о flat scalar Turtle, подтверждена анализом исходников.
