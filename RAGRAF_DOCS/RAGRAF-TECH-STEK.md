# Технологический стек ПО «РАГРАФ»

> Документ оформлен по образцу отраслевого «Технологического стека ПО» ЦИИ НГУ
> (ср. ИИ 11711-2025-66-1-ИИ-01). Версии и лицензии — фактические, из
> `backend/requirements.txt`, `frontend/package.json`, `Dockerfile` (на 2026-06-05).

---

## Лист утверждения

**МИНИСТЕРСТВО НАУКИ И ВЫСШЕГО ОБРАЗОВАНИЯ РОССИЙСКОЙ ФЕДЕРАЦИИ**

Тема разработки: **СИГМА**
Основание: Договор о предоставлении средств юридическому лицу, индивидуальному
предпринимателю на безвозмездной и безвозвратной основе в форме гранта, источником
финансового обеспечения которых полностью или частично является субсидия, предоставленная
из федерального бюджета, от 27 декабря 2023 г. № 70-2023-001318.
Грантодатель: Министерство экономического развития Российской Федерации.

Программное обеспечение **«РАГРАФ»** — стенд для уточнения требований и пользовательских
требований к фреймворку Σ СИГМА (замкнутый контур «датчик → регламент → задача → человек»).

**ТЕХНОЛОГИЧЕСКИЙ СТЕК ПО**
Идентификатор документа: РАГРАФ-ТС-2026-01 *(присваивается при регистрации)*
Листов: настоящий документ.

| | СОГЛАСОВАНО | УТВЕРЖДАЮ |
|---|---|---|
| Организация | — | ФГАОУ ВО «Новосибирский национальный исследовательский государственный университет» (НГУ) |
| Должность | Директор ЦИИ НГУ | Ректор НГУ |
| Подпись / дата | ______________ / ____.2026 | ______________ / ____.2026 |

| Роль | Должность | Подпись / дата |
|---|---|---|
| Руководитель разработки | Ведущий научный сотрудник ЦИИ НГУ | ______________ / ____.2026 |
| Нормоконтролёр | Ведущий специалист ЦИИ НГУ | ______________ / ____.2026 |

Новосибирск, 2026

---

## Содержание

1. [Введение](#1-введение)
2. [Технологический стек](#2-технологический-стек)
   - 2.1 [Перечень используемых ОС, СУБД, платформ, средств виртуализации](#21-перечень-используемых-ос-субд-платформ-средств-виртуализации)
   - 2.2 [Перечень библиотек и фреймворков с версиями](#22-перечень-библиотек-и-фреймворков-с-версиями)
   - 2.3 [Технические средства хранения исходного текста и объектного кода](#23-технические-средства-хранения-исходного-текста-и-объектного-кода)
3. [Список компонентов с лицензиями](#3-список-компонентов-с-лицензиями)
4. [Примечания: анализ рисков и рекомендации](#4-примечания-анализ-рисков-и-рекомендации)

---

## 1. Введение

Настоящий документ содержит подробное описание технологического стека, состоящего из данных:

- перечень используемых ОС, СУБД, платформ, средств виртуализации;
- перечень библиотек и фреймворков с версиями;
- перечень необходимых сторонних компонентов и систем;
- сведения о технических средствах хранения исходного текста и объектного кода ПО;
- сведения о технических средствах компиляции/сборки исходного текста в объектный код ПО,

необходимых для установки и работы ПО **«РАГРАФ»** — программного стенда замкнутого контура
управления инфраструктурой, целью которого является приём событий от датчиков, таск-трекеров и
внешних систем, сверка их с машиноисполняемыми регламентами и формирование объяснимых
рекомендаций/задач исполнителю с проверкой исхода.

Документ предназначен для архитекторов, технических специалистов, экспертов по безопасности, а
также представителей органов регистрации и сопровождения программных продуктов — как основание
для подтверждения корректного ведения, лицензирования, хранения и обновления исходного и
объектного кода ПО на всех стадиях жизненного цикла.

**Архитектурная особенность.** РАГРАФ — лёгкий стенд: единый процесс `uvicorn` (FastAPI отдаёт
REST API и SPA-статику), встраиваемое хранилище DuckDB, регламенты как файлы (`flow.json` +
W3C-Turtle/SHACL/PROV-O). **Без GPU/CUDA, без оркестратора, без отдельной серверной СУБД** —
разворачивается на одном узле (минимально 2 CPU / 4 ГБ / 50 ГБ). Это отличает стенд от тяжёлых
ML-детекторов и снижает инфраструктурные и санкционные риски (см. § 4).

> **Статус ПО.** «РАГРАФ» — проприетарный лицензируемый продукт, созданный в рамках НИОКТР
> (тема СИГМА). Открытость используемых компонентов (§ 3) не делает ПО «РАГРАФ» в целом
> открытым. Право использования предоставляется по отдельной лицензии (см. файл `LICENSE`).

---

## 2. Технологический стек

### 2.1 Перечень используемых ОС, СУБД, платформ, средств виртуализации

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| РЕД ОС | Российская ОС на базе Linux, сертифицирована ФСТЭК России, в Реестре российского ПО. Рекомендуемая целевая ОС для импортозамещающего развёртывания | 7.3+ | https://redos.red-soft.ru/downloads/ |
| Astra Linux | Альтернативная российская ОС (сертификация ФСТЭК/МО). Поддерживается как целевая (Docker) | SE 1.7+ | https://astralinux.ru |
| Ubuntu Server | Дистрибутив Linux LTS для серверов; среда разработки и совместимая среда исполнения | 22.04+ LTS | https://ubuntu.com/download/server |
| python:3.12-slim | Базовый образ контейнера времени исполнения (Debian-slim). Реальная runtime-среда стенда | 3.12 | https://hub.docker.com/_/python |
| DuckDB | Встраиваемая аналитическая СУБД (in-process, один файл). **Источник истины** для регламентов, параметров, триггеров, метрик, истории версий. Отдельный серверный процесс не требуется | ≥ 1.2, < 2 | https://duckdb.org |
| Docker | Контейнеризация: сборка переносимого образа и запуск стенда (моноконтейнер SPA + FastAPI) | Engine 24+ | https://docs.docker.com/engine/install/ |

> Долговременное хранение пользовательского состояния — локальный filesystem на Volume
> (`${DATA_DIR}`): файл `regulations.duckdb`, `data/flows/*.json`, `data/versions/`,
> `data/knowledge/`. Без S3 и облачных СУБД. Регламенты переносятся файлами (W3C-стек).

### 2.2 Перечень библиотек и фреймворков с версиями

#### 2.2.1 Backend — веб-фреймворк и API

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| FastAPI | Веб-фреймворк REST API: автогенерация OpenAPI, асинхронность. Ядро всех серверных эндпоинтов | 0.115.4 | https://pypi.org/project/fastapi/ |
| Uvicorn | ASGI-сервер (`[standard]`), единый рабочий процесс приложения | 0.32.0 | https://pypi.org/project/uvicorn/ |
| python-multipart | Разбор multipart/form-data (загрузка документов аналитиком) | 0.0.12 | https://pypi.org/project/python-multipart/ |
| HTTPX | Асинхронный HTTP-клиент для интеграций (ETL-источники, LLM-провайдеры) | 0.27.2 | https://pypi.org/project/httpx/ |
| Pydantic | Валидация и типизация моделей предметной области (регламент, узлы Flow) | 2.9.2 | https://pypi.org/project/pydantic/ |
| pydantic-settings | Конфигурация из окружения/`.env` | 2.6.1 | https://pypi.org/project/pydantic-settings/ |
| python-dotenv | Загрузка переменных окружения из `.env` | 1.0.1 | https://pypi.org/project/python-dotenv/ |

#### 2.2.2 Backend — семантический слой и знания

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| rdflib | Работа с RDF/Turtle: экспорт/импорт регламентов, PROV-O происхождение (interop с Σ СИГМА) | 7.1.1 | https://pypi.org/project/rdflib/ |
| pySHACL | Валидация данных по SHACL-формам (формальная проверяемость регламентов) | 0.26.0 | https://pypi.org/project/pyshacl/ |
| NetworkX | Графовые операции (анализ/обход графов регламентов и валидатор) | 3.4.2 | https://pypi.org/project/networkx/ |
| PyYAML | Разбор YAML (конфигурация, вспомогательные импортеры) | 6.0.2 | https://pypi.org/project/PyYAML/ |

#### 2.2.3 Backend — хранение данных

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| DuckDB (python) | Драйвер встраиваемой СУБД (источник истины) | ≥ 1.2, < 2 | https://pypi.org/project/duckdb/ |

#### 2.2.4 Backend — опциональный ИИ и обработка документов

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| OpenAI Python SDK | Клиент любого OpenAI-совместимого endpoint (Cerebras/Groq/Ollama/OpenAI). Импорт ленивый; **опционален** — без него стенд работает в mock-режиме | 2.36.0 | https://pypi.org/project/openai/ |
| pypdf | Извлечение текста из PDF (загрузка нормативных документов как контекст) | 5.1.0 | https://pypi.org/project/pypdf/ |
| python-docx | Извлечение текста из DOCX | 1.1.2 | https://pypi.org/project/python-docx/ |

#### 2.2.4.1 Локальный ИИ-контур (опционально, импортозамещающий)

ИИ-контур построен как тонкий слой над OpenAI-совместимым API (`llm_provider` по умолчанию —
**`ollama`**, локальный), что позволяет запускать **локально скачиваемые открытые модели** без
обращения к зарубежным облачным сервисам. Контур опционален: при `EMBEDDINGS_ENABLED=false` и
отсутствии LLM-ключа (значения по умолчанию) стенд работает в mock-режиме. Конфигурация —
`backend/app/config.py` (`llm_provider`, `llm_model`, `embed_model`).

| Компонент | Описание | Версия / модель | Ссылка |
|---|---|---|---|
| Ollama | Локальный сервер инференса LLM/эмбеддингов (OpenAI-совместимый REST). Запускает скачанные модели на CPU/GPU без облака | Latest stable | https://ollama.com |
| Qwen (LLM) | Открытые веса (Alibaba/Qwen), квантизация GGUF; скачивается через Ollama. Локальная генерация в чате аналитика | Qwen2.5-Instruct / Qwen3 (напр. `qwen2.5:7b-instruct`) | https://ollama.com/library/qwen2.5 |
| bge-m3 (эмбеддинги) | Открытый мультиязычный эмбеддер (BAAI), хорош для русского; локально через Ollama. Retrieval по корпусу регламентов | bge-m3 | https://huggingface.co/BAAI/bge-m3 |
| Qwen3-Embedding (эмбеддинги) | Альтернативный открытый эмбеддер (Alibaba/Qwen); поле `embed_model` (напр. `qwen3-embedding-8b`) | Qwen3-Embedding | https://huggingface.co/Qwen/Qwen3-Embedding-8B |

> Модели — **скачиваемые офлайн артефакты с открытыми весами** (Apache 2.0 / MIT). Это
> импортозамещающая альтернатива зарубежным облачным LLM: локальный приватный инференс, без
> передачи данных третьим лицам, работа в изолированном контуре заказчика. См. § 4.1.

#### 2.2.5 Backend — тестирование

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| pytest | Каркас модульных/интеграционных тестов | 8.3.4 | https://pypi.org/project/pytest/ |
| pytest-asyncio | Поддержка async-тестов | 0.24.0 | https://pypi.org/project/pytest-asyncio/ |

#### 2.2.6 Frontend — UI / SPA

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| React | Библиотека пользовательского интерфейса (SPA) | 18.3.1 | https://www.npmjs.com/package/react |
| react-dom | Рендеринг React в DOM | 18.3.1 | https://www.npmjs.com/package/react-dom |
| react-router-dom | Клиентская маршрутизация | 6.28.0 | https://www.npmjs.com/package/react-router-dom |
| TypeScript | Статически типизированный надмножество JS (исходный язык фронтенда) | 5.6.3 | https://www.npmjs.com/package/typescript |
| Vite | Сборщик и dev-сервер фронтенда | 5.4.10 | https://www.npmjs.com/package/vite |
| Tailwind CSS | Утилитарный CSS-фреймворк | 3.4.14 | https://www.npmjs.com/package/tailwindcss |
| React Flow (reactflow) | Канвас визуального редактора регламентов (Flow DSL) | 11.11.4 | https://www.npmjs.com/package/reactflow |
| Zustand | Управление клиентским состоянием | 5.0.1 | https://www.npmjs.com/package/zustand |
| TanStack Query | Управление серверным состоянием (кэш запросов) | 5.59.16 | https://www.npmjs.com/package/@tanstack/react-query |
| Zod | Валидация схем данных на клиенте | 4.4.3 | https://www.npmjs.com/package/zod |
| Cytoscape (+cola) | Визуализация графа знаний | 3.30.2 / 2.5.1 | https://www.npmjs.com/package/cytoscape |
| dnd-kit (core + sortable) | Drag-and-drop в интерфейсе | 6.3.1 / 8.0.0 | https://www.npmjs.com/package/@dnd-kit/core |
| lucide-react | Набор иконок | 0.469.0 | https://www.npmjs.com/package/lucide-react |

#### 2.2.7 Frontend — инструменты сборки и контроля качества

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| @vitejs/plugin-react | Плагин Vite для React | 4.3.3 | https://www.npmjs.com/package/@vitejs/plugin-react |
| ESLint + typescript-eslint | Статический анализ кода | 9.39.4 / 8.59.3 | https://www.npmjs.com/package/eslint |
| PostCSS + Autoprefixer | Постобработка CSS | 8.4.47 / 10.4.20 | https://www.npmjs.com/package/postcss |
| Vitest | Юнит-тесты фронтенда | 2.1.9 | https://www.npmjs.com/package/vitest |

### 2.3 Технические средства хранения исходного текста и объектного кода

#### 2.3.1 Системы контроля версий (хранение исходного текста)

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| Git | Распределённая СКВ: история, ветвление, слияние | Latest stable | https://git-scm.com/downloads |
| GitHub | Текущее размещение репозитория стенда | — | https://github.com/Barbashin1970/RAGRAF |
| GitLab (on-premise НГУ) | **Рекомендуемое** размещение лицензируемого ПО: code review, CI/CD, локальная инфраструктура | On-premise | https://gitlab.ci.nsu.ru |

#### 2.3.2 Реестры контейнеров и хранилища (хранение объектного кода)

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| GitLab Container Registry (НГУ) | **Рекомендуемое** хранилище образов в локальной инфраструктуре | Latest stable | https://gitlab.ci.nsu.ru |
| Docker Registry / Railway | Текущее хранилище/деплой образа стенда | — | https://docs.docker.com/registry/ |

#### 2.3.3 Средства сборки и пакетирования (компиляция исходного текста в объектный код)

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| Docker (multi-stage) | Сборка образа: stage 1 (Node) — Vite-build SPA; stage 2 (python:3.12-slim) — runtime + статика | Engine 24+ | https://docs.docker.com/engine/install/ |
| Node.js | Среда сборки фронтенда (только build-time) | 20 (alpine) | https://nodejs.org |
| npm | Менеджер пакетов фронтенда (`npm ci`); lock-файл `package-lock.json` | 10+ | https://docs.npmjs.com/cli |
| Vite build (`tsc -b` + `vite build`) | Транспиляция TS и сборка SPA в `frontend/dist` | 5.4.10 | https://vitejs.dev |
| pip | Менеджер пакетов Python; зависимости из `requirements.txt` | 25.x | https://pip.pypa.io/ |
| `backend/requirements.txt` | Декларативное описание Python-зависимостей с фиксированными версиями | N/A (текстовый файл) | в составе проекта |
| `frontend/package.json` + `package-lock.json` | Декларативное описание JS-зависимостей с фиксацией версий | N/A (текстовые файлы) | в составе проекта |
| `Dockerfile` | Декларативное описание сборки контейнера (базовый образ, зависимости, точка входа) | N/A (текстовый файл) | в составе проекта |
| `start.sh` | Идемпотентный entrypoint: privilege-drop, сидинг Volume, проверка целостности DuckDB, запуск uvicorn | N/A (скрипт) | в составе проекта |

#### 2.3.4 Перечень необходимых сторонних компонентов и систем

| Компонент | Описание | Версия | Ссылка |
|---|---|---|---|
| Bash | Командная оболочка Unix для скриптов развёртывания (`start.sh`, инсталляторы) | Включён в Linux/Unix | предустановлен в Unix-системах |
| YAML / JSON | Форматы структурированных данных: конфигурация, регламенты (`flow.json`), манифесты | N/A (текстовые форматы) | встроенная поддержка в Python |
| W3C Turtle / SHACL / PROV-O | Стандартизированные форматы обмена регламентами и происхождения (interop с Σ СИГМА) | W3C Rec | https://www.w3.org/TR/turtle/ |
| Внешние API-источники (опционально) | Open-Meteo, Yandex Traffic, NSK OpenData, Plan-R, LEYKA — источники событий ETL. **Не обязательны** для базовой работы | — | по договорённости |
| LLM-endpoint (опционально) | OpenAI-совместимый сервис (Cerebras/Groq/Ollama). При отсутствии — mock-режим | — | https://ollama.com (локально) |

---

## 3. Список компонентов с лицензиями

> Все используемые сторонние компоненты — под **разрешительными (permissive)** лицензиями
> (MIT / BSD / Apache 2.0 / ISC / PSF / Public Domain). **Сильного копилефта (GPL/AGPL) в составе
> исполняемого ПО нет.** Покупка лицензий на компоненты не требуется.

| Компонент | Тип лицензии | Коммерческое использование | Ссылка на лицензию |
|---|---|---|---|
| Python | PSF License | Да | https://docs.python.org/3/license.html |
| FastAPI | MIT | Да | https://github.com/fastapi/fastapi/blob/master/LICENSE |
| Uvicorn | BSD-3-Clause | Да | https://github.com/encode/uvicorn/blob/master/LICENSE.md |
| Starlette (через FastAPI) | BSD-3-Clause | Да | https://github.com/encode/starlette/blob/master/LICENSE.md |
| python-multipart | Apache 2.0 | Да | https://github.com/Kludex/python-multipart/blob/master/LICENSE.txt |
| HTTPX | BSD-3-Clause | Да | https://github.com/encode/httpx/blob/master/LICENSE.md |
| Pydantic | MIT | Да | https://github.com/pydantic/pydantic/blob/main/LICENSE |
| pydantic-settings | MIT | Да | https://github.com/pydantic/pydantic-settings/blob/main/LICENSE |
| python-dotenv | BSD-3-Clause | Да | https://github.com/theskumar/python-dotenv/blob/main/LICENSE |
| rdflib | BSD-3-Clause | Да | https://github.com/RDFLib/rdflib/blob/main/LICENSE |
| pySHACL | Apache 2.0 | Да | https://github.com/RDFLib/pySHACL/blob/master/LICENSE.txt |
| NetworkX | BSD-3-Clause | Да | https://github.com/networkx/networkx/blob/main/LICENSE.txt |
| PyYAML | MIT | Да | https://github.com/yaml/pyyaml/blob/main/LICENSE |
| DuckDB | MIT | Да | https://github.com/duckdb/duckdb/blob/main/LICENSE |
| OpenAI Python SDK | Apache 2.0 | Да (OpenAI, США) — **опционален** | https://github.com/openai/openai-python/blob/main/LICENSE |
| Ollama (локальный инференс) | MIT | Да — **опционален** | https://github.com/ollama/ollama/blob/main/LICENSE |
| Qwen (LLM, локальные веса) | Apache 2.0 | Да (Alibaba/Qwen, КНР) — **опционален** | https://github.com/QwenLM/Qwen2.5/blob/main/LICENSE |
| bge-m3 (эмбеддинги) | MIT | Да (BAAI, КНР) — **опционален** | https://huggingface.co/BAAI/bge-m3 |
| Qwen3-Embedding (эмбеддинги) | Apache 2.0 | Да (Alibaba/Qwen, КНР) — **опционален** | https://huggingface.co/Qwen/Qwen3-Embedding-8B |
| pypdf | BSD-3-Clause | Да | https://github.com/py-pdf/pypdf/blob/main/LICENSE |
| python-docx | MIT | Да | https://github.com/python-openxml/python-docx/blob/master/LICENSE |
| certifi (транзитивно через httpx) | MPL 2.0 | Да (файловый копилефт; использование как есть — без ограничений) | https://github.com/certifi/python-certifi/blob/master/LICENSE |
| pytest | MIT | Да | https://github.com/pytest-dev/pytest/blob/main/LICENSE |
| pytest-asyncio | Apache 2.0 | Да | https://github.com/pytest-dev/pytest-asyncio/blob/main/LICENSE |
| React / react-dom | MIT | Да (Meta, США) | https://github.com/facebook/react/blob/main/LICENSE |
| react-router-dom | MIT | Да | https://github.com/remix-run/react-router/blob/main/LICENSE.md |
| TypeScript | Apache 2.0 | Да (Microsoft, США) — build-time | https://github.com/microsoft/TypeScript/blob/main/LICENSE.txt |
| Vite | MIT | Да | https://github.com/vitejs/vite/blob/main/LICENSE |
| Tailwind CSS | MIT | Да | https://github.com/tailwindlabs/tailwindcss/blob/master/LICENSE |
| React Flow (reactflow) | MIT | Да | https://github.com/xyflow/xyflow/blob/main/LICENSE |
| Zustand | MIT | Да | https://github.com/pmndrs/zustand/blob/main/LICENSE |
| TanStack Query | MIT | Да | https://github.com/TanStack/query/blob/main/LICENSE |
| Zod | MIT | Да | https://github.com/colinhacks/zod/blob/main/LICENSE |
| Cytoscape (+cola) | MIT | Да | https://github.com/cytoscape/cytoscape.js/blob/master/LICENSE |
| dnd-kit (core + sortable) | MIT | Да | https://github.com/clauderic/dnd-kit/blob/master/LICENSE |
| lucide-react | ISC | Да | https://github.com/lucide-icons/lucide/blob/main/LICENSE |
| ESLint / typescript-eslint | MIT | Да (build-time) | https://github.com/eslint/eslint/blob/main/LICENSE |
| PostCSS / Autoprefixer | MIT | Да (build-time) | https://github.com/postcss/postcss/blob/main/LICENSE |
| Vitest | MIT | Да (build-time) | https://github.com/vitest-dev/vitest/blob/main/LICENSE.md |
| Node.js | MIT (+ зависимости) | Да (build-time) | https://github.com/nodejs/node/blob/main/LICENSE |
| Docker / Moby | Apache 2.0 | Да (Docker Inc., США) | https://github.com/moby/moby/blob/master/LICENSE |
| Git | GPL v2 | Да (инструмент, не линкуется в продукт) | https://github.com/git/git/blob/master/COPYING |
| РЕД ОС | Проприетарная | Да (с лицензией) | https://redos.red-soft.ru |

> Сводно: **runtime ПО «РАГРАФ»** опирается только на permissive-лицензии (MIT/BSD/Apache/ISC/PSF/
> MIT-эквивалент DuckDB). GPL-компоненты (`Git`) и Apache-инструменты (`Docker`) — это инструменты
> сборки/эксплуатации, не входящие в объектный код продукта. Само ПО «РАГРАФ» — проприетарное
> (см. `LICENSE`).

---

## 4. Примечания: анализ рисков и рекомендации

### 4.1. Геополитические / санкционные риски сторонних компонентов

1. **Зарубежные правообладатели компонентов.** Часть компонентов сопровождается компаниями
   недружественных юрисдикций: React/react-dom (Meta, США), TypeScript/ONNX-экосистема/VS Code-tooling
   (Microsoft, США), OpenAI SDK (OpenAI, США), Docker/Moby (Docker Inc., США). Риск — ограничение
   доступа к обновлениям/реестрам (PyPI, npm, Docker Hub, GitHub) при ужесточении санкций.
   **Снижение:** все версии зафиксированы (`requirements.txt`, `package-lock.json`); поднять
   локальные зеркала PyPI/npm и приватный Docker Registry; форкнуть критичные библиотеки на
   GitLab НГУ; иметь офлайн-копии образов.

2. **LLM-зависимость — опциональна, изолирована и заменяема локальными моделями.** `openai` SDK
   импортируется лениво; при `EMBEDDINGS_ENABLED=false` и отсутствии LLM-ключа (значения по умолчанию)
   стенд работает в mock-режиме. **Снижение:** боевой ИИ-контур разворачивается **локально через Ollama**
   на **скачиваемых открытых весах** — LLM **Qwen** (Qwen2.5/Qwen3, Apache 2.0) и эмбеддинги **bge-m3**
   (BAAI, MIT) / **Qwen3-Embedding** (Apache 2.0). Это приватный офлайн-инференс без передачи данных
   третьим лицам и без зависимости от облачных сервисов США; модели — из КНР (вне санкционного периметра
   против РФ) под открытыми лицензиями. Зарубежные облачные endpoint'ы (OpenAI/Cerebras/Groq) —
   не обязательны и используются только как ускорение на демо.

3. **Отсутствие GPU/CUDA/NVIDIA-зависимости.** В отличие от ML-детекторов, РАГРАФ не использует
   PyTorch/TensorRT/Triton/CUDA и работает на CPU (2 CPU / 4 ГБ). Это **исключает** аппаратный
   vendor-lock-in NVIDIA и связанные санкционные риски. Преимущество стенда для импортозамещения.

### 4.2. Лицензионная чистота (FOSS-compliance)

4. **Сильного копилефта в продукте нет.** Прямые зависимости — permissive (MIT/BSD/Apache/ISC/PSF).
   `certifi` под MPL 2.0 — файловый копилефт, используется «как есть» без модификаций → ограничений
   на распространение продукта не создаёт. `Git` (GPL v2) и `Node.js` — инструменты, не входящие в
   объектный код. **Снижение:** включить в CI проверку лицензий зависимостей (`pip-licenses`,
   `license-checker`) и фиксировать SBOM при каждом релизе.

5. **Коммерческие тиры внешних библиотек.** Следить за условиями коммерческих/Pro-версий
   (например, React Flow Pro) — в продукте используется только бесплатная MIT-редакция `reactflow`.
   **Снижение:** не вводить Pro-компоненты без отдельной правовой оценки.

### 4.3. Импортозамещение и размещение

6. **Целевая среда — российская.** Рекомендуемое production-развёртывание: **РЕД ОС** (ФСТЭК, Реестр
   российского ПО) или Astra Linux; СУБД — встраиваемая DuckDB (без иностранных серверных СУБД);
   размещение — на отечественной инфраструктуре (Selectel / Cloud.ru) или on-premise заказчика.
   Зависимости от AWS/Azure/GCP — отсутствуют.

7. **Перенос инфраструктуры разработки в контур НГУ.** Текущий репозиторий — на GitHub (публичный,
   использовался на этапе стенда). **Рекомендация для сдаваемого лицензируемого ПО:** перенести
   исходники в **GitLab on-premise НГУ** (`gitlab.ci.nsu.ru`), образы — в **GitLab Container Registry**;
   ограничить доступ; вести аудит изменений. Это согласуется с проприетарным статусом ПО (см. § 4.5).

### 4.4. Совместимость, воспроизводимость, сопровождение

8. **Все компоненты проверены на совместимость** в рамках единого технологического стека.
9. **Версии зафиксированы** на текущий период (`requirements.txt`, `package-lock.json`); для
   production использовать утверждённые в настоящем документе версии, обновления — через
   контролируемый цикл с регресс-тестированием.
10. **Воспроизводимая сборка** — multi-stage `Dockerfile`; стартовый набор регламентов сеется из
    фикстур при первом запуске (`init_db`/`start.sh`). Для production обязательны: Linux (РЕД ОС/Ubuntu),
    Python 3.12, Docker.

### 4.5. Статус и правообладание ПО «РАГРАФ»

11. **«РАГРАФ» — проприетарное ПО**, созданное в рамках НИОКТР (тема СИГМА, договор № 70-2023-001318).
    Открытость компонентов не делает продукт открытым. **Модель монетизации — сервисная:** лицензия на
    **РАГРАФ Lite** для заказчиков НГУ близка к нулю (фактически покрывается обучением одного специалиста
    по программе ДПО); доход — с услуг внедрения, обучения методологов и небольшой ежегодной платы за
    поддержку и обновление кода. Σ СИГМА — **отдельный продукт** со своей моделью. Условия — `LICENSE`.
12. **Правообладание** определяется условиями НИОКТР и трудовых/гражданско-правовых договоров
    (служебное произведение — ст. 1295 ГК РФ; права на РИД по гранту/госконтракту — ст. 1296–1298,
    1373 ГК РФ). Подлежит оформлению до распоряжения правами. Подробнее —
    [`FLOW-DSL-IP-STRATEGY.md`](GOTOMARKET/FLOW-DSL-IP-STRATEGY.md).
13. **Рекомендуется** государственная регистрация программы для ЭВМ (Роспатент) и включение в Реестр
    российского ПО (после переноса в контур НГУ и оформления прав).

---

*Документ подготовлен для сдачи ПО заказчику/грантодателю. Версии и лицензии актуальны на 2026-06-05;
перед регистрацией сверить с текущими `requirements.txt`/`package-lock.json` и текстами лицензий.*
