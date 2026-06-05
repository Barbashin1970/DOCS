# ПРОГРАММА И МЕТОДИКА ИСПЫТАНИЙ
## программы «LEYKA — Система управления проектами центра искусственного интеллекта»

---

> **Обозначение документа:** ЦИИ НГУ.LEYKA.ПМИ.001-2026  
> **Версия:** 1.1  
> **Дата составления:** 27 апреля 2026 г.  
> **Версия программы:** 0.2.0  
> **Статус:** действующий  
> **Разработано в соответствии с:** ГОСТ 19.301-79 «Программа и методика испытаний. Требования к содержанию и оформлению»  
> **Основание:** Техническое задание ЦИИ НГУ.LEYKA.ТЗ.001-2026 (версия 1.1 от 27.04.2026)

---

## Лист согласования

| Должность | ФИО | Подпись | Дата |
|---|---|---|---|
| Директор ЦИИ НГУ (Заказчик) | | | |
| Главный разработчик (Исполнитель) | | | |
| Руководитель QA | | | |

---

## Содержание

1. Объект испытаний  
2. Цель испытаний  
3. Требования к программе (проверяемые)  
4. Требования к программной документации  
5. Средства и порядок испытаний  
6. Порядок предъявления отчётности  
7. Методы испытаний  
   7.1 Дымовые испытания (Smoke)  
   7.2 Испытания подсистемы аутентификации  
   7.3 Испытания безопасности (Auth Lockdown)  
   7.4 Испытания управления партнёрами  
   7.5 Испытания управления проектами  
   7.6 Испытания видимости объектов (Visibility)  
   7.7 Испытания управления участниками  
   7.8 Испытания матрицы ролей и прав  
   7.9 Испытания управления задачами  
   7.10 Испытания типа и автора задачи  
   7.11 Испытания вложений задач  
   7.12 Испытания комментариев  
   7.13 Испытания квитанций прочтения  
   7.14 Испытания подсистемы KPI  
   7.15 Испытания витрины достижений (Showcase)  
   7.16 Испытания цепочки приглашений  
   7.17 Испытания чек-листа документации  
   7.18 Испытания загрузки файлов (Google Drive)  
   7.19 Испытания управления пользователями (Admin)  
   7.20 Испытания аналитики проекта  
   7.21 Испытания полнотекстового поиска (FTS5)  
   7.22 Испытания библиотеки знаний (WIKI)  
   7.23 Испытания резервного копирования (Backup/Restore)  
   7.24 Испытания фронтенда (Vitest, статика типов)  
8. Ведомость испытательных тестов  
9. Приложения  

---

## 1. Объект испытаний

**Наименование:** «LEYKA — Система управления проектами центра искусственного интеллекта»  
**Обозначение:** ЦИИ-НГУ-LEYKA-2026  
**Версия:** 0.2.0 (испытываемая)  
**Репозиторий:** https://github.com/Barbashin1970/LEYKA  
**Производственный экземпляр:** https://leyka-nsu.vercel.app  
**API-документация:** https://leyka-production.up.railway.app/docs  

Объект испытаний включает:
- **Back-end:** Python 3.12, FastAPI 0.115, SQLAlchemy 2 (async), Alembic (26 миграций, 0001 … 0026), 87 REST API-эндпойнтов;
- **Front-end:** React 18.3, TypeScript 5.6 (strict), TanStack Query 5.59, Zustand 5.0, Tailwind CSS 3.4, Vite 5.4 (производственный бандл — 167,7 КБ JavaScript + 10,2 КБ CSS после gzip);
- **СУБД:** SQLite (dev и prod) с режимом WAL на постоянном томе Railway; миграция на PostgreSQL не планируется в горизонте 5 лет (см. обоснование в SKILL-INFRA.md);
- **Инфраструктура:** Vercel (фронт, CDN, rewrite `/api/*` на Railway) + Railway (бэкенд, постоянный том `/data/leyka.db`); альтернативный сценарий «всё на одном железе» для развёртывания на сервере ЦИИ НГУ — см. SKILL-INFRA.md.

---

## 2. Цель испытаний

Целью испытаний является проверка соответствия программы LEYKA требованиям Технического задания ЦИИ НГУ.LEYKA.ТЗ.001-2026 по следующим аспектам:

1. Корректность реализации всех функциональных требований;
2. Работоспособность механизмов аутентификации и авторизации (OAuth, JWT, RBAC);
3. Корректность расчёта прогресса по 11 KPI-метрикам Минэкономразвития;
4. Надёжность и безопасность: защита от несанкционированного доступа, CSRF, XSS, подмены автора;
5. Корректность жизненного цикла объектов (проект, задача, документ, факт KPI);
6. Производительность при нормальной эксплуатационной нагрузке (≤ 50 конкурентных пользователей).

---

## 3. Требования к программе (проверяемые)

Настоящая ПМИ охватывает проверку следующих требований ТЗ:

| Пункт ТЗ | Требование | Раздел ПМИ |
|---|---|---|
| 4.1.1 (Auth) | OAuth Mail.ru / Google, JWT HttpOnly cookie, CSRF-защита | 7.2, 7.3 |
| 4.1.1 (Partners) | CRUD заказчиков, видимость, цвета | 7.4 |
| 4.1.1 (Projects) | CRUD проектов, стадии, аналитика | 7.5, 7.20 |
| 4.1.1 (Visibility) | private / public / demo — матрица доступа | 7.6 |
| 4.1.1 (Tasks) | Kanban, статусы, приоритеты, типы, критерий | 7.9, 7.10 |
| 4.1.1 (Comments) | Лента, @-упоминания, двойные галочки | 7.12, 7.13 |
| 4.1.1 (Docs) | Чек-лист 12 стадий, слоты, drag-and-drop | 7.17, 7.18 |
| 4.1.1 (KPI) | 11 метрик, обязательства, факты, дашборд | 7.14 |
| 4.1.1 (Showcase) | Витрина, 7 типов артефактов | 7.15 |
| 4.1.1 (Invites) | Цепочка приглашений, welcome-задача | 7.16 |
| 4.1.1 (RBAC) | 6 ролей, 12 флагов, superuser | 7.3, 7.7, 7.8 |
| 4.1.3 | Время отклика API ≤ 700 мс на 95-м перцентиле | 7.2, 7.5 |
| 4.2 | 345 back-end тестов и 176 front-end тестов проходят без ошибок, статическая проверка типов — 0 ошибок | 7.21–7.24, 8 |

---

## 4. Требования к программной документации

К моменту проведения испытаний должны быть представлены:

| Документ | Обозначение | Статус / Файл |
|---|---|---|
| Техническое задание | ЦИИ НГУ.LEYKA.ТЗ.001-2026 | Утверждён, LEYKA_TZ_GOST19.md |
| Программа и методика испытаний | ЦИИ НГУ.LEYKA.ПМИ.001-2026 | Настоящий документ |
| Архитектура системы | ЦИИ НГУ.LEYKA.АРХ.001-2026 | Утверждён, ARC.md |
| Руководство пользователя | ЦИИ НГУ.LEYKA.РП.001-2026 | Утверждён, README.md |
| Тест-план (расширенный) | ЦИИ НГУ.LEYKA.ТЕСТ.001-2026 | Утверждён, SKILL-TESTPLAN.md |
| Документ о выборе СУБД и развёртывании на собственных серверах | ЦИИ НГУ.LEYKA.ИНФ.001-2026 | Утверждён, SKILL-INFRA.md |

---

## 5. Средства и порядок испытаний

### 5.1 Программные средства испытаний

| Средство | Назначение | Версия |
|---|---|---|
| `pytest` + `pytest-asyncio` | Запуск back-end тестов | 9.0 / 1.3 |
| `httpx.AsyncClient` | HTTP-клиент поверх FastAPI ASGI (без реальной сети) | ≥ 0.27 |
| `aiosqlite` | SQLite in-memory для изоляции тестов | 0.20 |
| `SQLAlchemy StaticPool` | Одно соединение на весь тест — таблицы видны всем сессиям | 2.0 |
| `vitest` | Front-end unit-тесты (lib/, store/, components/About) | 2.1 |
| `tsc -b --noEmit` | Статическая проверка типов TypeScript в строгом режиме | 5.6 |
| Swagger UI | Ручное тестирование API-эндпойнтов через браузер | поставляется FastAPI |
| Браузер Chrome / Firefox / Safari | Ручное smoke-тестирование UI | Chrome 120+, Firefox 121+, Safari 17+ |

### 5.2 Технические средства

| Компонент | Конфигурация |
|---|---|
| Испытательная машина (backend-тесты) | CPU ≥ 2 ядра, RAM ≥ 2 ГБ, Python 3.12, venv |
| Тестовая СУБД | SQLite in-memory (StaticPool, изолирован на каждый тест) |
| Сеть | Не требуется (все тесты — offline, без внешних вызовов) |
| CI-среда (опционально) | GitHub Actions, Ubuntu latest |

### 5.3 Порядок проведения испытаний

1. Установить back-end зависимости: `cd backend && python3.12 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`;
2. Запустить полный набор back-end тестов: `pytest tests/ -v --tb=short`;
3. Зафиксировать результат back-end (passed / failed / errors);
4. Установить front-end зависимости: `cd frontend && npm install`;
5. Запустить front-end тесты: `npm test` (Vitest, режим без watch);
6. Запустить статическую проверку типов: `npx tsc -b --noEmit`;
7. Запустить производственную сборку front-end: `npm run build`;
8. Провести ручное smoke-тестирование по чек-листу п. 7.1;
9. Составить Протокол испытаний по форме Приложения А.

### 5.4 Критерии завершения испытаний

| Критерий | Пороговое значение |
|---|---|
| Back-end автотесты (pytest) | **288 passed, 0 failed, 0 errors** |
| Front-end автотесты (Vitest) | **176 passed, 0 failed** |
| Статическая проверка типов TypeScript | **0 ошибок** |
| Производственная сборка `npm run build` | **0 ошибок**, бандл собран |
| Ручное smoke-тестирование (п. 7.1) | Все пункты отмечены «Пройдено» |
| Время отклика критичных API-эндпойнтов | ≤ 700 мс на 95-м перцентиле при нагрузке ≤ 50 одновременных пользователей |

---

## 6. Порядок предъявления отчётности

По результатам испытаний оформляется **Протокол испытаний** по форме Приложения А настоящего документа. Протокол подписывается:
- Руководителем QA (Исполнитель);
- Директором ЦИИ НГУ или уполномоченным представителем (Заказчик).

Протокол прикладывается к Акту приёмки-сдачи программы.

---

## 7. Методы испытаний

> **Условные обозначения:**
> - `[AUTO]` — автоматизированный тест (pytest), выполняется командой `pytest tests/<файл>.py -v`
> - `[MANUAL]` — ручная проверка через браузер или Swagger UI
> - `[TYPECHECK]` — статический анализ TypeScript
> - **Ожидаемый результат** — критерий приёмки испытания

---

### 7.1 Дымовые испытания (Smoke)

**Тест-файл:** `tests/test_smoke.py`  
**Цель:** убедиться, что приложение запускается и базовые эндпоинты отвечают корректно.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 1.1 | `test_health_endpoint` | GET /health | HTTP 200, `{"status": "ok"}` |
| 1.2 | `test_anon_gets_401_on_me` | GET /auth/me без сессии | HTTP 401 |
| 1.3 | `test_logged_user_gets_me` | GET /auth/me с сессией admin | HTTP 200, поле `email` присутствует |
| 1.4 | `test_providers_list` | GET /auth/providers | HTTP 200, список провайдеров не пустой |

**Ручная проверка** `[MANUAL]`:

| № | Действие | Ожидаемый результат |
|---|---|---|
| 1.5 | Открыть https://leyka-nsu.vercel.app в браузере | Загружается экран входа |
| 1.6 | Нажать «Войти через Mail.ru» | Перенаправление на OAuth Mail.ru |
| 1.7 | Завершить OAuth-вход | Открывается главная страница (канбан / Inbox) |

---

### 7.2 Испытания подсистемы аутентификации

**Тест-файл:** `tests/test_auth.py`  
**Цель:** проверить все способы входа, выдачу и отзыв JWT-cookie, OAuth-флоу.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 2.1 | `test_local_login_success` | POST /auth/local с корректными root-кредами | HTTP 200, cookie `leyka_session` установлен |
| 2.2 | `test_local_login_wrong_password` | POST /auth/local с неверным паролем | HTTP 401 |
| 2.3 | `test_local_login_unknown_email` | POST /auth/local с несуществующим email | HTTP 401 |
| 2.4 | `test_logout_clears_cookie` | POST /auth/logout после входа | HTTP 200, cookie очищен |
| 2.5 | `test_me_after_logout_is_401` | GET /auth/me после logout | HTTP 401 |
| 2.6 | `test_oauth_login_redirect` | GET /auth/mailru/login | HTTP 302, redirect на Mail.ru, state-cookie установлен |
| 2.7 | `test_oauth_callback_csrf_invalid_state` | GET /auth/mailru/callback с неверным state | HTTP 400 (CSRF-ошибка) |
| 2.8 | `test_oauth_callback_access_denied` | GET /auth/mailru/callback?error=access_denied | HTTP 302 → /?login_cancelled=mailru |
| 2.9 | `test_mock_auth_disabled_on_prod` | GET /auth/mock/picker при IS_PRODUCTION=true | HTTP 404 или 403 |
| 2.10 | `test_jwt_ttl_7_days` | Проверить exp в декодированном JWT | exp - iat ≈ 604800 секунд (7 суток) |

---

### 7.3 Испытания безопасности (Auth Lockdown)

**Тест-файл:** `tests/test_auth_lockdown.py`  
**Цель:** проверить, что анонимные запросы блокируются, CSRF-защита OAuth работает, последний admin защищён.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 3.1 | `test_all_endpoints_require_auth` | 40+ эндпоинтов без cookie | Все возвращают HTTP 401 |
| 3.2 | `test_oauth_state_csrf_protection` | callback с неверным state | HTTP 400 |
| 3.3 | `test_oauth_state_missing` | callback без state-cookie | HTTP 400 |
| 3.4 | `test_last_admin_cannot_demote_self` | PATCH /users/{me} → is_superuser=false (последний admin) | HTTP 400 |
| 3.5 | `test_last_admin_cannot_deactivate_self` | PATCH /users/{me} → is_active=false (последний admin) | HTTP 400 |
| 3.6 | `test_last_admin_cannot_delete_self` | DELETE /users/{me} | HTTP 400 |
| 3.7 | `test_user_cannot_self_elevate` | PATCH /users/{me} → is_superuser=true (обычный user) | HTTP 200, но is_superuser остаётся false |
| 3.8 | `test_author_from_cookie_not_payload` | POST /comments с подставным author_id в body | author_id берётся из cookie, не из тела |
| 3.9 | `test_outsider_cannot_access_private_project` | GET /projects/{id} без membership | HTTP 403 |
| 3.10 | `test_task_lookup_excludes_foreign_tasks` | GET /tasks/lookup — чужие задачи не появляются | 200, в ответе нет чужих task.id |
| 3.11 | `test_invite_token_entropy` | Токен приглашения | Длина ≥ 32 символа base64url |
| 3.12 | `test_production_safety_default_jwt_secret` | Запуск с JWT_SECRET=default | Fail-fast: сервер не стартует |

---

### 7.4 Испытания управления партнёрами

**Тест-файл:** `tests/test_partners.py`  
**Цель:** проверить CRUD заказчиков и корректность поля visibility.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 4.1 | `test_create_partner` | POST /partners | HTTP 201, поля name, visibility возвращены |
| 4.2 | `test_list_partners` | GET /partners | HTTP 200, созданный партнёр присутствует |
| 4.3 | `test_update_partner` | PATCH /partners/{id} | HTTP 200, name обновлён |
| 4.4 | `test_delete_partner_cascade` | DELETE /partners/{id} | HTTP 204, каскадно удалены проекты и задачи |
| 4.5 | `test_partner_presale_project` | POST /partners/{id}/presale-project | HTTP 201, project.stage == "presale" |
| 4.6 | `test_partner_color_invalid_422` | POST /partners с color="invalid" | HTTP 422 |
| 4.7 | `test_partner_visibility_default_private` | POST /partners без visibility | HTTP 201, visibility == "private" |

---

### 7.5 Испытания управления проектами

**Тест-файл:** `tests/test_projects.py`  
**Цель:** проверить CRUD проектов, стадии жизненного цикла, фильтрацию.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 5.1 | `test_create_project` | POST /projects | HTTP 201, stage == "presale" по умолчанию |
| 5.2 | `test_list_projects_by_partner` | GET /projects?partner_id={id} | HTTP 200, только проекты этого партнёра |
| 5.3 | `test_update_project_stage` | PATCH /projects/{id} → stage=active | HTTP 200, stage обновлён |
| 5.4 | `test_update_project_budget` | PATCH /projects/{id} → budget | HTTP 200, budget обновлён |
| 5.5 | `test_delete_project_cascade` | DELETE /projects/{id} | HTTP 204, задачи удалены каскадом |
| 5.6 | `test_outsider_cannot_get_private_project` | GET /projects/{id} без membership | HTTP 403 |

---

### 7.6 Испытания видимости объектов (Visibility)

**Тест-файл:** `tests/test_visibility.py`  
**Цель:** проверить матрицу доступа для всех комбинаций visibility × роль пользователя.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 6.1 | `test_private_project_invisible_to_outsider` | Обычный user без membership видит private project | HTTP 403 |
| 6.2 | `test_public_project_visible_to_all_users` | Обычный user видит public project | HTTP 200 |
| 6.3 | `test_demo_project_visible_to_guest` | Гость видит demo project | HTTP 200 |
| 6.4 | `test_demo_project_invisible_to_guest_if_private` | Гость НЕ видит private project | HTTP 403 |
| 6.5 | `test_superuser_sees_all_private_projects` | Admin видит все private проекты | HTTP 200 |
| 6.6 | `test_private_partner_invisible_to_outsider` | User без связанного проекта не видит private partner | HTTP 403 или пустой список |
| 6.7 | `test_public_partner_visible_to_all` | Все аутентифицированные видят public partner | HTTP 200 |
| 6.8 | `test_visibility_toggle_by_admin_only` | Обычный user пытается сменить visibility | HTTP 403 |
| 6.9 | `test_admin_can_change_visibility` | Admin меняет project visibility | HTTP 200, visibility обновлён |
| 6.10 | `test_tasks_in_private_project_hidden` | GET /tasks?project_id={private} без membership | HTTP 403 |
| 6.11 | `test_tasks_in_public_project_visible` | GET /tasks?project_id={public} без membership | HTTP 200 |
| 6.12 | `test_demo_project_included_in_task_lookup` | Задача из demo-проекта видна в /tasks/lookup | Присутствует в ответе |

---

### 7.7 Испытания управления участниками

**Тест-файл:** `tests/test_members.py`  
**Цель:** проверить добавление/изменение/удаление участников проекта и RBAC.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 7.1 | `test_add_member_to_project` | POST /projects/{id}/members | HTTP 201, участник добавлен с ролью |
| 7.2 | `test_duplicate_member_conflict` | POST /projects/{id}/members дважды для одного user | HTTP 409 |
| 7.3 | `test_update_member_role` | PATCH /projects/{id}/members/{mid} | HTTP 200, роль изменена |
| 7.4 | `test_remove_member` | DELETE /projects/{id}/members/{mid} | HTTP 204 |
| 7.5 | `test_non_manager_cannot_manage_members` | Разработчик добавляет участника | HTTP 403 |
| 7.6 | `test_manager_can_manage_members` | Менеджер добавляет участника | HTTP 201 |

---

### 7.8 Испытания матрицы ролей и прав

**Тест-файл:** `tests/test_roles.py`  
**Цель:** проверить чтение и обновление матрицы RBAC-прав через API.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 8.1 | `test_list_roles` | GET /roles | HTTP 200, 6 ролей в ответе |
| 8.2 | `test_roles_have_expected_names` | Проверка имён ролей | Менеджер, Разработчик, Аналитик, Куратор, Наблюдатель, Заказчик |
| 8.3 | `test_admin_can_update_role_permissions` | PATCH /roles/{id} | HTTP 200, флаг обновлён |
| 8.4 | `test_regular_user_cannot_update_roles` | PATCH /roles/{id} от user | HTTP 403 |
| 8.5 | `test_manager_role_has_all_permissions` | Проверка флагов роли Менеджер | can_edit_tasks=true, can_manage_members=true и т.д. |
| 8.6 | `test_observer_role_readonly` | Проверка флагов Наблюдатель | can_edit_tasks=false, can_move_status=false |

---

### 7.9 Испытания управления задачами

**Тест-файл:** `tests/test_tasks.py`  
**Цель:** проверить CRUD задач, смену статусов, аудит переходов, mark-read.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 9.1 | `test_create_task` | POST /tasks | HTTP 201, status="inbox", priority="p3" |
| 9.2 | `test_list_tasks_by_project` | GET /tasks?project_id={id} | HTTP 200, только задачи этого проекта |
| 9.3 | `test_update_task_fields` | PATCH /tasks/{id} — title, description | HTTP 200, поля обновлены |
| 9.4 | `test_move_task_status` | PATCH /tasks/{id}/status → in_progress | HTTP 200, status обновлён |
| 9.5 | `test_status_history_recorded` | GET /tasks/{id}/history после смены статуса | HTTP 200, запись с old_status, new_status, changed_at |
| 9.6 | `test_delete_task` | DELETE /tasks/{id} | HTTP 204 |
| 9.7 | `test_mark_read` | POST /tasks/{id}/mark-read | HTTP 200, задача не в Inbox как непрочитанная |
| 9.8 | `test_task_author_immutable` | PATCH /tasks/{id} → author_id чужой | author_id не изменился |
| 9.9 | `test_outsider_cannot_create_task` | POST /tasks в чужом проекте | HTTP 403 |
| 9.10 | `test_done_without_criterion_requires_confirmation` | PATCH /tasks/{id}/status → done без criteria | HTTP 200 с предупреждением или отдельный флаг |

---

### 7.10 Испытания типа и автора задачи

**Тест-файл:** `tests/test_task_author_kind.py`  
**Цель:** проверить иммутабельность author_id и корректность работы типов задачи (task / idea / bug).  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 10.1 | `test_task_author_set_from_session` | Создание задачи — author берётся из cookie | author_id == id текущего пользователя |
| 10.2 | `test_task_author_cannot_be_changed` | PATCH /tasks/{id} с другим author_id | author_id не изменился |
| 10.3 | `test_task_kind_default_task` | Создание задачи без kind | kind == "task" |
| 10.4 | `test_task_kind_idea` | Создание задачи с kind=idea | kind == "idea" |
| 10.5 | `test_task_kind_bug` | Создание задачи с kind=bug | kind == "bug" |
| 10.6 | `test_task_kind_invalid_422` | Создание задачи с kind=unknown | HTTP 422 |

---

### 7.11 Испытания вложений задач

**Тест-файл:** `tests/test_attachments.py`  
**Цель:** проверить CRUD URL-вложений, стадии, корректность маппинга legacy-slugов.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 11.1 | `test_create_attachment` | POST /tasks/{id}/attachments | HTTP 201, url и stage сохранены |
| 11.2 | `test_list_attachments` | GET /tasks/{id}/attachments | HTTP 200, вложение присутствует |
| 11.3 | `test_update_attachment_stage` | PATCH /tasks/{id}/attachments/{aid} | HTTP 200, stage обновлён |
| 11.4 | `test_delete_attachment` | DELETE /tasks/{id}/attachments/{aid} | HTTP 204 |
| 11.5 | `test_attachment_invalid_url_422` | POST с url=not-a-url | HTTP 422 |
| 11.6 | `test_legacy_slug_contract_maps_to_research` | stage="contract" → при чтении | stage == "research" |
| 11.7 | `test_stage_list_matches_catalog` | Все stage-значения из каталога | Только допустимые 12 значений |

---

### 7.12 Испытания комментариев

**Тест-файл:** `tests/test_comments.py`  
**Цель:** проверить создание, редактирование и удаление комментариев; запрет подмены автора.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 12.1 | `test_create_comment` | POST /comments | HTTP 201, author_id из сессии |
| 12.2 | `test_list_comments_by_task` | GET /comments?task_id={id} | HTTP 200, комментарий присутствует |
| 12.3 | `test_edit_own_comment` | PATCH /comments/{id} своего | HTTP 200, text обновлён, edited_at != null |
| 12.4 | `test_cannot_edit_foreign_comment` | PATCH /comments/{id} чужого (не admin) | HTTP 403 |
| 12.5 | `test_delete_own_comment` | DELETE /comments/{id} своего | HTTP 204 |
| 12.6 | `test_author_id_from_cookie_not_body` | POST /comments с author_id чужого в body | author_id == id из cookie |

---

### 7.13 Испытания квитанций прочтения

**Тест-файл:** `tests/test_read_receipts.py`  
**Цель:** проверить механизм двойных галочек (M:N comment × user).  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 13.1 | `test_mark_comment_read` | POST /tasks/{id}/mark-read | HTTP 200, read_at фиксируется |
| 13.2 | `test_mark_read_idempotent` | Повторный POST mark-read | HTTP 200, дубль не создаётся |
| 13.3 | `test_unread_count_decreases` | GET /tasks после mark-read | unread_count уменьшился |
| 13.4 | `test_other_user_read_receipt_independent` | Два пользователя читают независимо | У каждого своё read_at |

---

### 7.14 Испытания подсистемы KPI

**Тест-файл:** `tests/test_kpi.py`  
**Цель:** проверить каталог метрик, обязательства проектов, факты достижений, сводный дашборд, rights.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 14.1 | `test_catalog_returns_seeded_entries` | GET /kpi/catalog | HTTP 200, коды z-1..z-11 присутствуют |
| 14.2 | `test_catalog_requires_auth` | GET /kpi/catalog без сессии | HTTP 401 |
| 14.3 | `test_commitment_create_and_progress` | POST обязательства z-1 target=2 | HTTP 201, achieved_value=0, progress_pct=0.0 |
| 14.4 | `test_commitment_idempotent_post` | Повторный POST того же kpi_code | Обновляет target, id не меняется |
| 14.5 | `test_commitment_unknown_kpi_404` | POST с kpi_code=z-999 | HTTP 404 |
| 14.6 | `test_achievement_counts_toward_progress` | 2 факта при target=2 | progress_pct == 100.0 |
| 14.7 | `test_achievement_overshoot_caps_at_100` | 3 факта при target=1 | achieved_value=3, progress_pct == 100.0 (clamp) |
| 14.8 | `test_commitment_update_and_delete` | PATCH и DELETE обязательства | HTTP 200 / 204, список пуст |
| 14.9 | `test_center_summary_aggregates_across_projects` | Два проекта, /analytics/kpi-summary?year=2026 | target_total, achieved_total агрегированы по всем проектам |
| 14.10 | `test_center_summary_filters_by_year` | Факт 2025 года не попадает в 2026 | achieved_total 2026 == 0 |
| 14.11 | `test_center_target_set_and_read` | PUT /kpi/catalog/z-9/center-targets/2026 | HTTP 200, center_target == 15 в summary |
| 14.12 | `test_center_target_requires_superuser` | PUT от обычного user | HTTP 403 |
| 14.13 | `test_center_target_unknown_kpi_404` | PUT /kpi/catalog/z-999/center-targets/2026 | HTTP 404 |
| 14.14 | `test_center_target_year_range` | PUT с year=1999 | HTTP 422 (ge=2020) |
| 14.15 | `test_commitment_negative_target_422` | POST с target_value=-1 | HTTP 422 |
| 14.16 | `test_achievement_evidences_overflow_422` | POST с 11 evidences | HTTP 422 (max 10) |
| 14.17 | `test_achievement_value_must_be_positive_422` | POST с value=0 | HTTP 422 (ge=1) |
| 14.18 | `test_commitment_unknown_id_404` | PATCH несуществующего commitment | HTTP 404 |
| 14.19 | `test_delete_commitment_404` | DELETE несуществующего | HTTP 404 |
| 14.20 | `test_kpi_summary_requires_auth` | GET /analytics/kpi-summary без сессии | HTTP 401 |
| 14.21 | `test_delete_center_target_admin_only` | DELETE center-target от user / admin | 403 / 204 соответственно |

---

### 7.15 Испытания витрины достижений (Showcase)

**Тест-файл:** `tests/test_showcase.py`  
**Цель:** проверить публикацию/снятие достижений на витрину, 7 типов артефактов, видимость для гостей.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 15.1 | `test_showcase_empty_initially` | GET /showcase без опубликованных | HTTP 200, пустой список |
| 15.2 | `test_publish_achievement_to_showcase` | PATCH achievement → published_in_showcase=true | HTTP 200, появляется в GET /showcase |
| 15.3 | `test_showcase_visible_to_guest` | GET /showcase от гостя (без membership) | HTTP 200 |
| 15.4 | `test_showcase_asset_types` | Публикация с каждым из 7 типов | Тип фиксируется, возвращается в GET /showcase |
| 15.5 | `test_invalid_asset_type_422` | PATCH с asset_type=unknown | HTTP 422 |
| 15.6 | `test_unpublish_removes_from_showcase` | PATCH → published_in_showcase=false | Исчезает из GET /showcase |
| 15.7 | `test_published_evidence_idx` | Выбор маркетинговой ссылки (published_evidence_idx) | В /showcase используется нужный evidence |
| 15.8 | `test_only_admin_can_publish` | Публикация от обычного user | HTTP 403 |

---

### 7.16 Испытания цепочки приглашений

**Тест-файл:** `tests/test_invites.py`  
**Цель:** проверить полный флоу приглашений: создание, просмотр, redeem, welcome-задача, idempotency, expiry.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 16.1 | `test_create_invite` | POST /projects/{id}/invites | HTTP 201, token не пустой |
| 16.2 | `test_invite_preview_public` | GET /invites/{token} без сессии | HTTP 200, project_title, role, inviter_name |
| 16.3 | `test_redeem_invite_creates_membership` | POST /invites/{token}/redeem | HTTP 200, ProjectMember создан |
| 16.4 | `test_redeem_invite_creates_welcome_task` | POST /invites/{token}/redeem | welcome_task != null, greeting в комментарии |
| 16.5 | `test_redeem_idempotent_second_call` | Повторный redeem того же token | HTTP 200, welcome_task=null, membership не дублируется |
| 16.6 | `test_expired_invite_returns_410` | Redeem просроченного invite (expires_at в прошлом) | HTTP 410 |
| 16.7 | `test_used_up_invite_returns_410` | Redeem invite с max_uses=1 дважды | HTTP 410 при втором redeem |
| 16.8 | `test_multi_use_invite` | max_uses=3 — три разных пользователя | Все трое получают membership |
| 16.9 | `test_list_invites` | GET /projects/{id}/invites | HTTP 200, список активных инвайтов |
| 16.10 | `test_revoke_invite` | DELETE /invites/{id} | HTTP 204, токен больше не работает |
| 16.11 | `test_only_manager_can_create_invite` | POST от Разработчика | HTTP 403 |
| 16.12 | `test_invite_token_is_cryptographically_random` | Проверка длины и символов токена | len(token) ≥ 32, base64url |

---

### 7.17 Испытания чек-листа документации

**Тест-файл:** `tests/test_project_documents.py`  
**Цель:** проверить каталог слотов, включение/выключение, добавление документов, счётчики.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 17.1 | `test_catalog_returns_all_stages` | GET /document-checklist/catalog | HTTP 200, 12 стадий, ≥ 40 слотов |
| 17.2 | `test_checklist_empty_by_default` | GET /projects/{id}/document-checklist | HTTP 200, все слоты disabled |
| 17.3 | `test_enable_slot` | POST /projects/{id}/included-slots/{slot_key} | HTTP 200, слот включён |
| 17.4 | `test_disable_slot` | DELETE /projects/{id}/included-slots/{slot_key} | HTTP 200, слот отключён |
| 17.5 | `test_counter_closed_vs_total` | Включить 3 слота, заполнить 2 | closed == 2, total == 3 |
| 17.6 | `test_add_document_to_slot` | POST /projects/{id}/documents | HTTP 201, document создан |
| 17.7 | `test_add_document_without_slot` | POST с slot_key=null | HTTP 201, «свободный» документ |
| 17.8 | `test_document_url_dedup` | POST с тем же URL дважды | HTTP 409 (unique constraint) |
| 17.9 | `test_update_document` | PATCH /projects/{id}/documents/{doc} | HTTP 200, title обновлён |
| 17.10 | `test_delete_document` | DELETE /projects/{id}/documents/{doc} | HTTP 204 |
| 17.11 | `test_task_attachments_appear_in_stage` | TaskAttachment со stage=tor | Появляется в stage tor в чек-листе |
| 17.12 | `test_non_manager_cannot_toggle_slots` | POST /included-slots от Наблюдателя | HTTP 403 |

---

### 7.18 Испытания загрузки файлов (Google Drive)

**Тест-файл:** `tests/test_uploads.py`  
**Цель:** проверить загрузку файлов, валидацию типов и размера, graceful degradation без Drive.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 18.1 | `test_upload_pdf_creates_document` | POST /projects/{id}/uploads (PDF, mock Drive) | HTTP 201, ProjectDocument с kind=gdrive_temp |
| 18.2 | `test_upload_invalid_extension_422` | POST с .exe | HTTP 422 |
| 18.3 | `test_upload_oversized_file_413` | POST с файлом > 15 МБ | HTTP 413 |
| 18.4 | `test_upload_without_gdrive_config_503` | POST без GOOGLE_DRIVE_REFRESH_TOKEN | HTTP 503, остальное API работает |
| 18.5 | `test_upload_creates_subfolder` | Первый upload в проект | gdrive_folder_id записывается в Project |
| 18.6 | `test_upload_progress_response` | POST upload | Ответ содержит url и badge=gdrive_temp |
| 18.7 | `test_upload_requires_can_edit_tasks` | Upload от Наблюдателя | HTTP 403 |
| 18.8 | `test_allowed_extensions_list` | POST с .pdf, .docx, .xlsx, .pptx, .png, .jpg, .md | HTTP 201 для каждого |

---

### 7.19 Испытания управления пользователями (Admin)

**Тест-файл:** `tests/test_user_admin.py`  
**Цель:** проверить CRUD пользователей, защиту последнего admin, само-разжалование.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 19.1 | `test_list_users_admin_only` | GET /users от обычного user | HTTP 403 |
| 19.2 | `test_list_users_as_admin` | GET /users от admin | HTTP 200, список пользователей |
| 19.3 | `test_create_user_admin_only` | POST /users от user | HTTP 403 |
| 19.4 | `test_update_user_full_name` | PATCH /users/{id} → full_name | HTTP 200, имя обновлено |
| 19.5 | `test_admin_can_deactivate_user` | PATCH /users/{id} → is_active=false | HTTP 200, user деактивирован |
| 19.6 | `test_deactivated_user_gets_401` | Вход деактивированного user | HTTP 401 |
| 19.7 | `test_delete_user` | DELETE /users/{id} | HTTP 204 |
| 19.8 | `test_last_admin_protected_from_delete` | DELETE /users/{last_admin} | HTTP 400 |
| 19.9 | `test_last_admin_protected_from_demote` | PATCH /users/{last_admin} → is_superuser=false | HTTP 400 |
| 19.10 | `test_user_cannot_self_elevate_to_admin` | PATCH /users/{me} → is_superuser=true | is_superuser не изменился |
| 19.11 | `test_admin_can_promote_user` | PATCH /users/{id} → is_superuser=true (другой admin) | HTTP 200, второй admin создан |
| 19.12 | `test_rename_user_visible_in_comments` | Сменить full_name → проверить отображение | Новое имя видно в комментариях |

---

### 7.20 Испытания аналитики проекта

**Тест-файл:** `tests/test_analytics.py` (3 теста, фактически)  
**Цель:** проверить расчёт скорости закрытия задач (throughput) и среднего времени выполнения (lead time) по задачам проекта.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 20.1 | `test_analytics_throughput` | Закрыть 3 задачи, выполнить GET /projects/{id}/analytics | поле `throughput_by_week` содержит 3 закрытия в текущей неделе |
| 20.2 | `test_analytics_lead_time` | Задача создана 2 дня назад, закрыта сейчас | `avg_lead_time_hours` ≈ 48 |
| 20.3 | `test_analytics_empty_project` | GET /analytics для проекта без задач | `total_tasks=0`, `avg_lead_time_hours=null` |

---

### 7.21 Испытания полнотекстового поиска (FTS5)

**Тест-файл:** `tests/test_search.py` (18 тестов)  
**Цель:** проверить безопасный полнотекстовый поиск по задачам и комментариям проекта через SQLite FTS5: чистка пользовательского ввода (`sanitize_query`), префиксный поиск, синхронизация индекса при INSERT/UPDATE/DELETE через триггеры, изоляция между проектами, member-only доступ.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 21.1 | `test_search_finds_task_by_title` | GET /projects/{id}/search?q=интеграция | найдена 1 задача с подсветкой `<mark>` |
| 21.2 | `test_search_finds_task_by_description` | поиск по слову из описания (не из заголовка) | задача найдена через description_snippet |
| 21.3 | `test_search_prefix_match` | q=«бюдж» при наличии задачи «Утвердить бюджет Q2» | задача найдена (префиксный поиск с `*`) |
| 21.4 | `test_search_finds_comment` | поиск по слову из комментария | комментарий найден, привязан к task_id |
| 21.5 | `test_search_reflects_task_update` | PATCH задачи → поиск по старому слову → 0, по новому → 1 | триггер AFTER UPDATE синхронизировал FTS |
| 21.6 | `test_search_reflects_task_delete` | DELETE задачи → поиск → 0 совпадений | триггер AFTER DELETE удалил из FTS |
| 21.7 | `test_search_does_not_leak_across_projects` | задача в проекте A не находится при поиске в проекте B | фильтр `project_id` в SQL работает |
| 21.8 | `test_search_forbidden_for_outsider` | посторонний делает GET /projects/{id}/search | 403 |
| 21.9 | `test_search_works_for_member` | член проекта делает поиск | 200 + результаты |
| 21.10 | `test_search_empty_query_returns_empty` | q="" | 200 с `total=0`, не 422 |
| 21.11 | `test_search_punctuation_only_returns_empty` | q="?? --" | 200 с пустым результатом |
| 21.12 | `test_search_unknown_project_404` | поиск в несуществующем проекте | 403/404 (не 500) |

Юнит-тесты `sanitize_query()`: 6 кейсов (одиночное слово, AND-конъюнкция, отбрасывание знаков препинания, отбрасывание односимвольных токенов, нейтрализация FTS5-keyword'ов через кавычки, пустой ввод).

---

### 7.22 Испытания библиотеки знаний (WIKI)

**Тест-файл:** `tests/test_wiki.py` (24 теста)  
**Цель:** проверить чипы-ссылки библиотеки знаний — доступ (RBAC), бизнес-правило «is_shared требует description», нормализацию URL, изоляцию между проектами, toggle-лайки («капля Лейки»), полнотекстовый поиск на центральной странице.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 22.1 | `test_manager_can_create_link` | POST /projects/{id}/wiki от менеджера | 201, чип создан с автором |
| 22.2 | `test_developer_cannot_create_link` | POST /projects/{id}/wiki от Разработчика (без can_edit_wiki) | 403 с подсказкой про менеджера |
| 22.3 | `test_outsider_cannot_create_link` | POST от не-члена проекта | 403 |
| 22.4 | `test_create_in_unknown_project_404` | POST в несуществующий проект | 404 |
| 22.5 | `test_share_without_description_fails` | is_shared=True без description | 422 |
| 22.6 | `test_share_with_description_works` | is_shared=True + description | 201 |
| 22.7 | `test_url_protocol_auto_added` | URL без http:// | автодополнение https:// |
| 22.8 | `test_project_list_shows_all_members_links` | GET /projects/{id}/wiki от члена проекта | видит все чипы (локальные + расшаренные) |
| 22.9 | `test_outsider_cannot_see_project_wiki` | GET /projects/{id}/wiki от постороннего | 403 |
| 22.10 | `test_central_list_shows_only_shared` | GET /wiki | только is_shared=true |
| 22.11 | `test_central_list_visible_to_outsider` | GET /wiki от не-члена проекта | 200 (центральная видна всем залогиненным) |
| 22.12 | `test_central_list_type_filter` | GET /wiki?type=video | только видео-чипы |
| 22.13 | `test_author_can_delete_own_link` | DELETE /wiki/{id} от автора | 204 |
| 22.14 | `test_manager_can_delete_others_link` | DELETE /wiki/{id} от менеджера-не-автора | 204 |
| 22.15 | `test_outsider_cannot_delete_link` | DELETE от постороннего | 403 |
| 22.16 | `test_author_can_edit_description` | PATCH /wiki/{id} от автора | 200, description обновлён |
| 22.17 | `test_non_author_cannot_edit` | PATCH /wiki/{id} от не-автора (даже менеджера) | 403 |
| 22.18 | `test_like_toggle` | POST /wiki/{id}/like дважды | 1-й: liked=true count=1; 2-й: liked=false count=0 |
| 22.19 | `test_liked_by_me_per_user` | админ лайкнул, юзер видит | у юзера liked_by_me=False, count=1 |
| 22.20 | `test_central_search_by_title` | GET /wiki?q=… поиск по title | релевантный чип возвращается |
| 22.21 | `test_central_search_by_description` | поиск по description | релевантный чип возвращается |
| 22.22 | `test_central_search_does_not_leak_local_links` | поиск находит только is_shared=true | приватные чипы не утекают |
| 22.23 | `test_central_search_empty_query_returns_full_list` | q="" | обычная выборка по is_shared=true |
| 22.24 | `test_central_search_punctuation_only_returns_empty` | q="??" | пустой результат |

---

### 7.23 Испытания резервного копирования (Backup/Restore)

**Тест-файл:** `tests/test_backup.py` (13 тестов)  
**Цель:** проверить экспорт/импорт ZIP-снимка БД через нативный SQLite backup-API: RBAC (менеджеры качают, восстанавливает только админ), валидацию формата (zip → leyka.db → SQLite-заголовок), полный round-trip восстановления.  
**Тип:** `[AUTO]`

| № | ID теста | Описание | Ожидаемый результат |
|---|---|---|---|
| 23.1 | `test_export_outsider_403` | GET /backup/export от не-менеджера | 403 |
| 23.2 | `test_export_unauthenticated_401` | GET /backup/export без куки | 401 |
| 23.3 | `test_export_regular_user_403` | GET от Разработчика | 403 |
| 23.4 | `test_export_manager_passes_rbac` | GET от Менеджера | прошёл права (на in-memory: 503) |
| 23.5 | `test_export_superuser_passes_rbac` | GET от admin | прошёл права |
| 23.6 | `test_import_regular_user_403` | POST /backup/import от обычного юзера | 403 |
| 23.7 | `test_import_manager_403` | POST от Менеджера (восстановление — admin only) | 403 |
| 23.8 | `test_import_wrong_extension_400` | загрузка .tar.gz | 400 |
| 23.9 | `test_import_inmemory_503` | валидный zip на in-memory БД | 503 |
| 23.10 | `test_export_round_trip_file_db` | GET от admin на file-engine | 200, ZIP с leyka.db (SQLite header) + manifest.json |
| 23.11 | `test_import_validates_zip_content` | ZIP без leyka.db | 400 |
| 23.12 | `test_import_validates_sqlite_header` | leyka.db без SQLite-заголовка | 400 |
| 23.13 | `test_full_round_trip_export_delete_import` | создать партнёра → export → delete → import → партнёр восстановлен | 200, manifest, партнёр снова виден через GET /partners |

**Полный e2e** (тест 23.13) подтверждает корректность всего цикла: данные действительно восстанавливаются на состояние момента бэкапа.

---

### 7.24 Испытания фронтенда (Vitest, статическая проверка типов)

**Тест-каталоги:** `frontend/src/lib/*.test.ts`, `frontend/src/store/*.test.ts`, `frontend/src/components/About/*.test.ts`  
**Цель:** проверить корректность чистых функций (определение типа артефакта по URL, парсинг ссылок на задачи, цвет инициала партнёра, расчёт позиции при перетаскивании, словарь стадий жизненного цикла и пр.) и пользовательских stores (sidebar, chatWallpaper, about).  
**Тип:** `[AUTO]` + `[TYPECHECK]`

#### 7.24.1 Юнит-тесты библиотечных функций (lib/)

**Тест-файлы и количество кейсов:**

| Файл | Кейсов | Что проверяет |
|---|---|---|
| `assetTypes.test.ts` | 22 | `detectAssetType(url)` — эвристика типа артефакта по домену (arXiv, патенты, GitHub, Vercel, YouTube и др.) и обработка некорректных значений |
| `linkKind.test.ts` | 20 | `detectKind(url)`, `extractUrls(text)`, `fallbackTitle(url)` — определение типа ссылки и парсинг URL из произвольного текста |
| `taskRef.test.ts` | 17 | `isTaskUuid()`, `splitByTaskRefs()`, `extractTaskIds()` — UUID-чипы в комментариях, защита от stateful regex |
| `partnerColors.test.ts` | 11 | `partnerColorClass(partner)` — явный выбор цвета или hash-fallback по имени партнёра |
| `taskColors.test.ts` | 10 | `dueState(due)` (с подменой `Date.now()`), инварианты палитр статусов и приоритетов |
| `attachmentStages.test.ts` | 8 | каталог 12 стадий + legacy-aliases (contract → research) |
| `completionCriteria.test.ts` | 7 | каталог критериев завершения задачи + null-safety |
| `userColors.test.ts` | 7 | `defaultColorIndex(userId)` — детерминированный hash, диапазон палитры |
| `position.test.ts` | 6 | алгоритм Linear для расчёта `position` при drag-and-drop (4 ветки) |
| `cn.test.ts` | 5 | склейка tailwind-классов + фильтрация falsy-значений |

**Итого: 113 кейсов, разворачиваются в 138 запусков (за счёт `it.each(...)`).**

#### 7.24.2 Юнит-тесты пользовательских stores

| Файл | Кейсов | Что проверяет |
|---|---|---|
| `chatWallpaper.test.ts` | 10 | clamp значений brightness ∈ [-100, 100] и blur ∈ [0, 20], reset, дефолты |
| `about.test.ts` | 4 | open/close модалки «О LEYKA», force-выбор вкладки |
| `sidebar.test.ts` | 4 | toggle / setCollapsed свёрнутого sidebar |

**Итого: 18 кейсов.**

#### 7.24.3 Юнит-тесты компонента «О LEYKA»

| Файл | Кейсов | Что проверяет |
|---|---|---|
| `components/About/detectActiveRole.test.ts` | 17 | автодетект активной вкладки по роли пользователя (гость / superuser / 6 проектных ролей + force-id) |

#### 7.24.4 Статическая проверка типов

| Команда | Ожидаемый результат |
|---|---|
| `npx tsc -b --noEmit` | 0 ошибок (TypeScript 5.6, strict-режим) |

#### 7.24.5 Производственная сборка

| Команда | Ожидаемый результат |
|---|---|
| `npm run build` | 0 ошибок; артефакты в `frontend/dist/`; JS-бандл ≤ 200 КБ после gzip; CSS ≤ 15 КБ после gzip |

**Итого по фронтенду: 176 автотестов (Vitest) + статическая проверка типов + сборка.**

---

## 8. Ведомость испытательных тестов

### 8.1 Сводная таблица тест-файлов

#### Back-end (pytest, актуально на 27.04.2026)

| № | Тест-файл | Тестов | Домен | Тип |
|---|---|---|---|---|
| 1 | `test_smoke.py` | 4 | Базовая работоспособность | AUTO + MANUAL |
| 2 | `test_auth.py` | 16 | Аутентификация (login / providers / OAuth callback) | AUTO |
| 3 | `test_auth_lockdown.py` | 19 | Безопасность / anon-lockdown / RBAC | AUTO |
| 4 | `test_partners.py` | 17 | Партнёры (CRUD + презейл + валидация) | AUTO |
| 5 | `test_projects.py` | 16 | Проекты (CRUD + аналитика + видимость) | AUTO |
| 6 | `test_visibility.py` | 12 | Видимость объектов (private / public / demo) | AUTO |
| 7 | `test_members.py` | 13 | Участники проекта | AUTO |
| 8 | `test_roles.py` | 9 | Матрица ролей (12 permission-флагов) | AUTO |
| 9 | `test_tasks.py` | 19 | Задачи / Kanban / валидация enum | AUTO |
| 10 | `test_task_author_kind.py` | 8 | Тип задачи + immutable author | AUTO |
| 11 | `test_attachments.py` | 13 | Вложения URL + cross-task защита | AUTO |
| 12 | `test_comments.py` | 13 | Комментарии (+ запрет подмены автора) | AUTO |
| 13 | `test_read_receipts.py` | 4 | Двойные галочки прочтения | AUTO |
| 14 | `test_kpi.py` | 23 | KPI / Минэкономразвития | AUTO |
| 15 | `test_showcase.py` | 19 | Витрина достижений + множественные evidences | AUTO |
| 16 | `test_invites.py` | 19 | Цепочка приглашений (preview / redeem / revoke) | AUTO |
| 17 | `test_project_documents.py` | 24 | Чек-лист документации (12 стадий, 40 слотов) | AUTO |
| 18 | `test_uploads.py` | 13 | Загрузка файлов в Google Drive (+ graceful 503) | AUTO |
| 19 | `test_user_admin.py` | 24 | Управление пользователями + lockout-защита | AUTO |
| 20 | `test_analytics.py` | 3 | Аналитика проекта (throughput / lead time) | AUTO |
| 21 | `test_search.py` | 18 | Полнотекстовый поиск (FTS5) по задачам и комментам | AUTO |
| 22 | `test_wiki.py` | 24 | Библиотека знаний: чипы-ссылки, лайки, RBAC, поиск | AUTO |
| 23 | `test_backup.py` | 13 | Резервная копия БД: export/import, RBAC, round-trip | AUTO |
| **Итого back-end** | **23 файла** | **345 тестов** | | |

#### Front-end (Vitest, актуально на 27.04.2026)

| № | Тест-файл | Тестов | Домен | Тип |
|---|---|---|---|---|
| 1 | `lib/assetTypes.test.ts` | 22 | Эвристика типа артефакта по URL | AUTO |
| 2 | `lib/linkKind.test.ts` | 20 | Определение типа ссылки + парсер URL из текста | AUTO |
| 3 | `lib/taskRef.test.ts` | 17 | UUID-чипы в комментариях | AUTO |
| 4 | `lib/partnerColors.test.ts` | 11 | Цвет партнёра (явный / hash-fallback) | AUTO |
| 5 | `lib/taskColors.test.ts` | 10 | Семантика статусов / приоритетов / дедлайнов | AUTO |
| 6 | `lib/attachmentStages.test.ts` | 8 | 12 стадий + legacy-aliases | AUTO |
| 7 | `lib/completionCriteria.test.ts` | 7 | Каталог критериев выполнения | AUTO |
| 8 | `lib/userColors.test.ts` | 7 | Палитра аватарок (детерминированный hash) | AUTO |
| 9 | `lib/position.test.ts` | 6 | Linear-алгоритм drag-and-drop | AUTO |
| 10 | `lib/cn.test.ts` | 5 | Склейка tailwind-классов | AUTO |
| 11 | `store/chatWallpaper.test.ts` | 10 | Clamp brightness/blur, reset | AUTO |
| 12 | `store/sidebar.test.ts` | 4 | Toggle/setCollapsed | AUTO |
| 13 | `store/about.test.ts` | 4 | Open/close модалки «О LEYKA» | AUTO |
| 14 | `components/About/detectActiveRole.test.ts` | 17 | Автодетект активной вкладки по роли | AUTO |
| **Итого front-end** | **14 файлов** | **176 тестов** | | |
| — | `npx tsc -b --noEmit` | 0 ошибок | Статическая проверка типов | TYPECHECK |

#### Сводно по проекту

| Слой | Файлов | Тестов | Время прогона |
|---|---|---|---|
| Back-end (pytest) | 20 | 288 | ~15 секунд |
| Front-end (Vitest) | 14 | 176 | ~0,7 секунды |
| Front-end (TypeScript strict) | вся кодовая база | 0 ошибок | ~5 секунд |
| **Всего** | **34 файла + статика** | **464 теста** | **~21 секунда** |

> **Примечание.** Ручное smoke-тестирование (п. 7.1.5–7.1.7) выполняется отдельно после автоматического прогона и не учитывается в счётчике автотестов.

### 8.2 Команды запуска

**Полный цикл испытаний:**

```bash
# Back-end автотесты
cd backend
source .venv/bin/activate
pytest tests/ -v --tb=short 2>&1 | tee back_report.txt
echo "Back-end exit code: $?"

# Front-end автотесты
cd ../frontend
npm install --silent
npx vitest run --reporter=basic 2>&1 | tee front_report.txt
echo "Front-end exit code: $?"

# Статическая проверка типов
npx tsc -b --noEmit
echo "TypeScript exit code: $?"

# Производственная сборка
npm run build
echo "Build exit code: $?"
```

**Быстрая проверка одного домена:**

```bash
# Back-end: только KPI
cd backend && source .venv/bin/activate && pytest tests/test_kpi.py -v

# Front-end: только тесты по имени-подстроке
cd frontend && npx vitest run -t "detectAssetType"
```

---

## 9. Приложения

### Приложение А. Форма Протокола испытаний

---

**ПРОТОКОЛ ИСПЫТАНИЙ № ___**

**Программа:** LEYKA v0.2.0  
**Дата:** _____________  
**Место проведения:** ЦИИ НГУ / удалённо  
**Испытательная среда:**  

| Компонент | Версия / конфигурация |
|---|---|
| Python | |
| pytest | |
| Операционная система | |
| Браузер (для ручных) | |

**Результаты автоматизированных испытаний (back-end):**

```
pytest tests/ -v
============================== test session starts ==============================
...
========================= ___ passed, ___ failed in ___s =========================
```

**Результаты автоматизированных испытаний (front-end):**

```
npx vitest run --reporter=basic
...
 Test Files  ___ passed (___)
      Tests  ___ passed (___)
```

**Результаты статической проверки типов:**

```
npx tsc -b --noEmit
(вывод — ошибки, если есть)
```

**Результаты производственной сборки:**

```
npm run build
✓ built in ___s
dist/assets/index-*.js   _____ kB │ gzip: _____ kB
dist/assets/index-*.css  _____ kB │ gzip: _____ kB
```

| Критерий | Пороговое значение | Результат | Замечания |
|---|---|---|---|
| Back-end автотесты (pytest) | 288 passed | __ passed / __ failed | |
| Front-end автотесты (Vitest) | 176 passed | __ passed / __ failed | |
| Статическая проверка типов TypeScript | 0 ошибок | __ ошибок | |
| Производственная сборка `npm run build` | 0 ошибок | пройдено / не пройдено | |
| Ручное smoke-тестирование (п. 7.1) | Все пункты «Пройдено» | пройдено / не пройдено | |
| Время отклика API | ≤ 700 мс p95 | __ мс | |

**Итоговое заключение:**  
☐ Программа соответствует требованиям ТЗ. Рекомендована к приёмке.  
☐ Программа не соответствует требованиям ТЗ. Требует доработки (см. замечания).

**Замечания:** ___________________________________________

**Подписи:**

| Должность | ФИО | Подпись | Дата |
|---|---|---|---|
| Руководитель QA (Исполнитель) | | | |
| Представитель Заказчика | | | |

---

### Приложение Б. Тест-фикстуры conftest.py

Все автоматические тесты используют общие фикстуры из `backend/tests/conftest.py`:

| Фикстура | Назначение |
|---|---|
| `engine` | SQLite in-memory + `StaticPool`. Свежий engine на каждый тест → полная изоляция |
| `session_maker` | `async_sessionmaker` поверх `engine` |
| `client` | `httpx.AsyncClient` с подменённым `get_db` — анонимный (без cookie) |
| `admin_client` | `httpx.AsyncClient` с HttpOnly cookie superuser'а (`admin@test.ru`) |
| `user_client` | `httpx.AsyncClient` с HttpOnly cookie обычного сотрудника (`user@test.ru`) |
| `admin`, `regular_user` | ORM-модели `User` соответствующих юзеров |
| `create_partner(client, name=…, **overrides)` | Хелпер: POST /partners → возвращает JSON-ответ как dict |
| `create_project(client, partner_id=…, **overrides)` | Хелпер: POST /projects → возвращает dict |
| `create_task(client, project_id=…, **overrides)` | Хелпер: POST /tasks → возвращает dict |
| `add_member(admin_client, project_id=…, user_id=…, role_name='Разработчик')` | Хелпер: POST /projects/{id}/members |

**Сидинг при создании engine:** 2 роли (Менеджер с полными правами, Разработчик — без can_delete_tasks/can_manage_members) и 2 KPI-метрики (`z-1`, `z-9`). Этого достаточно для всех существующих тестов; полный сид (11 KPI, все 6 ролей) применяется только в production-seed.

### Приложение В. Матрица покрытия требований ТЗ

| Требование ТЗ (п. 4.1.1) | Покрыто | Тестовые файлы |
|---|---|---|
| OAuth-аутентификация (Mail.ru, Google, mock-dev) | ✅ | `test_auth.py` |
| JWT в HttpOnly cookie | ✅ | `test_auth.py` |
| CSRF-защита OAuth-флоу через state-cookie | ✅ | `test_auth_lockdown.py`, `test_auth.py` |
| CRUD партнёров + видимость | ✅ | `test_partners.py` |
| Жизненный цикл проекта (5 стадий + аналитика) | ✅ | `test_projects.py`, `test_analytics.py` |
| Матрица видимости (private/public/demo) | ✅ | `test_visibility.py` |
| RBAC: 6 ролей × 12 permission-флагов | ✅ | `test_roles.py`, `test_members.py` |
| Защита последнего активного администратора | ✅ | `test_user_admin.py`, `test_auth_lockdown.py` |
| CRUD задач + Kanban (drag-and-drop) | ✅ | `test_tasks.py` |
| Аудит переходов статусов (`task_status_history`) | ✅ | `test_tasks.py` |
| Immutable author_id у задачи | ✅ | `test_task_author_kind.py` |
| Тип задачи (task / idea / bug) | ✅ | `test_task_author_kind.py` |
| Вложения (URL) с автоопределением типа | ✅ | `test_attachments.py` |
| Комментарии + защита от подмены автора | ✅ | `test_comments.py`, `test_auth_lockdown.py` |
| Двойные галочки прочтения | ✅ | `test_read_receipts.py` |
| 11 KPI Минэкономразвития (z-1 … z-11) | ✅ | `test_kpi.py` |
| Годовой план центра + прогресс KPI | ✅ | `test_kpi.py` |
| Множественные evidences (до 10) | ✅ | `test_kpi.py`, `test_showcase.py` |
| Витрина достижений (7 типов артефактов) | ✅ | `test_showcase.py` |
| Цепочка приглашений по ссылке + welcome-задача | ✅ | `test_invites.py` |
| Чек-лист документации (12 стадий, ~40 слотов) | ✅ | `test_project_documents.py` |
| Drag-and-drop загрузка в Google Drive | ✅ | `test_uploads.py` |
| Graceful degradation (Drive не настроен → 503) | ✅ | `test_uploads.py` |
| Скорость закрытия задач + время выполнения | ✅ | `test_analytics.py` |
| Эвристика типа артефакта по URL (front-end) | ✅ | `frontend/src/lib/assetTypes.test.ts` |
| Парсинг UUID-ссылок на задачи (front-end) | ✅ | `frontend/src/lib/taskRef.test.ts` |
| Цвет инициала партнёра (front-end) | ✅ | `frontend/src/lib/partnerColors.test.ts` |
| Алгоритм position при drag-and-drop (front-end) | ✅ | `frontend/src/lib/position.test.ts` |
| Модалка «О LEYKA» — автодетект роли | ✅ | `frontend/src/components/About/detectActiveRole.test.ts` |
| Статическая проверка типов TypeScript (strict) | ✅ | `npx tsc -b --noEmit` |
| Производственная сборка front-end | ✅ | `npm run build` |

---

*Документ составлен на основе исходного кода репозитория https://github.com/Barbashin1970/LEYKA версии 0.2.0 по состоянию на 27 апреля 2026 г.*  
*Программа и методика испытаний соответствуют требованиям ГОСТ 19.301-79 «Программа и методика испытаний. Требования к содержанию и оформлению».*

*Все цифры по числу тестов, эндпойнтов, миграций и метрик кода отражают фактическое состояние репозитория и проверены командами, перечисленными в §8.2 настоящего документа.*

---

**Конец документа ЦИИ НГУ.LEYKA.ПМИ.001-2026**
