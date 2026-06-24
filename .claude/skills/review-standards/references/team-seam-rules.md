# Team Seam Rules — Review Checklist

These are the load-bearing architectural constraints from the `frontend-react-engineer`, `fastapi-backend`, and `rpa-integration` skills. The team's design only holds if these hold, so a violation is a **blocker** unless the author has an explicit, justified exception. Check the diff against each.

## Backend (fastapi-backend)

| Rule | Violation looks like |
|------|----------------------|
| Router stays thin | Business logic or query-building inside a route function |
| Router never touches the DB | `session`/ORM used directly in a router instead of via a repository |
| Service has no HTTP/ORM session lifecycle | Service builds a `Response`, sets status codes, or opens its own session |
| Repository is the only DB layer | ORM queries outside `repositories/`/`crud/` |
| Models ≠ schemas | A SQLAlchemy model returned from a route; no `response_model` on a route |
| Request vs response schemas separated | One schema used for both input and output; client-settable id/status/timestamps |
| Sessions injected | Module-global session; session not closed |
| Contract ownership | A non-backend/architect change to a `response_model` or route shape |
| Breaking change unversioned | Field removed/renamed/retyped without a version bump or handoff note |

## Frontend (frontend-react-engineer)

| Rule | Violation looks like |
|------|----------------------|
| Repository seam | `fetch`/axios called inside a component instead of the `api/` layer |
| Container/Presentational | A leaf presentational component owning fetching/effects |
| Three states | An async view with no loading / error / empty handling |
| Tokens over arbitrary values | Raw `text-[#hex]` / `mt-[13px]` instead of config tokens |
| Validation parity | Form validates differently from the backend contract |
| Accessibility floor | Non-semantic `div` buttons, missing labels, no visible focus |

## Integration (rpa-integration)

| Rule | Violation looks like |
|------|----------------------|
| Anti-corruption boundary | Raw vendor JSON (Graph/Orchestrator) passed into domain code |
| Adapter owns external I/O | External HTTP calls outside `integrations/` |
| Domain changes delegated | An adapter writing the ORM directly instead of calling a backend service |
| Inbound verified | A webhook/callback handler acting without signature/secret verification |
| Inbound idempotent | No dedupe on event/job id — duplicate delivery causes double side effects |
| Fast-ack | Slow work done inline in the webhook handler before returning 2xx |
| Timeouts | An external call with no timeout |
| Retry discipline | Unbounded retries, or retrying non-retriable 4xx |
| Inbound route ownership | Integration agent mounting its own FastAPI route instead of handing the contract to backend |

## Cross-cutting

| Rule | Violation looks like |
|------|----------------------|
| Secrets | Hardcoded credentials/keys; secrets written to logs |
| PII | Full payloads/bodies logged when they may contain personal data |
| Handoff currency | Contract/endpoint changed but the relevant `*_HANDOFF.md` not updated |

When you flag one of these, name the rule and the skill it comes from so the author can read the rationale. If the author has a deliberate, documented reason to break a rule, downgrade from blocker to a discussion comment — the rules serve the design, they aren't dogma.
