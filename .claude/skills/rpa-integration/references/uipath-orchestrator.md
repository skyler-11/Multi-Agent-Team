# UiPath Orchestrator (both directions)

A round trip: your app **starts a job** (outbound) → the bot runs (minutes) → the bot **posts its result back** (inbound), correlated by an id you generate. Neither side blocks on the other.

## Auth — OAuth client credentials (External Application)

Register an External App in Orchestrator/Automation Cloud and use client-credentials. Cache the token; refresh on expiry. Never hardcode secrets.

```python
import httpx, time

class OrchestratorAuth:
    def __init__(self, s):  # s = Settings
        self.s, self._tok, self._exp = s, None, 0
    async def token(self) -> str:
        if self._tok and time.time() < self._exp - 60:
            return self._tok
        async with httpx.AsyncClient(timeout=10) as c:
            r = await c.post(self.s.uipath_token_url, data={
                "grant_type": "client_credentials",
                "client_id": self.s.uipath_client_id,
                "client_secret": self.s.uipath_client_secret,
                "scope": "OR.Jobs OR.Queues",
            })
            r.raise_for_status()
            d = r.json()
        self._tok, self._exp = d["access_token"], time.time() + d["expires_in"]
        return self._tok
```

Cloud calls also need tenant/org context in the URL and an `X-UIPATH-OrganizationUnitId` (folder) header — read those from settings, don't hardcode.

## Outbound — start a job

Expose intent-named methods in *your* vocabulary, not Orchestrator's:

```python
class OrchestratorClient:
    def __init__(self, auth, s): self.auth, self.s = auth, s

    async def start_request_processing(self, correlation_id: str, payload: dict) -> str:
        """Domain-named. Returns Orchestrator job id."""
        headers = {
            "Authorization": f"Bearer {await self.auth.token()}",
            "X-UIPATH-OrganizationUnitId": str(self.s.uipath_folder_id),
        }
        body = {"startInfo": {
            "ReleaseKey": self.s.uipath_release_key,
            "Strategy": "ModernJobsCount", "JobsCount": 1,
            "InputArguments": json.dumps({"correlationId": correlation_id, **payload}),
        }}
        async with httpx.AsyncClient(timeout=15) as c:
            r = await c.post(f"{self.s.uipath_base}/odata/Jobs/UiPath.Server."
                             f"Configuration.OData.StartJobs", json=body, headers=headers)
            r.raise_for_status()
            return str(r.json()["value"][0]["Id"])
```

You pass `correlationId` *into* the bot as an input argument so it can echo it back on callback. That id is how you reconnect the async result to the originating request.

**Alternative: Orchestrator Queues.** For high-volume or fire-and-forget, add a queue item instead of starting a job directly (`/odata/Queues(...)/UiPathODataSvc.AddQueueItem`); a bot picks it up on its own schedule. Choose queues when you don't need a job handle and want Orchestrator to manage throughput.

## Inbound — receive the bot's callback

The bot finishes and POSTs its result to your API. You own the handler + verification; `backend-developer` mounts the route. Verify a shared secret/HMAC, dedupe on the job/correlation id, then delegate to the backend service.

```python
async def handle_bot_callback(payload: BotResult, sig: str) -> None:
    verify_hmac(payload.raw, sig, settings.uipath_callback_secret)   # reject if bad
    if await seen(payload.correlation_id):     # idempotency — bots can retry
        return
    await mark_seen(payload.correlation_id)
    # delegate the domain change to the backend, don't touch the ORM here
    await request_service.complete(payload.correlation_id, payload.result)
```

Define the callback contract (the `BotResult` shape + the header carrying the signature) and put it in the handoff so backend mounts the route and the RPA team builds the bot to match.

## Reliability notes specific to Orchestrator

- Starting a job is quick (sync + short timeout + retry on 5xx/timeout). The *work* is async — never poll-block a request waiting for a job to finish; rely on the callback (or poll `Jobs(id)` from a background task if no callback is possible).
- Job-start can fail with no available robots — treat as retriable/backoff, surface a clean "queued/unavailable" state, not a 500.
- Always pass and log the correlation id; never log `InputArguments` if they carry sensitive data.
