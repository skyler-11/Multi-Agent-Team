# Inbound Webhooks

An external system POSTs to your API on its schedule, with retries you don't control. Three rules make inbound safe: **verify, dedupe, fast-ack.** You own this logic; `backend-developer` mounts the route.

## 1 — Verify authenticity (treat unverified as hostile)

Most providers sign the raw body with a shared secret (HMAC-SHA256) and send it in a header. Verify against the **raw bytes**, before parsing, in constant time.

```python
import hmac, hashlib

def verify_hmac(raw: bytes, signature: str, secret: str) -> None:
    expected = hmac.new(secret.encode(), raw, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, signature):   # constant-time
        raise HTTPException(401, "Bad signature")
```

If a provider can't sign, fall back to a secret path token + IP allowlist — but signed bodies are the standard. Never act on an unverified webhook.

## 2 — Idempotency (the same event will arrive twice)

Senders retry on any non-2xx or timeout, so you *will* receive duplicates. Dedupe on the provider's event id; make the check-and-record atomic.

```python
async def already_processed(event_id: str) -> bool:
    # INSERT ... ON CONFLICT DO NOTHING — returns False if it was new
    return not await events_repo.try_insert(event_id)
```

## 3 — Fast-ack, then work

Verify → record → return 2xx **immediately**; do the slow part after. A slow handler makes the sender time out and retry, multiplying load and risking duplicate side effects.

```python
@router.post("/webhooks/{provider}", status_code=202)   # route owned by backend
async def receive(provider: str, request: Request, bg: BackgroundTasks):
    raw = await request.body()
    verify_hmac(raw, request.headers.get("X-Signature", ""), secret_for(provider))
    event = parse_event(provider, raw)              # normalize to YOUR shape
    if await already_processed(event.id):
        return {"status": "duplicate"}              # still 2xx
    bg.add_task(process_event, event)               # slow work off the request
    return {"status": "accepted"}
```

If the project has a queue/worker, enqueue instead of `BackgroundTasks` so the event survives a restart between ack and processing (match the repo's approach).

## 4 — Normalize at the boundary

`parse_event` turns each provider's payload into *your* internal event model. Downstream code (and the backend service you delegate to) never sees the provider's raw shape — that's the anti-corruption layer.

## Contract handoff

Define and hand off: the route path, the expected payload shape, the signature header + algorithm, and the secret's env var name. `backend-developer` mounts the route; the external/RPA side builds to the same contract. Record it in `INTEGRATION_HANDOFF.md`.
