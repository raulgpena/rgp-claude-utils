# CLAUDE.python.md — Python Conventions
## Microservices · Clean & Hexagonal Architecture

> **Companion to [CLAUDE.md](./CLAUDE.md)** — read both files when working on a Python service.
> Rules in `CLAUDE.md` take precedence; this file adds Python-specific detail.

---

## 1. Stack & Versions

| Concern        | Choice |
|----------------|--------|
| Language       | Python 3.12+ |
| HTTP framework | FastAPI 0.111+ (infrastructure only) |
| Validation     | Pydantic v2 (DTOs and config only; not in domain) |
| Persistence    | SQLAlchemy 2.x async ORM (infrastructure only) |
| Migrations     | Alembic |
| Messaging      | `aiokafka` (Kafka) or `aio-pika` (RabbitMQ) |
| HTTP client    | `httpx` (async) |
| DI             | Manual constructor injection or `dependency-injector` library |
| Testing        | `pytest`, `pytest-asyncio`, `pytest-mock`, `testcontainers-python` |
| Observability  | `opentelemetry-sdk`, `structlog`, `prometheus-client` |
| Type checking  | `mypy --strict` enforced in CI |
| Linting        | `ruff` + `black` (pre-commit hooks) |
| Architecture   | `import-linter` enforces layer boundaries in CI |

---

## 2. Package Structure

```
<service>/
├── src/
│   ├── domain/
│   │   ├── __init__.py
│   │   ├── model/
│   │   │   ├── __init__.py
│   │   │   └── order.py          # Entity, Value Objects, Aggregate
│   │   ├── events/
│   │   │   ├── __init__.py
│   │   │   └── order_events.py
│   │   ├── ports/
│   │   │   ├── __init__.py
│   │   │   ├── inbound.py        # Protocol classes for use cases
│   │   │   └── outbound.py       # Protocol classes for repos, publishers
│   │   └── exceptions.py
│   │
│   ├── application/
│   │   ├── __init__.py
│   │   ├── use_cases/
│   │   │   ├── __init__.py
│   │   │   └── place_order.py    # One file per use case
│   │   └── dto/
│   │       ├── __init__.py
│   │       ├── commands.py       # Pydantic BaseModel command classes
│   │       └── responses.py
│   │
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   ├── inbound/
│   │   │   ├── http/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── router.py     # FastAPI APIRouter
│   │   │   │   └── schemas.py    # Pydantic request/response schemas
│   │   │   └── messaging/
│   │   │       └── consumer.py
│   │   └── outbound/
│   │       ├── persistence/
│   │       │   ├── orm_models.py  # SQLAlchemy mapped classes
│   │       │   ├── repository.py  # Implements domain port
│   │       │   └── mapper.py
│   │       ├── messaging/
│   │       │   └── producer.py
│   │       └── client/
│   │           └── inventory_client.py
│   │
│   └── config/
│       ├── __init__.py
│       ├── container.py          # DI wiring; creates FastAPI app
│       └── settings.py           # Pydantic BaseSettings
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── alembic/
├── main.py                       # Entry point: imports container.create_app()
├── pyproject.toml
└── .importlinter                 # import-linter configuration
```

---

## 3. Domain Layer

> **Zero framework imports** — no `fastapi`, `sqlalchemy`, `pydantic`, or `aiokafka` inside `domain/`.

### Entities
- Use `@dataclass(eq=False)` — identity equality by `id`, not by value.
- Expose state mutation only through domain methods; no public attribute assignment from outside.
- Enforce invariants in `__post_init__` or `@classmethod` factory methods.

```python
# domain/model/order.py
from __future__ import annotations
from dataclasses import dataclass, field
from domain.exceptions import DomainException


@dataclass(eq=False)
class Order:
    id: OrderId
    lines: list[OrderLine]
    status: str = field(default="PENDING", init=False)

    def __post_init__(self) -> None:
        if not self.lines:
            raise DomainException("Order must have at least one line.")

    @classmethod
    def create(cls, order_id: OrderId, lines: list[OrderLine]) -> Order:
        return cls(id=order_id, lines=list(lines))  # defensive copy

    def confirm(self) -> None:
        if self.status != "PENDING":
            raise DomainException("Only PENDING orders can be confirmed.")
        self.status = "CONFIRMED"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Order):
            return NotImplemented
        return self.id == other.id

    def __hash__(self) -> int:
        return hash(self.id)
```

### Value Objects
- Use `@dataclass(frozen=True)` — immutable, value equality, hashable.
- Validate in `__post_init__`.

```python
# domain/model/order.py (continued)
@dataclass(frozen=True)
class OrderId:
    value: str

    def __post_init__(self) -> None:
        if not self.value or not self.value.strip():
            raise DomainException("OrderId cannot be blank.")

    @classmethod
    def generate(cls) -> OrderId:
        import uuid
        return cls(value=str(uuid.uuid4()))


@dataclass(frozen=True)
class OrderLine:
    sku: str
    quantity: int

    def __post_init__(self) -> None:
        if self.quantity <= 0:
            raise DomainException("Quantity must be positive.")
```

### Domain Events
```python
# domain/events/order_events.py
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime, timezone
from domain.model.order import OrderId


@dataclass(frozen=True)
class DomainEvent:
    occurred_at: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc), compare=False
    )


@dataclass(frozen=True)
class OrderPlaced(DomainEvent):
    order_id: OrderId = field(default_factory=lambda: OrderId(""))  # always provide
```

### Port Interfaces (Protocols)
- Use `typing.Protocol` — structural, no inheritance required from implementations.
- Keep protocols in `domain/ports/` — they belong to the domain, not the infrastructure.

```python
# domain/ports/outbound.py
from __future__ import annotations
from typing import Protocol
from domain.model.order import Order, OrderId
from domain.events.order_events import DomainEvent


class OrderRepository(Protocol):
    async def save(self, order: Order) -> None: ...
    async def find_by_id(self, order_id: OrderId) -> Order | None: ...


class OrderEventPublisher(Protocol):
    async def publish(self, event: DomainEvent) -> None: ...


# domain/ports/inbound.py
from __future__ import annotations
from typing import Protocol
from domain.model.order import OrderId
from application.dto.commands import PlaceOrderCommand


class PlaceOrderUseCase(Protocol):
    async def execute(self, command: PlaceOrderCommand) -> OrderId: ...
```

### Domain Exceptions
```python
# domain/exceptions.py
class DomainException(Exception):
    """Base class for all domain exceptions."""

class OrderNotFoundException(DomainException):
    def __init__(self, order_id: str) -> None:
        super().__init__(f"Order '{order_id}' not found.")
```

---

## 4. Application Layer

- Use case classes receive ports via constructor injection (`__init__`).
- Commands and responses are **Pydantic `BaseModel`** classes (DTO layer only; do not pass them into domain).
- Application code may use `asyncio` but must **not** import FastAPI, SQLAlchemy, or any broker library.
- Raise `ApplicationException` for flow errors; let `DomainException` propagate as-is.

```python
# application/dto/commands.py
from pydantic import BaseModel, Field


class OrderLineInput(BaseModel):
    sku: str
    quantity: int = Field(gt=0)


class PlaceOrderCommand(BaseModel):
    lines: list[OrderLineInput] = Field(min_length=1)


# application/use_cases/place_order.py
from __future__ import annotations
from domain.model.order import Order, OrderId, OrderLine
from domain.ports.outbound import OrderRepository, OrderEventPublisher
from domain.events.order_events import OrderPlaced
from application.dto.commands import PlaceOrderCommand


class PlaceOrderUseCase:
    def __init__(
        self,
        repo: OrderRepository,
        publisher: OrderEventPublisher,
    ) -> None:
        self._repo = repo
        self._publisher = publisher

    async def execute(self, command: PlaceOrderCommand) -> OrderId:
        order_id = OrderId.generate()
        lines = [OrderLine(sku=l.sku, quantity=l.quantity) for l in command.lines]
        order = Order.create(order_id, lines)
        await self._repo.save(order)
        await self._publisher.publish(OrderPlaced(order_id=order_id))
        return order_id
```

---

## 5. Infrastructure Layer

### Persistence
- SQLAlchemy mapped classes (`MappedColumn`, `mapped_column`) live **only** in `infrastructure/outbound/persistence/orm_models.py`.
- Repository class implements the domain `OrderRepository` Protocol.
- `mapper.py` contains pure functions that convert between ORM model ↔ domain object.
- Use Alembic for all schema migrations; **never** call `Base.metadata.create_all()` in production.

```python
# infrastructure/outbound/persistence/orm_models.py
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class OrderModel(Base):
    __tablename__ = "orders"
    id: Mapped[str] = mapped_column(primary_key=True)
    status: Mapped[str]
    lines: Mapped[list["OrderLineModel"]] = relationship(cascade="all, delete-orphan")


# infrastructure/outbound/persistence/repository.py
from sqlalchemy.ext.asyncio import AsyncSession
from domain.model.order import Order, OrderId
from .orm_models import OrderModel
from .mapper import to_orm, to_domain

class SqlAlchemyOrderRepo:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def save(self, order: Order) -> None:
        orm = to_orm(order)
        await self._session.merge(orm)

    async def find_by_id(self, order_id: OrderId) -> Order | None:
        result = await self._session.get(OrderModel, order_id.value)
        return to_domain(result) if result else None
```

### HTTP Adapter (FastAPI)
- `APIRouter` lives in `infrastructure/inbound/http/router.py`.
- Inject the **use case** via FastAPI's `Depends` — never inject SQLAlchemy session into a router directly.
- Pydantic `schemas.py` handles HTTP serialisation; never expose domain objects directly to HTTP.
- Map `DomainException` → HTTP status codes in an exception handler registered in `config/container.py`.

```python
# infrastructure/inbound/http/router.py
from fastapi import APIRouter, Depends, status
from application.use_cases.place_order import PlaceOrderUseCase
from infrastructure.inbound.http.schemas import PlaceOrderRequest, PlaceOrderResponse
from config.container import get_place_order_use_case

router = APIRouter(prefix="/api/v1/orders", tags=["orders"])

@router.post("/", status_code=status.HTTP_201_CREATED, response_model=PlaceOrderResponse)
async def place_order(
    request: PlaceOrderRequest,
    use_case: PlaceOrderUseCase = Depends(get_place_order_use_case),
) -> PlaceOrderResponse:
    command = request.to_command()
    order_id = await use_case.execute(command)
    return PlaceOrderResponse(id=order_id.value)
```

### DI Wiring
- Wire everything in `config/container.py`; use FastAPI's dependency injection system.
- `main.py` only calls `create_app()` and starts uvicorn.

```python
# config/container.py
from fastapi import FastAPI
from fastapi.responses import JSONResponse
from domain.exceptions import DomainException
from infrastructure.outbound.persistence.repository import SqlAlchemyOrderRepo
from infrastructure.outbound.messaging.producer import KafkaOrderPublisher
from application.use_cases.place_order import PlaceOrderUseCase
from infrastructure.inbound.http.router import router as order_router

def create_app() -> FastAPI:
    app = FastAPI(title="Order Service")

    @app.exception_handler(DomainException)
    async def domain_exception_handler(request, exc: DomainException):
        return JSONResponse(status_code=422, content={"detail": str(exc)})

    app.include_router(order_router)
    return app

# Dependency providers
def get_place_order_use_case() -> PlaceOrderUseCase:
    # In production, inject session from async_session_factory
    repo = SqlAlchemyOrderRepo(session=get_async_session())
    publisher = KafkaOrderPublisher()
    return PlaceOrderUseCase(repo=repo, publisher=publisher)
```

---

## 6. Testing

### Unit tests
- `pytest` + `pytest-asyncio` — zero I/O, mock ports with `unittest.mock.AsyncMock`.

```python
# tests/unit/test_place_order.py
import pytest
from unittest.mock import AsyncMock
from application.use_cases.place_order import PlaceOrderUseCase
from application.dto.commands import PlaceOrderCommand, OrderLineInput
from domain.exceptions import DomainException

@pytest.fixture
def repo():   return AsyncMock()
@pytest.fixture
def publisher(): return AsyncMock()
@pytest.fixture
def use_case(repo, publisher): return PlaceOrderUseCase(repo=repo, publisher=publisher)

@pytest.mark.asyncio
async def test_publishes_event_on_success(use_case, repo, publisher):
    command = PlaceOrderCommand(lines=[OrderLineInput(sku="A", quantity=1)])
    await use_case.execute(command)
    publisher.publish.assert_awaited_once()

@pytest.mark.asyncio
async def test_rejects_empty_lines(use_case):
    with pytest.raises(Exception):  # Pydantic or DomainException
        PlaceOrderCommand(lines=[])
```

### Integration tests
- `testcontainers-python` for Postgres.
- `pytest-asyncio` with `asyncio_mode = "auto"` in `pyproject.toml`.

```python
# tests/integration/test_sqlalchemy_order_repo.py
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:16") as pg:
        yield pg

@pytest.mark.asyncio
async def test_save_and_find(postgres):
    engine = create_async_engine(postgres.get_connection_url())
    async with AsyncSession(engine) as session:
        repo = SqlAlchemyOrderRepo(session)
        order = Order.create(OrderId.generate(), [OrderLine(sku="X", quantity=2)])
        await repo.save(order)
        found = await repo.find_by_id(order.id)
        assert found is not None
        assert found.id == order.id
```

### Architecture enforcement (`import-linter`)
```ini
# .importlinter
[importlinter]
root_package = src

[importlinter:contract:domain-isolation]
name = Domain must not import application or infrastructure
type = forbidden
source_modules = src.domain
forbidden_modules = src.application, src.infrastructure, fastapi, sqlalchemy, pydantic

[importlinter:contract:application-isolation]
name = Application must not import infrastructure
type = forbidden
source_modules = src.application
forbidden_modules = src.infrastructure, fastapi, sqlalchemy
```

Run in CI: `lint-imports`

---

## 7. Observability Setup

```python
# config/container.py (add to create_app)
import structlog
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from prometheus_client import make_asgi_app

structlog.configure(
    processors=[
        structlog.processors.add_log_level,
        structlog.contextvars.merge_contextvars,
        structlog.processors.JSONRenderer(),
    ]
)

# Mount Prometheus metrics
metrics_app = make_asgi_app()
app.mount("/metrics", metrics_app)

# Health endpoint
@app.get("/health")
async def health() -> dict:
    return {"status": "UP"}
```

- Propagate `traceparent` header via OpenTelemetry middleware: `from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor`.
- Always bind `traceId` and `spanId` to structlog context via middleware.

---

## 8. Python-Specific "Never Do"

- ❌ Import `fastapi`, `sqlalchemy`, `pydantic`, or any broker library inside `domain/` or `application/`.
- ❌ Use mutable default arguments in dataclasses — always use `field(default_factory=...)`.
- ❌ Use `@dataclass` without `frozen=True` for Value Objects.
- ❌ Use `dict` or raw `str` to pass structured data between layers — use typed dataclasses or Pydantic models.
- ❌ Use `print()` anywhere — always use `structlog` or `logging`.
- ❌ Catch bare `except:` or `except Exception:` without re-raising or explicit intent.
- ❌ Skip `mypy --strict` on domain and application packages.
- ❌ Run database migrations automatically at startup in production — use Alembic CLI in a deployment step.
- ❌ Use synchronous blocking I/O (`psycopg2`, `requests`) inside `async def` functions.
- ❌ Expose SQLAlchemy ORM objects outside the persistence package — map to domain or DTO first.

---

*Read [CLAUDE.md](./CLAUDE.md) for language-agnostic rules. This file extends those rules for Python.*
*Last updated: 2026 — engineering commons.*
