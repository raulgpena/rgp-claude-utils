# CLAUDE.go.md — Go Conventions
## Microservices · Clean & Hexagonal Architecture

> **Companion to [CLAUDE.md](./CLAUDE.md)** — read both files when working on a Go service.
> Rules in `CLAUDE.md` take precedence; this file adds Go-specific detail.

---

## 1. Stack & Versions

| Concern        | Choice |
|----------------|--------|
| Language       | Go 1.22+ |
| HTTP router    | `chi` v5 (lightweight, idiomatic) or `net/http` stdlib for simple services |
| Database       | `pgx/v5` (PostgreSQL); `database/sql` compatible |
| Migrations     | `golang-migrate/migrate` |
| Messaging      | `segmentio/kafka-go` or `confluentinc/confluent-kafka-go` |
| HTTP client    | `net/http` stdlib + optional `resty` |
| DI / Wiring    | Manual constructor injection in `config/wire.go` — no DI framework |
| Testing        | `testing` stdlib + `testify/assert` + `testify/mock` + Testcontainers-go |
| Observability  | OpenTelemetry Go SDK, `log/slog` (stdlib), `prometheus/client_golang` |
| Architecture   | `go vet`, `staticcheck`, `golangci-lint` |

---

## 2. Package Structure

Use Go's `internal/` directory to enforce package visibility at the toolchain level.

```
<service>/
├── internal/
│   ├── domain/
│   │   ├── order.go             # Entity + aggregate + value objects
│   │   ├── order_events.go      # Domain events
│   │   ├── ports.go             # ALL port interfaces (or split: inbound.go / outbound.go)
│   │   └── errors.go            # Sentinel errors and DomainError type
│   │
│   ├── application/
│   │   ├── place_order.go       # One file per use case
│   │   └── dto.go               # Command + Response structs
│   │
│   └── infrastructure/
│       ├── http/
│       │   ├── handler.go       # HTTP handlers (one per aggregate or use-case group)
│       │   ├── router.go        # Route registration
│       │   └── middleware.go    # Auth, logging, tracing middleware
│       ├── messaging/
│       │   ├── consumer.go
│       │   └── producer.go
│       ├── persistence/
│       │   ├── postgres_order_repo.go
│       │   └── mapper.go
│       └── client/
│           └── inventory_client.go
│
├── config/
│   └── wire.go                  # Manual DI wiring; returns initialised app
│
├── cmd/
│   └── main.go                  # Entry point: calls wire.Initialize(), starts server
│
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

> `internal/` prevents other modules from importing these packages. The domain package imports **only the standard library**.

---

## 3. Domain Layer

### Entities & Aggregates
- Plain Go structs with **unexported fields**.
- Expose state through methods — no direct field access from outside the package.
- Enforce invariants in constructors (`New*` functions); return `(*Type, error)`.

```go
// internal/domain/order.go
package domain

import "errors"

type OrderID string
type OrderStatus string

const (
    StatusPending   OrderStatus = "PENDING"
    StatusConfirmed OrderStatus = "CONFIRMED"
)

type Order struct {
    id     OrderID
    status OrderStatus
    lines  []OrderLine
}

// NewOrder is the only way to create a valid Order.
func NewOrder(id OrderID, lines []OrderLine) (*Order, error) {
    if len(lines) == 0 {
        return nil, ErrOrderMustHaveLines
    }
    return &Order{
        id:     id,
        status: StatusPending,
        lines:  append([]OrderLine{}, lines...), // defensive copy
    }, nil
}

func (o *Order) Confirm() error {
    if o.status != StatusPending {
        return ErrOrderNotPending
    }
    o.status = StatusConfirmed
    return nil
}

func (o *Order) ID() OrderID         { return o.id }
func (o *Order) Status() OrderStatus { return o.status }
func (o *Order) Lines() []OrderLine  { return append([]OrderLine{}, o.lines...) } // defensive copy
```

### Value Objects
- Structs with all fields comparable (no slices/maps); implement an `Equal` method.
- Use `type` aliases for primitive IDs to prevent accidental mixing.

```go
// internal/domain/order.go (continued)
type OrderLine struct {
    SKU      string
    Quantity int
}

func (l OrderLine) Equal(other OrderLine) bool {
    return l.SKU == other.SKU && l.Quantity == other.Quantity
}
```

### Domain Errors
```go
// internal/domain/errors.go
package domain

import "errors"

var (
    ErrOrderMustHaveLines = errors.New("order must have at least one line")
    ErrOrderNotPending    = errors.New("order is not in PENDING status")
    ErrOrderNotFound      = errors.New("order not found")
)
```

### Domain Events
```go
// internal/domain/order_events.go
package domain

import "time"

type DomainEvent interface {
    EventName() string
    OccurredAt() time.Time
}

type OrderPlacedEvent struct {
    OrderID    OrderID
    occurredAt time.Time
}

func NewOrderPlacedEvent(id OrderID) OrderPlacedEvent {
    return OrderPlacedEvent{OrderID: id, occurredAt: time.Now().UTC()}
}

func (e OrderPlacedEvent) EventName() string  { return "order.placed" }
func (e OrderPlacedEvent) OccurredAt() time.Time { return e.occurredAt }
```

### Port Interfaces
- Keep interfaces **small** — Go's interface segregation is naturally powerful.
- Define in `domain/ports.go`; only use standard library types in signatures.

```go
// internal/domain/ports.go
package domain

import "context"

// --- Outbound ports ---

type OrderRepository interface {
    Save(ctx context.Context, order *Order) error
    FindByID(ctx context.Context, id OrderID) (*Order, error)
}

type OrderEventPublisher interface {
    Publish(ctx context.Context, event DomainEvent) error
}

// --- Inbound ports ---

type PlaceOrderUseCase interface {
    PlaceOrder(ctx context.Context, cmd PlaceOrderCommand) (OrderID, error)
}
```

---

## 4. Application Layer

- Use case structs receive outbound ports as interface fields — constructor injection.
- No global variables; no `init()` side-effects.
- Return errors using domain sentinel errors or wrapped errors (`fmt.Errorf("place order: %w", err)`).
- Commands and responses are plain structs in `application/dto.go`.

```go
// internal/application/dto.go
package application

type OrderLineInput struct {
    SKU      string
    Quantity int
}

// internal/application/place_order.go
package application

import (
    "context"
    "fmt"

    "github.com/org/service/internal/domain"
)

type PlaceOrderService struct {
    repo      domain.OrderRepository
    publisher domain.OrderEventPublisher
}

func NewPlaceOrderService(repo domain.OrderRepository, pub domain.OrderEventPublisher) *PlaceOrderService {
    return &PlaceOrderService{repo: repo, publisher: pub}
}

func (s *PlaceOrderService) PlaceOrder(ctx context.Context, cmd domain.PlaceOrderCommand) (domain.OrderID, error) {
    lines := make([]domain.OrderLine, len(cmd.Lines))
    for i, l := range cmd.Lines {
        lines[i] = domain.OrderLine{SKU: l.SKU, Quantity: l.Quantity}
    }

    id := domain.OrderID(generateID())
    order, err := domain.NewOrder(id, lines)
    if err != nil {
        return "", fmt.Errorf("place order: %w", err)
    }

    if err := s.repo.Save(ctx, order); err != nil {
        return "", fmt.Errorf("place order: save: %w", err)
    }

    _ = s.publisher.Publish(ctx, domain.NewOrderPlacedEvent(order.ID())) // best-effort
    return order.ID(), nil
}
```

---

## 5. Infrastructure Layer

### Persistence
- `pgx/v5` pool is scoped to the persistence package.
- Map between DB rows and domain objects in `mapper.go` functions — no domain methods touch SQL.
- Use `pgx` named struct scans; never use `SELECT *`.

```go
// internal/infrastructure/persistence/postgres_order_repo.go
package persistence

import (
    "context"
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/org/service/internal/domain"
)

type PostgresOrderRepo struct {
    pool   *pgxpool.Pool
    mapper OrderMapper
}

func NewPostgresOrderRepo(pool *pgxpool.Pool) *PostgresOrderRepo {
    return &PostgresOrderRepo{pool: pool}
}

func (r *PostgresOrderRepo) Save(ctx context.Context, order *domain.Order) error {
    row := r.mapper.ToRow(order)
    _, err := r.pool.Exec(ctx,
        `INSERT INTO orders (id, status) VALUES ($1, $2)
         ON CONFLICT (id) DO UPDATE SET status = EXCLUDED.status`,
        row.ID, row.Status)
    return err
}

func (r *PostgresOrderRepo) FindByID(ctx context.Context, id domain.OrderID) (*domain.Order, error) {
    // ... query + mapper.ToDomain(row)
}
```

### HTTP Handlers
- Handlers receive the **inbound port interface** — never the concrete service struct.
- Decode request → call use case → encode response. Zero business logic in handlers.
- Map domain errors → HTTP status codes only here.

```go
// internal/infrastructure/http/handler.go
package http

import (
    "encoding/json"
    "errors"
    "net/http"

    "github.com/org/service/internal/domain"
)

type OrderHandler struct {
    placeOrder domain.PlaceOrderUseCase
}

func NewOrderHandler(placeOrder domain.PlaceOrderUseCase) *OrderHandler {
    return &OrderHandler{placeOrder: placeOrder}
}

func (h *OrderHandler) PlaceOrder(w http.ResponseWriter, r *http.Request) {
    var req PlaceOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request body", http.StatusBadRequest)
        return
    }

    id, err := h.placeOrder.PlaceOrder(r.Context(), req.ToCommand())
    if err != nil {
        switch {
        case errors.Is(err, domain.ErrOrderMustHaveLines):
            http.Error(w, err.Error(), http.StatusUnprocessableEntity)
        default:
            http.Error(w, "internal error", http.StatusInternalServerError)
        }
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]string{"id": string(id)})
}
```

### DI Wiring
- All wiring lives in `config/wire.go`; `main.go` only calls `wire.Initialize()`.
- Never use a DI framework — manual wiring is explicit and fast to compile.

```go
// config/wire.go
package config

import (
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/org/service/internal/application"
    infrahttp "github.com/org/service/internal/infrastructure/http"
    "github.com/org/service/internal/infrastructure/persistence"
)

type App struct {
    Router http.Handler
}

func Initialize(pool *pgxpool.Pool) *App {
    orderRepo     := persistence.NewPostgresOrderRepo(pool)
    publisher     := messaging.NewKafkaPublisher(/* ... */)
    placeOrderSvc := application.NewPlaceOrderService(orderRepo, publisher)
    orderHandler  := infrahttp.NewOrderHandler(placeOrderSvc)
    router        := infrahttp.NewRouter(orderHandler)
    return &App{Router: router}
}
```

---

## 6. Testing

### Unit tests
- `testing` stdlib + `testify/assert`; zero I/O.
- Use table-driven tests for all domain rule variations.

```go
func TestNewOrder(t *testing.T) {
    tests := []struct {
        name    string
        lines   []domain.OrderLine
        wantErr error
    }{
        {"valid order", []domain.OrderLine{{SKU: "A", Quantity: 1}}, nil},
        {"no lines",    []domain.OrderLine{},                        domain.ErrOrderMustHaveLines},
    }
    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            _, err := domain.NewOrder("id-1", tc.lines)
            assert.ErrorIs(t, err, tc.wantErr)
        })
    }
}
```

### Integration tests
- Testcontainers-go: spin up a real Postgres container.
- Use `t.Cleanup()` to tear down containers; avoid `TestMain` global state.

```go
func TestPostgresOrderRepo_Save(t *testing.T) {
    ctx := context.Background()
    pgContainer, _ := postgres.RunContainer(ctx, testcontainers.WithImage("postgres:16"))
    t.Cleanup(func() { pgContainer.Terminate(ctx) })

    pool := mustConnectPool(ctx, pgContainer.ConnectionString(ctx))
    repo := persistence.NewPostgresOrderRepo(pool)
    // ... test assertions
}
```

### CI checks
```bash
go vet ./...
staticcheck ./...
golangci-lint run
go test -race ./...          # always run with -race
go test -coverprofile=coverage.out ./internal/domain/... ./internal/application/...
go tool cover -func=coverage.out  # assert ≥ 80%
```

---

## 7. Observability Setup

```go
// Structured logging with slog
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
slog.SetDefault(logger)

// Usage in handlers/services — always pass ctx for trace propagation
slog.InfoContext(ctx, "order placed", "orderId", string(id))

// OpenTelemetry — initialise in main.go
tp := sdktrace.NewTracerProvider(sdktrace.WithBatcher(otlpExporter))
otel.SetTracerProvider(tp)
otel.SetTextMapPropagator(propagation.TraceContext{})
```

- Expose `/metrics` via `promhttp.Handler()` on a separate admin port.
- Expose `/health` returning `{"status":"UP"}` on the main port.

---

## 8. Go-Specific "Never Do"

- ❌ Import a non-stdlib package inside `internal/domain/` — domain must be stdlib-only.
- ❌ Use global variables or `init()` for dependency setup.
- ❌ Use `panic` in domain or application code — return errors.
- ❌ Define interfaces in the package that **implements** them — define in the **consumer** package (domain).
- ❌ Return concrete struct types from repository methods — return domain types.
- ❌ Use `interface{}` or `any` in domain or application code — use typed parameters.
- ❌ Skip the `-race` flag in tests; always run `go test -race`.
- ❌ Use named return variables in public API functions — they obscure intent.
- ❌ Ignore errors with `_`; handle or explicitly document why they are safe to ignore.

---

*Read [CLAUDE.md](./CLAUDE.md) for language-agnostic rules. This file extends those rules for Go.*
*Last updated: 2026 — engineering commons.*
