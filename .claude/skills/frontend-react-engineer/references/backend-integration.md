# Backend Integration (contract-first)

The goal of this file: build a React frontend that is fully functional *today* against mock data, and connects to a real backend *tomorrow* by changing one layer. FastAPI + Pydantic is the canonical worked example because Pydantic models give you a typed contract for free — but every principle here applies to any REST/JSON backend (Express, Django REST, Spring, a serverless function). When the backend is unknown, default to backend-agnostic REST/JSON and keep the seam clean.

## The core idea: one seam

All network access lives in a single layer (an `api/` module or a set of hooks). Components never call `fetch` directly. Swapping mock → real backend means editing only that layer.

```
components/  → call hooks/selectors, never fetch()
hooks/       → useRequests() owns data/loading/error, calls the api module
api/         → requestApi.list(), requestApi.create() — the ONLY place fetch lives
api/mock/    → same interface, returns fake data (used until backend exists)
types/       → the shared data contract (see below)
```

Because `api/` and `api/mock/` expose the **same function signatures**, switching is a one-line change (an env flag or a single import swap).

## Step 1 — Define the contract first

Write the data shape before building UI or backend. This shape is the agreement between both sides.

**On the backend (FastAPI + Pydantic):**

```python
# schemas.py
from pydantic import BaseModel
from datetime import datetime
from enum import Enum

class RequestStatus(str, Enum):
    pending = "pending"
    approved = "approved"
    rejected = "rejected"

class ShopfloorRequest(BaseModel):
    id: int
    requester: str
    item: str
    quantity: int
    status: RequestStatus
    created_at: datetime

class ShopfloorRequestCreate(BaseModel):
    requester: str
    item: str
    quantity: int
```

**On the frontend (mirror the contract):**

```typescript
// types/request.ts
export type RequestStatus = "pending" | "approved" | "rejected";

export interface ShopfloorRequest {
  id: number;
  requester: string;
  item: string;
  quantity: number;
  status: RequestStatus;
  created_at: string; // ISO string over the wire
}

export type ShopfloorRequestCreate = Pick<
  ShopfloorRequest, "requester" | "item" | "quantity"
>;
```

**Keep the two in sync automatically when possible.** FastAPI serves an OpenAPI schema at `/openapi.json`. Generate TS types from it instead of hand-writing them:

```bash
npx openapi-typescript http://localhost:8000/openapi.json -o src/types/api.ts
```

This makes the backend the single source of truth — when a Pydantic model changes, regenerate and the compiler shows you every frontend break.

## Step 2 — Build the api module behind the seam

```typescript
// api/requestApi.ts
import type { ShopfloorRequest, ShopfloorRequestCreate } from "../types/request";

const BASE = import.meta.env.VITE_API_URL ?? "/api";

async function handle<T>(res: Response): Promise<T> {
  if (!res.ok) {
    // surface a typed, useful error — not a raw throw
    const detail = await res.json().catch(() => ({}));
    throw new ApiError(res.status, detail?.detail ?? res.statusText);
  }
  return res.json() as Promise<T>;
}

export const requestApi = {
  list: () =>
    fetch(`${BASE}/requests`).then(handle<ShopfloorRequest[]>),
  create: (body: ShopfloorRequestCreate) =>
    fetch(`${BASE}/requests`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    }).then(handle<ShopfloorRequest>),
};

export class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message);
  }
}
```

## Step 3 — Mock the same interface

Until the backend is live (or for tests/Storybook), provide a mock with an *identical signature*:

```typescript
// api/mock/requestApi.ts
import type { ShopfloorRequest, ShopfloorRequestCreate } from "../../types/request";

let db: ShopfloorRequest[] = [
  { id: 1, requester: "Zan", item: "Cutting fluid", quantity: 5,
    status: "pending", created_at: new Date().toISOString() },
];

const delay = (ms = 400) => new Promise(r => setTimeout(r, ms));

export const requestApi = {
  list: async () => { await delay(); return [...db]; },
  create: async (body: ShopfloorRequestCreate) => {
    await delay();
    const row = { id: db.length + 1, status: "pending" as const,
      created_at: new Date().toISOString(), ...body };
    db.push(row);
    return row;
  },
};
```

Swap with one switch — e.g. `api/index.ts` re-exports the real or mock module based on `import.meta.env.VITE_USE_MOCK`. Components import from `api/index` and never know the difference. The artificial `delay()` is deliberate: it forces you to build real loading states instead of assuming instant data.

## Step 4 — Wrap it in a hook that owns the three states

```typescript
// hooks/useRequests.ts  (custom-hook version; prefer TanStack Query in real apps)
import { useState, useEffect, useCallback } from "react";
import { requestApi } from "../api";
import type { ShopfloorRequest } from "../types/request";

export function useRequests() {
  const [data, setData] = useState<ShopfloorRequest[] | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(true);

  const refetch = useCallback(async () => {
    setLoading(true); setError(null);
    try { setData(await requestApi.list()); }
    catch (e) { setError(e as Error); }
    finally { setLoading(false); }
  }, []);

  useEffect(() => { refetch(); }, [refetch]);
  return { data, error, loading, refetch };
}
```

In real projects prefer **TanStack Query** (`useQuery`/`useMutation`) — it gives caching, dedup, retries, and background refetch for free, and removes most of this boilerplate. The hand-rolled version above is the fallback when a library isn't available, and it makes the three-state contract explicit.

## Cross-cutting concerns to design in from day one

- **Auth:** keep token/session handling in the api layer (an interceptor or a wrapped `fetch`), never in components. If the backend uses Keycloak/OIDC, the api module attaches the bearer token; components stay unaware.
- **CORS / proxy:** in dev, proxy `/api` to the FastAPI origin (Vite `server.proxy`) so the frontend code uses relative URLs and doesn't hardcode `localhost:8000`.
- **Error normalization:** FastAPI returns errors as `{ "detail": ... }` (string or validation array). Normalize these into one frontend error shape in `handle()` so the UI renders errors uniformly.
- **Validation parity:** validate forms against the same constraints as the Pydantic model (required fields, min/max, enums). Mismatched validation = the user passes the frontend check then gets a 422.
- **Optimistic updates:** for create/update, optionally update the UI before the server confirms, then roll back on error — TanStack Query's mutation API supports this cleanly.

## Backend-agnostic checklist

When the backend is something other than FastAPI, the seam doesn't change — only the contract source does:

| Concern | FastAPI/Pydantic | Generic REST/JSON |
|---|---|---|
| Contract source | `/openapi.json` (auto) | Hand-written types or a shared schema (OpenAPI/JSON Schema) |
| Error shape | `{ detail }` | Normalize whatever the API returns in `handle()` |
| Type generation | `openapi-typescript` | Same, if an OpenAPI doc exists; else manual |
| Validation parity | mirror Pydantic models | mirror the documented contract |

The rule that never changes: **components depend on the contract and the hooks; only the api layer depends on the actual backend.**
