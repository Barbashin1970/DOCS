# SKILL: Release & Deploy LEYKA

> Финальная инструкция для разработчика и devops: **как опубликовать LEYKA в прод и поддерживать его в живом состоянии**.
> Содержит всё, что я (как системный аналитик) выстрадал в первой публикации 2026-04-23 — чтобы следующий человек не повторял те же шаги вслепую.

---

## 1. Архитектура и философия

```
┌─────────────────────┐         ┌──────────────────────────┐
│  Vercel (Frontend)  │         │   Railway (Backend)       │
│  ─────────────────  │         │   ──────────────────      │
│  React + Vite       │  ───▶   │   FastAPI + uvicorn       │
│  leyka-nsu          │ rewrite │   leyka-production        │
│  .vercel.app        │  /api/* │   .up.railway.app         │
└─────────────────────┘         │                           │
                                │   SQLite на Volume /data  │
                                │   (вживую переживает      │
                                │    редеплои)              │
                                └──────────────────────────┘
```

### Принципы

1. **Same-origin для cookies.** Браузер видит **только Vercel-домен**. Vercel rewrite-правилом проксирует `/api/*` и `/health` на Railway. Это убирает CORS-проблемы и `SameSite=None` навсегда. Куки `leyka_session` живут как `Secure; HttpOnly; SameSite=lax`.
2. **Persistent storage = Railway Volume.** SQLite-файл лежит на **/data** (mount-path Volume'а). Без Volume каждый редеплой = чистая БД. С Volume = данные живут между деплоями.
3. **Fail-fast в проде.** Переменная `IS_PRODUCTION=true` включает stress-проверки конфига при старте. Слабый JWT_SECRET / включённый mock-auth / небезопасные cookies → бэкенд **не стартует**. Молча работать в небезопасном режиме невозможно.
4. **Watch Paths изолируют деплои.** Railway ребилдит бэк **только** когда меняется `backend/`, `nixpacks.toml`, `railway.json`. Правки фронта триггерят только Vercel.
5. **Секреты живут в env-переменных, не в коде.** `.env` локально (gitignored), Railway Variables — на проде. Никогда в коммитах, никогда в чатах.

---

## 2. Что нужно подготовить до деплоя

| Что | Где взять / сделать |
|---|---|
| Репозиторий запушен в GitHub | `git push` ветки `main` |
| Аккаунт Vercel | [vercel.com](https://vercel.com), привязать GitHub |
| Аккаунт Railway | [railway.com](https://railway.com), привязать GitHub |
| `JWT_SECRET` сгенерирован | `openssl rand -hex 32` → сохранить в менеджер паролей |
| `ROOT_EMAIL` + `ROOT_PASSWORD` | Email суперпользователя (есть в seed.py) + пароль на твой выбор |
| Mail.ru OAuth-приложение (опц.) | [account.mail.ru/developer](https://account.mail.ru/developer) |
| Google OAuth-приложение (опц.) | [console.cloud.google.com](https://console.cloud.google.com) |

> Если OAuth-провайдеров пока нет — можно деплоиться, фронт покажет только «Войти по email и паролю». OAuth добавляется позже в любой момент.

---

## 3. Backend → Railway

### 3.1. Создание сервиса

1. На [railway.com](https://railway.com) → **New Project** → **Deploy from GitHub repo** → выбираешь `LEYKA`.
2. Railway создаст сервис из всего репо. **Никаких настроек Root Directory не нужно** — корневые `nixpacks.toml` и `railway.json` сами расскажут, как собирать.

### 3.2. Volume для persistence

1. Service → вкладка **Volumes** → **+ New Volume**
2. **Mount Path: `/data`** ← критично, по этому пути ходит `DATABASE_URL`
3. Размер `1 GB` хватит на годы. Имя любое (например `leyka-volume`).

### 3.3. Env-переменные

Service → **Variables** → копируй построчно (значения замени):

```env
# === База ===
# Четыре слэша — абсолютный путь /data/leyka.db (на Volume)
DATABASE_URL=sqlite+aiosqlite:////data/leyka.db
DB_ECHO=false

# === Прод-маркер: ВКЛЮЧАЕТ fail-fast'ы безопасности в main.py ===
IS_PRODUCTION=true

# === Auth ===
JWT_SECRET=<64-символьный hex от openssl rand -hex 32>
JWT_EXPIRE_HOURS=168
AUTH_COOKIE_NAME=leyka_session
AUTH_COOKIE_SECURE=true
AUTH_COOKIE_SAMESITE=lax

# === Резервный root (логин по паролю на /auth/local) ===
ROOT_EMAIL=banksnab@gmail.com
ROOT_PASSWORD=<твой пароль>

# === Mock-провайдер: ОБЯЗАТЕЛЬНО false на проде ===
ENABLE_MOCK_AUTH=false

# === Vercel-домен (заполни после Этапа 4 — поначалу placeholder) ===
CORS_ORIGINS=["https://leyka-nsu.vercel.app"]
AUTH_SUCCESS_REDIRECT=https://leyka-nsu.vercel.app

# === OAuth: Mail.ru (опционально) ===
MAILRU_CLIENT_ID=<из кабинета Mail.ru ID>
MAILRU_CLIENT_SECRET=<из кабинета>
MAILRU_REDIRECT_URI=https://leyka-nsu.vercel.app/api/v1/auth/mailru/callback

# === OAuth: Google (опционально) ===
GOOGLE_CLIENT_ID=<из Google Cloud Console>
GOOGLE_CLIENT_SECRET=<оттуда же>
GOOGLE_REDIRECT_URI=https://leyka-nsu.vercel.app/api/v1/auth/google/callback
```

#### ⚠ Грабли env-переменных Railway

| Переменная | Тип в коде | Формат значения |
|---|---|---|
| `CORS_ORIGINS` | `list[str]` | **JSON-массив**: `["https://x.vercel.app"]` (с квадратными скобками и двойными кавычками!) |
| `IS_PRODUCTION`, `AUTH_COOKIE_SECURE`, `ENABLE_MOCK_AUTH` | `bool` | `true` / `false` (без кавычек) |
| `JWT_EXPIRE_HOURS` | `int` | `168` |
| Всё остальное | `str` | как есть, без кавычек |

> **Самый частый баг:** забыть скобки в `CORS_ORIGINS`. Pydantic не сможет распарсить, бэк упадёт с `ValidationError` при следующем редеплое. Бывает не сразу — потому что значение читается на старте, а не на каждом запросе. Плата за лень — крах посреди ночи.

### 3.4. Watch Paths (Settings → Build → Watch Paths)

Чтобы Railway не ребилдил бэк при правках фронта, добавь три паттерна:

```
/backend/**
/nixpacks.toml
/railway.json
```

После настройки: коммит, который меняет только `frontend/`, **полностью пропустит Railway** — никаких failed-меток на коммитах, никаких лишних минут билда.

### 3.5. Публичный домен

**Settings → Networking → Generate Domain** → получишь `<имя>-production.up.railway.app`. **Запиши** — он понадобится для `vercel.json`.

### 3.6. Healthcheck (опционально, рекомендую)

`railway.json` уже задаёт healthcheck `/health`. Если хочешь подкрутить через UI: **Settings → Deploy** → Healthcheck Path = `/health`, Timeout = 60.

### 3.7. Первый деплой и проверка

После сохранения переменных Railway автоматически передеплоит. В логах должны быть:

```
==> python: /opt/venv/bin/python (Python 3.12.7)
==> alembic upgrade head
INFO  [alembic.runtime.migration] Running upgrade  -> 0001_initial, ...
==> seed (no-op если БД уже заполнена)
✓ Партнёров: 4
✓ Проектов: 5
✓ Пользователей: 8 (root=banksnab@gmail.com, local-логин активен)
==> set_root_password (идемпотентно)
✓ banksnab@gmail.com: пароль обновлён, oauth_provider='local'.
==> uvicorn on port 8080
INFO:     Application startup complete.
```

И **никаких `RuntimeError: Небезопасная конфигурация для IS_PRODUCTION=true`** — значит fail-fast прошёл, конфиг безопасный.

Smoke-test:
```bash
curl https://<railway-host>/health
# {"status":"ok","service":"leyka-api","version":"0.2.0"}

curl https://<railway-host>/api/v1/auth/providers
# {"providers":[...],"local_enabled":true}
```

---

## 4. Frontend → Vercel

### 4.1. Подставь Railway-host в `vercel.json`

Открой [`frontend/vercel.json`](frontend/vercel.json), замени плейсхолдер `REPLACE_WITH_RAILWAY_HOST` на свой Railway-домен (без `https://`, без слэша на конце):

```json
{
  "rewrites": [
    { "source": "/api/:path*",
      "destination": "https://<твой>-production.up.railway.app/api/:path*" },
    { "source": "/health",
      "destination": "https://<твой>-production.up.railway.app/health" },
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

Запушь:
```bash
git add frontend/vercel.json
git commit -m "deploy: railway-host в vercel rewrites"
git push
```

### 4.2. Создание Vercel-проекта

1. [vercel.com](https://vercel.com) → **Add New** → **Project** → выбираешь `LEYKA`
2. **Configure Project**:
   - Framework Preset: **Vite** (auto)
   - **Root Directory: `frontend`** ← обязательно, иначе Vercel попытается билдить весь репо
   - Build/Output: дефолты (`npm run build`, output `dist`)
   - Environment Variables: **никаких не нужно** (всё через rewrite)
3. **Deploy** → ждёшь 1-2 минуты.

### 4.3. Кастомный домен

**Settings → Domains** → Add Domain → впиши желаемый `*.vercel.app` поддомен (например `leyka-nsu.vercel.app`). Если свободен — Vercel закрепит мгновенно. Старый авто-генерированный (типа `leyka-three.vercel.app`) можно удалить или оставить как редирект.

### 4.4. Проверка
```bash
curl https://leyka-nsu.vercel.app/health
# {"status":"ok","service":"leyka-api"} — rewrite на Railway работает

curl -I https://leyka-nsu.vercel.app/12345678-1234-1234-1234-123456789012
# HTTP/2 200 + content-type text/html — SPA-fallback работает (deeplinks)
```

### 4.5. Дозаполни Railway env под Vercel-домен

Если в `CORS_ORIGINS` / `AUTH_SUCCESS_REDIRECT` / `*_REDIRECT_URI` стоял placeholder — обнови на реальный Vercel-host. Railway автоматически передеплоит.

---

## 5. OAuth-провайдеры

### 5.1. Mail.ru ID

1. [account.mail.ru/developer](https://account.mail.ru/developer) → твоё приложение (или создай новое).
2. **Redirect URI:** `https://<vercel-host>/api/v1/auth/mailru/callback` (НЕ Railway!)
3. Скопируй `Client ID` + `Client Secret` → в Railway Variables: `MAILRU_CLIENT_ID`, `MAILRU_CLIENT_SECRET`, `MAILRU_REDIRECT_URI`.

### 5.2. Google Cloud

1. [console.cloud.google.com](https://console.cloud.google.com) → New Project (или существующий).
2. **APIs & Services → OAuth consent screen** → External → заполни App name, support email, developer email. Scopes: `userinfo.email`, `userinfo.profile`, `openid`. **Test users:** добавь свой email (без этого Google заблокирует).
3. **Credentials → Create Credentials → OAuth client ID** → Web application:
   - Authorized redirect URIs (добавь оба):
     ```
     https://<vercel-host>/api/v1/auth/google/callback
     http://localhost:8000/api/v1/auth/google/callback
     ```
4. Сохрани `Client ID` + `Client Secret` → в Railway Variables: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI`.

### 5.3. Локальный `.env` для dev

Та же пара Client ID + Secret должна быть в `backend/.env`. **Только `*_REDIRECT_URI` отличается** — локально это `http://localhost:8000/...`, на проде — `https://<vercel-host>/...`. В Google Cloud мы прописали оба URI, поэтому пара credentials работает и там и там.

### 5.4. Если кнопки нет на экране входа

```bash
curl https://<vercel-host>/api/v1/auth/providers
```
Если в массиве `providers` нет `mailru` или `google` — значит CLIENT_ID или CLIENT_SECRET пустой. Проверь Railway Variables.

---

## 6. Чек-лист безопасности (must-have для прода)

- [ ] **`IS_PRODUCTION=true`** на Railway
- [ ] **`JWT_SECRET`** ≥ 32 символа, без подстрок `dev`/`change-me`/`local`/`test`
- [ ] **`AUTH_COOKIE_SECURE=true`** (HTTPS-only куки)
- [ ] **`ENABLE_MOCK_AUTH=false`** (двойная защита: код в `auth.py` ещё раз проверяет `IS_PRODUCTION`)
- [ ] **`CORS_ORIGINS`** содержит **только** реальный Vercel-домен в JSON-массиве
- [ ] **OAuth `client_secret`** уникальный per-environment, не закоммичен
- [ ] **Volume на `/data`** смонтирован, `DATABASE_URL` указывает туда
- [ ] **Healthcheck `/health`** возвращает 200
- [ ] **`/api/v1/auth/mock/picker`** возвращает 404 на проде

Все эти проверки автоматизированы в `app/main.py:_enforce_production_safety()` — если что-то не так, бэк не стартанёт.

---

## 7. Lifecycle деплоев и тригеры пересборки

### Что триггерит что

| Push в main изменил | Vercel ребилдит? | Railway ребилдит? |
|---|---|---|
| `frontend/**` | ✅ | ❌ (отсекается Watch Paths) |
| `backend/**` | ✅ (впустую) | ✅ |
| `nixpacks.toml`, `railway.json` | ✅ (впустую) | ✅ |
| `*.md`, `frontend/vercel.json` | ✅ если vercel.json | ❌ |
| Empty commit | ✅ | ✅ (empty не попадает под Watch Paths) |

### Принудительная пересборка бэка

Если нужно переподнять Railway без правок кода:

```bash
git commit --allow-empty -m "trigger railway rebuild"
git push
```

Empty commit обходит Watch Paths и заставляет Railway пересобрать `HEAD` ветки.

### Принудительная пересборка успешного деплоя

В Railway → Deployments → ACTIVE деплой → троеточие `⋯` → **Redeploy**. Просто переподнимет тот же образ из registry — без пересборки.

### Failed-деплой ре-деплоить нельзя

У failed-деплоев в Railway нет «Redeploy» (только «Delete»). Чтобы повторить — empty commit (Способ выше).

---

## 8. Локальная разработка vs прод

| Слой | Локально (dev) | Прод |
|---|---|---|
| Backend host | `http://localhost:8000` | `https://<railway>.up.railway.app` |
| Frontend host | `http://localhost:5173` | `https://leyka-nsu.vercel.app` |
| `DATABASE_URL` | `sqlite+aiosqlite:///./leyka.db` (в `backend/`) | `sqlite+aiosqlite:////data/leyka.db` (на Volume) |
| `IS_PRODUCTION` | не задан → `false` | `true` |
| `AUTH_COOKIE_SECURE` | `false` (HTTP) | `true` (HTTPS) |
| `ENABLE_MOCK_AUTH` | `true` (удобство) | `false` (безопасность) |
| OAuth `*_REDIRECT_URI` | `http://localhost:8000/...` | `https://<vercel>/...` |
| `JWT_SECRET` | `local-dev-jwt-secret-replace-me-in-prod` (в `.env`) | сгенерированный 64-символьный hex |
| Запуск | `./leyka.command` | автоматический Railway-deploy на `git push` |

### Запуск локально
```bash
./leyka.command       # стартует backend + frontend, открывает браузер
./leyka.command --reset-db  # с перезаливкой данных через seed
./leyka.command --no-open   # без авто-открытия браузера
```

---

## 9. Диагностика

### Бэкенд жив?
```bash
curl https://<railway-host>/health
# {"status":"ok","service":"leyka-api","version":"0.2.0"}
```

### Auth работает?
```bash
curl https://<vercel-host>/api/v1/auth/providers
# {"providers":[...], "local_enabled":true}
```

### Логин по паролю работает?
```bash
curl -X POST https://<vercel-host>/api/v1/auth/local \
  -H "Content-Type: application/json" \
  -d '{"email":"banksnab@gmail.com","password":"<пароль>"}' \
  -i
# HTTP/2 200 + Set-Cookie: leyka_session=...
```

### Чтение Railway-логов

В логах Railway помечает потоки так:
- `[inf]` = stdout
- `[err]` = stderr (НЕ обязательно ошибка!)
- `[wrn]` = build-time warnings (Docker BuildKit, можно игнорировать)
- `[dbg]` = отладочные

`uvicorn` и `alembic` пишут свой `INFO:` лог в **stderr** (это норма для Python). Реальные ошибки содержат `ERROR`, `Traceback`, `Exception`. Если только `INFO:` — спокойно.

---

## 10. Типовые ошибки (грабли, на которые мы наступили)

### `error: externally-managed-environment` при `pip install`

**Где:** Railway build log, шаг `[8/9] RUN /opt/venv/bin/pip install ...`

**Причина:** Nix-Python (PEP 668) запрещает `pip install` напрямую в системную инсталляцию.

**Фикс:** уже в `nixpacks.toml` — install-фаза создаёт `/opt/venv` и ставит туда. `start.sh` активирует его через `export PATH="/opt/venv/bin:$PATH"`. Если случайно сломаешь эту цепочку — ошибка вернётся.

### `pydantic_core._pydantic_core.ValidationError: CORS_ORIGINS`

**Причина:** значение в Railway записано без квадратных скобок, например `https://app.vercel.app` вместо `["https://app.vercel.app"]`.

**Фикс:** Variables → CORS_ORIGINS → отредактировать значение в JSON-формат.

### `RuntimeError: Небезопасная конфигурация для IS_PRODUCTION=true`

**Причина:** `IS_PRODUCTION=true`, но один или несколько security-параметров в дефолте.

**Фикс:** в сообщении перечислено, что именно не так. Поправь Variables, Railway передеплоит.

### Failed-деплой с обрывом лога на `image push`

**Причина:** Build удалось, но push образа в Railway-registry упал (сетевой блип). Лог обрывается, потому что push-ошибка не успевает залогироваться.

**Фикс:** empty commit (см. §7).

### Mock-picker открыт публично

**Симптом:** `https://<host>/api/v1/auth/mock/picker` возвращает HTML со списком юзеров.

**Причина:** `IS_PRODUCTION` не выставлен или `false`, а `ENABLE_MOCK_AUTH=true`.

**Фикс:** `IS_PRODUCTION=true` + `ENABLE_MOCK_AUTH=false` на Railway. После редеплоя `mock/picker` отдаёт 404.

### `502 Application failed to respond` посреди использования

**Причина:** Railway мидл-деплой — старый контейнер уже остановлен, новый ещё не поднят. Окно 5-30 секунд.

**Фикс:** ждать → обновить страницу. Если повторяется — смотреть логи последнего деплоя, мог упасть startup.

### «Крестик» на коммите в GitHub после успешного деплоя

**Причина:** на коммит подписаны несколько checks (Vercel, Railway, GitHub Actions). Один из них упал, другие зелёные. Сервис при этом может работать.

**Фикс:** на странице коммита в GitHub → Checks → посмотреть, какой именно check красный. Чаще всего — GitHub Actions CI на каком-то локальном тесте, не блокирующий прод.

### OAuth: `state mismatch — попробуй войти ещё раз`

**Причина:** браузер потерял cookie с OAuth-state (incognito + сторонние cookies заблокированы, либо TTL > 10 минут истёк).

**Фикс:** перезагрузить страницу логина и попробовать заново.

### OAuth: `Error 400: redirect_uri_mismatch`

**Причина:** Redirect URI в кабинете провайдера (Mail.ru/Google) не совпадает с тем, что в Railway env.

**Фикс:** сравнить **посимвольно** — слеш на конце, http vs https, домен.

---

## 11. Ротация секретов

### Если `JWT_SECRET` утёк

1. Сгенерируй новый: `openssl rand -hex 32`
2. Railway → Variables → `JWT_SECRET` → новое значение → Save
3. Railway автоматически передеплоит — **все существующие сессии инвалидируются**, юзеры разлогинятся.

Это и есть «kill switch» — если кто-то утянул токены, ротация секрета мгновенно их обнуляет.

### Если OAuth `client_secret` утёк (Mail.ru / Google)

1. В кабинете провайдера → отозвать старый секрет, выпустить новый
2. Railway → Variables → `MAILRU_CLIENT_SECRET` (или `GOOGLE_CLIENT_SECRET`) → новое значение
3. Локальный `backend/.env` тоже обновить (для dev-сервера)

### Если `ROOT_PASSWORD` утёк

1. Railway → Variables → `ROOT_PASSWORD` → новый пароль → Save
2. После редеплоя `set_root_password.py` идемпотентно обновит хеш в БД.

---

## 12. Бэкап БД (что есть, что добавить)

### Что есть сейчас
- **Railway Volume** хранит `leyka.db` независимо от контейнера. Редеплой бэка не сносит данные.
- **`set_root_password.py`** идемпотентен — root-юзер не теряется.
- **`seed.py`** не перезаливает, если БД не пустая.

### Чего нет (запланировано в [backlog.md](backlog.md))
- **Фаза 2:** `VACUUM INTO`-снапшоты + admin-эндпоинт `/admin/backup` + расписание (каждые 6 часов, retain 30) + дублирование на Яндекс.Диск.
- **Фаза 3:** JSON export/import per-project — для миграции отдельного проекта между средами (dev → prod, прод → демо).

Без них Volume — единственная защита. Если Railway-storage сломается, восстановиться будет нечем. Реализовать Phase 2 — приоритет №1 после релиза.

---

## 13. Чек-лист «готов к боевому использованию»

### Инфраструктура
- [ ] Railway: Volume `/data` смонтирован
- [ ] Railway: публичный домен поднят, `/health` отвечает 200
- [ ] Railway: Watch Paths настроены (`/backend/**`, `/nixpacks.toml`, `/railway.json`)
- [ ] Vercel: Root Directory = `frontend`, домен привязан
- [ ] Vercel: `vercel.json` указывает на реальный Railway-host (не на placeholder)

### Безопасность (см. §6)
- [ ] `IS_PRODUCTION=true` + все fail-fast'ы прошли (бэк стартует без `RuntimeError`)
- [ ] `JWT_SECRET` уникальный и сгенерированный
- [ ] OAuth `client_secret` ротированы (если утекали локально/в чате)
- [ ] `ENABLE_MOCK_AUTH=false` на проде
- [ ] `mock/picker` возвращает 404

### Функциональность
- [ ] Резервный root-логин работает (`banksnab@gmail.com` + пароль)
- [ ] Mail.ru OAuth работает (если настроен)
- [ ] Google OAuth работает (если настроен)
- [ ] Тестовая задача создаётся и **переживает редеплой бэка** (это критично — проверка Volume)
- [ ] Deeplink на задачу `https://<vercel>/<uuid>` открывает её

### Операционные процессы
- [ ] У тебя в менеджере паролей: `JWT_SECRET`, `ROOT_PASSWORD`, OAuth client secrets
- [ ] Команда знает, как читать Railway-логи (`[inf]`/`[err]`/`[wrn]` — см. §9)
- [ ] Команда знает, как пересобрать бэк (empty commit — см. §7)
- [ ] **Процесс бэкапа есть** (на момент релиза — нет, см. §12 — поставить в priority)

---

## TL;DR — самые ёмкие правила

1. **`IS_PRODUCTION=true`** — единственный переключатель безопасности. Без него бэк работает как dev.
2. **Vercel rewrite — лучший друг.** Same-origin = нет CORS, нет SameSite-болей, куки просто работают.
3. **Volume mount-path должен совпадать с DATABASE_URL.** `/data` ↔ `sqlite:////data/leyka.db`.
4. **Watch Paths экономят время.** `frontend/`-правки не должны триггерить Railway.
5. **Empty commit пересобирает любой failed.** Запомни эту команду — выручит часто.
6. **CORS_ORIGINS — JSON-массив.** Самый коварный баг.
7. **`set_root_password.py` идемпотентен.** Можно гонять при каждом старте, ничего не сломает.
8. **Логи `[err]` ≠ ошибки.** Это просто stderr. Реальная ошибка содержит `ERROR`/`Traceback`.

---

*Этот документ — финал работ 2026-04-23 по подъёму прода LEYKA. Если столкнёшься с чем-то новым — добавляй сюда раздел в стиле «§10. Типовые ошибки», чтобы следующий человек не наступил.*
