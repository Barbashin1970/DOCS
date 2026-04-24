# ARC.md — Архитектура системы LEYKA

> **L**EYKA · *Lightweight Engineering Yard for Knowledge Activities*
> Корпоративный task-tracker для **ЦИИ НГУ** (Центр искусственного интеллекта НГУ).
> Отслеживает проекты с заказчиками/грантодателями, обязательства центра по
> 11 KPI Минобрнауки и реальную работу команд по канбану.
>
> **Версия документа:** 1.0 · **Дата:** 2026-04-24
> **Статус:** active · **Прод:** [leyka-nsu.vercel.app](https://leyka-nsu.vercel.app)

[![Backend tests](https://img.shields.io/badge/backend%20tests-132%20passing-brightgreen)]()
[![Frontend typecheck](https://img.shields.io/badge/frontend-typecheck%20clean-brightgreen)]()
[![Migrations](https://img.shields.io/badge/migrations-16-blue)]()
[![Bundle](https://img.shields.io/badge/bundle-~250KB%20gzip-blue)]()

---

## 📑 Оглавление

1. [Обзор системы](#1-обзор-системы)
2. [Слоистая архитектура](#2-слоистая-архитектура)
3. [Доменная модель](#3-доменная-модель)
4. [Жизненный цикл проекта](#4-жизненный-цикл-проекта)
5. [RBAC: два уровня ролей + visibility](#5-rbac-два-уровня-ролей--visibility)
6. [Аутентификация и авторизация](#6-аутентификация-и-авторизация)
7. [Цепочка приглашений](#7-цепочка-приглашений)
8. [KPI-подсистема](#8-kpi-подсистема)
9. [API-контракт](#9-api-контракт)
10. [Деплой и инфраструктура](#10-деплой-и-инфраструктура)
11. [Безопасность](#11-безопасность)
12. [Метрики проекта](#12-метрики-проекта)
13. [Тестовая стратегия](#13-тестовая-стратегия)
14. [Принятые архитектурные решения (ADR)](#14-принятые-архитектурные-решения-adr)
15. [Дальнейшее развитие](#15-дальнейшее-развитие)

---

## 1. Обзор системы

### 1.1 High-level диаграмма

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         🌐 КЛИЕНТ (SPA, Vercel)                           │
│   React 19 · TypeScript strict · TanStack Query · Zustand · Tailwind     │
│   leyka-nsu.vercel.app                                                    │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │ HTTPS · Cookie-сессия (HttpOnly, SameSite=Lax)
                                  │ Same-origin через Vercel rewrite /api/* → Railway
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    🛡 ПРОКСИ-СЛОЙ (Vercel Edge, rewrite)                  │
│   /api/v1/*   →  https://leyka-production.up.railway.app/api/v1/*        │
│   /health     →  …/health                                                 │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     ⚙ BACKEND (FastAPI, Railway)                          │
│   Python 3.12 · FastAPI 0.115 · SQLAlchemy 2 (async) · Alembic           │
│   Uvicorn · CORS · Pydantic v2 · python-jose (JWT) · passlib (bcrypt)    │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
                  ┌───────────────┼───────────────┐
                  ▼               ▼               ▼
        ┌─────────────────┐ ┌──────────┐ ┌──────────────────┐
        │ 💾 SQLite       │ │ 🔐 OAuth │ │ 📨 SMTP (план,   │
        │ /data/leyka.db  │ │ Mail.ru  │ │    Phase 6)      │
        │ Railway Volume  │ │ Google   │ │                  │
        └─────────────────┘ └──────────┘ └──────────────────┘
```

### 1.2 Ключевые свойства

| Свойство            | Значение                                         | Обоснование                                                |
|---------------------|--------------------------------------------------|------------------------------------------------------------|
| **Async-only**      | SQLAlchemy AsyncSession, FastAPI async endpoints | Веб-IO bound, единый стиль                                 |
| **Same-origin**     | Vercel rewrite вместо CORS preflight             | Cookie-auth работает без `withCredentials` плясок          |
| **HttpOnly-cookie** | JWT в `leyka_session`, не localStorage           | Защита от XSS-кражи токена                                 |
| **Type-safe API**   | OpenAPI → openapi-typescript → `types.gen.ts`    | Изменение схемы ломает фронт-билд → рано видно             |
| **SQLite на проде** | Persistent volume на Railway                     | <50 юзеров, не нужен Postgres; миграция → Postgres готова  |
| **Stateless API**   | Никаких in-memory сессий                         | Можно горизонтально скейлить (когда понадобится)           |

### 1.3 Что система НЕ делает

- ❌ Не хранит файлы — только URL (Yandex.Disk, Google Drive, GitHub).
- ❌ Не отправляет email (Phase 6).
- ❌ Не имеет mobile-приложения (PWA-обёртка — на потом).
- ❌ Не интегрируется с 1С/НГУ-учёткой (только OAuth).
- ❌ Не делает нормальный realtime — обновление через TanStack Query polling.

---

## 2. Слоистая архитектура

### 2.1 Backend (Python · FastAPI · SQLAlchemy 2)

```
┌──────────────────────────────────────────────────────────────────┐
│  app/main.py                                                      │
│  Создание FastAPI · CORS · request-id middleware · 500-handler   │
│  Production safety asserts (JWT_SECRET, IS_PRODUCTION флаги)     │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  🌐 API ROUTERS (app/api/v1/*.py) · 11 модулей                    │
│                                                                   │
│  auth.py            · /auth/{providers,me,login,logout,callback}  │
│  users.py           · /users + /users/{id} + admin actions        │
│  partners.py        · /partners (CRUD + presale-project helper)   │
│  projects.py        · /projects (+ /attachments, /analytics)      │
│  project_members.py · /projects/{id}/members                      │
│  roles.py           · /roles (RBAC matrix)                        │
│  tasks.py           · /tasks (+ status history, mark-read, lookup)│
│  task_attachments.py· /tasks/{id}/attachments                     │
│  comments.py        · /comments                                   │
│  invites.py         · /invites + /projects/{id}/invites           │
│  kpi.py             · /kpi/* + /projects/{id}/kpi/*               │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  📦 SCHEMAS (app/schemas/*.py · Pydantic v2)                      │
│  Create/Update/Response triples для каждой сущности +             │
│  специальные: analytics (lead-time, throughput), invite (preview),│
│  auth (providers, me).                                            │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  🗄 MODELS (app/models/*.py · SQLAlchemy 2 declarative)           │
│  base.py · GUID type + Base + TimestampMixin                      │
│  enums.py· UserRole · TaskStatus · ProjectStage · Visibility ...  │
│  ─────────────────────────────────────────────────────────────    │
│  user · partner · project · project_member · role · project_invite│
│  task · task_attachment · task_status_history                     │
│  comment · comment_read_receipt                                   │
│  kpi (Catalog · CenterTarget · ProjectCommitment · Achievement)   │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  🛡 DEPENDENCIES (app/api/dependencies.py)                        │
│  current_user_optional · current_user · ensure_project_access     │
│  require_project_member · require_permission(perm)                │
└──────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
   ┌─────────────────┐ ┌────────────┐ ┌───────────────────┐
   │ 🔐 AUTH         │ │ 💾 DATABASE│ │ ⚙ CONFIG          │
   │ providers/      │ │            │ │ pydantic-settings │
   │  · base         │ │ engine +   │ │  · DATABASE_URL   │
   │  · mailru       │ │ async_     │ │  · JWT_SECRET     │
   │  · google       │ │ session_   │ │  · OAuth creds    │
   │  · mock (dev)   │ │ maker      │ │  · ROOT_*         │
   │ security.py     │ │            │ │  · IS_PRODUCTION  │
   │ (JWT + bcrypt)  │ └────────────┘ └───────────────────┘
   └─────────────────┘
```

**Правило слоёв:** стрелки зависимостей идут только **вниз**. API → Schemas → Models → Database. Любая бизнес-логика, которая нужна в нескольких роутерах, выносится в `dependencies.py` или helper-модуль.

### 2.2 Frontend (React 19 · TypeScript · TanStack Query · Zustand)

```
┌────────────────────────────────────────────────────────────────────┐
│  src/App.tsx                                                        │
│  Auth-guard · /invite/:token routing · 5 view modes:                │
│  inbox · kpi-center · roles · users-admin · project (5 tabs)        │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│  🖼 SCREENS / VIEWS                                                  │
│  InboxView · CenterKpiView · RoleMatrixView · UsersAdminPanel       │
│  Project tabs:  Board · MembersTab · ProjectDocs · ProjectKpiTab    │
│                 · ProjectAnalyticsView                              │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│  🧩 COMPONENTS (по доменам)                                          │
│                                                                     │
│  Admin/    · UsersAdminPanel (table + edit-modal + delete-confirm)  │
│  Inbox/    · InboxView (фильтры «Для меня» / «Непрочитанные» / All) │
│  Kanban/   · Board · Column · TaskCard (dnd-kit drag&drop)          │
│  Login/    · LoginScreen · InvitePage                               │
│  Members/  · MembersTab · UserPickerButton · InviteByLinkDialog     │
│            · InvitesList                                            │
│  TaskDrawer/ · TaskDrawer (header + Meta + ActivityFeed) · ...      │
│                                                                     │
│  Standalone: PartnerCardDialog · ProjectSettingsDialog ·            │
│              CreateProjectDialog · CreatePartnerDialog ·            │
│              UserAvatar · UserProfileMenu · DeepLinkTaskOverlay     │
│              MentionText · MentionTextarea · TaskChip · StageBadge  │
│              PriorityChip · StatusPill · VisibilityToggle ·         │
│              GuestEmptyState · LeykaLoader · DangerZone             │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│  🔌 API LAYER (src/api/)                                             │
│  client.ts · 90+ wrapped fetch-функций (api.partners, api.tasks,    │
│              api.invites, api.kpi, api.users …)                     │
│  types.gen.ts · 1 700 строк, генерится из FastAPI OpenAPI           │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│  🗃 STORES (src/store/)                                              │
│  auth.ts          · useAuthMe (TanStack Query hook), useLogout      │
│  currentUser.ts   · текущий userId (для пометки автора комментов)   │
│  chatWallpaper.ts · персистентные настройки чата (Zustand+persist)  │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│  🛠 UTILS (src/lib/)                                                 │
│  cn · projectPermissions · taskColors · taskRef · linkKind          │
│  position (drag-drop fractional indexing) · userColors · userRole   │
│  completionCriteria · attachmentStages · kpiLabels · chatWallpapers │
└────────────────────────────────────────────────────────────────────┘
```

**Правило компонентов:** компонент знает только про свой проп-API. Глобальное состояние — через TanStack Query (server state) и Zustand (client UI prefs). Никаких prop-drilling больше двух уровней.

---

## 3. Доменная модель

### 3.1 Сущности и их связи

```
                                   ┌──────────────┐
                                   │   Partner    │
                                   │ (заказчик /  │
                                   │  грантодатель│
                                   └──────┬───────┘
                                          │ 1:N
                          ┌───────────────┴──────────────┐
                          ▼                              ▼
                   ┌──────────────┐              ┌──────────────┐
                   │   Project    │              │   Project    │
                   │  visibility: │              │              │
                   │  private |   │              │              │
                   │  public |    │              │              │
                   │  demo        │              │              │
                   └──────┬───────┘              └──────────────┘
                          │
        ┌─────────────────┼──────────────────┬─────────────┐
        │ 1:N             │ 1:N              │ 1:N         │ 1:N
        ▼                 ▼                  ▼             ▼
   ┌─────────┐     ┌──────────────┐  ┌──────────────┐  ┌──────────┐
   │  Task   │     │ ProjectMember│  │ProjectInvite │  │ Project  │
   │ status: │     │ user_id +    │  │ token, role, │  │KpiCommit │
   │ inbox.. │     │ role_id      │  │ inviter,     │  │ ment     │
   │ done    │     │ (M:N user-   │  │ greeting,    │  │          │
   └────┬────┘     │  project)    │  │ expires      │  └────┬─────┘
        │          └──────┬───────┘  └──────┬───────┘       │
        │ 1:N             │ N:1             │ N:1           │
        ▼                 ▼                 ▼               ▼
   ┌─────────┐     ┌──────────────┐  ┌──────────────┐  ┌──────────┐
   │ Comment │     │     User     │  │     User     │  │KpiAchieve│
   │ author  │◄────│ full_name,   │  │   (inviter)  │  │ ment     │
   │ +read_by│ N:1 │ email, role, │  └──────────────┘  └──────────┘
   └─────────┘     │ is_superuser │           ▲
                   └──────┬───────┘           │
                          │ N:M (через project_members)
                          └────────────────────┐
                                               │
                                          ┌────▼─────┐
                                          │   Role   │
                                          │ Менеджер │
                                          │Разработчик│
                                          │ Аналитик │
                                          │ Куратор  │
                                          │Наблюдатель│
                                          │ Заказчик │
                                          │ + 12 perm│
                                          │  flags   │
                                          └──────────┘

           Task ───1:N──→ TaskAttachment (URL + stage + title)
           Task ───1:N──→ TaskStatusHistory (old/new + changed_by)
           Comment ──N:M──→ User (через CommentReadReceipt)

           KpiCatalog (z-1..z-11, справочник 11 KPI центра)
                  │
                  ├──1:N──→ KpiCenterTarget (план центра по годам)
                  └──1:N──→ ProjectKpiCommitment (обязательство проекта)
                                  │
                                  └──1:N──→ KpiAchievement (факт)
```

### 3.2 Все таблицы

| Таблица                  | Назначение                                              | Ключевые поля                                                   |
|--------------------------|---------------------------------------------------------|------------------------------------------------------------------|
| `users`                  | Аккаунты (OAuth + root)                                 | email, full_name, role, is_superuser, oauth_provider             |
| `partners`               | Заказчики/грантодатели                                  | name, type, status, **visibility** (private/public)              |
| `projects`               | Инициативы под партнёра                                 | title, stage (presale→active→…), **visibility**, budget          |
| `project_members`        | M:N user×project с ролью в проекте                      | project_id, user_id, role_id (UNIQUE pair)                       |
| `roles`                  | RBAC-роли проекта (12 permission flags)                 | name, can_view_tasks, can_edit_project, can_manage_members …    |
| `project_invites`        | Ссылки-приглашения                                      | token, role_id, inviter_id, expires_at, max_uses, used_count    |
| `tasks`                  | Карточки канбана                                        | title, status, priority, assignee_id, completion_criterion       |
| `task_status_history`    | Аудит переходов статусов                                | task_id, old_status, new_status, changed_by_id, changed_at       |
| `task_attachments`       | URL-ссылки (Я.Диск, Docs, GitHub …)                     | task_id, url, title, kind, stage                                 |
| `comments`               | Чат внутри задачи                                       | task_id, author_id, text, edited_at                              |
| `comment_read_receipts`  | M:N comment×user — кто прочитал                         | comment_id, user_id, read_at                                     |
| `kpi_catalog`            | Справочник 11 KPI Минобрнауки                           | code (z-1..z-11), title, unit, sort_order                        |
| `kpi_center_targets`     | План центра на год по каждой KPI                        | catalog_id, year, target_value                                   |
| `project_kpi_commitments`| Обязательство проекта                                   | project_id, catalog_id, target_value, due_date                   |
| `kpi_achievements`       | Факты выполнения (Роспатент, статья, акт)               | commitment_id, value, title, evidence_url, occurred_at           |

### 3.3 Enum'ы

| Enum                  | Значения                                                                          |
|-----------------------|-----------------------------------------------------------------------------------|
| `UserRole`            | `pm` · `developer` · `analyst` · `designer` · `curator` · `observer` · `customer` · `guest` |
| `TaskStatus`          | `inbox` · `todo` · `in_progress` · `waiting` · `review` · `done`                  |
| `TaskPriority`        | `p1` · `p2` · `p3` · `p4`                                                         |
| `ProjectStage`        | `presale` · `active` · `paused` · `completed` · `archived`                        |
| `ProjectVisibility`   | `private` · `public` · `demo`                                                     |
| `PartnerVisibility`   | `private` · `public`                                                              |
| `PartnerType`         | `enterprise` · `government` · `startup` · `research`                              |
| `PartnerStatus`       | `lead` · `negotiation` · `active` · `paused` · `archived`                         |

---

## 4. Жизненный цикл проекта

### 4.1 Стадии (Project.stage)

```
   ┌──────────────────────────────────────────────────────────────────┐
   │  presale     активный обмен ТЗ-набросками; нет команды и бюджета │
   │              ▼ договор подписан                                   │
   │  active      бюджет выделен, команда работает по канбану          │
   │              ▼ временная заморозка (правки, ожидание)             │
   │  paused      работа приостановлена, состояние сохранено           │
   │              ▼ возобновлено                                       │
   │              ▲                                                    │
   │  active                                                           │
   │              ▼ цели достигнуты, отчёт сдан                        │
   │  completed   проект закрыт, сумма потрачена, KPI зафиксированы    │
   │              ▼ устарел, не нужен в дашбордах                      │
   │  archived    скрыт из всех аналитик                               │
   └──────────────────────────────────────────────────────────────────┘
```

### 4.2 Жизнь одной задачи (Task.status)

```
        ┌─────────┐    user добавил ↓
        │  inbox  │ ◄───────────────────── создание (по умолчанию)
        └────┬────┘
             │ взяли в работу
             ▼
        ┌─────────┐                        ┌─────────────┐
        │  todo   │──────────────────────► │ in_progress │
        └─────────┘                        └──────┬──────┘
                          assignee начал ↑        │ ждём чего-то
                                                  ▼
        ┌─────────┐    исполнитель готов    ┌─────────┐
        │ review  │ ◄────────────────────── │ waiting │
        └────┬────┘                          └─────────┘
             │ принято
             ▼
        ┌─────────┐
        │  done   │     ◄── completion_criterion enforce'ится модалкой
        └─────────┘
```

Каждый переход пишется в `task_status_history` (SQLAlchemy event listener в `app/models/task.py`) → используется для аналитики throughput / lead time.

---

## 5. RBAC: два уровня ролей + visibility

> Ключевое решение Phase 1 / 1.5: разделить **глобальную должность** и
> **проектные права**. Списки лейблов синхронизированы, чтобы админ не
> путался, но семантика разная.

### 5.1 Уровень 1 — глобальная должность (`User.role`)

Информационный тег. Задаётся в карточке профиля. На авторизацию **не влияет**.

| Роль        | Кто это                                                              |
|-------------|----------------------------------------------------------------------|
| `pm`        | Менеджер проекта (PM)                                                |
| `developer` | Разработчик/инженер                                                  |
| `analyst`   | Аналитик / data scientist                                            |
| `designer`  | Дизайнер интерфейсов                                                 |
| `curator`   | Куратор (надзорная роль, см. §5.3)                                   |
| `observer`  | Наблюдатель                                                          |
| `customer`  | Внешний контактный (заказчик)                                        |
| `guest`     | Default для свежих OAuth-юзеров до первого вступления в проект       |

### 5.2 Уровень 2 — роль в проекте (`ProjectMember.role` → `Role`)

Реальные permission'ы. У одного юзера могут быть **разные роли в разных проектах**.

| Роль          | view_tasks | edit_tasks | move_status | delete_tasks | manage_members | edit_project | manage_budget | view_confidential |
|---------------|:----------:|:----------:|:-----------:|:------------:|:--------------:|:------------:|:-------------:|:-----------------:|
| Менеджер      | ✅          | ✅          | ✅           | ✅            | ✅              | ✅            | ✅             | ✅                 |
| Разработчик   | ✅          | ✅          | ✅           | ❌            | ❌              | ❌            | ❌             | ❌                 |
| Аналитик      | ✅          | ✅          | ✅           | ❌            | ❌              | ❌            | ❌             | ❌                 |
| Куратор       | ✅          | ❌          | ❌           | ❌            | ❌              | ❌            | ❌             | ✅                 |
| Наблюдатель   | ✅          | ❌          | ❌           | ❌            | ❌              | ❌            | ❌             | ❌                 |
| Заказчик      | ✅          | ❌          | ❌           | ❌            | ❌              | ❌            | ❌             | ❌                 |

> Полная матрица из 12 флагов настраивается админом через UI **«Управление доступом»** ([RoleMatrixView](frontend/src/components/RoleMatrixView.tsx)).

### 5.3 Куратор — надзорная роль

Подключается к проекту чтобы **наблюдать**, не правит. Типовые сценарии:

- **Директор центра** — проверка отчётности.
- **Директор грантодателя** — контроль исполнения по гранту.
- **Главбух НГУ** — сверка устава и расходов.
- **Научный руководитель** — прогресс аспирантов.
- **Внутренний аудитор** — выборочные проверки.

### 5.4 Над всем — `is_superuser`

Флаг администратора центра. **Обходит обе модели.** Защищён от само-разжалования: последний активный admin не может ни снять с себя `is_superuser`, ни задеактивировать ([проверка в `update_user`](backend/app/api/v1/users.py#L131)). Защищён от само-удаления (`DELETE /users/{me}` → 400).

### 5.5 Visibility — третий слой над RBAC

| Сущность   | Значения                  | Кто видит                                                         |
|------------|---------------------------|-------------------------------------------------------------------|
| `Project`  | private (по умолчанию)    | члены проекта + superuser                                          |
|            | public                    | все аккаунты центра                                                |
|            | demo                      | все, включая гостей; безопасен для тренировок                     |
| `Partner`  | private (по умолчанию)    | сотрудники с доступом ≥ к одному проекту партнёра + superuser     |
|            | public                    | все аккаунты центра                                                |

Менять может только superuser — переключатель «🔒 закрытый / 🌍 открытый / ✨ демо» в карточке заказчика и в настройках проекта ([VisibilityToggle.tsx](frontend/src/components/VisibilityToggle.tsx)).

---

## 6. Аутентификация и авторизация

### 6.1 Способы входа

```
┌─────────────────────────────────────────────────────────────────┐
│  1. OAuth Mail.ru   — основной для сотрудников НГУ              │
│  2. OAuth Google    — для коллег с корпоративным Google         │
│  3. Local (root)    — резервный вход (ROOT_EMAIL/ROOT_PASSWORD) │
│  4. Mock (dev only) — выбор юзера из dropdown без реального SSO │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 OAuth-флоу (с CSRF и graceful cancel)

```
   [Frontend]                       [Backend]                       [Provider]
       │                                │                                │
       │ click "Войти через Mail.ru"    │                                │
       ├───────────────────────────────►│                                │
       │ GET /auth/mailru/login         │                                │
       │                                │ generate state, set cookie     │
       │                                │ build authorize URL            │
       │ ◄──── 302 redirect ────────────┤                                │
       │                                                                 │
       │ ─────────────── 302 ──────────────────────────────────────────► │
       │                                                                 │
       │           ┌───────── user grants OR cancels ──────┐             │
       │           ▼                                       ▼             │
       │  redirect ?code=ABC&state=XYZ         redirect ?error=denied    │
       │           │                                       │             │
       │ ──────────┴───────────────────────────────────────┴───────────► │
       │                                │                                │
       │                                │ GET /auth/mailru/callback      │
       │                                │ ┌──────────────────────────┐   │
       │                                │ │ if error || no code:     │   │
       │                                │ │   302 → /?login_cancelled│   │
       │                                │ │ else:                    │   │
       │                                │ │   verify state           │   │
       │                                │ │   exchange code → token  │   │
       │                                │ │   fetch user info        │   │
       │                                │ │   upsert User by         │   │
       │                                │ │   (oauth_provider,       │   │
       │                                │ │    oauth_subject)        │   │
       │                                │ │   issue JWT in cookie    │   │
       │                                │ │   302 → frontend root    │   │
       │                                │ └──────────────────────────┘   │
       │ ◄────────── 302 + Set-Cookie ──┤                                │
       │                                                                 │
       │ if was on /invite/<token>:                                      │
       │   read localStorage pendingInviteToken                          │
       │   POST /invites/<token>/redeem                                  │
       │   → switch view to project + welcome task                       │
```

### 6.3 Сессия

- JWT в HttpOnly cookie `leyka_session`, `SameSite=Lax`, `Secure=true` на проде.
- TTL — 7 дней (`JWT_EXPIRE_HOURS=168`), без refresh-token (для простоты — на проде логинимся раз в неделю).
- На каждом API-запросе `current_user_optional` декодирует JWT, грузит User, проверяет `is_active`. Деактивированный юзер мгновенно теряет доступ.

### 6.4 Защиты на уровне приложения

| Защита                           | Где                                                           |
|----------------------------------|---------------------------------------------------------------|
| Production safety asserts        | [main.py](backend/app/main.py) — fail-fast при дефолтном JWT_SECRET, mock-auth, не-secure cookies на проде |
| OAuth state CSRF                 | [auth.py](backend/app/api/v1/auth.py) — random state в short-TTL cookie, проверка через `compare_digest` |
| Last-admin lockout protection    | [users.py](backend/app/api/v1/users.py) — нельзя разжаловать/задеактивировать последнего admin'а |
| Self-edit field stripping        | [users.py](backend/app/api/v1/users.py) — юзер не может выдать себе `is_superuser` через PATCH |
| Author always from cookie        | [comments.py](backend/app/api/v1/comments.py) — `author_id` форсится из сессии, нельзя притвориться другим |
| Project member-only writes       | [dependencies.py](backend/app/api/dependencies.py) — `ensure_project_access` |

---

## 7. Цепочка приглашений

> Phase 4. Реализует основной онбординг-сценарий: «admin зовёт менеджера → менеджер зовёт разработчиков».

### 7.1 Модель `ProjectInvite`

```
┌────────────────────────────────────────────────────────────┐
│ id            UUID                                          │
│ project_id    FK projects (CASCADE)                         │
│ role_id       FK roles (RESTRICT)  ─ какую роль получит     │
│ inviter_id    FK users (SET NULL)  ─ кто пригласил          │
│ token         VARCHAR(64) UNIQUE   ─ secrets.token_urlsafe  │
│ email_hint    String?              ─ для UI «приглашение …» │
│ greeting      String?              ─ первый коммент         │
│ expires_at    DateTime             ─ default now + 7 дней   │
│ max_uses      Int                  ─ default 1              │
│ used_count    Int                  ─ инкрементится в redeem │
└────────────────────────────────────────────────────────────┘
```

### 7.2 Полный поток

```
   [Inviter]                  [Backend]                     [Invitee]
       │                          │                              │
       │ create invite            │                              │
       ├─────────────────────────►│                              │
       │   POST /projects/{id}    │                              │
       │   /invites               │                              │
       │ ◄──── { url } ───────────┤                              │
       │ скопировал, отправил                                    │
       │ ──── e-mail / Telegram ─────────────────────────────►   │
       │                                                         │
       │                          │  открывает /invite/<token>   │
       │                          │ ◄────────────────────────────┤
       │                          │  GET /invites/<token>        │
       │                          ├─ public preview ────────────►│
       │                          │  показывает «X приглашает в  │
       │                          │  проект Y, роль Z»           │
       │                                                         │
       │                          │  кликает «Войти через Mail»  │
       │                          │  (token сохранён в localStorage) │
       │                          │  ◄─── OAuth flow ───────────►│
       │                          │  возвращается с cookie       │
       │                          │                              │
       │                          │  App.tsx читает localStorage │
       │                          │  POST /invites/<token>/redeem│
       │                          │ ◄────────────────────────────┤
       │                          │ ┌────────────────────────┐   │
       │                          │ │ if already member:     │   │
       │                          │ │   return existing      │   │
       │                          │ │   (idempotent)         │   │
       │                          │ │ else:                  │   │
       │                          │ │   check is_active      │   │
       │                          │ │   create ProjectMember │   │
       │                          │ │   create welcome Task: │   │
       │                          │ │     "Добро пожаловать,│   │
       │                          │ │      <name>!"          │   │
       │                          │ │   add first Comment    │   │
       │                          │ │     from inviter       │   │
       │                          │ │     (greeting)         │   │
       │                          │ │   used_count++         │   │
       │                          │ └────────────────────────┘   │
       │                          ├ { project_id, welcome_task }│
       │                          │ ─────────────────────────────►│
       │                          │  view → project + open       │
       │                          │  welcome task drawer         │
       │                          │                              │
       │                          │  invitee пишет ответ inviter │
       │                          │  → unread comment в Inbox    │
       │ ◄──────────────── inviter получает уведомление ──────── │
```

Тесты потока: [test_invites.py](backend/tests/test_invites.py) — 19 кейсов (создание, права, preview, expired/used, идемпотентность, multi-use, list, revoke).

---

## 8. KPI-подсистема

### 8.1 Концепция

Всё крутится вокруг **11 обязательных KPI Минобрнауки** (Приложение №2 к Правилам экспертизы). Они хранятся в `kpi_catalog` со стабильными кодами `z-1`…`z-11` — не меняются, к ним привязываются обязательства проектов и факты.

```
┌──────────────────────────────────────────────────────────────┐
│  KpiCatalog            (справочник, seed на 11 строк)         │
│       │                                                       │
│       │ 1:N (по годам)                                        │
│       ▼                                                       │
│  KpiCenterTarget       (директор: «z-9 = 10 статей в 2026»)   │
│       │                                                       │
│       │ независимо                                            │
│       ▼                                                       │
│  ProjectKpiCommitment  (менеджер проекта:                     │
│       │                 «Сигма обязуется по z-9 = 2 статьи»)  │
│       │ 1:N                                                   │
│       ▼                                                       │
│  KpiAchievement        (факт: «Опубликовано в IEEE ITAI       │
│                         2026, evidence_url=…»)                │
└──────────────────────────────────────────────────────────────┘
```

### 8.2 Подсчёт прогресса (центральная сводка)

```
для каждой KPI z-N:
   plan_for_year   = sum(KpiCenterTarget where year=Y)
   committed       = sum(ProjectKpiCommitment.target_value where catalog=N)
   achieved        = sum(KpiAchievement.value where commitment.catalog=N
                         and occurred_at within year Y)

   цвет плитки:
      🔴 если committed = 0                         → ищи заказчика/грант
      🟠 если 0 < achieved < plan*0.8               → в работе
      🟢 если plan*0.8 ≤ achieved < plan            → почти закрыт
      🟢🟢 если achieved ≥ plan                      → перевыполнено
```

UI: [CenterKpiView](frontend/src/components/CenterKpiView.tsx) — плитки с цветовыми маркерами.

---

## 9. API-контракт

### 9.1 Все эндпоинты (72 route'а)

| Группа         | Эндпоинты                                                                       |
|----------------|---------------------------------------------------------------------------------|
| **Auth**       | `GET /auth/providers` · `GET /auth/me` · `POST /auth/local` · `POST /auth/logout` · `GET /auth/{provider}/login` · `GET /auth/{provider}/callback` · `GET /auth/mock/picker` |
| **Users**      | `GET /users` · `POST /users` · `GET /users/{id}` · `PATCH /users/{id}` · `DELETE /users/{id}` · `GET /users/me/memberships` |
| **Partners**   | `GET /partners` · `POST /partners` · `GET /partners/{id}` · `PATCH /partners/{id}` · `DELETE /partners/{id}` · `POST /partners/{id}/presale-project` |
| **Projects**   | `GET /projects` · `POST /projects` · `GET /projects/{id}` · `PATCH /projects/{id}` · `DELETE /projects/{id}` · `GET /projects/{id}/attachments` · `GET /projects/{id}/analytics` |
| **Members**    | `GET /projects/{id}/members` · `POST /projects/{id}/members` · `PATCH /projects/{id}/members/{mid}` · `DELETE /projects/{id}/members/{mid}` |
| **Roles**      | `GET /roles` · `PATCH /roles/{id}`                                              |
| **Invites**    | `POST /projects/{id}/invites` · `GET /projects/{id}/invites` · `GET /invites/{token}` · `POST /invites/{token}/redeem` · `DELETE /invites/{id}` |
| **Tasks**      | `GET /tasks` · `POST /tasks` · `GET /tasks/{id}` · `PATCH /tasks/{id}` · `DELETE /tasks/{id}` · `PATCH /tasks/{id}/status` · `GET /tasks/{id}/history` · `POST /tasks/{id}/mark-read` · `GET /tasks/lookup` |
| **Attachments**| `GET /tasks/{id}/attachments` · `POST /tasks/{id}/attachments` · `PATCH /tasks/{id}/attachments/{aid}` · `DELETE /tasks/{id}/attachments/{aid}` |
| **Comments**   | `GET /comments` · `POST /comments` · `PATCH /comments/{id}` · `DELETE /comments/{id}` |
| **KPI**        | `GET /kpi/catalog` · `GET /kpi/catalog/{code}/center-targets` · `PUT /kpi/catalog/{code}/center-targets/{year}` · `DELETE /kpi/catalog/{code}/center-targets/{year}` · `GET /kpi/catalog/{code}/achievements` · `GET /analytics/kpi-summary` · `/projects/{id}/kpi/...` |
| **Health**     | `GET /health` (для Railway probe и frontend banner)                             |

### 9.2 Type-safety end-to-end

```
1. FastAPI → OpenAPI 3.1 spec (автогенерится из Pydantic schemas + endpoints)
2. openapi-typescript /openapi.json → src/api/types.gen.ts
3. src/api/client.ts вручную тонкие wrapper'ы — typed по components['schemas']
4. TanStack Query useQuery/useMutation с typed return → React-компоненты
   получают полные TS-типы без any
```

Любое изменение бэк-схемы → `npm run gen:api` → ломается фронт-компиляция → ловим до коммита.

---

## 10. Деплой и инфраструктура

### 10.1 Прод-стек

```
┌─────────────────────────────────────────────────────────────────┐
│  🌐 FRONTEND — Vercel                                            │
│     · Static SPA build (Vite)                                    │
│     · CDN (edge caching)                                         │
│     · Rewrite /api/* → Railway (same-origin для cookie)          │
│     · vercel.json: SPA fallback `/(.*)` → /index.html            │
│     · Custom domain: leyka-nsu.vercel.app                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  ⚙ BACKEND — Railway                                              │
│     · Nixpacks build (Python 3.12, venv в /opt/venv)             │
│     · uvicorn app.main:app --host 0.0.0.0 --port $PORT           │
│     · Environment: ROOT_EMAIL/PASSWORD, JWT_SECRET, OAuth creds  │
│     · Health probe: /health                                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  💾 DATA — Railway Persistent Volume                              │
│     · /data/leyka.db (SQLite)                                    │
│     · alembic upgrade head запускается на каждом старте          │
│     · Бэкапы: Phase 6 (snapshots на Yandex.Disk)                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 CI/CD

```
git push origin main
        │
        ├─► Vercel webhook — фронт build (~1 мин)
        │   ▶ Watch Paths настроены: пересобирается ТОЛЬКО при
        │     изменении frontend/** (пропускает чисто бэк-PR)
        │
        └─► Railway webhook — бэк build (~2 мин)
            ▶ Watch Paths: пересобирается ТОЛЬКО при backend/**
            ▶ start.sh: source venv → alembic upgrade head → uvicorn
```

Подробности: [DEPLOY.md](DEPLOY.md), [SKILL-RELEASE.md](SKILL-RELEASE.md).

### 10.3 Файлы конфигурации

| Файл                       | Назначение                                  |
|----------------------------|---------------------------------------------|
| `frontend/vercel.json`     | SPA fallback + rewrites на Railway          |
| `nixpacks.toml`            | Python 3.12 + venv setup для Railway        |
| `railway.json`             | startCommand, healthcheck path              |
| `backend/start.sh`         | activate venv → migrate → uvicorn           |
| `backend/.env`             | Локальные секреты (НЕ в git)                |
| `backend/alembic.ini`      | Конфиг миграций                             |

---

## 11. Безопасность

### 11.1 Чек-лист (зафиксирован в audit'е до релиза)

| Категория              | Защита                                                  | Статус |
|------------------------|---------------------------------------------------------|:------:|
| **Auth bypass**        | Все API под `Depends(current_user)` → 401 для анонимов  | ✅      |
| **JWT**                | HttpOnly cookie, SameSite=Lax, Secure на проде          | ✅      |
| **Production secrets** | Fail-fast при дефолтном JWT_SECRET / mock-auth на проде | ✅      |
| **CSRF (OAuth)**       | Random state в short-TTL cookie, `compare_digest`       | ✅      |
| **Author spoofing**    | `comments.author_id` форсится из cookie                 | ✅      |
| **Last-admin lockout** | Нельзя разжаловать/удалить последнего активного admin'а | ✅      |
| **Self-elevation**     | Self-edit режет `is_superuser`/`is_active`/`role`       | ✅      |
| **Direct task access** | `ensure_project_access` → 403 для не-членов             | ✅      |
| **Project leakage**    | Outsider lookup исключает чужие задачи                  | ✅      |
| **Invite enumeration** | Token = 24 байта base64url (`secrets.token_urlsafe`)    | ✅      |
| **XSS**                | Нет `dangerouslySetInnerHTML`, MentionText escape'ит    | ✅      |
| **CVE-аудит deps**     | python-dotenv 1.2.2, python-jose 3.5.0, bcrypt 4.0.1    | ✅      |
| **Starlette CVE**      | Документированный риск-acceptance: не используем FileResponse / multipart | ⚠ doc'd |

Регресс-тесты: [test_auth_lockdown.py](backend/tests/test_auth_lockdown.py) — 19 кейсов.

---

## 12. Метрики проекта

| Метрика                            | Значение                |
|------------------------------------|-------------------------|
| **Backend LOC** (`app/`)           | ~5 000                  |
| **Frontend LOC** (`src/`)          | ~18 000                 |
| **Backend файлов** (`*.py`)        | 50+                     |
| **Frontend компонентов** (`*.tsx`) | 35+                     |
| **API эндпоинтов**                 | 72                      |
| **Доменных моделей**               | 15 таблиц               |
| **Pydantic схем**                  | 11 модулей              |
| **Alembic миграций**               | 16                      |
| **Backend тестов**                 | **132 (all green)**     |
| **Frontend typecheck**             | strict, 0 errors        |
| **Bundle size (prod, gzip)**       | ~250 KB                 |
| **Языков i18n**                    | 1 (RU; EN — на потом)   |
| **OAuth провайдеров**              | 3 (Mail.ru, Google, mock) |
| **RBAC ролей в seed'е**            | 6 (12 permission flags) |
| **KPI центра**                     | 11 (z-1..z-11)          |
| **Visibility-режимов**             | 3 для проекта, 2 для партнёра |

---

## 13. Тестовая стратегия

### 13.1 Тестовая пирамида (текущая)

```
                ┌──────────────────────────┐
                │   Manual smoke (browser) │   ← после каждого релиза
                ├──────────────────────────┤
                │       (нет E2E)          │   ← на потом, Playwright
                ├──────────────────────────┤
                │  API integration (132)   │   ← основной слой
                ├──────────────────────────┤
                │  Unit (через API)        │   ← всё через httpx + AsyncSession
                └──────────────────────────┘
```

### 13.2 Покрытие по доменам

| Файл                       | Тестов | Что покрывает                                        |
|----------------------------|:------:|------------------------------------------------------|
| `test_smoke.py`            | 4      | Health, /auth/me для anon/logged                     |
| `test_auth_lockdown.py`    | 19     | Anon-blocking, OAuth state CSRF, cancel flow         |
| `test_partners.py`         | 5      | CRUD заказчиков                                      |
| `test_projects.py`         | 6      | CRUD проектов, фильтр по партнёру                    |
| `test_visibility.py`       | 12     | private/public/demo матрица доступа                  |
| `test_user_admin.py`       | 15     | DELETE /users, last-admin lockout, rename            |
| `test_members.py`          | 6      | Add/update/remove participants                       |
| `test_roles.py`            | 6      | Permission matrix update                             |
| `test_tasks.py`            | 7      | CRUD + status history + mark-read                    |
| `test_attachments.py`      | 7      | URL-attachments lifecycle                            |
| `test_comments.py`         | 6      | Comment CRUD, edited_at, author from cookie          |
| `test_read_receipts.py`    | 4      | M:N read tracking                                    |
| `test_kpi.py`              | 13     | Catalog, commitments, achievements, year aggregation |
| `test_analytics.py`        | 3      | Project analytics: throughput, lead time             |
| `test_invites.py`          | 19     | Цепочка приглашений (создание, redeem, list, revoke) |

### 13.3 Тест-фикстуры

```
tests/conftest.py
   ├── engine          ─ in-memory SQLite, StaticPool, fresh per test
   ├── session_maker   ─ async_sessionmaker над engine
   ├── client          ─ AsyncClient (анон) с подменённым get_db
   ├── admin_client    ─ AsyncClient с cookie superuser'а
   ├── user_client     ─ AsyncClient с cookie regular_user
   └── helpers         ─ create_partner, create_project, add_member, _mk_user
```

---

## 14. Принятые архитектурные решения (ADR)

> Решения с обоснованием. Если приходит соблазн «передумать» — сначала
> прочитай тред, почему сейчас именно так.

| ADR  | Решение                                          | Почему                                                        |
|------|--------------------------------------------------|---------------------------------------------------------------|
| 001  | UUID в качестве primary key                      | Безопасный URL-share, легко генерить на клиенте               |
| 002  | SQLite на проде, не Postgres                     | <50 юзеров; миграция на Postgres готова (ASYNC dialect)       |
| 003  | Same-origin через Vercel rewrite, не CORS        | Cookie-auth работает «само», нет preflight-задержек           |
| 004  | HttpOnly cookie, не localStorage для JWT         | Защита от XSS-кражи токена                                    |
| 005  | Один `User.full_name` вместо `display_name`      | Простота, single source of truth (см. ревизию 2026-04-24)     |
| 006  | Два уровня ролей (User.role + ProjectMember)     | Глобальная должность ≠ права в проекте; синхронизированы лейблы |
| 007  | Visibility поверх RBAC                           | Гибкость онбординга: гость видит только public/demo           |
| 008  | Цепочка инвайтов вместо self-signup              | Контроль доступа: админ зовёт менеджера → менеджер зовёт devs |
| 009  | TanStack Query, не Redux                         | Минимум boilerplate для server state, optimistic updates      |
| 010  | Zustand для UI prefs, не Context                 | 2KB, persist middleware, нет re-render каскадов               |
| 011  | URL-based attachments (без файлового хранилища)  | НГУ уже использует Я.Диск/Google Docs — не дублируем          |
| 012  | Last-admin protection в backend, не frontend     | Critical защита, нельзя обойти через прямой API-вызов         |
| 013  | OpenAPI → TypeScript codegen                     | Контракт — единственная правда, нет ручных типов              |
| 014  | Phase 4 invites без email-нотификации            | SMTP — отдельный риск (deliverability); URL копируется руками |
| 015  | Welcome-задача с greeting от inviter             | Замыкает петлю: invitee сразу пишет ответ в чат пригласившему |

---

## 15. Дальнейшее развитие

### 15.1 Backlog (приоритизировано)

| Phase  | Что                                                            | Сложность  |
|--------|----------------------------------------------------------------|------------|
| **6**  | SMTP-нотификации (приглашения, ответы в @-упоминаниях)         | M (config) |
| **7**  | Бэкап БД на Yandex.Disk + retention policy                     | M          |
| **8**  | Wiki-страницы для проектов (markdown + конфиденциальный flag)  | L          |
| **9**  | Per-project KPI report PDF (для отчётности в Минобрнауки)      | M          |
| **10** | Bulk export проекта (zip с задачами + комментами + ссылками)   | M          |
| **11** | I18n: английский UI                                            | S          |
| **12** | PWA: offline-режим, push-уведомления                           | L          |

### 15.2 Технический долг

- ❌ Нет E2E-тестов (Playwright). Манульный smoke после каждого релиза.
- ❌ TanStack Query refetch стратегия не оптимизирована (везде `refetchInterval: 15s` где нужно — overkill).
- ❌ Bundle не code-split'нут — все 250KB загружаются сразу.
- ❌ SQLite не enforce'ит FK constraints без `PRAGMA foreign_keys=ON` — компенсируем ORM cascade'ами.

---

## 16. Ссылки

| Документ              | Содержание                                                        |
|-----------------------|-------------------------------------------------------------------|
| [README.md](README.md)| Пользовательская инструкция: роли, KPI, типовые сценарии          |
| [DEPLOY.md](DEPLOY.md)| Шаги первичного деплоя на Vercel/Railway                          |
| [SKILL-RELEASE.md](SKILL-RELEASE.md)| Релизный процесс, обнаруженные подводные камни      |
| [SKILL-AUTH.md](SKILL-AUTH.md)| Детали OAuth-конфигурации провайдеров                     |
| [SKILL-UI-Master.md](SKILL-UI-Master.md)| UI-гайдлайны (цвета, тени, иконки)              |
| [TODONOW.md](TODONOW.md)| Текущий план работ, открытые вопросы                            |
| [backlog.md](backlog.md)| Долгосрочный бэклог идей                                         |

---

> *Документ создан 2026-04-24. Поддерживается параллельно с кодом.*
> *При значимых изменениях архитектуры — обновлять одновременно с PR.*
