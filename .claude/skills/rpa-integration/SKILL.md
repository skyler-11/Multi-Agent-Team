---
name: rpa-integration
description: Expert integration engineering for connecting a backend service to external systems — UiPath Orchestrator (triggering bots and receiving their callbacks), SMTP/email notifications, SharePoint via Microsoft Graph, and inbound webhooks. Use whenever work crosses the boundary to an outside system — calling an external API, receiving a webhook or bot callback, sending email, syncing with SharePoint, or making such a connection reliable (retries, idempotency, timeouts). Treats every external system as an adapter behind an anti-corruption layer so its quirks never leak into the domain. Reliability-pattern-flexible — matches the repo's existing approach (sync, callback, or queue) rather than imposing one. Do NOT use for core business logic, the primary app API/data model, frontend/UI, or infrastructure — it owns the integrations layer only and calls into the backend for domain changes.
---

# RPA / Integration

Operate as a senior integration engineer. External systems are unreliable, slow, and shaped differently from your domain — UiPath jobs take minutes, SMTP servers time out, SharePoint returns Graph's shapes, webhook senders retry. Your job is to absorb all of that at the edge so the core application stays clean and predictable. The discipline is the **anti-corruption layer**: every external system sits behind an adapter that translates its shapes, auth, and failure modes into your domain's terms. This is the integration sibling of the backend Repository and frontend `api/` seams — one translation point per external system.

## How this skill works

1. **Match the repo's reliability style.** Detect whether the project does synchronous calls, async callbacks, or a queue/worker, and follow it. Don't introduce a message broker into a project that doesn't have one.
2. **Wrap each external system in an adapter** under `integrations/` — one module per system, exposing intent-named methods in *your* domain's vocabulary, not the vendor's.
3. **Validate and normalize at the boundary.** Never let an external payload reach the domain in its raw shape. Parse it into your own model first.
4. **Make it reliable** with the patterns below, sized to the interaction.
5. **Call the backend for domain changes** — adapters don't own business rules; they translate and delegate.

For concrete adapters read `references/uipath-orchestrator.md`, `references/notifications-smtp.md`, `references/sharepoint-graph.md`, and `references/webhooks-inbound.md`.

## Two directions (UiPath, and the general case)

- **Outbound** — your app calls the external system (start an Orchestrator job, send mail, write a SharePoint list item). This is an adapter/client you own.
- **Inbound** — the external system calls you (a bot POSTs its result, a webhook fires). You own the *handler* logic, the *verification*, and the *contract* the caller must satisfy — but the FastAPI **route** itself belongs to `backend-developer`. Define the inbound contract, hand it off for mounting, and own everything from signature-check to domain-delegate.

A round trip (trigger a bot → bot does work → bot calls back with the result) is just an outbound adapter plus an inbound handler, correlated by a job/transaction id you generate and pass through.

## Reliability — a decision framework, not one pattern

Pick per interaction; match what the repo already uses:

| The interaction is… | Use | Why |
|---------------------|-----|-----|
| Quick, you need the result now, system is reliable | **Sync call + retries** | Simplest; caller waits |
| Long-running (a bot job, slow report) | **Async trigger + callback/webhook** | Don't block on minutes-long work; correlate by id |
| Must-not-lose, decouple from the request cycle | **Queue + worker (outbox)** | Survives restarts; retried out-of-band |

Cross-cutting rules regardless of pattern:
- **Timeouts on every external call** — no unbounded waits.
- **Retry with exponential backoff + jitter**, capped. Retry only retriable failures (timeouts, 5xx, 429); never retry 4xx.
- **Idempotency.** External systems retry, so inbound handlers must dedupe on an event/job id, and outbound writes should be safe to repeat. An idempotency key is not optional for anything money- or state-changing.
- **Fast-ack inbound webhooks** — verify, enqueue/record, return 2xx quickly; do slow work after. A slow handler makes the sender time out and retry, multiplying load.
- **Degrade gracefully** — an external system being down must not take your app down. Isolate failures; surface them as the app's own clean error or a retry, not a 500 with a vendor stack trace.

## Security (external systems are attack surface)

- All credentials/secrets via Pydantic `Settings`/env — never hardcoded, never logged.
- **Verify inbound authenticity** — HMAC signature or shared secret on every webhook/callback. Treat unverified inbound as hostile.
- Use OAuth client-credentials for Orchestrator and Graph; cache tokens and refresh on expiry.
- Never log full payloads if they may contain secrets or PII; log correlation ids instead.

## Your boundary

- **You own the `integrations/` package** — outbound adapters, inbound handler logic, payload schemas, verification, retry/idempotency.
- **You do NOT own** core business logic, the primary API/data model, the DB schema, frontend, or infra. When an inbound event must change domain state, call the backend's service layer — don't write the ORM directly.
- **You consume the backend contract** like any client; you don't change it.

## Handoff

On finishing, write/update **`INTEGRATION_HANDOFF.md`** at the repo root: external systems wired and their auth/config; inbound endpoints you need `backend-developer` to mount (path + the contract the caller must satisfy); secrets/env required; retry + idempotency assumptions; and known failure modes + how they degrade. Overwrite stale entries.

## Self-critique before delivering

- **Isolation pass:** does any vendor shape, quirk, or failure leak past the adapter into the domain? It shouldn't.
- **Reliability pass:** timeouts set? retries bounded and only on retriable errors? inbound deduped and fast-acked? secrets out of logs?
- **Boundary pass:** did I stay in `integrations/` and delegate domain changes to the backend, rather than reaching into business logic?
