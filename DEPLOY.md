# Деплой LEYKA: Vercel (фронт) + Railway (бэк)

> **Архитектура.** Фронт раздаёт Vercel. Все запросы к API/health он
> **прокидывает rewrite-правилом** на Railway-бэкенд (см. `frontend/vercel.json`).
> С точки зрения браузера всё под одним origin'ом — куки работают без
> CORS-плясок и без `SameSite=None`.

> **Структура репо для деплоев.**
> - `railway.json` в корне — высокоуровневая Railway-конфигурация: start-команда
>   `cd backend && ./start.sh`, healthcheck, политика рестарта. В Railway
>   dashboard поля Custom Build/Start Command будут подсвечены как «The value is
>   set in railway.json» — это нормально, всё управляется из репо.
> - `nixpacks.toml` в корне — низкоуровневая инструкция Nixpacks'у: «это Python
>   3.12, ставь зависимости из `backend/requirements.txt`». Без него Nix не
>   обнаружил бы Python (наш `requirements.txt` лежит не в корне).
> - Деплоится **весь репо** — никакой «Root Directory» в Railway трогать не надо.
> - `frontend/vercel.json` — для Vercel; на Vercel выставляется только
>   **Root Directory = `frontend`** в Project Settings.

---

## Этап 0. Что иметь под рукой

- Репозиторий запушен в GitHub.
- Аккаунты на [vercel.com](https://vercel.com) и [railway.com](https://railway.com).
- Mail.ru ID developer-кабинет (если хочешь логин через Mail.ru) и/или Google
  Cloud Console (для логина через Google). Если пока ни того, ни другого —
  можно жить только на резервном root-логине: `banksnab@gmail.com` / `2012199600`.
- Готовый `JWT_SECRET`. Сгенерируй один раз:
  ```bash
  openssl rand -hex 32
  ```
  Вставишь в env Railway. Сохрани в менеджере паролей.

---

## Этап 1. Backend на Railway

### 1.1. Создаём сервис
1. На [railway.com](https://railway.com) → **New Project** → **Deploy from GitHub repo** → выбираешь `LEYKA`.
2. Railway создаст сервис **из всего репо**. Откроется dashboard. В **Settings**
   ничего трогать не надо — `railway.json` и `nixpacks.toml` в корне всё уже
   рассказали:
   - В **Settings → Build → Custom Build Command** будет пусто или серым с
     пометкой «set in railway.json» (в нашем случае build=NIXPACKS, без override
     команды — install из `nixpacks.toml`).
   - В **Settings → Deploy → Custom Start Command** будет `cd backend && ./start.sh`
     с пометкой «The value is set in `railway.json`» — ровно как на твоих
     прежних проектах.
   - Healthcheck Path = `/health`, Restart Policy = On Failure (всё из railway.json).

### 1.2. Подключаем persistent Volume
1. В сервисе → вкладка **Volumes** → **+ New Volume**.
2. **Mount Path: `/data`**. Имя любое (например `leyka-data`). Размер `1 GB`
   хватит на годы (SQLite-файл с тысячей задач — единицы мегабайт).
3. Без этого `git push` будет пересоздавать контейнер, и БД исчезнет на каждом
   деплое — это и есть та самая боль, ради которой всё затевалось.

### 1.3. Конфигурируем env-переменные
В сервисе → **Variables**. Скопируй построчно (значения замени на свои):

```env
# === База ===
# Четыре слэша = абсолютный путь /data/leyka.db (на смонтированный Volume).
DATABASE_URL=sqlite+aiosqlite:////data/leyka.db
DB_ECHO=false

# === Прод-маркер: включает fail-fast'ы безопасности ===
IS_PRODUCTION=true

# === Auth ===
# Сгенерировал openssl rand -hex 32 — вставь сюда.
JWT_SECRET=ЗАМЕНИ_НА_СГЕНЕРИРОВАННЫЙ_СЕКРЕТ_64_СИМВОЛА
JWT_EXPIRE_HOURS=168

AUTH_COOKIE_NAME=leyka_session
AUTH_COOKIE_SECURE=true
AUTH_COOKIE_SAMESITE=lax

# === Резервный root (логин по паролю на /auth/local) ===
ROOT_EMAIL=banksnab@gmail.com
ROOT_PASSWORD=2012199600

# === Mock-провайдер: ОТКЛЮЧЁН на проде. fail-fast его проверяет. ===
ENABLE_MOCK_AUTH=false

# === Vercel-домен. Можно поставить заглушку и обновить после Этапа 2. ===
CORS_ORIGINS=["https://leyka-nsu.vercel.app"]
AUTH_SUCCESS_REDIRECT=https://leyka-nsu.vercel.app

# === OAuth: Mail.ru.
# Callback-URL УКАЗЫВАЕТ НА VERCEL — Vercel rewrite'ом пробросит его на Railway.
MAILRU_CLIENT_ID=ИЗ_КАБИНЕТА_MAILRU
MAILRU_CLIENT_SECRET=ИЗ_КАБИНЕТА_MAILRU
MAILRU_REDIRECT_URI=https://leyka-nsu.vercel.app/api/v1/auth/mailru/callback

# === OAuth: Google (если нужен) ===
# GOOGLE_CLIENT_ID=
# GOOGLE_CLIENT_SECRET=
# GOOGLE_REDIRECT_URI=https://leyka-nsu.vercel.app/api/v1/auth/google/callback
```

> **Важно.** `IS_PRODUCTION=true` запускает stress-проверки в
> [`app/main.py`](backend/app/main.py) `_enforce_production_safety`. Если
> `JWT_SECRET` короче 32 символов или содержит подстроки `dev`/`change-me`/`local`,
> или `ENABLE_MOCK_AUTH=true`, или `AUTH_COOKIE_SECURE=false` — бэкенд
> **упадёт на старте** с понятной ошибкой. Это защита, а не баг.

### 1.4. Первый деплой
1. После сохранения переменных Railway автоматически передеплоит сервис.
   Жди 2–3 минуты.
2. Логи смотри на **Deployments → последний → View logs**. Должно пройти:
   ```
   ==> alembic upgrade head
   INFO  [alembic.runtime.migration] Running upgrade  -> 0001_initial, ...
   ==> seed (no-op если БД уже заполнена)
   ✓ Партнёров: 4
   ✓ Проектов: 5
   ✓ Пользователей: 8 (root=banksnab@gmail.com, local-логин активен)
   ...
   ==> set_root_password (идемпотентно)
   ✓ banksnab@gmail.com: пароль уже актуален, ничего не меняю.
   ==> uvicorn on port 8080
   INFO:     Application startup complete.
   ```
3. Подними публичный домен: **Settings → Networking → Generate Domain**.
   Получишь что-то вроде `leyka-backend-production.up.railway.app`.
   **Запиши его** — он понадобится в Этапе 2.
4. Smoke-test: открой `https://<твой-railway-host>/health` — должен ответить
   `{"status":"ok","service":"leyka-api","version":"0.2.0"}`.

---

## Этап 2. Frontend на Vercel

### 2.1. Подставляем Railway-host в `vercel.json`
1. Открой [`frontend/vercel.json`](frontend/vercel.json), замени строку-плейсхолдер
   `REPLACE_WITH_RAILWAY_HOST` на свой Railway-домен (без `https://`, без слэша
   в конце). Например:
   ```json
   "destination": "https://leyka-backend-production.up.railway.app/api/:path*"
   ```
   Поправь обе rewrite-строки (`/api/...` и `/health`).
2. Закоммить и запушь:
   ```bash
   git add frontend/vercel.json
   git commit -m "deploy: подставил railway-host во vercel rewrites"
   git push
   ```

### 2.2. Создаём проект Vercel
1. На [vercel.com](https://vercel.com) → **Add New** → **Project** → выбираешь `LEYKA`.
2. **Configure Project**:
   - Framework Preset: **Vite** (auto-detect, обычно сам определяется).
   - **Root Directory: `frontend`** ← это критично, иначе Vercel попытается билдить весь репо.
   - Build & Output Settings — оставь дефолты (`npm run build`, output = `dist`).
   - Environment Variables — никаких не нужно (rewrite в `vercel.json` уже всё связывает).
3. **Deploy**. Жди 1–2 минуты.

### 2.3. Привязываем домен
- Если хочешь именно `leyka-nsu.vercel.app` — иди в **Settings → Domains** и
  добавь его (Vercel разрешит, если свободен).
- Если занят — Vercel выдаст что-то типа `leyka-nsu-<hash>.vercel.app`. **Запиши**.

### 2.4. Smoke-test
- Открой `https://<твой-vercel-host>` — должен загрузиться UI с экраном входа.
- Открой `https://<твой-vercel-host>/health` — должен ответить тем же JSON,
  что Railway (значит rewrite работает).

---

## Этап 3. Сводим домены

Если выше ты ставил placeholder `leyka-nsu.vercel.app`, а реальный домен другой —
обнови переменные на Railway:

| Переменная Railway | Чему равняется |
|---|---|
| `CORS_ORIGINS` | `["https://<реальный-vercel-host>"]` |
| `AUTH_SUCCESS_REDIRECT` | `https://<реальный-vercel-host>` |
| `MAILRU_REDIRECT_URI` | `https://<реальный-vercel-host>/api/v1/auth/mailru/callback` |
| `GOOGLE_REDIRECT_URI` | `https://<реальный-vercel-host>/api/v1/auth/google/callback` |

Railway автоматически передеплоит сервис после сохранения переменных.

---

## Этап 4. OAuth-провайдеры (если используешь)

### Mail.ru ID
1. [account.mail.ru/developer](https://account.mail.ru/developer) → твоё приложение.
2. **Redirect URI** добавь: `https://<реальный-vercel-host>/api/v1/auth/mailru/callback`.
3. **Client Secret тот, что лежал в локальном `.env`, обязательно отозви и
   перевыпусти** — он засветился в файле. Новый секрет вставь в Railway →
   Variables → `MAILRU_CLIENT_SECRET`. Локальный `.env` тоже обнови.

### Google
1. [console.cloud.google.com](https://console.cloud.google.com) → твой проект →
   APIs & Services → Credentials.
2. У OAuth 2.0 Client ID добавь **Authorized redirect URI**:
   `https://<реальный-vercel-host>/api/v1/auth/google/callback`.

---

## Этап 5. Финальный smoke-test

1. Открой `https://<твой-vercel-host>` в incognito-окне.
2. Войди по резервному root: email `banksnab@gmail.com`, пароль `2012199600`.
   Должен залогиниться, увидеть сайдбар с заказчиками и проектами (из seed'а).
3. Войди через Mail.ru (если настроил OAuth). Должен пройти flow без ошибок
   «state mismatch».
4. Открой любую задачу → скопируй её UUID → вставь в адресную строку как
   `https://<vercel-host>/<uuid>`. Должна открыться задача — это проверка
   SPA-fallback'а.
5. Создай тестовый коммент. Сделай `git push` (любую правку) — Railway передеплоит
   бэк, данные должны **остаться**. Это проверка Volume.

Если всё работает — деплой завершён.

---

## FAQ / типовые проблемы

**🔥 На Railway упало с `RuntimeError: Небезопасная конфигурация для IS_PRODUCTION=true`.**
Это не баг, это `_enforce_production_safety`. В сообщении перечислено, что
именно не так — починить env-переменные и редеплой.

**🔥 На Vercel вижу `404 Not Found` при попытке открыть `/api/...`.**
`REPLACE_WITH_RAILWAY_HOST` остался в `vercel.json`. Подставь реальный
Railway-host, закоммить, запушь — Vercel передеплоит.

**🔥 Логин через Mail.ru возвращает `OAuth state mismatch — попробуй войти ещё раз`.**
Cookie state не сохранилось. Через rewrite это same-origin, обычно работает.
Если не работает — проверь, что `AUTH_COOKIE_SECURE=true` и фронт ходит
через **https** (не http).

**🔥 На Vercel я редактировал `vercel.json`, но изменения не видны.**
Vercel кеширует static-конфиги. После правки — push в git, Vercel сам
передеплоит. Если не помогает — **Settings → Deployments → Redeploy →
без кеша**.

**🔥 На Railway я не вижу публичного домена.**
Settings → Networking → **Generate Domain**. После этого появится `*.up.railway.app`.
Если нужен кастомный — там же **Custom Domain**.

**🔥 Тесты в CI падают, потому что pytest-asyncio обновился.**
Пины в `requirements.txt` обновлены, тесты зелёные локально (84/84).
Если CI красный — посмотри в GHA логи.

---

## Чек-лист готовности к боевому использованию

- [ ] Railway: Volume `/data` смонтирован, `DATABASE_URL` ссылается на него.
- [ ] Railway: `IS_PRODUCTION=true`, `JWT_SECRET` сгенерирован,
      `AUTH_COOKIE_SECURE=true`, `ENABLE_MOCK_AUTH=false`.
- [ ] Railway: публичный домен поднят, `/health` отвечает.
- [ ] Vercel: `vercel.json` указывает на реальный Railway-host (не на placeholder).
- [ ] Vercel: Root Directory выставлен в `frontend`.
- [ ] Mail.ru `client_secret` ротирован, новый прописан только на Railway
      (нигде в коде/git'е).
- [ ] OAuth redirect URIs прописаны во всех провайдерах под Vercel-host.
- [ ] Резервный root (`banksnab@gmail.com`) реально логинится через `/login`.
- [ ] Тестовая задача после редеплоя бэка не пропала — Volume работает.
