# SharePoint (Microsoft Graph)

Talk to SharePoint through the Microsoft Graph API, not legacy SOAP/CSOM. Auth is OAuth client-credentials against an Azure AD app registration with the right application permissions (e.g. `Sites.ReadWrite.All`), admin-consented.

## Auth — cache the token

```python
import httpx, time

class GraphAuth:
    def __init__(self, s): self.s, self._tok, self._exp = s, None, 0
    async def token(self) -> str:
        if self._tok and time.time() < self._exp - 60:
            return self._tok
        url = f"https://login.microsoftonline.com/{self.s.tenant_id}/oauth2/v2.0/token"
        async with httpx.AsyncClient(timeout=10) as c:
            r = await c.post(url, data={
                "grant_type": "client_credentials",
                "client_id": self.s.graph_client_id,
                "client_secret": self.s.graph_client_secret,
                "scope": "https://graph.microsoft.com/.default",
            })
            r.raise_for_status(); d = r.json()
        self._tok, self._exp = d["access_token"], time.time() + d["expires_in"]
        return self._tok
```

## Adapter — domain methods over Graph shapes

Resolve the site/list ids once (from config or a cached lookup); expose methods in your vocabulary and map Graph's `fields` envelope into your own model at the boundary.

```python
class SharePointClient:
    def __init__(self, auth, s): self.auth, self.s = auth, s

    async def _h(self): return {"Authorization": f"Bearer {await self.auth.token()}"}

    async def add_request_item(self, item: dict) -> str:
        url = (f"https://graph.microsoft.com/v1.0/sites/{self.s.site_id}"
               f"/lists/{self.s.list_id}/items")
        async with httpx.AsyncClient(timeout=15) as c:
            r = await c.post(url, json={"fields": item}, headers=await self._h())
            r.raise_for_status()
            return r.json()["id"]

    async def list_open_requests(self) -> list[Request]:
        url = (f"https://graph.microsoft.com/v1.0/sites/{self.s.site_id}"
               f"/lists/{self.s.list_id}/items?$expand=fields"
               f"&$filter=fields/Status eq 'Open'")
        async with httpx.AsyncClient(timeout=15) as c:
            r = await c.get(url, headers=await self._h()); r.raise_for_status()
            # normalize Graph's {value:[{fields:{...}}]} into YOUR model here
            return [Request.from_sharepoint(i["fields"]) for i in r.json()["value"]]
```

## Reliability

- **Throttling (429) is normal on Graph.** Honor the `Retry-After` header — wait exactly that long, then retry. This is the single most common Graph failure.
- Timeout every call; retry 429/5xx/timeout with backoff; never retry 4xx (except 429).
- For large reads, follow `@odata.nextLink` pagination rather than assuming one page.
- Files vs list items use different endpoints (`/drive/items` vs `/lists/items`) — keep them in separate adapter methods.
- Map Graph errors to your app's clean errors; don't surface Graph's `error.code` to end users.

## Boundary

The adapter returns *your* models, never raw Graph JSON. If a SharePoint write must also change app state, the adapter delegates to the backend service — it doesn't write the DB itself.
