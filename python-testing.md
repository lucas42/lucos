# Python / FastAPI Testing Patterns

Conventions established in lucos_photos. Apply to any Python FastAPI + SQLAlchemy service.

## Test stack

- **pytest** — test runner (`api/requirements-test.txt`)
- **FastAPI TestClient** — HTTP request simulation (uses `httpx`, already a prod dep)
- **SQLite in-memory** — replaces PostgreSQL for unit tests; no Docker needed
- `pytest.ini` in `api/` with `pythonpath = .` and `testpaths = tests`

## File layout

```
api/
  pytest.ini
  requirements-test.txt     # just: pytest
  tests/
    __init__.py
    conftest.py             # fixtures: db_session, client
    test_main.py
```

## conftest.py essentials

```python
import os
# Must be set before app is imported — database.py constructs the SQLAlchemy
# engine at module load time by reading these from the environment.
# The Postgres engine is never actually used in tests (get_db is overridden).
os.environ.setdefault("POSTGRES_USER", "test")
os.environ.setdefault("POSTGRES_PASSWORD", "test")

import lucos_photos_common.models  # noqa: F401 — registers models with Base.metadata
from lucos_photos_common.database import Base
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool  # required — see note below

@pytest.fixture
def db_session():
    engine = create_engine(
        "sqlite:///:memory:",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,   # required — see note below
    )
    Base.metadata.create_all(engine)
    session = sessionmaker(bind=engine)()
    yield session
    session.close()
    Base.metadata.drop_all(engine)

@pytest.fixture
def client(db_session, tmp_path, monkeypatch):
    monkeypatch.setenv("CLIENT_KEYS", "test:development=validkey")
    monkeypatch.setattr(main_module, "UPLOADS_DIR", tmp_path)

    def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

## Gotchas

**`StaticPool` is required for SQLite in-memory.** Without it, SQLAlchemy opens a
new connection for each operation. Each new connection to `sqlite:///:memory:` gets
a fresh empty database, so tables created by `create_all` are invisible to the
session. `StaticPool` forces all requests through the same underlying connection.

**Import-time engine construction.** `database.py` calls `URL.create()` and
`create_engine()` at module level, reading `POSTGRES_USER` and `POSTGRES_PASSWORD`
from the environment immediately. Set dummy values via `os.environ.setdefault`
*before* any import of the app module — fixtures run too late.

**Explicit model import before `create_all`.** Even if `app.main` imports the models,
add `import lucos_photos_common.models` explicitly in conftest.py. This guarantees
the models are registered with `Base.metadata` regardless of import order.

**`python-multipart` is a runtime dep.** FastAPI raises a `RuntimeError` at startup
(not at request time) if `python-multipart` is missing and any route accepts file
uploads. Add it to `api/requirements.txt`, not just test deps.
