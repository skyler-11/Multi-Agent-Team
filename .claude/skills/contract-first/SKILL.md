---
name: contract-first
description: Expert discipline for treating the API/data contract as a published interface that one side serves and the others consume — the shared rule behind the team's contract border. Use this skill whenever work crosses the seam between backend, frontend, and integrations: defining or changing a request/response shape, generating client types, deciding whether a change is additive or breaking, mirroring validation across layers, or recording a contract change in a handoff. It loads into every builder agent so all sides build to the same agreed shapes instead of drifting. Do NOT use it for the role-specific depth — serving the contract lives in the fastapi-backend skill and consuming it in the frontend-react-engineer skill; this skill is the agent-neutral principle they all share.
---

# Contract-First

The contract — the request/response shapes and the routes that carry them — is a **published
interface**, not a side effect of whoever happened to write the endpoint. Multiple sides build
against it at once: the frontend generates its types from it, integrations depend on its shapes. So
it has to be defined deliberately, owned clearly, and changed carefully. A silent contract change
is a broken client that fails confidently — worse than no contract at all.

This is the team's shared baseline for everything that crosses the seam. The role-specific depth
lives in the specialist skills (linked below); this skill is the principle all sides share.

## The contract border (who may do what)

One rule prevents three-way drift:

- **Backend *serves* the contract.** It is the single source — the shapes and routes it exposes
  *are* the contract.
- **Frontend and integration *consume* the contract.** They build to it and read it; they don't
  redefine it on their side.
- **Only the architect (or backend) may *change* the contract.** A consumer that needs a different
  shape requests the change at the border — it doesn't fork its own version.

If you find yourself inventing a shape a consumer "wishes" the backend had, stop: that's a change
request to the border, not a local decision.

## How this skill works

1. **Define the shape before building.** Agree the request and response shapes first; they're a
   design input for both sides, not an afterthought.
2. **One source of truth.** The backend's OpenAPI document (`/openapi.json`) is the canonical
   contract. Everything else derives from it.
3. **Generate, don't hand-sync.** Produce client types from OpenAPI rather than maintaining a
   parallel hand-written copy that silently rots.
4. **Classify every change** as additive or breaking before you ship it.
5. **Record it in the handoff** so consumers know to regenerate.

## Define the shape first

Separate the shapes by role — collapsing them leaks internals and breaks clients:

- **Request shapes** (`ThingCreate`, `ThingUpdate`) carry only client-settable fields; inputs
  exclude server-controlled fields (id, timestamps, status).
- **Response shapes** (`Thing`) include everything safe to expose.
- The persistence model is **not** the wire shape — it never crosses the border directly.

The agreement on these shapes precedes the implementation on either side. Both sides can then build
in parallel against the frozen shape — the frontend against a mock, the backend against the real
thing — and meet in the middle.

## Single source of truth

- The backend puts an explicit response shape and status on **every** route so the OpenAPI document
  is accurate. An endpoint with no declared response shape generates as `any` and the contract is
  worthless.
- Consumers generate types from it, e.g. `npx openapi-typescript http://localhost:8000/openapi.json
  -o src/types/api.ts`, instead of typing shapes by hand. When a shape changes, regenerate and the
  compiler points at every place that must update.
- Don't maintain two hand-written copies of the same shape on two sides; that's drift waiting to
  happen.

## Changing the contract safely

- **Additive** (new optional field, new endpoint) — safe; no version bump.
- **Breaking** (remove/rename a field, change a type, tighten a constraint) — version the route
  (`/api/v2/...`) and keep the old one until consumers migrate. A breaking change to a shared shape
  breaks every generated client silently.

## Validation parity

Validate the same constraints on both sides — required fields, min/max, enums, formats. When the
consumer's validation is looser or stricter than the contract's, the user passes the client-side
check and then gets rejected at the server (a 422), or vice versa. Mirror the constraints so the
two agree.

## Handoff currency

Every contract change is recorded in the relevant `*_HANDOFF.md` at the repo root: the endpoint,
the shape, and a one-line **breaking?** flag (with "consumers must regenerate types" on breaking
ones). A contract that changed without a handoff note is a change consumers will discover at
runtime.

## Role-specific depth (don't duplicate — link)

- **Serving the contract** (schema layering, OpenAPI accuracy, error envelopes, versioning) →
  `fastapi-backend/references/contract.md`.
- **Consuming the contract** (the `api/` seam, type generation, mocking, error/loading
  conventions) → `frontend-react-engineer/references/backend-integration.md`.

## Quality floor (non-negotiable, never announced)

Shapes are separated by role; every route declares its response shape and status; consumers
generate from `/openapi.json` rather than hand-syncing; breaking changes are versioned; validation
matches across sides; and every change lands in the handoff with a breaking flag. None of this is
announced — a clean, stable contract is the expectation.

## Self-critique before delivering

- **Source pass:** is there exactly one source of truth, and does it reflect reality? Could a
  consumer generate types from it and get exactly the right shapes and statuses?
- **Change pass:** is every change classified additive vs breaking, breaking ones versioned, and
  the handoff updated? Would any consumer be surprised at runtime by something I didn't record?
