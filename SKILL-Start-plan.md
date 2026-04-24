<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# перепиши этот файл на нашем стеке технологий и замени Инициативы на Проекты и т д - просто чтобы у меня был хотя бы псевдо код а лучше заготовка как референс для клод-кода

Я подготовил для вас заготовку — идеальный референс (псевдокод), который переводит вашу логику из DuckDB (`task_store.py`) на нашу новую целевую архитектуру (FastAPI + SQLAlchemy 2.0 Async + Pydantic V2).[^1][^2][^3]

Этот код демонстрирует, как правильно связать БД, ORM-модели, схемы валидации и маршрутизацию (роуты) без блокировок.

### 1. Структура папок (Clean Architecture)

В промышленной разработке все файлы не сваливают в один `task_store.py`. Для вашего агента (или для вас) создайте такую структуру:

```text
project_root/
├── app/
│   ├── main.py                  # Точка входа FastAPI
│   ├── database.py              # Настройки подключения к PostgreSQL (asyncpg)
│   ├── models/                  # SQLAlchemy ORM (Таблицы БД)
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── project.py
│   │   ├── task.py
│   │   └── user.py
│   ├── schemas/                 # Pydantic модели (Валидация Request/Response)
│   │   ├── __init__.py
│   │   ├── project.py
│   │   └── task.py
│   ├── api/                     # Эндпоинты (Routers)
│   │   ├── dependencies.py      # Dependency Injection (get_db, auth)
│   │   └── v1/
│   │       └── tasks.py
│   └── services/                # Бизнес-логика (например, Drag&Drop)
│       └── task_service.py
├── requirements.txt
└── railway.json                 # Конфиг деплоя
```


### 2. Настройка БД (`app/database.py`)

Здесь мы настраиваем асинхронный движок (Engine) и генератор сессий.[^3][^4]

```python
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

# Для локальной разработки можно использовать SQLite с aiosqlite, 
# а на Railway подхватывать PostgreSQL из ENV-переменной.
DATABASE_URL = "postgresql+asyncpg://postgres:pass@localhost:5432/aicenter"

engine = create_async_engine(
    DATABASE_URL, 
    echo=True, # Включаем логирование SQL
    pool_size=20,
    max_overflow=10
)

# Фабрика асинхронных сессий
async_session_maker = async_sessionmaker(
    engine, 
    class_=AsyncSession, 
    expire_on_commit=False
)

# Зависимость (Dependency) для FastAPI
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```


### 3. Слой Моделей SQLAlchemy (`app/models/task.py`)

Заменяем сырой SQL из вашего `TASKSDDL` на декларативные классы SQLAlchemy 2.0.[^3]

```python
import enum
import uuid
from datetime import datetime
from sqlalchemy import String, Float, Enum, ForeignKey, DateTime
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.models.base import Base # Base = declarative_base()

class TaskStatus(str, enum.Enum):
    inbox = "inbox"
    todo = "todo"
    in_progress = "in_progress"
    waiting = "waiting"
    review = "review"
    done = "done"

class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    project_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("projects.id"))
    assignee_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("users.id"))
    
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(String)
    status: Mapped[TaskStatus] = mapped_column(Enum(TaskStatus), default=TaskStatus.inbox)
    position: Mapped[float] = mapped_column(Float, default=0.0) # Для Drag&Drop Канбана
    
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    
    # Связи (Relationships) для авто-подгрузки
    project = relationship("Project", back_populates="tasks")
    assignee = relationship("User", back_populates="tasks")
```


### 4. Слой Схем Pydantic (`app/schemas/task.py`)

Здесь мы описываем, как данные должны валидироваться при входе от фронтенда (React/Vue) и как отдаваться обратно.[^5]

```python
from pydantic import BaseModel, ConfigDict, Field
from uuid import UUID
from datetime import datetime
from app.models.task import TaskStatus

# Схема для создания задачи (Request)
class TaskCreate(BaseModel):
    title: str = Field(..., min_length=3, max_length=200)
    description: str | None = None
    project_id: UUID

# Схема для перемещения карточки в Канбане (Request)
class TaskMove(BaseModel):
    status: TaskStatus
    position: float

# Схема для отдачи задачи фронтенду (Response)
class TaskResponse(BaseModel):
    id: UUID
    title: str
    status: TaskStatus
    position: float
    created_at: datetime
    
    # Магия Pydantic: разрешаем читать данные из SQLAlchemy-объекта
    model_config = ConfigDict(from_attributes=True) 
```


### 5. Роуты (API) с бизнес-логикой (`app/api/v1/tasks.py`)

Вот как выглядит правильный асинхронный CRUD (создание и перемещение карточек), защищенный от блокировок.[^5][^6]

```python
from typing import List
from uuid import UUID
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.database import get_db
from app.models.task import Task
from app.schemas.task import TaskCreate, TaskResponse, TaskMove

router = APIRouter(prefix="/tasks", tags=["Tasks"])

@router.post("", response_model=TaskResponse, status_code=status.HTTP_201_CREATED)
async def create_task(
    payload: TaskCreate, 
    db: AsyncSession = Depends(get_db)
):
    # Создаем SQLAlchemy модель из Pydantic схемы
    new_task = Task(**payload.model_dump())
    db.add(new_task)
    await db.commit()
    await db.refresh(new_task)
    return new_task

@router.get("", response_model=List[TaskResponse])
async def list_tasks(
    project_id: UUID | None = None,
    db: AsyncSession = Depends(get_db)
):
    # SQLAlchemy 2.0 Select Statement
    stmt = select(Task).order_by(Task.position)
    
    if project_id:
        stmt = stmt.where(Task.project_id == project_id)
        
    result = await db.execute(stmt)
    tasks = result.scalars().all()
    return tasks

@router.patch("/{task_id}/move", response_model=TaskResponse)
async def move_task(
    task_id: UUID, 
    payload: TaskMove, 
    db: AsyncSession = Depends(get_db)
):
    # select(...).with_for_update() гарантирует, что пока мы двигаем 
    # карточку, другой юзер не сможет ее изменить (защита от Race Condition)
    stmt = select(Task).where(Task.id == task_id).with_for_update()
    result = await db.execute(stmt)
    task = result.scalar_one_or_none()
    
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")
        
    # Логика обновления (Drag & Drop)
    task.status = payload.status
    task.position = payload.position
    
    await db.commit()
    await db.refresh(task)
    
    # (Здесь в будущем нужно будет вызвать сервис для записи в TaskStatusHistory)
    
    return task
```


### 6. Точка входа (`app/main.py`)

Собираем всё вместе.

```python
from fastapi import FastAPI
from app.api.v1 import tasks

app = FastAPI(
    title="NSU AI Center Task Tracker API",
    description="FastAPI + PostgreSQL + SQLAlchemy 2.0",
    version="1.0.0"
)

# Подключаем наши роуты
app.include_router(tasks.router, prefix="/api/v1")

@app.get("/health")
async def health_check():
    return {"status": "ok"}
```

Вы можете скопировать этот код и отдать Клоду вместе с `SKILL.md`. Он сразу поймет заданный стандарт качества (использование `Mapped`, `async_sessionmaker`, `with_for_update`, `Depends`) и легко допишет остальные сущности (Projects, Partners, Users) и аналитику переходов.
<span style="display:none">[^10][^11][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://blog.petrousoft.com/using-sqlalchemy-2-0-to-implement-crud-operations-with-fastapi-a56b040571ba

[^2]: https://www.c-sharpcorner.com/article/how-to-build-a-rest-api-with-fastapi-and-async-operations/

[^3]: https://oneuptime.com/blog/post/2026-02-06-instrument-async-sqlalchemy-2-opentelemetry/view

[^4]: https://praciano.com.br/fastapi-and-async-sqlalchemy-20-with-pytest-done-right.html

[^5]: https://berkkaraal.com/blog/2024/09/19/setup-fastapi-project-with-async-sqlalchemy-2-alembic-postgresql-and-docker/

[^6]: https://chaoticengineer.hashnode.dev/fastapi-sqlalchemy

[^7]: task_store.py

[^8]: https://testdriven.io/blog/fastapi-sqlmodel/

[^9]: https://dev.to/akarshan/asynchronous-database-sessions-in-fastapi-with-sqlalchemy-1o7e

[^10]: https://www.youtube.com/watch?v=I8WiIXMDydw

[^11]: https://stackoverflow.com/questions/76539549/pydantic-with-asyncattrs-sqlalchemy

