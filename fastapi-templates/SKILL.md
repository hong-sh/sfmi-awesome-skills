---
name: fastapi-templates
description: "Create and extend SFMI-standard FastAPI AI services based on the fastapi-skeleton (ai-dev-light): app_config.yaml + scaffold code generation, Executable-based services/pipelines, dispatcher routing by SERVICE_CODE, inbound/outbound adapters, and asset-managed prompts. Use when building a new FastAPI AI service or adding APIs/services/pipelines/adapters to one."
risk: unknown
source: community
date_added: "2026-06-15"
---

# FastAPI Project Templates (SFMI AI Service Skeleton)

A common skeleton that lets every AI service be built **on the same structure**. You declare APIs, services, and pipelines in `app_config.yaml`, `scripts/scaffold.py` generates the code, and the team fills in only the schema fields and `execute()` logic.

## Use this skill when

- Starting a new FastAPI-based AI service from the skeleton
- Adding an API endpoint, service, or pipeline to an existing skeleton-based project
- Adding outbound integrations (LLM, embedding, reranker, storage, DB, vector DB, etc.)
- Managing prompts or other static assets for an AI service
- Reviewing whether code follows the skeleton's layering and conventions

## Do not use this skill when

- The task is unrelated to FastAPI AI service structure
- The project intentionally uses a different architecture (don't force this structure onto it)

## Core Principles

- **Declare → Generate**: declare in `app_config.yaml`, then run `python scripts/scaffold.py`. It generates schema stubs, services, pipelines, endpoints, and tests, and registers them in `ServiceCode` (enums) and `_REGISTRY` (dispatcher). Do not hand-write what scaffold generates.
- **Two-schema separation**: API schemas (inbound, exposed in Swagger) are separate from domain schemas (execution I/O). Conversion lives in the `Executable`'s `validate` / `serialize`.
- **Single `Executable` contract**: both `service` (single responsibility) and `pipeline` (composition of services) inherit `Executable[In, Out]` **directly** and implement `execute`. There is no separate BaseService/BasePipeline. The dispatcher runs both the same way.
- **Deployment chooses the unit**: which Executable runs is decided by the deployment setting `SERVICE_CODE` (`.env`), not by the request.
- **Outbound via providers**: external integrations live in `adapters/outbound/<capability>/` and are accessed through cached `get_*()` provider functions. Static files such as prompts live in `assets/`.

## Project Structure

You MUST follow this directory structure and layer responsibilities:

```
project-root/
├── README.md
├── pyproject.toml              # pytest (asyncio_mode=auto), ruff config
├── requirements.txt            # runtime + test + lint dependencies
├── app_config.yaml             # scaffold input: api_list / services / pipelines
├── .env.example                # SERVICE_CODE, ENVIRONMENT, API_PREFIX, LLM_*, ASSETS_DIR ...
├── scripts/
│   └── scaffold.py             # code generator (idempotent)
├── assets/                     # static assets committed with source
│   └── prompts/
│       ├── manifest.json       # prompt name → active version pin
│       └── <name>/vN.txt       # versioned prompt bodies
├── tests/                      # pytest (TestClient) + run_server.py / run_client.py
└── app/
    ├── main.py                 # entry point: create_app() assembly only
    ├── common/                 # config(settings) · enums(ServiceCode, Environment) · errors(AppError hierarchy)
    ├── util/                   # pure utilities — files · paths
    ├── domain/                 # ★ business core
    │   ├── executable.py       #   Executable[In, Out] — shared contract for services and pipelines
    │   ├── dispatcher.py       #   ServiceCode → _REGISTRY[Executable]
    │   ├── schemas/            #   domain I/O schemas (<name>_schema.py: *Input / *Output)
    │   ├── entity/             #   pure domain value objects (1:1 with DB tables, no ORM)
    │   ├── services/           #   <name>.py — single-responsibility Executables
    │   └── pipelines/          #   <name>.py — Executables composing services
    └── adapters/
        ├── inbound/            #   routers · request/response (API schemas) · middleware
        └── outbound/           #   external connections — _transport.py · llm/ · storage/ · ...
```

### Layer responsibilities

- **`app/main.py`**: assembly only — create the app, register middlewares, exception handlers, and `api_router` (with `settings.api_prefix`), and close outbound clients in `lifespan`. No business logic.
- **`adapters/inbound`**: thin HTTP layer. Validate the API request → `Dispatcher().handle(settings.service_code, request.model_dump())` → map to the API response model. No envelope; the body is the schema itself.
- **`domain/dispatcher.py`**: looks up the `Executable` for a `ServiceCode` and runs `validate → execute → serialize`. Unknown code → `NotFoundError` (404).
- **`domain/pipelines`**: orchestrate multiple services (sequence/branching/merging) in `execute`. Services are created in `__init__`.
- **`domain/services`**: single business use case. Calls outbound providers as needed.
- **`domain/schemas`**: typed use-case I/O contracts (Pydantic). **`domain/entity`**: persisted value objects (dataclasses, no SQLAlchemy/infra knowledge). Keep the two roles separate.
- **`adapters/outbound`**: real connections to external systems. Raw transport (httpx) is an internal detail in `_transport.py`.
- **`common`**: `settings`, enums, error types. **`util`**: pure functions only (no external systems).

**Dependency direction:** `inbound → domain (dispatcher → pipeline → service) → outbound`. Domain code must never import from `adapters/inbound`.

## Workflow: Adding a New Service

1. Add entries to `app_config.yaml` (`*_cls` values are **full dotted paths**):
   ```yaml
   api_list:
     - api_type: POST
       api_path: /summary/run                       # final path: /api/v1/summary/run
       summary: "문서 요약"                           # optional, OpenAPI
       tags: [summary]                              # optional, default ["service"]
       api_request_cls: "app.adapters.inbound.request.SummaryRequest"
       api_response_cls: "app.adapters.inbound.response.SummaryResponse"
   services:
     - service_name: "summary_service"              # ServiceCode value + file name
       service_cls: "SummaryService"
       input_schema_cls: "app.domain.schemas.summary_schema.SummaryInput"
       output_schema_cls: "app.domain.schemas.summary_schema.SummaryOutput"
   pipelines:                                       # optional
     - pipeline_name: "summary_pipeline"
       pipeline_cls: "SummaryPipeline"
       steps: [summary_service]                     # optional; generates __init__/execute
       input_schema_cls: "app.domain.schemas.summary_schema.SummaryInput"
       output_schema_cls: "app.domain.schemas.summary_schema.SummaryOutput"
   ```
2. Run the generator (idempotent; existing files, registrations, and schema classes are skipped):
   ```bash
   python scripts/scaffold.py                # generate + register
   python scripts/scaffold.py -c other.yaml  # another config file
   python scripts/scaffold.py --force        # overwrite generated files (schemas excluded)
   ```
3. Fill in the **schema fields** and the **`execute()` logic** in the generated files. If the API schema differs from the domain schema, override `validate` / `serialize`.
4. Set `SERVICE_CODE` in `.env` to the name of the Executable this deployment serves.
5. Use `get_*()` providers from `adapters/outbound/<capability>/` for external calls.
6. Run `pytest tests/` and `ruff check .`.

Never delete or edit the `# scaffold:*` marker comments in `routers.py`, `enums.py`, and `dispatcher.py` — scaffold inserts code above them.

## Implementation Patterns

### Executable contract

```python
class Executable(ABC, Generic[TReq, TResp]):
    name: str
    request_model: ClassVar[type[BaseModel]]
    async def validate(self, payload: dict) -> TReq:   # default: request_model.model_validate
    @abstractmethod
    async def execute(self, request: TReq) -> TResp:   # implement this
    async def serialize(self, result: TResp) -> dict:  # default: result.model_dump()
```

Service:
```python
class SummaryService(Executable[SummaryInput, SummaryOutput]):
    name = "summary_service"
    request_model = SummaryInput

    async def execute(self, request: SummaryInput) -> SummaryOutput:
        system = get_prompt_provider().render("summary_system", lang="ko")
        answer = get_llm().chat([{"role": "system", "content": system},
                                 {"role": "user", "content": request.text}])
        return SummaryOutput(summary=answer)
```

Pipeline:
```python
class SummaryPipeline(Executable[SummaryInput, SummaryOutput]):
    name = "summary_pipeline"
    request_model = SummaryInput

    def __init__(self) -> None:
        self._summary_service = SummaryService()

    async def execute(self, request: SummaryInput) -> SummaryOutput:
        return await self._summary_service.execute(request)
```

Schema mapping when API ≠ domain schema:
```python
async def validate(self, payload):  return SummaryInput(text=payload["userText"])
async def serialize(self, result):  return {"data": result.summary}
```

### Outbound adapters

- Put each capability in its own folder: `adapters/outbound/<capability>/` (e.g. `llm/`, `storage/`, `embedding/`, `vectordb/`).
- Expose a process-wide singleton via an `@lru_cache` `get_*()` provider so connection pools are reused. Export it from the folder's `__init__.py`.
- HTTP model clients extend `BaseModelClient` in `_transport.py`, which wraps httpx errors into `ExternalServiceError`.
- If a client holds connections, add cleanup to `close_all()` so `lifespan` can close it on shutdown.
- No separate port layer; rely on structural typing. For another backend, add a new `*_client.py` with the same method signatures in the same folder.

```python
from app.adapters.outbound.llm import get_llm, get_prompt_provider
from app.adapters.outbound.storage import get_asset_loader
```

### Prompts and static assets

- Never hard-code prompt strings in code. Store them in `assets/prompts/<name>/vN.txt` and pin the active version in `manifest.json`.
- Load prompts with `get_prompt_provider().get(name)` / `.render(name, **vars)`. Variables use `$var` (`string.Template`), so `{}` in LLM prompts stays safe.
- For per-service prompts, use `ServicePrompts(namespace, fixed={...})`: `fixed` holds prompts tied to code/parsers, and anything else loads from `assets/prompts/<namespace>/<name>`.
- `PROMPT_VERSION_OVERRIDE` forces a specific version.

### Config, errors, logging

- Settings: `from app.common.config import settings` (pydantic-settings, `.env`/env vars). Add new env vars to both `Settings` and `.env.example`.
- Errors: raise from the shared hierarchy; `register_exception_handlers` turns them into consistent JSON (`error_code`, `message`, `detail`). Don't return error dicts manually.
  ```
  AppError ├ DomainError (Validation 400 / Unauthorized 401 / Forbidden 403 / NotFound 404 / Conflict 409)
           ├ InfraError  (ExternalService 502 / Database 503 / Storage 503)
           └ ConfigError (500)
  ```
- Logging/tracing (`sfmi_ai_log`): the SDK integration is commented out in `main.py`, `errors.py`, `_transport.py`, `prompt_provider.py`, `middleware.py`, and `dispatcher.py`. Keep the same pattern (`get_logger`, `set_attr`, `@traced`) when adding code, and uncomment it once the package is available.
- Middleware: request-id (`X-Request-ID`) and request/response logging are in `adapters/inbound/middleware.py`. Prefer router `Depends` for auth.

### Coding conventions

- Python ≥ 3.11, `from __future__ import annotations`, full type hints. Use Pydantic models instead of raw dicts inside the domain; dict ↔ model conversion happens only at the dispatcher boundary.
- Use `async def` for Executable methods and I/O; use `def` for pure functions.
- Ruff: line length 100, rules `E, F, I, UP, B` (`B008` ignored for `Depends`).
- Tests: `pytest` with in-process `TestClient` (`asyncio_mode = "auto"`), no running server needed. scaffold creates `tests/test_<slug>.py` per API; add real assertions to it.

## Local Run

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
uvicorn app.main:app --reload        # http://localhost:8000/docs
curl localhost:8000/api/v1/health
pytest tests/ && ruff check .
```

## Limitations

- Use this skill only when the task clearly matches the scope described above.
- Do not treat the output as a substitute for environment-specific validation, testing, or expert review.
- Stop and ask for clarification if required inputs, permissions, safety boundaries, or success criteria are missing.
