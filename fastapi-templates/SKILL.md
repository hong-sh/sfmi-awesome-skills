---
name: fastapi-templates
description: "Create production-ready FastAPI projects with async patterns, dependency injection, and comprehensive error handling. Use when building new FastAPI applications or setting up backend API projects."
risk: unknown
source: community
date_added: "2026-06-15"
---

# FastAPI Project Templates

Production-ready FastAPI project structures with async patterns, dependency injection, middleware, and best practices for building high-performance APIs.

## Use this skill when

- Starting new FastAPI projects from scratch
- Implementing async REST APIs with Python
- Building high-performance web services and microservices
- Creating async applications with PostgreSQL, MongoDB, or Vector DBs (e.g., OpenSearch, Milvus)
- Setting up API projects with proper structure and testing

## Do not use this skill when
 
- The task is unrelated to fastapi project templates
- You need a different domain or tool outside this scope

## Instructions

- Clarify goals, constraints, and required inputs.
- Apply relevant best practices and validate outcomes.
- Provide actionable steps and verification.
- Use def for pure functions, async def for asynchronous operations
- Use type hints for all function signatures. Prefer Pydantic models over raw dictionaries

### Architecture & Implementation Guidelines
- **Dependency Rule:** Dependencies MUST point inward: `app -> pipelines -> services -> domain`.
- **Domain Independence:** The `domain` layer must not depend on external frameworks or libraries. It should contain pure business logic and schemas.
- **Ports & Adapters:** External system clients (LLM APIs, Vector DBs, RDS, etc.) must be implemented as Adapters conforming to Port interfaces defined internally. Isolate core logic from infrastructure changes.
- **Dependency Injection:** Actively use FastAPI's `Depends` to decouple layers and inject dependencies cleanly.

### Project Structure & Layer Responsibilities
You MUST adhere strictly to the following directory structure and layer responsibilities. Do not alter this structure:

```
project-root/
├── README.md 
├── pyproject.toml
├── uv.lock
├── .gitignore
│
├── app/                    # Delivery Layer (API Entry Point)
│   ├── middlewares/        # FastAPI middlewares
│   ├── routers/            # FastAPI routers (Endpoints)
│   ├── main.py             # FastAPI app creation, router inclusion
│   └── settings.py         # API settings
│
├── configs/                # Configuration Files
│   ├── prd/                
│   └── prompt_config.yaml                  
│   
├── src/                    # Application Source Code
│   ├── domain/             # Core Layer: Domain models, Pydantic schemas, Port interfaces
│   │
│   ├── pipelines/          # Orchestration Layer: Complex workflows (e.g., RAG, Agent chains) combining multiple services
│   │  
│   ├── services/           # Application Layer: Single business use cases (e.g., search, generation)
│   │  
│   ├── adapters/           # Infrastructure Layer: External system adapters (DBs, Retrievers, LLMs)
│   │
│   └── common/             # Common Utilities: Loggers, singletons, config/prompt loaders, exceptions
│
├── tests/                  # Test Codes
```

- app: Handles HTTP requests/responses and DI container setup. Must NOT contain business logic.
- src/pipelines: Orchestrates multi-step workflows.
- src/services: Implements core business use cases using injected external dependencies.
- src/adapters: Implements actual connections to external systems.
- src/domain: Defines the core data structures and interfaces.

## Limitations

- Use this skill only when the task clearly matches the scope described above.
- Do not treat the output as a substitute for environment-specific validation, testing, or expert review.
- Stop and ask for clarification if required inputs, permissions, safety boundaries, or success criteria are missing.
