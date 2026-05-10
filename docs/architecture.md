[Русский](./architecture.ru.md) · **English**

# Architecture — Content Generation Pipeline

> **Disclaimer.** This is a public architectural description of a real
> system the author worked on. Specific clients, domain names,
> financial indicators, source code, and proprietary implementation
> details are not disclosed. The content is limited to architectural
> decisions and principles publicly discussed for systems of this kind.

Extended architectural description. Companion to [README.md](../README.md).

## 1. System layers

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

## 2. Domain model

High-level entities and relations. Three semantic clusters can be
distinguished:

- **Project configuration** — what the user configures: brand profiles,
  AI configs, platform integrations, media assets, pipeline templates.
- **Execution** — a concrete pipeline run and its stages.
- **Result** — generated content and its publications with analytics.

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

### Entity purposes

- **USER / PROJECT** — account and isolated working context.
- **BRAND_PROFILE** — brand voice, visual guidelines (used by validator
  stages).
- **AI_CONFIG** — LLM provider credentials and parameters for the
  project. Sensitive fields are encrypted.
- **SOCIAL_INTEGRATION** — credentials and configuration for a specific
  social platform. Sensitive fields are encrypted.
- **ASSET** — media files available to pipeline stages (logos,
  templates, photos).
- **PIPELINE_TEMPLATE / STAGE_DEFINITION** — user-built pipeline
  template and a description of its stages with configuration.
- **PIPELINE_RUN** — a single run of a template against a specific
  brief.
- **STAGE_EXECUTION** — execution of a single stage within a run;
  stores input and output context for traceability.
- **CONTENT_RESULT** — the final generation result.
- **PUBLICATION / PUBLICATION_STATS** — publication of a result to a
  specific channel and metrics from there.

## 3. Pipeline-run lifecycle

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

## 4. Pipeline state machine — detailed

Each `STAGE_EXECUTION` goes through these states:

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

Each transition is a separate DB transaction, providing atomicity and
visibility in the UI.

## 5. Stage registry — extensibility

Stages register in the registry at app startup. The registry is a
dictionary `{stage_type: handler_class}`. To add a new stage:

1. Define input / output contracts (Pydantic models).
2. Implement a handler class with
   `async def execute(context, config) -> output`.
3. Register in the registry via decorator or explicit call.

After that the stage is available in the in-UI pipeline builder — the
user can select and configure it.

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

## 6. Security

### Authentication and sessions

- Session tokens are JWT with HMAC signing (HS256). Short-lived access
  token, refresh token in an HttpOnly cookie.
- User passwords in the DB are stored as bcrypt hashes only.

### Sensitive data

The system has three classes of sensitive information that need
protection from leakage:

- LLM keys (billing surface)
- OAuth tokens / API credentials of social platforms (access to the
  customer's corporate accounts)
- Object-storage access keys

All these fields are encrypted at the ORM layer (Fernet) and stored in
the DB already encrypted. The encryption key is held in the server's
env config. Any tool reading the DB directly — backup, dump,
log-collector, dba session — sees only ciphertext.

### Project isolation

In the UI a user only sees data from projects they have access to.
Queries are filtered by `project_id` at the repository layer.

### Audit log

A separate table records user actions on key entities: authentication,
project config changes, operations on pipeline runs, approval actions,
publications. The set is extended as new compliance requirements
emerge.

## 7. Deployment

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

The target installation is a single-server Linux. Backend runs under
uvicorn workers behind a reverse proxy (nginx) with TLS termination;
Celery workers and the beat scheduler are separate processes;
everything is managed by a process supervisor.

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

The frontend static bundle is served by nginx. All service processes
(backend, workers, beat) are run by a process supervisor and write
structured logs to the system journal.

### CI

GitLab CI with typical stages for a Python + JS stack:

- **Lint**: static code checks (ruff / black / mypy for Python; eslint
  for frontend).
- **Test**: pytest with PG and Redis running in the CI runner. Unit
  and integration.
- **Build**: frontend bundle and backend wheel.
- **Artifact**: packaged release.

Release rollout to the target server is a short sequence of pull,
dependency installation, frontend build, DB migration, and service
restart.

## 8. Monitoring and observability

- **Structured logs** in JSON, with the aggregator on the installation
  side.
- **Health endpoints**: `/health` (liveness), `/health/deps`
  (readiness — PG, Redis, Storage check).
- **Pipeline metrics**: number of runs by status, average pass-through
  time, error rate per stage.
- **Publishing metrics**: publishing success rate per platform,
  latency.
- **Audit log** in the DB.
