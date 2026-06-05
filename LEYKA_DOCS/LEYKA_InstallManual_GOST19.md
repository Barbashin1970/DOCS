# ИНСТРУКЦИЯ ПО УСТАНОВКЕ ПРОГРАММНОГО ОБЕСПЕЧЕНИЯ
## «LEYKA — Система управления проектами центра искусственного интеллекта»

---

> **Обозначение документа:** ЦИИ НГУ.LEYKA.ИУ.001-2026  
> **Версия:** 1.1  
> **Дата составления:** 27 апреля 2026 г.  
> **Версия программы:** 0.2.0  
> **Статус:** действующий  
> **Разработано в соответствии с:** ГОСТ 19.502-78 «Описание применения. Требования к содержанию и оформлению» (в части установки)  
> **Основание:** ТЗ ЦИИ НГУ.LEYKA.ТЗ.001-2026 (версия 1.1 от 27.04.2026); DEPLOY.md, SKILL-RELEASE.md, SKILL-INFRA.md репозитория

---

## Лист согласования

| Должность | ФИО | Подпись | Дата |
|---|---|---|---|
| Директор ЦИИ НГУ (Заказчик) | | | |
| Главный разработчик (Исполнитель) | | | |

---

## Содержание

1. Общие сведения  
2. Требования к техническим средствам  
   2.1 Вариант А: облачное развёртывание (Vercel + Railway)  
   2.2 Вариант Б: развёртывание на собственном сервере (НГУ)  
   2.3 Требования к рабочим местам операторов  
3. Структура поставки  
4. Предварительные действия перед установкой  
   4.1 Получение OAuth-ключей Mail.ru  
   4.2 Получение OAuth-ключей Google  
   4.3 Генерация JWT_SECRET  
   4.4 Настройка Google Drive (опционально)  
5. Установка — Вариант А: Vercel + Railway  
   5.1 Развёртывание бэкенда на Railway  
   5.2 Развёртывание фронтенда на Vercel  
   5.3 Согласование доменов  
   5.4 Настройка OAuth redirect URI  
   5.5 Финальная проверка  
6. Установка — Вариант Б: собственный сервер ЦИИ НГУ  
   6.1 Использование готового bootstrap-скрипта  
   6.2 Что делает скрипт  
   6.3 Настройка переменных окружения после установки  
   6.4 Резервное копирование (включается автоматически)  
   6.5 Альтернатива: ручная установка (для специальных случаев)  
   6.6 Проверка после установки  
   6.7 Перенос данных с Railway на сервер ЦИИ НГУ  
7. Запуск локально (для разработки)  
8. Проверка корректности установки  
9. Возможные неисправности и методы их устранения  
10. Приложения  
    Приложение А. Полный шаблон .env для Railway  
    Приложение Б. Чек-лист готовности к боевому использованию  

---

## 1. Общие сведения

**Наименование программы:** «LEYKA — Система управления проектами ЦИИ НГУ»  
**Обозначение:** ЦИИ-НГУ-LEYKA-2026  
**Версия:** 0.2.0  
**Репозиторий:** https://github.com/Barbashin1970/LEYKA  

Программа реализована как **клиент-серверное веб-приложение**:

| Компонент | Технология | Место развёртывания |
|---|---|---|
| Фронтенд (SPA) | React 18.3 + TypeScript 5.6 (strict) + Vite 5.4; производственный бандл — 167,7 КБ JavaScript + 10,2 КБ CSS после gzip | Vercel (CDN) либо Nginx на сервере ЦИИ НГУ |
| Бэкенд (API) | Python 3.12 + FastAPI 0.115 + Alembic 1.14 (24 миграции) | Railway PaaS либо собственный сервер ЦИИ НГУ |
| База данных | SQLite (асинхронный драйвер aiosqlite), режим WAL — единый режим для dev и prod (миграция на PostgreSQL не планируется в горизонте 5 лет, обоснование — SKILL-INFRA.md) | Persistent Volume Railway или каталог `/var/lib/leyka/` сервера |

Архитектура проксирования: **Vercel rewrite** проксирует все запросы `/api/*` и `/health` на Railway-бэкенд. С точки зрения браузера всё — один origin, cookie работают без CORS-настройки и без `SameSite=None`.

---

## 2. Требования к техническим средствам

### 2.1 Вариант А: облачное развёртывание (Vercel + Railway)

| Ресурс | Требование |
|---|---|
| Аккаунт GitHub | Репозиторий должен быть доступен публично или через GitHub-интеграцию |
| Аккаунт Vercel | Free tier достаточен (до 100 ГБ трафика/мес.) |
| Аккаунт Railway | Starter: 512 МБ RAM, 1 виртуальный CPU, 1 ГБ Persistent Volume |
| Интернет-соединение (оператор) | Не менее 2 Мбит/с для первоначальной настройки |

### 2.2 Вариант Б: развёртывание на собственном сервере (НГУ)

| Компонент | Минимальные требования | Рекомендуемые |
|---|---|---|
| CPU | 2 ядра x86-64 | 4 ядра |
| RAM | 2 ГБ | 4 ГБ |
| Диск | 20 ГБ SSD | 50 ГБ SSD |
| ОС | Ubuntu 22.04 LTS / Debian 12 | Ubuntu 24.04 LTS |
| Python | 3.12 | 3.12 |
| Node.js | 20 LTS | 22 LTS |
| Nginx | 1.24 | 1.26 |
| Открытые порты | 80 (HTTP) + 443 (HTTPS) | + 22 (SSH) |

### 2.3 Требования к рабочим местам операторов

Установка дополнительного ПО на рабочее место оператора **не требуется**. Для работы достаточен современный браузер:

- Google Chrome ≥ 120, Firefox ≥ 120, Safari ≥ 17, Edge ≥ 120.
- Разрешение экрана ≥ 1280 × 720 пикселей.
- JavaScript и cookies должны быть включены.

---

## 3. Структура поставки

Программа поставляется в виде исходного кода в репозитории:

```
LEYKA/
├── backend/                    # Python FastAPI бэкенд
│   ├── app/                    # Исходный код приложения
│   │   ├── main.py             # Точка входа, настройка FastAPI, проверки прод-окружения
│   │   ├── config.py           # Настройки из переменных окружения (pydantic-settings)
│   │   ├── database.py         # async-engine SQLAlchemy + сессии
│   │   ├── api/v1/             # REST-эндпойнты по доменам (11 модулей, 73 эндпойнта)
│   │   ├── auth/               # OAuth-провайдеры (mailru, google, mock) + JWT/bcrypt
│   │   ├── models/             # SQLAlchemy ORM-модели (15 таблиц)
│   │   ├── schemas/            # Pydantic v2 схемы (Create/Update/Response)
│   │   ├── checklist/          # Каталог слотов документации (12 стадий, ~40 слотов)
│   │   └── services/           # Сервисы (Google Drive — drag-and-drop в чек-лист)
│   ├── migrations/versions/    # Alembic-миграции (24 файла: 0001_initial … 0024_*)
│   ├── tests/                  # Автотесты pytest (20 файлов, 288 тестов)
│   ├── scripts/                # seed.py, set_root_password.py, authorize_gdrive.py
│   ├── requirements.txt        # Python-зависимости
│   └── start.sh                # Скрипт запуска (alembic + seed + uvicorn) — Railway entrypoint
├── frontend/                   # React SPA фронтенд
│   ├── src/                    # Исходный код (TypeScript strict)
│   │   ├── api/                # Сгенерированные типы и клиент REST API
│   │   ├── components/         # React-компоненты (49 файлов *.tsx)
│   │   ├── lib/                # Чистые функции и каталоги (для unit-тестов)
│   │   ├── store/              # Zustand-stores (sidebar, chatWallpaper, about)
│   │   └── *.test.ts           # Vitest-тесты рядом с реализацией (176 тестов)
│   ├── vercel.json             # Конфигурация Vercel + rewrite-правила /api/* → Railway
│   ├── vite.config.ts          # Vite + Vitest configuration
│   └── package.json            # Node.js зависимости
├── deploy/
│   └── nsu-server/             # Bootstrap-скрипты для развёртывания на сервере ЦИИ НГУ:
│       ├── setup.sh            #   главный установочный скрипт (11 шагов, idempotent)
│       ├── leyka-api.service   #   systemd unit для uvicorn
│       ├── leyka-backup.sh     #   ежесуточный бэкап БД через rclone
│       ├── leyka-backup.timer  #   systemd timer (каждые 6 часов)
│       └── nginx-leyka.conf    #   шаблон nginx-сайта (TLS + SPA + reverse-proxy)
├── docs/
│   └── GOOGLE_DRIVE_SETUP.md   # 11-шаговая настройка Google Drive (OAuth user-flow)
├── railway.json                # Конфигурация Railway PaaS (start, healthcheck)
├── nixpacks.toml               # Nixpacks-конфигурация для Railway (Python 3.12, venv)
├── leyka.command               # Скрипт локального запуска (macOS/Linux)
├── README.md                   # Руководство пользователя (расширенное)
├── ARC.md                      # Архитектурная документация (диаграммы Mermaid)
├── DEPLOY.md                   # Краткая инструкция по развёртыванию
├── SKILL-RELEASE.md            # Подробный релизный процесс (Vercel + Railway)
├── SKILL-INFRA.md              # Развёртывание на собственном сервере + бэкап-стратегия
├── SKILL-AUTH.md               # OAuth-конфигурация Mail.ru / Google
├── SKILL-TESTPLAN.md           # Тест-план (back + front)
├── SKILL-MERMAID.md            # Гайдлайны диаграмм
├── SKILL-UI-Master.md          # UI-гайдлайны (цвета, тени, иконки)
├── LEYKA_TZ_GOST19.md          # Техническое задание (ГОСТ 19.201)
├── LEYKA_PMI_GOST19.md         # Программа и методика испытаний (ГОСТ 19.301)
├── LEYKA_UserManual_GOST19.md  # Руководство пользователя (ГОСТ 19.505)
└── LEYKA_InstallManual_GOST19.md  # Настоящий документ (ГОСТ 19.502)
```

---

## 4. Предварительные действия перед установкой

### 4.1 Получение OAuth-ключей Mail.ru

1. Перейти на https://account.mail.ru/developer.
2. Создать или выбрать существующее OAuth-приложение.
3. В разделе **Redirect URI** добавить:
   ```
   https://<домен-фронтенда>/api/v1/auth/mailru/callback
   ```
4. Скопировать **Client ID** и **Client Secret** — они потребуются в разделе 5.1.

> **Важно.** Client Secret должен быть ротирован перед вводом в боевую среду — не использовать секрет из локального `.env`-файла (он мог попасть в историю Git).

### 4.2 Получение OAuth-ключей Google (при необходимости)

1. Перейти в https://console.cloud.google.com → IAM & Admin → ваш проект.
2. APIs & Services → Credentials → **+ Create Credentials** → **OAuth 2.0 Client ID**.
3. В разделе **Authorized redirect URIs** добавить:
   ```
   https://<домен-фронтенда>/api/v1/auth/google/callback
   ```
4. Скопировать **Client ID** и **Client Secret**.

### 4.3 Генерация JWT_SECRET

Выполнить в терминале:

```bash
openssl rand -hex 32
```

Сохранить результат (64 hex-символа) в менеджере паролей. Это значение будет использовано в переменной `JWT_SECRET`.

> **Требование безопасности.** При `IS_PRODUCTION=true` бэкенд не запустится, если `JWT_SECRET` короче 32 символов или содержит подстроки `dev`, `change-me`, `local`.

### 4.4 Настройка Google Drive для drag-and-drop (опционально)

Для включения функции перетаскивания файлов в чек-лист необходимо пройти one-time setup по инструкции [`docs/GOOGLE_DRIVE_SETUP.md`](https://github.com/Barbashin1970/LEYKA/blob/main/docs/GOOGLE_DRIVE_SETUP.md). Занимает ~5 минут. Без этого шага drag-and-drop возвращает 503, остальное работает.

---

## 5. Установка — Вариант А: Vercel + Railway

### 5.1 Развёртывание бэкенда на Railway

**Шаг 1. Создание сервиса:**

1. Открыть https://railway.com → **New Project** → **Deploy from GitHub repo** → выбрать репозиторий `LEYKA`.
2. Railway создаст сервис из всего репозитория. Конфигурация уже задана в `railway.json` и `nixpacks.toml`:
   - Start-команда: `cd backend && ./start.sh`
   - Healthcheck: `GET /health`
   - Restart Policy: On Failure

**Шаг 2. Подключение Persistent Volume:**

1. В сервисе открыть вкладку **Volumes** → **+ New Volume**.
2. Установить **Mount Path: `/data`**, имя — `leyka-data`, размер — **1 ГБ**.

> **Критично.** Без Volume база данных пересоздаётся при каждом деплое и все данные теряются.

**Шаг 3. Переменные окружения:**

В сервисе открыть **Variables**, добавить построчно (шаблон — Приложение А):

| Переменная | Значение |
|---|---|
| `DATABASE_URL` | `sqlite+aiosqlite:////data/leyka.db` |
| `IS_PRODUCTION` | `true` |
| `JWT_SECRET` | *сгенерированный в п.4.3* |
| `JWT_EXPIRE_HOURS` | `168` |
| `AUTH_COOKIE_NAME` | `leyka_session` |
| `AUTH_COOKIE_SECURE` | `true` |
| `AUTH_COOKIE_SAMESITE` | `lax` |
| `ROOT_EMAIL` | *email администратора* |
| `ROOT_PASSWORD` | *пароль администратора* |
| `ENABLE_MOCK_AUTH` | `false` |
| `CORS_ORIGINS` | `["https://<домен-vercel>"]` |
| `AUTH_SUCCESS_REDIRECT` | `https://<домен-vercel>` |
| `MAILRU_CLIENT_ID` | *из п.4.1* |
| `MAILRU_CLIENT_SECRET` | *из п.4.1* |
| `MAILRU_REDIRECT_URI` | `https://<домен-vercel>/api/v1/auth/mailru/callback` |

**Шаг 4. Первый деплой:**

1. После сохранения переменных Railway автоматически выполнит деплой (2–3 минуты).
2. Проверить логи (**Deployments → View logs**). Ожидаемый вывод:
   ```
   ==> alembic upgrade head
   INFO [alembic.runtime.migration] Running upgrade -> 0001_initial, ...
   ==> seed (no-op если БД уже заполнена)
   ✓ Партнёров: 4, Проектов: 5, Пользователей: 8
   ==> uvicorn on port 8080
   INFO: Application startup complete.
   ```
3. Поднять публичный домен: **Settings → Networking → Generate Domain**.
4. Smoke-test: открыть `https://<railway-host>/health` — ожидаемый ответ:
   ```json
   {"status": "ok", "service": "leyka-api", "version": "0.2.0"}
   ```

### 5.2 Развёртывание фронтенда на Vercel

**Шаг 1. Подстановка Railway-хоста:**

1. Открыть файл `frontend/vercel.json`.
2. Заменить `REPLACE_WITH_RAILWAY_HOST` на реальный Railway-домен (без `https://` и без слэша):
   ```json
   "destination": "https://leyka-production.up.railway.app/api/:path*"
   ```
3. Сохранить, закоммитить и запушить:
   ```bash
   git add frontend/vercel.json
   git commit -m "deploy: Railway host in vercel.json"
   git push
   ```

**Шаг 2. Создание проекта Vercel:**

1. Открыть https://vercel.com → **Add New → Project** → выбрать репозиторий `LEYKA`.
2. Настройки:
   - Framework Preset: **Vite** (определяется автоматически);
   - **Root Directory: `frontend`** ← обязательно;
   - Build Command: `npm run build` (по умолчанию);
   - Output: `dist` (по умолчанию);
   - Environment Variables: не требуются.
3. Нажать **Deploy**. Ожидание: 1–2 минуты.

**Шаг 3. Smoke-test:**

- Открыть `https://<vercel-host>` — должен загрузиться экран входа.
- Открыть `https://<vercel-host>/health` — должен вернуть тот же JSON, что Railway (rewrite работает).

### 5.3 Согласование доменов

Если Vercel выдал хост, отличный от указанного в Railway-переменных, обновить в Railway:

| Переменная | Новое значение |
|---|---|
| `CORS_ORIGINS` | `["https://<реальный-vercel-host>"]` |
| `AUTH_SUCCESS_REDIRECT` | `https://<реальный-vercel-host>` |
| `MAILRU_REDIRECT_URI` | `https://<реальный-vercel-host>/api/v1/auth/mailru/callback` |
| `GOOGLE_REDIRECT_URI` | `https://<реальный-vercel-host>/api/v1/auth/google/callback` |

Railway автоматически выполнит повторный деплой.

### 5.4 Настройка OAuth redirect URI

- **Mail.ru:** https://account.mail.ru/developer → приложение → добавить Redirect URI = `https://<vercel-host>/api/v1/auth/mailru/callback`.
- **Google:** Cloud Console → OAuth 2.0 Client ID → добавить Authorized Redirect URI = `https://<vercel-host>/api/v1/auth/google/callback`.

### 5.5 Финальная проверка

1. Открыть `https://<vercel-host>` в режиме инкогнито.
2. Войти по резервному root: email и пароль из `ROOT_EMAIL` / `ROOT_PASSWORD`.
3. Убедиться, что отображается сайдбар с партнёрами и проектами (seed-данные).
4. Войти через Mail.ru (если OAuth настроен). Флоу должен завершиться без ошибки «state mismatch».
5. Открыть задачу по UUID в адресной строке — SPA-fallback работает.
6. Создать комментарий, затем выполнить `git push` любой правки — после передеплоя бэкенда данные должны сохраниться (проверка Volume).

---

## 6. Установка — Вариант Б: собственный сервер ЦИИ НГУ

### 6.1 Использование готового bootstrap-скрипта

В составе репозитория поставляется автоматический установочный скрипт [`deploy/nsu-server/setup.sh`](deploy/nsu-server/setup.sh) и сопутствующие конфигурационные файлы:

| Файл | Назначение |
|---|---|
| `setup.sh` | Главный bootstrap-скрипт (11 шагов, идемпотентен — повторный запуск безопасен) |
| `leyka-api.service` | systemd unit для FastAPI/uvicorn (с настройками безопасности `NoNewPrivileges`, `ProtectSystem=strict`) |
| `nginx-leyka.conf` | Шаблон nginx-сайта с TLS, SPA-fallback и reverse-proxy на uvicorn |
| `leyka-backup.sh` | Скрипт резервного копирования через `sqlite3 .backup` + сжатие + rclone |
| `leyka-backup.timer` | systemd timer (каждые 6 часов; в 03:00 — отгрузка на удалённое хранилище) |

**Минимальный сценарий** (свежеустановленная Ubuntu Server 24.04 LTS или Debian 12, доступ под `root` или `sudo`):

```bash
# 1. Клонировать репозиторий во временный каталог
git clone https://github.com/Barbashin1970/LEYKA.git /tmp/leyka
cd /tmp/leyka/deploy/nsu-server

# 2. Запустить установку (интерактивный режим — спросит домен и email)
sudo bash setup.sh

# Альтернатива: автоматический режим без интерактивных вопросов
sudo bash setup.sh \
    --domain leyka.nsu.ru \
    --email admin@nsu.ru \
    --repo https://github.com/Barbashin1970/LEYKA.git \
    --branch main
```

**Дополнительные опции скрипта:**

| Флаг | Назначение |
|---|---|
| `--tunnel` | Установить `cloudflared` для Cloudflare Tunnel вместо открытия портов 80/443 (если нет внешнего IP в сети ЦИИ НГУ) |
| `--skip-tls` | Пропустить шаг получения сертификата Let's Encrypt (для dev-окружений) |
| `--no-frontend` | Не собирать front-end на этом сервере (актуально, если фронт остаётся на Vercel) |

### 6.2 Что делает скрипт

В порядке выполнения (~10 минут на mini-PC с гигабитным интернетом):

1. **Установка пакетов** — `python3.12`, `nginx`, `sqlite3`, `git`, `ufw`, `fail2ban`, `rclone`, `unattended-upgrades`, `certbot` (если TLS не пропущен), `cloudflared` (если выбран `--tunnel`), `nodejs 20` (если собирается фронт).
2. **Создание системного пользователя `leyka`** — без shell-доступа (`/usr/sbin/nologin`).
3. **Клонирование репозитория в `/opt/leyka`** и `chown leyka:leyka`.
4. **Создание Python venv в `/opt/leyka/.venv`** и установка зависимостей из `backend/requirements.txt`.
5. **Подготовка каталогов**: `/var/lib/leyka/` (БД), `/var/backups/leyka/` (бэкапы), `/var/log/leyka/`.
6. **Генерация `.env`** — случайные `JWT_SECRET` (через `openssl rand -hex 32`) и `ROOT_PASSWORD` (через `openssl rand -base64 18`); пароль выводится в консоль один раз — обязательно сохранить.
7. **Запуск миграций** `alembic upgrade head` — все 24 миграции применяются на чистой БД.
8. **Установка systemd unit** `leyka-api.service` и его автозапуск.
9. **Сборка front-end** (если не указан `--no-frontend`): `npm ci && npm run build`, копирование `frontend/dist/` в `/var/www/leyka-frontend/dist/`.
10. **Настройка nginx** через шаблон `nginx-leyka.conf`, выпуск сертификата Let's Encrypt через `certbot --nginx`.
11. **Включение бэкап-таймера** `leyka-backup.timer`, настройка `ufw` (открыты только 22/80/443), активация `unattended-upgrades`. В конце — `curl http://127.0.0.1:8000/health` для контроля.

После завершения скрипт выводит сводку: ссылку на промышленный экземпляр, адрес Swagger UI, ключевые команды для админа (`journalctl -u leyka-api -f`, `systemctl list-timers leyka-backup`, путь к `.env` и `leyka.db`).

### 6.3 Настройка переменных окружения после установки

Сгенерированный скриптом `/opt/leyka/backend/.env` содержит безопасные дефолты. Для подключения OAuth-провайдеров (см. §4.1, §4.2) необходимо отредактировать файл:

```bash
sudo nano /opt/leyka/backend/.env
```

И добавить значения:

```env
MAILRU_CLIENT_ID=<из п.4.1>
MAILRU_CLIENT_SECRET=<из п.4.1>
MAILRU_REDIRECT_URI=https://<домен-сервера>/api/v1/auth/mailru/callback

# Опционально — Google
GOOGLE_CLIENT_ID=<из п.4.2>
GOOGLE_CLIENT_SECRET=<из п.4.2>
GOOGLE_REDIRECT_URI=https://<домен-сервера>/api/v1/auth/google/callback

# Опционально — Google Drive (drag-and-drop в чек-лист)
GOOGLE_DRIVE_REFRESH_TOKEN=<из п.4.4>
GOOGLE_DRIVE_ROOT_FOLDER_ID=<из п.4.4>
```

После правки `.env` перезапустить сервис:

```bash
sudo systemctl restart leyka-api
sudo systemctl status leyka-api
```

### 6.4 Резервное копирование (включается автоматически)

Скрипт активирует systemd-таймер `leyka-backup.timer`, который:

- каждые 6 часов выполняет `sqlite3 leyka.db ".backup …"` (атомарный snapshot, безопаснее `cp`) с последующим сжатием gzip;
- хранит локально 7 ежедневных копий + 4 еженедельных (rolling);
- в 03:00 ежесуточно загружает свежий snapshot в удалённое хранилище через `rclone` (Я.Диск, S3-совместимое, NAS).

**Настройка удалённого хранилища** выполняется один раз после установки:

```bash
sudo -u root rclone config
# Выбрать тип: yandex / s3 / sftp ...
# Пройти OAuth-флоу провайдера, назвать remote, например, 'yadisk'
```

Подробности — в [SKILL-INFRA.md §3.5](SKILL-INFRA.md) (бэкап-стратегия и план восстановления).

### 6.5 Альтернатива: ручная установка (для специальных случаев)

Если по требованиям ИТ-службы ЦИИ НГУ автоматический скрипт не подходит (например, предписан определённый системный пользователь, нестандартное расположение `/data`, корпоративный SSL-сертификат вместо Let's Encrypt) — детальная пошаговая инструкция приведена в [SKILL-INFRA.md §3.1–§3.5](SKILL-INFRA.md). Все шаги, выполняемые автоматическим скриптом, там разобраны отдельно с пояснением, почему именно так.

### 6.6 Проверка после установки

```bash
# Проверить статус сервиса
sudo systemctl status leyka-api.service

# Проверить health-endpoint (с самого сервера)
curl http://127.0.0.1:8000/health
# Ожидаемый ответ: {"status":"ok","service":"leyka-api","version":"0.2.0"}

# Проверить health через nginx (внешний доступ)
curl https://<домен-сервера>/health

# Логи бэкенда в реальном времени
sudo journalctl -u leyka-api.service -f

# Когда следующий бэкап
sudo systemctl list-timers leyka-backup
```

---

### 6.7 Перенос данных с Railway на сервер ЦИИ НГУ

Сценарий: проект работает на Railway, принято решение перенести его в инфраструктуру ЦИИ НГУ для импортозамещения / аудиторских требований / снижения зависимости от зарубежных сервисов. Все накопленные данные нужно сохранить и перенести без потерь.

В программе предусмотрен встроенный механизм экспорта/импорта резервной копии через REST API (эндпоинты `GET /api/v1/backup/export` и `POST /api/v1/backup/import`). Подробное описание для пользователя — в `LEYKA_UserManual_GOST19.md` §3.13. Здесь приводится процедура, ориентированная на администратора.

**Шаг 1. Скачать резервную копию с Railway-инсталляции**

Открыть текущую инсталляцию (например, `https://leyka-nsu.vercel.app`), войти как администратор центра. В правом верхнем углу нажать на чип своего имени → выбрать пункт **«Резервная копия»** (иконка БД) → в секции «Скачать резервную копию» нажать **«Скачать сейчас»**.

Браузер сохранит файл `leyka-backup-YYYY-MM-DD-HHMM.zip` (~1–10 МБ в зависимости от объёма данных).

**Содержимое архива:**
- `leyka.db` — атомарный снимок SQLite-файла, созданный через нативный backup-API SQLite (Connection.backup). Не блокирует живые транзакции.
- `manifest.json` — метаданные снимка: версия формата, версия приложения, дата создания (UTC), количество записей по основным таблицам.

**Шаг 2. Развернуть LEYKA на сервере ЦИИ НГУ**

Выполнить установку по разделу 6 настоящего руководства (`./deploy/nsu-server/setup.sh`). Дождаться запуска сервиса:

```bash
sudo systemctl status leyka-api.service
# active (running)
curl https://<домен-сервера>/health
# {"status":"ok","service":"leyka-api","version":"0.2.0"}
```

На этом этапе на сервере работает чистая инсталляция LEYKA с пустой базой (только семянные роли и каталог KPI).

**Шаг 3. Подготовить административный вход для процедуры восстановления**

Чтобы использовать веб-интерфейс восстановления, нужно войти как администратор. На свежей инсталляции уже есть root-админ — создаётся скриптом `scripts/seed.py` (запускается автоматически в составе `setup.sh`) на основе переменной `ROOT_EMAIL` из `.env`. Однако при первом запуске пароль не выставлен — admin может войти только через OAuth.

Самый быстрый путь: установить временный пароль root-админу и войти по email + паролю.

```bash
# Отредактировать /etc/leyka/.env и временно прописать ROOT_PASSWORD:
sudo nano /etc/leyka/.env
# ROOT_EMAIL=admin@nsu.ru
# ROOT_PASSWORD=<временный_пароль>

# Применить пароль к существующему root-админу
cd /opt/leyka
sudo -u leyka /opt/leyka/.venv/bin/python -m scripts.set_root_password
```

Скрипт идемпотентен — повторный запуск с тем же паролем не делает ничего.

**Важно:** этот пароль действует только до восстановления. После импорта (шаг 4) учётные записи будут перезаписаны теми, что были в бэкапе с Railway. Войдёте под прежним email и прежним паролем (как было на Railway).

После переноса не забудьте удалить `ROOT_PASSWORD` из `.env` (или оставить, но сменить на другой) — это страховка на случай повторного восстановления.

**Шаг 4. Восстановить из резервной копии через веб-интерфейс**

1. Открыть `https://<домен-сервера>` в браузере, войти под учётной записью администратора, созданной на шаге 3.
2. Чип имени в правом верхнем углу → **«Резервная копия»** → секция **«Восстановить из файла»**.
3. Нажать **«Выбрать ZIP-файл»** → указать `leyka-backup-YYYY-MM-DD-HHMM.zip`, скачанный на шаге 1.
4. Нажать **«Восстановить»** → подтвердить операцию в всплывающем окне.
5. Дождаться сообщения «Восстановление выполнено». В блоке-результате будут показаны:
   - содержимое manifest нового бэкапа (количество записей по таблицам — для сверки с тем, что было на Railway);
   - имя файла-страховки старой БД: `leyka.db.before-restore`;
   - предупреждение о необходимости перезапустить сервис.

**Шаг 5. Перезапустить серверный сервис**

```bash
sudo systemctl restart leyka-api.service
sudo systemctl status leyka-api.service
# active (running) после перезапуска
```

Перезапуск необходим, потому что SQLAlchemy кэширует connection pool, и новые HTTP-запросы должны открыть свежие соединения с восстановленным файлом БД.

**Шаг 6. Проверка переноса**

1. Очистить cookie сайта в браузере (важно, см. примечание ниже).
2. Войти заново под прежним email и паролем (как было на Railway).
3. Проверить, что в сайдбаре появились знакомые партнёры и проекты, в задачах — комментарии, в библиотеке знаний — материалы, в KPI — ранее зафиксированные показатели.
4. Открыть несколько задач и убедиться, что вложения (URL на Google Docs и пр.) продолжают работать.

> **Примечание про cookies.** При переезде HttpOnly-cookies сессий у пользователей перестанут работать: они подписаны секретом `JWT_SECRET`, который у новой инсталляции другой (на шаге 4.3 был сгенерирован свежий через `openssl rand -hex 32`). Это штатное поведение и часть безопасности переезда. Все пользователи должны будут пройти аутентификацию заново — это разумно сообщить им заранее.

**Откат от отката (если что-то пошло не так)**

Если после восстановления данные оказались некорректными, был выбран не тот архив, или администратор не может зайти в систему (например, в восстановленной БД нет его учётной записи) — есть три способа откатиться к состоянию «до восстановления». Выбор зависит от того, есть ли у тебя shell-доступ к серверу.

---

**Способ 1 (рекомендуемый, не требует shell-доступа): через Swagger UI бэкенда**

Подходит, если ты залочен в самой Лейке (не можешь войти после восстановления), но имеешь доступ к панели управления хостингом (Railway dashboard, переменные окружения сервера НГУ).

1. Открой панель управления хостингом → Variables / .env:
   - **Railway:** Project → Settings → Variables;
   - **Сервер ЦИИ НГУ:** `sudo nano /etc/leyka/.env`.
2. Добавь временную переменную с длинной случайной строкой:
   ```
   BACKUP_ROLLBACK_TOKEN=$(openssl rand -hex 32)   # сгенерируй и подставь реальное значение
   ```
3. Дождись redeploy (Railway — автоматически за ~30 сек; сервер НГУ — `sudo systemctl restart leyka-api`).
4. Открой Swagger UI бэкенда:
   - **Railway:** `https://leyka-production.up.railway.app/docs`;
   - **Сервер НГУ:** `https://<домен-сервера>/docs`.
5. Найди раздел **Backup** → **POST `/backup/rollback`** → нажми **«Try it out»** → введи свой токен в поле `token` → **Execute**.
6. Сервер ответит JSON с `rolled_back: true`, именами файлов и предупреждением «перезапусти сервис».
7. Перезапусти сервис (Railway Restart или `sudo systemctl restart leyka-api`).
8. Зайди в Лейку — данные на месте.
9. **Удали `BACKUP_ROLLBACK_TOKEN` из переменных** (защита от повторного использования). После удаления endpoint снова вернёт 403 на любой запрос.

> Endpoint `POST /backup/rollback` — единственный, который работает БЕЗ авторизации. Защита — секретный токен из env-переменной. Если переменная пустая (default) — endpoint возвращает 403 на любые запросы. Это сделано специально для аварии «не могу залогиниться»: восстановиться через UI Лейки в этом случае невозможно, и приходится использовать обходной канал.

---

**Способ 2 (через CLI на сервере): `python -m scripts.restore_rollback`**

Подходит, если есть shell-доступ к серверу и не хочется ставить env-переменную.

```bash
# Сервер ЦИИ НГУ:
cd /opt/leyka
sudo -u leyka /opt/leyka/.venv/bin/python -m scripts.restore_rollback
# Скрипт спросит подтверждение и выведет, что произойдёт.
# Опционально: --yes для скриптов, --db /var/lib/leyka/leyka.db для нестандартного пути.

# Railway (если установлен railway CLI):
railway run python -m scripts.restore_rollback --yes
```

После — перезапустить сервис.

---

**Способ 3 (вручную, переименование файлов): для случаев, когда Python недоступен**

```bash
# 1. Остановить сервис
sudo systemctl stop leyka-api.service

# 2. В каталоге БД (по умолчанию /var/lib/leyka/) лежат два файла:
#    leyka.db                   — текущая (неудачно восстановленная) БД
#    leyka.db.before-restore    — копия БД, которая была ДО восстановления
ls -la /var/lib/leyka/

# 3. Сохранить плохую БД на всякий случай и откатиться
sudo mv /var/lib/leyka/leyka.db /var/lib/leyka/leyka.db.failed
sudo mv /var/lib/leyka/leyka.db.before-restore /var/lib/leyka/leyka.db
sudo chown leyka:leyka /var/lib/leyka/leyka.db
# Удалить хвосты WAL/SHM от плохой БД, если есть
sudo rm -f /var/lib/leyka/leyka.db-wal /var/lib/leyka/leyka.db-shm

# 4. Запустить сервис обратно
sudo systemctl start leyka-api.service
```

Через минуту система работает в состоянии, которое было до неудачного восстановления. Никакие данные не потеряны.

> **Где лежит файл-страховка:**
> - **На сервере ЦИИ НГУ** — `/var/lib/leyka/leyka.db.before-restore` (рядом с основным `leyka.db`).
> - **На Railway** — `/data/leyka.db.before-restore` (рядом с основным `/data/leyka.db`, на персистентном томе).
> - **Локально (dev)** — `./backend/leyka.db.before-restore`.
>
> Файл-страховка перезаписывается при каждом следующем восстановлении (хранится только последняя версия). Для длительного хранения исторических снимков используется отдельный механизм — systemd-таймер `leyka-backup.timer` (см. §6.4), который ведёт rolling-набор «7 дневных + 4 еженедельных».

---

## 7. Запуск локально (для разработки)

```bash
# Клонировать репозиторий
git clone https://github.com/Barbashin1970/LEYKA.git
cd LEYKA

# Запустить одним скриптом (macOS/Linux):
./leyka.command              # стандартный запуск
./leyka.command --reset-db   # снести и залить демо-данные
./leyka.command --no-seed    # без seed (если БД уже наполнена)
./leyka.command --no-open    # без авто-открытия браузера

# После запуска:
# Фронтенд: http://localhost:5173
# Бэкенд API: http://localhost:8000
# Swagger UI: http://localhost:8000/docs
```

Войти: email из `.env` (`ROOT_EMAIL`) и пароль из `.env` (`ROOT_PASSWORD`).

---

## 8. Проверка корректности установки

После завершения развёртывания выполнить следующие проверки:

| № | Проверка | Ожидаемый результат |
|---|---|---|
| 1 | GET `https://<хост>/health` | `{"status":"ok","service":"leyka-api","version":"0.2.0"}` |
| 2 | Открыть `https://<хост>` в браузере | Загружается экран входа |
| 3 | Войти по ROOT_EMAIL / ROOT_PASSWORD | Открывается главный экран с сайдбаром |
| 4 | Войти через OAuth Mail.ru | Завершается без ошибки «state mismatch» |
| 5 | Открыть `https://<хост>/docs` | Загружается Swagger UI с 73 эндпойнтами, сгруппированными по тегам |
| 6 | Создать задачу → обновить страницу | Задача сохранена (Volume работает) |
| 7 | `git push` любой правки → задача после передеплоя | Данные не потеряны |

---

## 9. Возможные неисправности и методы их устранения

| Неисправность | Причина | Метод устранения |
|---|---|---|
| Railway: `RuntimeError: Небезопасная конфигурация` | `IS_PRODUCTION=true` + некорректные env | В сообщении перечислено что не так; исправить и передеплоить |
| Vercel: `404 Not Found` при GET `/api/...` | В `vercel.json` остался `REPLACE_WITH_RAILWAY_HOST` | Подставить реальный Railway-хост, запушить |
| `OAuth state mismatch` | Cookie state не сохранился между запросами | Проверить `AUTH_COOKIE_SECURE=true`; фронт должен ходить по HTTPS |
| Vercel не подхватывает правки `vercel.json` | Кеширование | Settings → Deployments → Redeploy → без кеша |
| Railway: нет публичного домена | Не создан сетевой endpoint | Settings → Networking → Generate Domain |
| Backend упал сразу после старта | JWT_SECRET слабый / ENABLE_MOCK_AUTH=true | Исправить env, передеплоить |
| `503 — загрузка временно недоступна` при drag-and-drop | Google Drive не настроен | Пройти `GOOGLE_DRIVE_SETUP.md` |
| Данные пропали после передеплоя (Railway) | Volume не подключён или Database_URL не на `/data` | Проверить Volume mount path `/data`; `DATABASE_URL` должен содержать `////data/leyka.db` |
| Nginx: `502 Bad Gateway` | Бэкенд не запущен или слушает другой порт | `sudo systemctl status leyka`; проверить порт 8000 |

---

## 10. Приложения

### Приложение А. Полный шаблон .env для Railway

```env
# === База данных ===
DATABASE_URL=sqlite+aiosqlite:////data/leyka.db
DB_ECHO=false

# === Прод-маркер ===
IS_PRODUCTION=true

# === Auth / JWT ===
JWT_SECRET=СЮДА_ВСТАВИТЬ_64_HEX_СИМВОЛА
JWT_EXPIRE_HOURS=168
AUTH_COOKIE_NAME=leyka_session
AUTH_COOKIE_SECURE=true
AUTH_COOKIE_SAMESITE=lax

# === Резервный root-вход ===
ROOT_EMAIL=admin@example.com
ROOT_PASSWORD=НАДЁЖНЫЙ_ПАРОЛЬ

# === Mock-провайдер: ОТКЛЮЧЁН на проде ===
ENABLE_MOCK_AUTH=false

# === Vercel-домен ===
CORS_ORIGINS=["https://<vercel-host>"]
AUTH_SUCCESS_REDIRECT=https://<vercel-host>

# === OAuth Mail.ru ===
MAILRU_CLIENT_ID=
MAILRU_CLIENT_SECRET=
MAILRU_REDIRECT_URI=https://<vercel-host>/api/v1/auth/mailru/callback

# === OAuth Google (опционально) ===
# GOOGLE_CLIENT_ID=
# GOOGLE_CLIENT_SECRET=
# GOOGLE_REDIRECT_URI=https://<vercel-host>/api/v1/auth/google/callback

# === Google Drive для drag-and-drop (опционально) ===
# GOOGLE_DRIVE_CLIENT_ID=
# GOOGLE_DRIVE_CLIENT_SECRET=
# GOOGLE_DRIVE_REFRESH_TOKEN=
# GOOGLE_DRIVE_FOLDER_ID=
```

---

### Приложение Б. Чек-лист готовности к боевому использованию

| # | Пункт | Статус |
|---|---|---|
| 1 | Railway: Volume `/data` смонтирован | ☐ |
| 2 | Railway: `DATABASE_URL` ссылается на `/data/leyka.db` | ☐ |
| 3 | Railway: `IS_PRODUCTION=true` | ☐ |
| 4 | Railway: `JWT_SECRET` — 64 hex-символа, без слова «dev» | ☐ |
| 5 | Railway: `AUTH_COOKIE_SECURE=true` | ☐ |
| 6 | Railway: `ENABLE_MOCK_AUTH=false` | ☐ |
| 7 | Railway: публичный домен создан, `/health` отвечает | ☐ |
| 8 | Vercel: `vercel.json` указывает на реальный Railway-хост | ☐ |
| 9 | Vercel: Root Directory выставлен в `frontend` | ☐ |
| 10 | Mail.ru Client Secret ротирован; новый только в Railway | ☐ |
| 11 | OAuth redirect URI прописан у всех провайдеров | ☐ |
| 12 | Резервный root (`ROOT_EMAIL`) успешно логинится | ☐ |
| 13 | Данные после передеплоя не пропали (Volume проверен) | ☐ |

---

*Документ составлен на основе DEPLOY.md, SKILL-RELEASE.md, SKILL-INFRA.md и скриптов deploy/nsu-server/ репозитория https://github.com/Barbashin1970/LEYKA версии 0.2.0 по состоянию на 27 апреля 2026 г.*  
*Инструкция по установке соответствует требованиям ГОСТ 19.502-78 «Описание применения. Требования к содержанию и оформлению» (в части установки и развёртывания).*

*При обновлении установочных скриптов (`deploy/nsu-server/setup.sh`, `leyka-api.service`, `nginx-leyka.conf`) настоящая инструкция должна синхронизироваться с актуальной версией скриптов одновременно с выпуском новой версии программы.*

---

**Конец документа ЦИИ НГУ.LEYKA.ИУ.001-2026**
