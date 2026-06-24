# Test Database — in-memory SQLite + transactional rollback

Goal: every test gets a clean database, runs fast, and shares nothing with its neighbors. The pattern: one in-memory SQLite engine, each test wrapped in a transaction that is **rolled back** on teardown, with the app's `get_session` dependency overridden to use it.

## Async version (SQLAlchemy 2.0)

```python
import pytest_asyncio
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from httpx import AsyncClient, ASGITransport
from app.main import app
from app.db import Base, get_session

@pytest_asyncio.fixture
async def engine():
    # StaticPool keeps a single in-memory DB across connections in the test
    eng = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        connect_args={"check_same_thread": False},
        poolclass=__import__("sqlalchemy").pool.StaticPool,
    )
    async with eng.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield eng
    await eng.dispose()

@pytest_asyncio.fixture
async def session(engine) -> AsyncSession:
    """Each test runs in a transaction that is rolled back."""
    conn = await engine.connect()
    txn = await conn.begin()
    s = AsyncSession(bind=conn, expire_on_commit=False)
    yield s
    await s.close()
    await txn.rollback()          # undo everything the test did
    await conn.close()

@pytest_asyncio.fixture
async def client(session) -> AsyncClient:
    app.dependency_overrides[get_session] = lambda: session   # app uses the test session
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

## Sync version

```python
import pytest
from sqlalchemy import create_engine, pool
from sqlalchemy.orm import sessionmaker
from fastapi.testclient import TestClient

@pytest.fixture
def engine():
    eng = create_engine("sqlite:///:memory:", connect_args={"check_same_thread": False},
                        poolclass=pool.StaticPool)
    Base.metadata.create_all(eng); yield eng; eng.dispose()

@pytest.fixture
def session(engine):
    conn = engine.connect(); txn = conn.begin()
    s = sessionmaker(bind=conn)()
    yield s
    s.close(); txn.rollback(); conn.close()

@pytest.fixture
def client(session):
    app.dependency_overrides[get_session] = lambda: session
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

## Why these choices

- **In-memory + StaticPool** — no file on disk, no cleanup, and StaticPool ensures every connection in the test sees the *same* in-memory DB (without it, each connection gets a fresh empty one).
- **Transaction rollback over `create_all`/`drop_all` per test** — rolling back a transaction is far faster than rebuilding the schema each test, and guarantees isolation even if a test forgets to clean up.
- **`dependency_overrides`** — the app code is unchanged; tests just swap the session (and `get_current_user`, the email sender, the integration adapters) for test doubles.

## Overriding auth and integrations

```python
app.dependency_overrides[get_current_user] = lambda: User(id="t", roles=["admin"])
app.dependency_overrides[get_orchestrator_client] = lambda: FakeOrchestrator()
```

Override at the **dependency/adapter boundary**, never by patching deep internals. If something is hard to override, that usually means a missing seam — flag it rather than reaching in with monkeypatch.

## SQLite caveat

SQLite enforces fewer constraints than Postgres (looser typing, FK off unless enabled). For schema/constraint-sensitive tests, enable `PRAGMA foreign_keys=ON` per connection, and run the contract/migration tests against Postgres in CI if the production target is Postgres.
