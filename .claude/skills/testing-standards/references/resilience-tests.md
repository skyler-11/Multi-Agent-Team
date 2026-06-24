# Resilience / Chaos Tests

A happy-path suite proves the system works when everything works. Resilience tests prove it survives when external systems don't — and external systems always eventually don't. You inject failure at the adapter boundary and assert the app degrades gracefully instead of cascading into a 500 or losing data.

The principle: **don't break real things — substitute a fake that misbehaves on demand.** This is chaos testing scoped to a test suite (deterministic, fast), not chaos engineering in production.

## The misbehaving fake

Give each external adapter a test double you can command to fail:

```python
class FakeOrchestrator:
    def __init__(self): self.mode = "ok"; self.calls = 0
    async def start_request_processing(self, correlation_id, payload):
        self.calls += 1
        if self.mode == "timeout":   raise httpx.TimeoutException("boom")
        if self.mode == "5xx":       raise httpx.HTTPStatusError("500", request=..., response=...)
        if self.mode == "no_robots": raise NoRobotsAvailable()
        return "job-123"
```

Inject it via `dependency_overrides` (see test-database.md).

## What to assert (the failure contract)

| Inject | Assert |
|--------|--------|
| External call **times out** | It's retried up to the cap, then surfaced as a clean app error (e.g. 503 "unavailable"), never a raw 500/stack trace |
| External returns **5xx** | Retried with backoff; **4xx is NOT retried** (assert `fake.calls == 1`) |
| Same **webhook delivered twice** | Second is deduped — side effect happens once (`fake.calls == 1`, one DB row) |
| **Email send fails** | The triggering request still returns success; failure is recorded, not propagated |
| External **down entirely** | App stays up; affected feature degrades to a clear state; unrelated endpoints unaffected |

```python
async def test_timeout_is_retried_then_surfaced_cleanly(client, fake_orch):
    fake_orch.mode = "timeout"
    r = await client.post("/requests", json={"item": "x", "quantity": 1})
    assert r.status_code == 503                 # clean, not 500
    assert "stack" not in r.text.lower()        # no leakage
    assert fake_orch.calls > 1                   # it retried

async def test_duplicate_webhook_is_idempotent(client):
    payload, sig = signed_event(event_id="evt-1")
    a = await client.post("/webhooks/uipath", content=payload, headers={"X-Signature": sig})
    b = await client.post("/webhooks/uipath", content=payload, headers={"X-Signature": sig})
    assert a.status_code == 202 and b.status_code in (200, 202)
    assert await count_processed("evt-1") == 1   # processed once, not twice
```

## Controlling time (so retry/backoff tests are fast and deterministic)

Never `sleep()` through real backoff. Patch the clock / sleep so retry tests run instantly:

```python
async def test_retry_gives_up_after_cap(client, fake_orch, monkeypatch):
    monkeypatch.setattr(asyncio, "sleep", lambda *_: asyncio.sleep(0))  # collapse backoff
    fake_orch.mode = "5xx"
    r = await client.post("/requests", json={"item": "x", "quantity": 1})
    assert fake_orch.calls == settings.max_retries + 1
    assert r.status_code == 503
```

## Boundary

These tests assert *your* resilience handling, so they belong with the code that owns the failure response — usually the integration and backend layers. They mock the external system; they never call a real Orchestrator/SMTP/Graph endpoint. One resilience test per external failure mode is the minimum bar before that integration is "done."
