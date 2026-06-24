# Auth (pluggable)

Auth is a **dependency at the router boundary**, never threaded through services or repositories. Business logic receives an already-authenticated principal; it never parses tokens. This keeps the core auth-agnostic and lets you swap schemes without touching logic.

```python
# routers/requests.py
@router.get("/requests", response_model=list[RequestOut])
async def list_requests(user: User = Depends(get_current_user)):
    ...
```

Only `get_current_user` (and an optional `require_role(...)`) changes between the two modules below. Pick one per project.

---

## Module A — Simple JWT (lightweight default)

For self-contained services that issue their own tokens. Use `python-jose` (or `pyjwt`) + `passlib` for hashing.

```python
from datetime import datetime, timedelta
from jose import jwt, JWTError
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2 = OAuth2PasswordBearer(tokenUrl="auth/token")

def create_access_token(sub: str, expires_min: int = 30) -> str:
    payload = {"sub": sub, "exp": datetime.utcnow() + timedelta(minutes=expires_min)}
    return jwt.encode(payload, settings.jwt_secret, algorithm="HS256")

async def get_current_user(token: str = Depends(oauth2)) -> User:
    cred_err = HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid credentials")
    try:
        payload = jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])
        sub = payload.get("sub")
        if sub is None:
            raise cred_err
    except JWTError:
        raise cred_err
    user = await user_repo.get(sub)         # look up the real user
    if user is None:
        raise cred_err
    return user
```

Notes: store only a hash of passwords (`passlib` bcrypt), keep `jwt_secret` in env via `Settings`, keep access tokens short-lived, and add refresh tokens only if the client needs long sessions.

---

## Module B — Keycloak / OIDC + RBAC

When an external identity provider (Keycloak) owns identity and you need role-based access. The API **validates** tokens it didn't issue, by verifying the signature against the provider's public keys (JWKS) — it does not check passwords.

```python
from jose import jwt
import httpx

# fetch + cache the realm's JWKS once
async def _jwks():
    url = f"{settings.keycloak_url}/realms/{settings.realm}/protocol/openid-connect/certs"
    async with httpx.AsyncClient() as c:
        return (await c.get(url)).json()

async def get_current_user(token: str = Depends(oauth2)) -> User:
    try:
        claims = jwt.decode(
            token, await _jwks(),
            algorithms=["RS256"],
            audience=settings.client_id,
            issuer=f"{settings.keycloak_url}/realms/{settings.realm}",
        )
    except Exception:
        raise HTTPException(401, "Invalid token")
    return User(id=claims["sub"], username=claims.get("preferred_username"),
                roles=claims.get("realm_access", {}).get("roles", []))
```

**RBAC** as a composable dependency:

```python
def require_role(*allowed: str):
    async def dep(user: User = Depends(get_current_user)) -> User:
        if not set(allowed) & set(user.roles):
            raise HTTPException(403, "Insufficient role")
        return user
    return dep

# usage
@router.delete("/requests/{id}", dependencies=[Depends(require_role("admin"))])
async def delete_request(id: int): ...
```

Notes: roles come from Keycloak (`realm_access.roles` or client roles), not your DB. Cache JWKS and refresh on key rotation. Verify `aud` and `iss` — skipping them is a common security hole. Never trust claims without signature verification.

---

## Which to use

| Situation | Module |
|-----------|--------|
| Standalone service, you own the users | A — Simple JWT |
| Org SSO, central identity, shared roles across systems | B — Keycloak / OIDC + RBAC |
| No auth needed yet (internal/portfolio) | Neither — leave the boundary, add later |

Whatever you choose, the dependency signature (`get_current_user`) stays the same, so routers and the rest of the app don't change when auth does.
