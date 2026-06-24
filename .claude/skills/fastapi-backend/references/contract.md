# The Contract (you are the source)

The frontend generates its TypeScript types from your `/openapi.json` (`npx openapi-typescript`), and integrations depend on your shapes. Your OpenAPI document is a published interface, not a side effect. Design it deliberately.

## Schema layering: three shapes, not one

For any resource, keep the shapes separate — collapsing them leaks internals and breaks clients:

| Shape | Role | Includes |
|-------|------|----------|
| `ThingCreate` | request (POST body) | only client-settable fields |
| `ThingUpdate` | request (PATCH body) | same, all optional |
| `Thing` (out) | response | everything safe to expose (id, timestamps, status) |
| ORM `Thing` | DB only | never crosses the wire |

```python
class RequestCreate(BaseModel):
    item: str = Field(min_length=1)
    quantity: int = Field(gt=0)

class RequestOut(BaseModel):
    id: int
    item: str
    quantity: int
    status: RequestStatus
    created_at: datetime
    model_config = ConfigDict(from_attributes=True)
```

The frontend's `RequestCreate`/`Request` types (from the contract-first skill) should mirror these exactly. They will, automatically, if you generate them from OpenAPI.

## Make OpenAPI accurate

- `response_model=...` and explicit `status_code=...` on **every** route. Without `response_model`, the generated client gets `any` and the contract is worthless.
- Type path/query params precisely (`int`, `Enum`, constrained types) so they appear correctly in the schema.
- Add `summary`/`description` and `examples` on schemas — they flow into the docs the frontend reads.
- Use enums (not free strings) for closed sets like status; they generate as TS string-literal unions.

## Error envelope: one shape, everywhere

FastAPI returns `{"detail": ...}` (a string for `HTTPException`, a structured array for validation errors). Don't invent a different error shape per route — the frontend normalizes one shape. If you need richer errors, standardize them and document the structure once:

```python
raise HTTPException(status_code=404, detail="Request not found")
# validation errors (422) are produced automatically by Pydantic — let them be
```

Never return stack traces or internal messages. Map domain errors to clean HTTP statuses (404 not found, 409 conflict, 403 forbidden, 422 validation).

## Versioning: protect existing clients

A breaking change to a shape or route breaks every generated client silently. Rules:
- **Additive changes** (new optional field, new endpoint) — safe, no version bump.
- **Breaking changes** (remove/rename a field, change a type, tighten a constraint) — version the route (`/api/v2/...`) and keep v1 until clients migrate.
- Record every shape change in `BACKEND_HANDOFF.md` with a one-line "consumers must regenerate types" flag.

## The handoff loop

1. You change/add a `response_model`.
2. You note it in `BACKEND_HANDOFF.md` (endpoint, shape, breaking?).
3. The frontend regenerates types from `/openapi.json` and the compiler shows every place that must update.

That loop only works if your OpenAPI is accurate. An inaccurate contract is worse than none — it makes clients confidently wrong.
