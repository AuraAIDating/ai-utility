---
name: python-development
description: "Python development patterns: project structure, FastAPI for HTTP APIs, async patterns, dependency injection, Pydantic models, pytest unit/component testing, BDD with behave/pytest-bdd, type hints, linting, packaging, and production best practices."
---

# Python Development Patterns

Standards and best practices for Python (3.11+) development in Polaris services.

## When to Activate

- Writing or reviewing Python code
- Creating FastAPI services or HTTP APIs
- Writing Python tests (unit, component, BDD)
- Structuring Python projects
- Working with async/await patterns
- Configuring Python tooling (linting, formatting, packaging)

## Project Structure

```
service-name/
├── src/
│   └── service_name/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app entry point
│       ├── config.py                # Settings via pydantic-settings
│       ├── dependencies.py          # Dependency injection providers
│       ├── api/
│       │   ├── __init__.py
│       │   ├── router.py            # Top-level router aggregation
│       │   └── v1/
│       │       ├── __init__.py
│       │       ├── endpoints/
│       │       │   ├── __init__.py
│       │       │   ├── orders.py
│       │       │   └── patients.py
│       │       └── schemas/         # Pydantic request/response models
│       │           ├── __init__.py
│       │           ├── orders.py
│       │           └── patients.py
│       ├── domain/
│       │   ├── __init__.py
│       │   ├── models.py            # Domain entities (dataclasses or Pydantic)
│       │   ├── services.py          # Business logic
│       │   └── exceptions.py        # Domain-specific exceptions
│       ├── infrastructure/
│       │   ├── __init__.py
│       │   ├── database.py          # SQLAlchemy engine/session setup
│       │   ├── repositories.py      # Data access layer
│       │   ├── kafka/
│       │   │   ├── __init__.py
│       │   │   ├── producer.py
│       │   │   └── consumer.py
│       │   └── external/            # External API clients
│       │       ├── __init__.py
│       │       └── cms_client.py
│       └── common/
│           ├── __init__.py
│           ├── logging.py           # Structured logging setup
│           └── middleware.py         # CORS, tracing, error handling
├── tests/
│   ├── __init__.py
│   ├── conftest.py                  # Shared fixtures
│   ├── unit/
│   │   ├── __init__.py
│   │   ├── conftest.py
│   │   ├── test_services.py
│   │   └── test_models.py
│   ├── component/
│   │   ├── __init__.py
│   │   ├── conftest.py              # Testcontainers, httpx AsyncClient
│   │   ├── test_orders_api.py
│   │   └── test_patients_api.py
│   └── bdd/
│       ├── __init__.py
│       ├── conftest.py
│       ├── features/
│       │   ├── orders.feature
│       │   └── patients.feature
│       └── step_defs/
│           ├── __init__.py
│           ├── test_orders.py
│           └── test_patients.py
├── alembic/                         # Database migrations
│   ├── env.py
│   └── versions/
├── alembic.ini
├── pyproject.toml                   # Single source for project config
├── Dockerfile
├── docker-compose.yml
└── README.md
```

### Key Principles

- **src layout** — all source code under `src/service_name/`, avoids import confusion
- **Layered architecture** — `api/` -> `domain/` -> `infrastructure/`
- **No business logic in endpoints** — endpoints call domain services
- **Schemas separate from domain models** — Pydantic schemas for API, domain models for business logic

## FastAPI Patterns

### Application Factory

```python
# src/service_name/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from service_name.api.router import api_router
from service_name.config import settings
from service_name.common.middleware import setup_middleware
from service_name.infrastructure.database import init_db, close_db


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown events."""
    await init_db()
    yield
    await close_db()


def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.app_name,
        version=settings.app_version,
        lifespan=lifespan,
    )
    setup_middleware(app)
    app.include_router(api_router, prefix="/api")
    return app


app = create_app()
```

### Configuration with pydantic-settings

```python
# src/service_name/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    app_name: str = "service-name"
    app_version: str = "0.1.0"
    debug: bool = False

    # Database
    database_url: str
    database_pool_size: int = 5
    database_max_overflow: int = 10

    # Kafka
    kafka_bootstrap_servers: str = "localhost:9092"
    kafka_consumer_group: str = "service-name-group"

    # External APIs
    cms_api_url: str
    cms_api_timeout: int = 30


settings = Settings()
```

### Dependency Injection

```python
# src/service_name/dependencies.py
from typing import Annotated, AsyncGenerator
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from service_name.infrastructure.database import get_session
from service_name.infrastructure.repositories import OrderRepository
from service_name.domain.services import OrderService


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with get_session() as session:
        yield session


async def get_order_repository(
    db: Annotated[AsyncSession, Depends(get_db)],
) -> OrderRepository:
    return OrderRepository(db)


async def get_order_service(
    repo: Annotated[OrderRepository, Depends(get_order_repository)],
) -> OrderService:
    return OrderService(repo)
```

### Endpoints

```python
# src/service_name/api/v1/endpoints/orders.py
from typing import Annotated
from uuid import UUID
from fastapi import APIRouter, Depends, HTTPException, status
from service_name.api.v1.schemas.orders import (
    OrderCreate,
    OrderResponse,
    OrderListResponse,
)
from service_name.dependencies import get_order_service
from service_name.domain.services import OrderService
from service_name.domain.exceptions import OrderNotFoundError

router = APIRouter(prefix="/orders", tags=["orders"])


@router.post(
    "",
    response_model=OrderResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create a new order",
)
async def create_order(
    payload: OrderCreate,
    service: Annotated[OrderService, Depends(get_order_service)],
) -> OrderResponse:
    order = await service.create(payload)
    return OrderResponse.model_validate(order)


@router.get("/{order_id}", response_model=OrderResponse)
async def get_order(
    order_id: UUID,
    service: Annotated[OrderService, Depends(get_order_service)],
) -> OrderResponse:
    try:
        order = await service.get_by_id(order_id)
    except OrderNotFoundError:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Order {order_id} not found",
        )
    return OrderResponse.model_validate(order)
```

### Pydantic Schemas

```python
# src/service_name/api/v1/schemas/orders.py
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel, ConfigDict, Field


class OrderCreate(BaseModel):
    patient_id: UUID
    provider_npi: str = Field(..., pattern=r"^\d{10}$")
    icd10_codes: list[str] = Field(..., min_length=1)


class OrderResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    patient_id: UUID
    provider_npi: str
    icd10_codes: list[str]
    status: str
    created_at: datetime
    updated_at: datetime


class OrderListResponse(BaseModel):
    items: list[OrderResponse]
    total: int
    page: int
    page_size: int
```

### Error Handling

```python
# src/service_name/common/middleware.py
import uuid
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
from service_name.domain.exceptions import DomainError


def setup_middleware(app: FastAPI) -> None:
    app.add_middleware(RequestIdMiddleware)
    app.add_exception_handler(DomainError, domain_error_handler)


class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-Id", str(uuid.uuid4()))
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers["X-Request-Id"] = request_id
        return response


async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    """RFC 7807 problem detail response."""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "type": exc.error_type,
            "title": exc.title,
            "status": exc.status_code,
            "detail": str(exc),
            "instance": str(request.url),
        },
    )
```

### Domain Exceptions

```python
# src/service_name/domain/exceptions.py
class DomainError(Exception):
    status_code: int = 500
    error_type: str = "about:blank"
    title: str = "Internal Server Error"


class OrderNotFoundError(DomainError):
    status_code = 404
    error_type = "urn:polaris:error:order-not-found"
    title = "Order Not Found"


class ValidationError(DomainError):
    status_code = 422
    error_type = "urn:polaris:error:validation"
    title = "Validation Error"
```

## Async Patterns

### SQLAlchemy Async

```python
# src/service_name/infrastructure/database.py
from contextlib import asynccontextmanager
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from service_name.config import settings

engine = create_async_engine(
    settings.database_url,
    pool_size=settings.database_pool_size,
    max_overflow=settings.database_max_overflow,
    echo=settings.debug,
)

async_session = async_sessionmaker(engine, expire_on_commit=False)


async def init_db() -> None:
    """Initialize database connection pool."""
    async with engine.begin() as conn:
        await conn.run_sync(lambda _: None)  # verify connectivity


async def close_db() -> None:
    await engine.dispose()


@asynccontextmanager
async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### Repository Pattern

```python
# src/service_name/infrastructure/repositories.py
from uuid import UUID
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from service_name.domain.models import Order
from service_name.domain.exceptions import OrderNotFoundError


class OrderRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def create(self, order: Order) -> Order:
        self._session.add(order)
        await self._session.flush()
        return order

    async def get_by_id(self, order_id: UUID) -> Order:
        result = await self._session.execute(
            select(Order).where(Order.id == order_id)
        )
        order = result.scalar_one_or_none()
        if order is None:
            raise OrderNotFoundError(f"Order {order_id} not found")
        return order
```

## Testing

### Unit Tests (pytest)

```python
# tests/unit/test_services.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4
from service_name.domain.services import OrderService
from service_name.domain.exceptions import OrderNotFoundError


@pytest.fixture
def mock_repo():
    return AsyncMock()


@pytest.fixture
def service(mock_repo):
    return OrderService(mock_repo)


class TestOrderService:
    async def test_create_order_returns_order(self, service, mock_repo):
        # Arrange
        mock_repo.create.return_value = MagicMock(id=uuid4(), status="pending")

        # Act
        result = await service.create(MagicMock())

        # Assert
        assert result.status == "pending"
        mock_repo.create.assert_awaited_once()

    async def test_get_by_id_raises_when_not_found(self, service, mock_repo):
        # Arrange
        mock_repo.get_by_id.side_effect = OrderNotFoundError("not found")

        # Act & Assert
        with pytest.raises(OrderNotFoundError):
            await service.get_by_id(uuid4())
```

### Component Tests (httpx + Testcontainers)

```python
# tests/component/conftest.py
import pytest
from httpx import ASGITransport, AsyncClient
from testcontainers.postgres import PostgresContainer
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from service_name.main import create_app
from service_name.infrastructure.database import get_session


@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture
async def app(postgres):
    """Create app with real database."""
    app = create_app()
    engine = create_async_engine(postgres.get_connection_url().replace("psycopg2", "asyncpg"))
    session_factory = async_sessionmaker(engine, expire_on_commit=False)

    async def override_session():
        async with session_factory() as session:
            yield session

    app.dependency_overrides[get_session] = override_session
    yield app
    app.dependency_overrides.clear()


@pytest.fixture
async def client(app):
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac
```

```python
# tests/component/test_orders_api.py
import pytest
from httpx import AsyncClient


class TestOrdersAPI:
    async def test_create_order_returns_201(self, client: AsyncClient):
        response = await client.post(
            "/api/v1/orders",
            json={
                "patient_id": "550e8400-e29b-41d4-a716-446655440000",
                "provider_npi": "1234567893",
                "icd10_codes": ["J06.9"],
            },
        )
        assert response.status_code == 201
        data = response.json()
        assert data["status"] == "pending"
        assert "id" in data

    async def test_get_nonexistent_order_returns_404(self, client: AsyncClient):
        response = await client.get(
            "/api/v1/orders/550e8400-e29b-41d4-a716-446655440000"
        )
        assert response.status_code == 404
        assert response.json()["type"] == "urn:polaris:error:order-not-found"
```

### BDD Tests (pytest-bdd)

```gherkin
# tests/bdd/features/orders.feature
Feature: Order Management

  Scenario: Successfully create an order
    Given a valid patient exists
    And a valid provider with NPI "1234567893"
    When I submit a new order with ICD-10 code "J06.9"
    Then the order is created with status "pending"
    And the order has a valid UUID

  Scenario: Reject order with invalid NPI
    Given a valid patient exists
    When I submit a new order with NPI "invalid"
    Then the request is rejected with status 422

  Scenario: Reject order with no diagnosis codes
    Given a valid patient exists
    And a valid provider with NPI "1234567893"
    When I submit a new order with no ICD-10 codes
    Then the request is rejected with status 422
```

```python
# tests/bdd/step_defs/test_orders.py
import pytest
from pytest_bdd import scenario, given, when, then, parsers

@scenario("../features/orders.feature", "Successfully create an order")
def test_create_order():
    pass


@given("a valid patient exists")
def patient(test_patient):
    return test_patient


@given(parsers.parse('a valid provider with NPI "{npi}"'))
def provider_npi(npi):
    return npi


@when(parsers.parse('I submit a new order with ICD-10 code "{code}"'))
def submit_order(client, patient, provider_npi, code):
    response = client.post(
        "/api/v1/orders",
        json={
            "patient_id": str(patient.id),
            "provider_npi": provider_npi,
            "icd10_codes": [code],
        },
    )
    return response


@then(parsers.parse('the order is created with status "{status}"'))
def order_created(submit_order, status):
    assert submit_order.status_code == 201
    assert submit_order.json()["status"] == status


@then("the order has a valid UUID")
def order_has_uuid(submit_order):
    assert "id" in submit_order.json()
```

## Type Hints

- **Always use type hints** on function signatures and class attributes
- Use `typing.Annotated` for FastAPI dependencies
- Use modern syntax: `list[str]` not `List[str]`, `str | None` not `Optional[str]`
- Use `TypeVar` and `Generic` for reusable base classes

```python
# Correct
def process(items: list[str], count: int = 10) -> dict[str, int]: ...
async def get_item(item_id: UUID) -> Item | None: ...

# Incorrect
def process(items, count=10): ...
def get_item(item_id) -> Optional[Item]: ...
```

## Tooling & Configuration

### pyproject.toml (single config file)

```toml
[project]
name = "service-name"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "pydantic>=2.0",
    "pydantic-settings>=2.0",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.29",
    "alembic>=1.13",
    "structlog>=24.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-bdd>=7.0",
    "pytest-cov>=5.0",
    "httpx>=0.27",
    "testcontainers[postgres,kafka]>=4.0",
    "ruff>=0.5",
    "mypy>=1.10",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-v --strict-markers --cov=src --cov-report=term-missing"

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM", "TCH", "RUF"]

[tool.mypy]
python_version = "3.11"
strict = true
plugins = ["pydantic.mypy"]
```

## Structured Logging

```python
# src/service_name/common/logging.py
import structlog


def setup_logging() -> None:
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.stdlib.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.stdlib.BoundLogger,
        logger_factory=structlog.stdlib.LoggerFactory(),
    )
```

- Use `structlog` for structured JSON logging
- Bind request_id and correlation_id to log context
- Never log PHI (patient names, SSNs, dates of birth, etc.)
- Log at appropriate levels: ERROR for failures, WARNING for degraded, INFO for business events, DEBUG for development

## Coding Standards

- **Formatting**: `ruff format` (Black-compatible)
- **Linting**: `ruff check` with auto-fix
- **Type checking**: `mypy --strict`
- **Naming**: `snake_case` for functions/variables, `PascalCase` for classes, `UPPER_SNAKE` for constants
- **Imports**: sorted by `ruff` (isort-compatible), absolute imports preferred
- **Docstrings**: Google style, required for public APIs
- **No bare `except`** — always catch specific exceptions
- **No mutable default arguments** — use `None` and assign in function body
- **Prefer `dataclasses` or Pydantic `BaseModel`** over plain dicts for structured data
- **Use `pathlib.Path`** over `os.path`
- **Use `enum.Enum`** for fixed sets of values
