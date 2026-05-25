
# CLAUDE.programming.md — Microservices Commons
## Clean Architecture & Hexagonal Architecture

---

## 1. Core Principles (Non-Negotiable)

### 1.1 Dependency Rule
- Dependencies ALWAYS point **inward**: Infrastructure → Application → Domain.
- The **Domain** layer has **zero external dependencies** — no frameworks, no ORMs, no HTTP libs.
- The **Application** layer depends only on Domain. It defines **Ports** (interfaces).
- The **Infrastructure** layer implements Ports and wires everything via Dependency Injection.

### 1.2 Hexagonal Ports & Adapters
- **Primary (Driving) Ports** — interfaces that expose use cases to the outside world (HTTP, gRPC, CLI, message consumers).
- **Secondary (Driven) Ports** — interfaces the application calls outward (repositories, messaging, external APIs).
- **Adapters** — concrete implementations of ports; they live exclusively in the Infrastructure layer.
- The Application Core (Domain + Application) must be fully testable **without spinning up any adapter**.

### 1.3 Ubiquitous Language
- Use domain vocabulary in all names. No generic names: `Manager`, `Helper`, `Utils`, `Data`, `Info`.
- Each microservice owns its **bounded context**; never share domain model classes across services.
- Communicate across service boundaries through **DTOs / events / contracts**, never domain objects.

### 1.4 Single Responsibility & Cohesion
- One use case = one application service / command handler class.
- One repository interface per aggregate root.
- Keep files small and purposeful; prefer many small focused files over few large ones.

---

## 2. Universal Folder Structure

Every microservice, regardless of language, maps to this conceptual layout:

```
<service-name>/
├── domain/                   # Enterprise business rules — NO external deps
│   ├── model/                # Entities, Value Objects, Aggregates
│   ├── event/                # Domain Events
│   ├── port/
│   │   ├── inbound/          # Primary port interfaces (use case contracts)
│   │   └── outbound/         # Secondary port interfaces (repo, messaging, etc.)
│   └── exception/            # Domain-specific exceptions / errors
│
├── application/              # Application business rules — orchestrates domain
│   ├── usecase/              # One class/file per use case
│   ├── dto/                  # Commands, Queries, Responses (no domain leakage)
│   └── service/              # Application Services implementing inbound ports
│
├── infrastructure/           # Adapters — frameworks, DB, HTTP, messaging
│   ├── inbound/
│   │   ├── http/             # REST / gRPC controllers / handlers
│   │   └── messaging/        # Message consumers (Kafka, RabbitMQ, etc.)
│   └── outbound/
│       ├── persistence/      # Repository implementations, ORM models, mappers
│       ├── messaging/        # Message producers
│       └── client/           # HTTP/gRPC clients to other services
│
├── config/                   # DI wiring, app configuration, framework bootstrap
└── tests/
    ├── unit/                 # Domain + Application tests — zero infrastructure
    ├── integration/          # Adapter tests with real DB/broker (Testcontainers)
    └── e2e/                  # Full-slice tests through the HTTP/gRPC entry point
```

> **Rule**: If you cannot place a file in one of these buckets without hesitation, the abstraction is wrong. Revisit the design.

See the language companion file for the exact directory/package names in each ecosystem.

---

## 3. Cross-Cutting Concerns

### 3.1 Error Handling
- Define domain errors close to the domain model; never in infrastructure.
- Map domain exceptions → HTTP status codes **only** inside the inbound HTTP adapter.
- Never swallow errors silently; always propagate or log with full context.
- Use structured logging with a correlation/trace ID on every log entry.

### 3.2 Observability
Every service must expose:

| Signal   | Contract |
|----------|----------|
| Health   | `GET /health` → `{"status":"UP"}` (liveness + readiness) |
| Metrics  | Prometheus-compatible `GET /metrics` |
| Tracing  | OpenTelemetry SDK; propagate `traceparent` header |
| Logging  | Structured JSON: `service`, `traceId`, `spanId`, `level`, `message` |

Specific library choices are in each language companion file.

### 3.3 Configuration
- All configuration via **environment variables** (12-Factor App — no config files in the image).
- Never hard-code URLs, credentials, ports, or timeouts anywhere.
- Use a typed config/settings object; fail fast at startup if required variables are missing.

### 3.4 API Design
- REST: follow **RFC 7807** for error responses (`application/problem+json`).
- Use versioned routes: `/api/v1/...`.
- gRPC: `.proto` files live in `proto/` at repo root; generate stubs — never edit generated code.
- Async messaging: publish domain events as **CloudEvents v1.0**.

### 3.5 Security
- Validate all input at the inbound adapter boundary — never trust raw HTTP/message payloads inside domain or application code.
- OAuth 2.0 / JWT validation belongs in middleware/filters, not in use cases.
- Never log PII, tokens, or passwords.
- Run dependency vulnerability scanning (`trivy` / OWASP Dependency-Check) in CI.

### 3.6 Data Persistence
- Each microservice owns its own database schema or catalog. **No shared DB tables** across services.
- Use optimistic locking for concurrent aggregate updates.
- Use the **Outbox Pattern** for reliable event publishing (write event + state in the same transaction).
- Repository interfaces return **domain objects only**; infrastructure mappers handle the translation.

---

## 4. Testing Strategy

```
         /\
        /  \      E2E        — few, happy path + critical error paths
       /----\
      /      \    Integration — some, one per adapter (Testcontainers)
     /--------\
    /          \ Unit        — many, all domain rules + all use cases
   /____________\
```

| Layer       | Scope                                      | Key rule                              |
|-------------|-------------------------------------------|---------------------------------------|
| Unit        | Domain model + Application use cases       | Zero I/O, zero framework context      |
| Integration | Each outbound/inbound adapter individually | Use Testcontainers, never mock the DB |
| E2E         | Full HTTP slice                            | Test behaviour, not implementation    |

**Mandatory CI gates:**
- All tests green.
- Coverage ≥ 80% on `domain/` + `application/` packages.
- Architecture fitness functions enforce the dependency rule automatically.
- Static analysis + linting with zero warnings.

Specific tooling per language is in the companion files.

---

## 5. Code Generation Checklist

When Claude Code generates a new microservice or feature, **always follow this order**:

1. **Domain first** — model the aggregate, value objects, domain events, and port interfaces. No framework code yet.
2. **Domain unit tests** — write tests, make them pass before moving on.
3. **Use case** — application service implementing the inbound port, injecting outbound ports.
4. **Application unit tests** — mock the outbound ports, test orchestration logic.
5. **Infrastructure adapters** — persistence, HTTP handler, message consumer/producer. Map to/from domain objects.
6. **Integration tests** — use Testcontainers, one test class per adapter.
7. **DI wiring** — in `config/` only; never inside domain or application layers.
8. **Observability** — health endpoint, metrics registration, structured logging, OTel tracing.
9. **E2E test** — at least the happy path through the HTTP entry point.

> **Hard stop**: If you are about to import an infrastructure library inside `domain/` or `application/`, stop. Define a port interface instead and inject it.

---

## 6. Naming Conventions

| Concept              | Java                | C#                   | Go                          | Python                      |
|----------------------|---------------------|----------------------|-----------------------------|-----------------------------|
| Entity               | `Order`             | `Order`              | `Order` (struct)            | `Order` (dataclass)         |
| Value Object         | `OrderId` (record)  | `OrderId` (record)   | `OrderID` (struct)          | `OrderId` (frozen dataclass)|
| Domain Event         | `OrderPlaced`       | `OrderPlaced`        | `OrderPlacedEvent`          | `OrderPlaced`               |
| Inbound Port         | `PlaceOrderUseCase` | `IPlaceOrderUseCase` | `PlaceOrderUseCase` (iface) | `PlaceOrderUseCase` (Protocol) |
| Outbound Port        | `OrderRepository`   | `IOrderRepository`   | `OrderRepository` (iface)   | `OrderRepository` (Protocol)|
| Application Service  | `PlaceOrderService` | `PlaceOrderHandler`  | `PlaceOrderService`         | `PlaceOrderUseCase`         |
| HTTP Adapter         | `OrderController`   | `OrdersController`   | `OrderHandler`              | `order_router`              |
| Persistence Adapter  | `JpaOrderRepo`      | `EfOrderRepository`  | `PostgresOrderRepo`         | `SqlAlchemyOrderRepo`       |
| Mapper               | `OrderMapper`       | `OrderMapper`        | `toOrder()` / `fromOrder()` | `OrderMapper`               |
| Command / DTO        | `PlaceOrderCommand` | `PlaceOrderCommand`  | `PlaceOrderCommand`         | `PlaceOrderCommand`         |

---

## 7. What Claude Must NEVER Do

- ❌ Import a framework (Spring, ASP.NET, FastAPI, `net/http`) inside `domain/` or `application/`.
- ❌ Put business logic inside a controller, handler, router, or consumer.
- ❌ Return ORM / persistence entities from a repository interface.
- ❌ Share a domain model class between two different microservices.
- ❌ Use static or global mutable state in domain or application code.
- ❌ Hard-code any configuration value (URL, timeout, credential, port).
- ❌ Skip unit tests for new domain or application code.
- ❌ Access another microservice's database directly.
- ❌ Use `SELECT *` in any query.
- ❌ Catch and silently swallow exceptions or errors.

---

*This file is language-agnostic. Always read the companion file for the language in use.*
*Last updated: 2026 — engineering commons.*:w

---

## 8. Git & Workflow
- Branch naming: `feat/`, `fix/`, `chore/`, `refactor/` + short description.
- Commits follow **Conventional Commits**: `feat(orders): add cancellation use case`.
- PRs must include: description, test evidence, migration notes if applicable.
- No commented-out code committed.


---

## 9. Database
- Write **Liquibase** migrations for all schema changes.