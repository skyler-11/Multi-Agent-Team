---
name: fastapi-backend
description: Expert backend engineering for FastAPI services — endpoints, Pydantic v2 schemas, SQLAlchemy models, business logic, and the OpenAPI contract that frontend and integrations consume. Use whenever the work is server-side — adding or refactoring an endpoint, modeling data, wiring the database, shaping request/response schemas, handling errors/validation, or making the API contract-clean for client codegen — even if the user doesn't say "FastAPI." Database- and auth-agnostic by default, it matches the repo's existing SQLAlchemy style (async or sync), keeps persistence swappable, and pulls auth/migration modules from references only when needed. Do NOT use for frontend/UI, external system adapters (RPA, SMTP, SharePoint), or infrastructure.
---

# FastAPI Backend

Operate as a senior backend engineer who treats the **API contract as a published product**. This service is the contract *source*: the frontend generates its types from `/openapi.json`, and integrations depend on these shapes. So two things are judged together — is the service correct and well-layered, and is the contract it exposes clean, accurate, and stable enough for others to build on without surprises.

## How this skill works

1. **Match the repo before imposing anything.** Detect the existing style — async (`async def` + `AsyncSession`) or sync (`Session`), project layout, naming — and follow it. Never convert sync↔async or restructure unprompted.
2. **Design the contract first.** Define Pydantic v2 schemas (request + response) before writing logic; these *are* the contract.
3. **Build in layers** behind that contract (router → service → repository → model).
4. **Keep persistence and auth pluggable** — never hardcode a database dialect or auth scheme into business logic.
5. **Verify the contract** is clean and the three concerns (validation, errors, types) are explicit before handing off.

For database setup + migrations read `references/persistence.md`; for auth read `references/auth.md`; for schema/response design and versioning read `references/contract.md`.

## Layered architecture (mirror of the frontend's seams)

Four layers, one job each — the backend analog of the frontend's Repository / Container split:

- **Router** (`routers/`) — thin. Declares path, method, `response_model`, status code, dependencies. No business logic, no raw DB access.
- **Service** (`services/`) — business logic and orchestration. Doesn't build HTTP responses or manage the ORM session beyond what's injected.
- **Repository** (`repositories/` or `crud/`) — the only layer that touches the ORM/session. Same idea as the frontend Repository: the single place that "talks to the database," so swapping DB or query strategy touches one layer.
- **Model vs. Schema** — SQLAlchemy models (DB shape) and Pydantic schemas (wire shape) are **separate**. Never return a SQLAlchemy model directly; map it to a response schema.

If a layer reaches across (a router querying the DB, a service building a `Response`), stop — that's the seam telling you to refactor.

## Pydantic v2 schemas = the contract

- **Separate request and response models.** `ThingCreate` (input) ≠ `Thing` (output). Inputs exclude server-controlled fields (id, timestamps, status); outputs include them. Never accept or leak fields the client shouldn't set or see.
- **Put `response_model` on every route** so FastAPI enforces and documents the output shape. This is what makes client codegen reliable.
- **`model_config = ConfigDict(from_attributes=True)`** on response schemas to map cleanly from ORM objects.
- Validate at the edge (`Field` constraints, validators) so invalid data never reaches the service. Mirror these constraints in what the frontend validates — matched validation prevents "frontend passes, server 422s."

See `references/contract.md` for response conventions, error envelopes, and versioning.

## Persistence — database-agnostic

Configure the engine from a single `DATABASE_URL` (Pydantic `Settings`), so SQLite, Postgres, or MySQL is a config change, not a code change. Use ORM constructs; avoid dialect-specific SQL unless guarded. Engine/session wiring differs for async vs sync — follow the repo's choice. Dialect gotchas (SQLite FK pragma + no native `ALTER`; Postgres-only types) and Alembic migrations live in `references/persistence.md`.

## Auth — a pluggable dependency, not a baked-in assumption

Treat auth as a FastAPI dependency at the router boundary (`Depends(get_current_user)`), never threaded through business logic. The core stays auth-agnostic; pull the concrete module from `references/auth.md` only when the project needs it:
- **Simple JWT** — lightweight default for self-contained services.
- **Keycloak / OIDC + RBAC** — when integrating with an identity provider and role-based access.

## Quality floor (non-negotiable, never announced)

Every endpoint: explicit `response_model` and status code; input validated by Pydantic; errors raised as `HTTPException` with useful, non-leaky detail; no secrets or stack traces in responses; DB sessions dependency-injected (never module-global); async routes never block on sync I/O. Secrets come from env via Pydantic `Settings`, never hardcoded.

## Handoff (the contract is the border)

What you expose is what the frontend and integration agents consume. Ensure `/openapi.json` reflects reality (correct `response_model`s, status codes, examples), and record new/changed endpoints and shapes in `BACKEND_HANDOFF.md` so consumers can regenerate types. A silent contract change is a broken client.

## Self-critique before delivering

- **Correctness pass:** layers respected? request/response schemas separated? sessions injected and closed? errors safe?
- **Contract pass:** would a frontend dev generating types from `/openapi.json` get exactly the right shapes and statuses? Anything under- or over-exposed? Is any breaking change versioned?
