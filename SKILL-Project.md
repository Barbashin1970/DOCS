# SKILL: FastAPI \& SQLAlchemy 2.0 Backend for NSU AI Center Task Tracker

## 1. Контекст проекта (Project Context)

Вы — Senior Python Backend Developer. Ваша задача — разработать отказоустойчивый и масштабируемый бэкенд для системы управления проектами Центра искусственного интеллекта НГУ (ЦИИ НГУ).
Система предназначена для административного управления (бюджеты, переговоры, проекты, ресурсы, задачи) и должна выдерживать высокую конкурентность записи (множество пользователей одновременно обновляют Канбан-доски).

## 2. Технологический стек (Tech Stack)

Всегда используйте следующие технологии и их современные стандарты:

- **Framework:** FastAPI (строго асинхронный).
- **Database:** PostgreSQL.
- **ORM:** SQLAlchemy 2.0 (строго асинхронный движок, использование `async_sessionmaker` и `AsyncSession`).
- **DB Driver:** `asyncpg` (для максимальной пропускной способности OLTP).
- **Migrations:** Alembic (с поддержкой асинхронных миграций).
- **Validation:** Pydantic v2.


## 3. Архитектурные принципы (Core Rules)

При генерации кода строго соблюдайте следующие правила:

1. **Асинхронность везде:** Все роуты, обращения к БД и фоновые задачи должны быть асинхронными (`async def`, `await`). Никогда не используйте синхронные вызовы, блокирующие Event Loop.
2. **Сессии БД:** Используйте Dependency Injection (через `Depends` в FastAPI) для передачи `AsyncSession` в эндпоинты. Сессия должна автоматически закрываться/откатываться при ошибках.
3. **SQLAlchemy 2.0 Style:** Используйте новый синтаксис 2.0. Никаких `session.query()`. Используйте конструкцию `select(...)`, `session.execute()`, `session.scalars()`.
4. **Слои абстракции:** Разделяйте слой моделей (Models - SQLAlchemy), слой схем (Schemas - Pydantic) и слой бизнес-логики (CRUD/Services). Не смешивайте бизнес-логику с эндпоинтами роутера.

## 4. Доменная модель (Database Schema)

Система строится вокруг 5 ключевых сущностей. При проектировании моделей SQLAlchemy реализуйте следующие таблицы и связи (Foreign Keys):

- **Partners (Заказчики/Партнеры):**
    - Поля: `id`, `name` (например, "МТС", "РТК"), `type`, `contact_person`, `status`.
- **Projects (Инициативы):**
    - Связь: `partner_id` (FK -> Partners).
    - Поля: `id`, `title` (например, "5G-ядро"), `budget_allocated`, `budget_spent`, `start_date`, `end_date`.
- **Users (Сотрудники ЦИИ):**
    - Поля: `id`, `full_name`, `role` (PM, Developer, Analyst), `department`.
- **Tasks (Канбан-задачи):**
    - Связи: `project_id` (FK -> Projects), `assignee_id` (FK -> Users).
    - Поля: `id`, `title`, `description`, `status` (Enum: Backlog, To Do, In Progress, Waiting, Review, Done), `priority`, `due_date`, `estimated_hours`.
- **TaskStatusHistory (Аналитика переходов):**
    - Связи: `task_id` (FK -> Tasks), `changed_by` (FK -> Users).
    - Поля: `id`, `old_status`, `new_status`, `changed_at` (timestamp).
    - *Правило триггера:* При любом обновлении поля `status` в таблице `Tasks`, автоматически (через SQLAlchemy Events или слой сервиса) создавайте запись в этой таблице.
- **Comments (Мессенджер/Обсуждения):**
    - Связи: `task_id` (FK -> Tasks), `author_id` (FK -> Users).
    - Поля: `id`, `text`, `created_at`.


## 5. API Design \& Kanban Logic

При реализации REST API роутов учитывайте логику работы Канбан-доски:

1. **Drag-and-Drop Endpoint:** Реализуйте метод `PATCH /tasks/{task_id}/status`, который будет атомарно менять статус задачи и сразу записывать лог в `TaskStatusHistory`.
2. **Очередность (Ordering):** У задач в колонке должно быть поле `position` (float или int) для сохранения пользовательской сортировки при перетаскивании.
3. **Фильтрация:** Эндпоинт `GET /tasks` должен поддерживать фильтрацию по `project_id`, `assignee_id` и `status` для отрисовки конкретной доски.

## 6. Транзакции и безопасность (Concurrency)

- Используйте механизмы изоляции транзакций PostgreSQL. При изменении критичных данных (например, бюджетов или статусов) используйте `select(...).with_for_update()` для предотвращения состояния гонки (Race Conditions) при параллельных запросах.
- При возникновении исключений (Exceptions) на уровне БД, отлавливайте их в глобальном `exception_handler` FastAPI, откатывайте транзакцию (`await session.rollback()`) и возвращайте клиенту понятный HTTP 400/500 ответ.
Это ТЗ описывает идеальную архитектуру, которая не "зависнет" при параллельном доступе и легко адаптируется под любые будущие процессы Центра ИИ.
