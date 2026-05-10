**Русский** · [English](./architecture.md)

# Architecture — Content Generation Pipeline

> **Дисклеймер.** Это публичное описание архитектуры реальной системы, в разработке которой автор участвовал. Конкретные клиенты, доменные имена, финансовые показатели, исходный код и проприетарные детали реализации не раскрываются. Содержание ограничено архитектурными решениями и принципами, обсуждаемыми в публичном поле для систем такого назначения.

Расширенное архитектурное описание. Дополняет [README.md](../README.md).

## 1. Слои системы

```mermaid
flowchart TB
    subgraph L1["Presentation"]
        SPA[React SPA: dashboard, pipeline editor, approval queue]
    end
    subgraph L2["API"]
        Routers[FastAPI routers]
        AuthMW[Auth middleware - JWT]
        ValMW[Validation - Pydantic]
    end
    subgraph L3["Application"]
        Orchestrator[Pipeline orchestrator]
        StageRegistry[Stage registry]
        Approval[Approval service]
        Publishing[Publishing service]
        Schedules[Schedule service]
    end
    subgraph L4["Workers"]
        Workers[Celery worker pool]
        Beat[Celery beat]
    end
    subgraph L5["Adapters"]
        LLMAdapter[LLM adapter]
        SocialAdapter[Social platform adapters]
        StorageAdapter[Object storage adapter]
    end
    subgraph L6["Persistence"]
        Repos[Repositories - SQLAlchemy]
        Cache[Redis cache]
    end
    subgraph L7["Infrastructure"]
        PG[(PostgreSQL)]
        Redis[(Redis)]
        FS[Object Storage]
    end

    SPA --> Routers
    Routers --> AuthMW --> ValMW
    ValMW --> Orchestrator & Approval & Publishing & Schedules
    Orchestrator --> StageRegistry
    Orchestrator --> Repos
    Orchestrator -->|enqueue| Cache
    Workers -->|consume| Cache
    Workers --> StageRegistry
    Workers --> LLMAdapter & SocialAdapter & StorageAdapter
    Workers --> Repos
    Repos --> PG
    Cache --> Redis
    StorageAdapter --> FS
    Beat --> Cache
    Schedules --> Beat
```

## 2. Доменная модель

Высокоуровневые сущности и связи. Можно выделить три семантических кластера:

- **Конфигурация проекта** — то, что пользователь настраивает: brand-профили, AI-конфиги, интеграции с платформами, медиа-assets, шаблоны пайплайнов.
- **Исполнение** — конкретный запуск пайплайна и его стадии.
- **Результат** — сгенерированный контент и его публикации с аналитикой.

```mermaid
erDiagram
    USER ||--o{ PROJECT : owns
    PROJECT ||--o{ PIPELINE_TEMPLATE : "has templates"
    PROJECT ||--o{ AI_CONFIG : "has configs"
    PROJECT ||--o{ BRAND_PROFILE : "has brand profiles"
    PROJECT ||--o{ SOCIAL_INTEGRATION : "has integrations"
    PROJECT ||--o{ ASSET : "has assets"

    PIPELINE_TEMPLATE ||--o{ PIPELINE_RUN : "instances"
    PIPELINE_RUN ||--|{ STAGE_EXECUTION : "passes through"
    PIPELINE_RUN ||--o| CONTENT_RESULT : produces
    CONTENT_RESULT ||--o{ PUBLICATION : "published as"
    PUBLICATION ||--o| PUBLICATION_STATS : has

    PIPELINE_TEMPLATE ||--|{ STAGE_DEFINITION : contains
    STAGE_DEFINITION ||--o{ STAGE_EXECUTION : "instantiated as"
```

### Назначение сущностей

- **USER / PROJECT** — учётка и изолированный контекст работы.
- **BRAND_PROFILE** — голос бренда, визуальные гайдлайны (используется стадиями-валидаторами).
- **AI_CONFIG** — credentials и параметры LLM-провайдеров для проекта. Чувствительные поля шифруются.
- **SOCIAL_INTEGRATION** — credentials и конфигурация для конкретной социальной платформы. Чувствительные поля шифруются.
- **ASSET** — медиа-файлы, доступные стадиям пайплайна (логотипы, шаблоны, фото).
- **PIPELINE_TEMPLATE / STAGE_DEFINITION** — пользовательский шаблон пайплайна и описание его стадий с конфигурацией.
- **PIPELINE_RUN** — единичный запуск шаблона на конкретном брифе.
- **STAGE_EXECUTION** — выполнение одной стадии в рамках run'а; хранит входной и выходной контекст для трассируемости.
- **CONTENT_RESULT** — финальный результат генерации.
- **PUBLICATION / PUBLICATION_STATS** — публикация результата в конкретный канал и метрики оттуда.

## 3. Жизненный цикл pipeline-run

```mermaid
sequenceDiagram
    participant UI as User UI
    participant API
    participant DB
    participant Queue
    participant Worker
    participant LLM
    participant Approval as Approval service

    UI->>API: POST /pipelines/{template_id}/run + brief
    API->>DB: INSERT pipeline_run (status=created)
    API->>DB: INSERT stage_executions (one per stage, status=pending)
    API->>Queue: enqueue first stage
    API-->>UI: 202 Accepted, run_id

    loop for each stage
        Worker->>Queue: consume stage
        Worker->>DB: SELECT stage_execution + load context
        Worker->>DB: UPDATE stage_execution status=running
        Worker->>StageRegistry: lookup stage handler
        StageRegistry-->>Worker: handler

        alt stage type = generation
            Worker->>LLM: generate(prompt, context)
            LLM-->>Worker: content
        end

        alt stage type = manual-approval
            Worker->>Approval: register pending approval
            Worker->>DB: UPDATE stage_execution status=awaiting_approval
            Note over Worker: pipeline pauses here
        else stage type = action
            Worker->>Worker: execute side-effect
        end

        Worker->>DB: UPDATE stage_execution status=completed, save output_context
        Worker->>Queue: enqueue next stage
    end

    UI->>Approval: GET pending approvals
    UI->>Approval: POST approve / edit / reject
    Approval->>DB: UPDATE stage_execution status=approved/edited/rejected
    Approval->>Queue: enqueue next stage (if approved/edited)
```

## 4. Pipeline state machine — детально

Каждый `STAGE_EXECUTION` проходит через состояния:

```mermaid
stateDiagram-v2
    [*] --> Pending
    Pending --> Running: worker picks up
    Running --> Completed: success
    Running --> Failed: error
    Running --> AwaitingApproval: stage type = manual gate
    AwaitingApproval --> Approved: user approves
    AwaitingApproval --> Edited: user edits and approves
    AwaitingApproval --> Rejected: user rejects
    Approved --> [*]: next stage queued
    Edited --> [*]: next stage queued
    Rejected --> [*]: pipeline_run terminated
    Completed --> [*]: next stage queued / pipeline finished
    Failed --> Retried: manual or auto retry
    Retried --> Pending
```

Каждый переход — отдельная транзакция в БД, что обеспечивает atomicity и видимость в UI.

## 5. Stage registry — расширяемость

Стадии регистрируются в реестре при старте приложения. Реестр — словарь `{stage_type: handler_class}`. Добавить новую стадию:

1. Описать input / output контракты (Pydantic-модели).
2. Реализовать handler-класс с методом `async def execute(context, config) -> output`.
3. Зарегистрировать в реестре через декоратор или явный вызов.

После этого стадия доступна в конструкторе пайплайнов в UI — пользователь может её выбрать и сконфигурировать.

```mermaid
classDiagram
    class StageHandler {
        <<interface>>
        +stage_type: str
        +input_schema: Type[BaseModel]
        +output_schema: Type[BaseModel]
        +config_schema: Type[BaseModel]
        +execute(context, config) Output
    }
    class NicheResearchHandler {
        +stage_type = "niche-research"
        +execute()
    }
    class GenerationHandler {
        +stage_type = "generation"
        +execute()
    }
    class ApprovalHandler {
        +stage_type = "manual-approval"
        +execute()
    }
    class PublishHandler {
        +stage_type = "publish-to-channel"
        +execute()
    }
    StageHandler <|-- NicheResearchHandler
    StageHandler <|-- GenerationHandler
    StageHandler <|-- ApprovalHandler
    StageHandler <|-- PublishHandler
```

## 6. Безопасность

### Аутентификация и сессии

- Сессионные токены — JWT с HMAC-подписью (HS256). Access-токен короткоживущий, refresh — в HttpOnly-cookie.
- Пароли пользователей в БД — только хеши через bcrypt.

### Чувствительные данные

В системе три класса чувствительной информации, которые требуют защиты от утечки:

- LLM-ключи (биллинговая поверхность)
- OAuth-токены / API-credentials социальных платформ (доступ к корпоративным аккаунтам клиента)
- Ключи доступа к медиа-хранилищам

Все эти поля шифруются на уровне ORM (Fernet) и хранятся в БД уже зашифрованными. Encryption key держится в env-конфигурации сервера. Любой инструмент, читающий БД напрямую — бэкап, дамп, log-collector, dba-сессия — видит только шифр.

### Изоляция проектов

В UI пользователь видит только данные тех проектов, на которые у него есть доступ. Запросы фильтруются по `project_id` на уровне репозиториев.

### Audit log

В отдельной таблице фиксируются пользовательские действия с ключевыми сущностями: аутентификация, изменение конфигурации проекта, операции над pipeline-runами, approval-действия, публикации. Состав расширяется по мере появления новых требований compliance.

## 7. Деплой

### Dev — Docker Compose

```mermaid
flowchart LR
    Dev["docker compose up"] --> Postgres
    Dev --> Redis
    Dev --> Backend["FastAPI - hot reload"]
    Dev --> Worker[Celery worker]
    Dev --> Beat[Celery beat]
    Dev --> Frontend["Vite dev server"]
```

### Production

Целевая инсталляция — single-server Linux. Backend под uvicorn-воркерами за reverse-proxy (nginx) с TLS-терминацией; Celery-воркеры и beat-планировщик — отдельные процессы; всё под управлением process-supervisor.

```mermaid
flowchart LR
    Edge[nginx + TLS] --> AppPool[uvicorn workers - FastAPI]
    AppPool --> PG[(PostgreSQL)]
    AppPool --> RDQ[(Redis)]
    Workers[Celery workers]
    Beat[Celery beat]
    Workers --> PG
    Workers --> RDQ
    Workers --> Storage[Object Storage]
    AppPool --> Storage
    Beat --> RDQ
```

Static-bundle фронтенда раздаётся nginx'ом. Все служебные процессы (backend, workers, beat) поднимаются process-supervisor'ом и пишут структурированные логи в системный журналер.

### CI

GitLab CI с типичными этапами для Python+JS-стека:

- **Lint**: статические проверки кода (ruff / black / mypy для Python; eslint для frontend).
- **Test**: pytest с поднятыми PG и Redis в CI-runner. Юнит и интеграционные.
- **Build**: frontend-bundle и backend-wheel.
- **Artifact**: упакованный релиз.

Раскатка релиза на целевой сервер — короткая последовательность из pull, установки зависимостей, сборки фронта, миграции БД и перезапуска сервисов.

## 8. Мониторинг и observability

- **Структурированные логи** в JSON, агрегатор на стороне инсталляции.
- **Health endpoints**: `/health` (liveness), `/health/deps` (readiness — проверка PG, Redis, Storage).
- **Метрики pipeline**: количество run'ов по статусам, среднее время прохождения, error rate по стадиям.
- **Метрики publishing**: успешность публикации по платформам, latency.
- **Audit log** в БД.

