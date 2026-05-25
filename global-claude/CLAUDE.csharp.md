# CLAUDE.csharp.md — C# / .NET Conventions
## Microservices · Clean & Hexagonal Architecture

> **Companion to [CLAUDE.md](./CLAUDE.md)** — read both files when working on a C# service.
> Rules in `CLAUDE.md` take precedence; this file adds C#-specific detail.

---

## 1. Stack & Versions

| Concern        | Choice |
|----------------|--------|
| Language       | C# 12+ / .NET 8+ |
| Framework      | ASP.NET Core (Minimal APIs preferred for new services) |
| CQRS           | MediatR 12+ |
| Validation     | FluentValidation |
| Persistence    | Entity Framework Core 8 (infrastructure only) |
| Migrations     | EF Core Migrations via `dotnet ef` |
| Messaging      | MassTransit (Kafka or RabbitMQ transport) |
| HTTP client    | `IHttpClientFactory` + Refit |
| Testing        | xUnit, NSubstitute, FluentAssertions, Testcontainers.NET, `WebApplicationFactory` |
| Observability  | OpenTelemetry .NET SDK, Serilog (JSON sink), `prometheus-net` |
| Architecture tests | ArchUnitNET |

---

## 2. Project / Namespace Structure

Use a **multi-project solution** to enforce layer boundaries at the compiler level:

```
<Service>.sln
├── src/
│   ├── <Service>.Domain/            # No NuGet dependencies except language primitives
│   │   ├── Model/
│   │   ├── Events/
│   │   ├── Ports/
│   │   │   ├── Inbound/             # Use case interfaces
│   │   │   └── Outbound/            # Repository + publisher interfaces
│   │   └── Exceptions/
│   │
│   ├── <Service>.Application/       # Depends only on Domain project
│   │   ├── UseCases/
│   │   ├── Commands/
│   │   ├── Queries/
│   │   └── DTOs/
│   │
│   ├── <Service>.Infrastructure/    # Depends on Application + Domain
│   │   ├── Inbound/
│   │   │   ├── Http/                # Controllers or Minimal API endpoint classes
│   │   │   └── Messaging/           # MassTransit consumers
│   │   └── Outbound/
│   │       ├── Persistence/         # DbContext, EF entities, repositories, mappers
│   │       ├── Messaging/           # MassTransit producers
│   │       └── Client/              # Refit clients
│   │
│   └── <Service>.Configuration/     # Entry point; DI wiring; depends on all layers
│       ├── DependencyInjection.cs   # IServiceCollection extension methods
│       └── Program.cs
│
└── tests/
    ├── <Service>.UnitTests/
    ├── <Service>.IntegrationTests/
    └── <Service>.E2ETests/
```

> The project reference graph enforces the dependency rule: `Infrastructure` → `Application` → `Domain`. `Domain` has **no project references**.

---

## 3. Domain Layer

### Entities
- Sealed classes with **private setters** and **private constructors**.
- Expose state mutation only through domain methods; no public setters.
- No EF Core, no ASP.NET, no MediatR references.

```csharp
// Domain/Model/Order.cs
public sealed class Order
{
    public OrderId Id { get; }
    public OrderStatus Status { get; private set; }

    private readonly List<OrderLine> _lines;
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    private Order(OrderId id, IEnumerable<OrderLine> lines)
    {
        Id = id;
        _lines = lines.ToList();
        Status = OrderStatus.Pending;
    }

    public static Order Create(OrderId id, IEnumerable<OrderLine> lines)
    {
        var lineList = lines.ToList();
        if (!lineList.Any())
            throw new DomainException("Order must have at least one line.");
        return new Order(id, lineList);
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Only pending orders can be confirmed.");
        Status = OrderStatus.Confirmed;
    }
}
```

### Value Objects
- Use C# `record` — structural equality, immutability, and `with` expressions for free.

```csharp
// Domain/Model/OrderId.cs
public record OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId From(string value) =>
        Guid.TryParse(value, out var g) ? new(g)
            : throw new DomainException($"'{value}' is not a valid OrderId.");
}
```

### Domain Events
```csharp
// Domain/Events/IDomainEvent.cs
public interface IDomainEvent { DateTimeOffset OccurredAt { get; } }

// Domain/Events/OrderPlaced.cs
public record OrderPlaced(OrderId OrderId, DateTimeOffset OccurredAt) : IDomainEvent
{
    public OrderPlaced(OrderId orderId) : this(orderId, DateTimeOffset.UtcNow) { }
}
```

### Port Interfaces
```csharp
// Domain/Ports/Outbound/IOrderRepository.cs
public interface IOrderRepository
{
    Task SaveAsync(Order order, CancellationToken ct = default);
    Task<Order?> FindByIdAsync(OrderId id, CancellationToken ct = default);
}

// Domain/Ports/Inbound/IPlaceOrderUseCase.cs
public interface IPlaceOrderUseCase
{
    Task<OrderId> PlaceOrderAsync(PlaceOrderCommand command, CancellationToken ct = default);
}
```

---

## 4. Application Layer

- Use **MediatR** `IRequestHandler<TCommand, TResult>` for use cases.
- Commands and Queries are `record` types — immutable.
- FluentValidation `AbstractValidator<T>` lives next to the command it validates.
- No EF Core, no ASP.NET Core, no MassTransit references.

```csharp
// Application/Commands/PlaceOrderCommand.cs
public record PlaceOrderCommand(IReadOnlyList<OrderLineDto> Lines) : IRequest<OrderId>;

// Application/Commands/PlaceOrderCommandValidator.cs
public sealed class PlaceOrderCommandValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderCommandValidator()
    {
        RuleFor(x => x.Lines).NotEmpty().WithMessage("Order must have at least one line.");
    }
}

// Application/UseCases/PlaceOrderHandler.cs
public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, OrderId>
{
    private readonly IOrderRepository _repository;
    private readonly IOrderEventPublisher _publisher;

    public PlaceOrderHandler(IOrderRepository repository, IOrderEventPublisher publisher)
        => (_repository, _publisher) = (repository, publisher);

    public async Task<OrderId> Handle(PlaceOrderCommand command, CancellationToken ct)
    {
        var lines = command.Lines.Select(l => l.ToDomain()).ToList();
        var order = Order.Create(OrderId.New(), lines);
        await _repository.SaveAsync(order, ct);
        await _publisher.PublishAsync(new OrderPlaced(order.Id), ct);
        return order.Id;
    }
}
```

---

## 5. Infrastructure Layer

### Persistence
- EF Core `DbContext` and entity type configurations live **only** in `Infrastructure/Outbound/Persistence/`.
- Map EF entities ↔ domain objects in dedicated `*Mapper` classes — no AutoMapper in domain.
- Use `IEntityTypeConfiguration<T>` for fluent EF configuration (no data annotations on EF entities).

```csharp
// Infrastructure/Outbound/Persistence/OrderEntityConfiguration.cs
public class OrderEntityConfiguration : IEntityTypeConfiguration<OrderDataModel>
{
    public void Configure(EntityTypeBuilder<OrderDataModel> builder)
    {
        builder.ToTable("orders");
        builder.HasKey(o => o.Id);
        builder.HasMany(o => o.Lines).WithOne().HasForeignKey("OrderId");
    }
}

// Infrastructure/Outbound/Persistence/EfOrderRepository.cs
public sealed class EfOrderRepository : IOrderRepository
{
    private readonly OrderDbContext _ctx;
    private readonly OrderMapper _mapper;

    public EfOrderRepository(OrderDbContext ctx, OrderMapper mapper)
        => (_ctx, _mapper) = (ctx, mapper);

    public async Task SaveAsync(Order order, CancellationToken ct)
    {
        var entity = _mapper.ToDataModel(order);
        _ctx.Orders.Update(entity);
        await _ctx.SaveChangesAsync(ct);
    }

    public async Task<Order?> FindByIdAsync(OrderId id, CancellationToken ct)
    {
        var entity = await _ctx.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id.Value, ct);
        return entity is null ? null : _mapper.ToDomain(entity);
    }
}
```

### HTTP / Minimal API
- Prefer **Minimal API endpoint classes** (`.WithTags`, `.WithName`, `MapGroup`) over full controllers.
- Inject `IMediator` — never inject use case implementations directly.

```csharp
// Infrastructure/Inbound/Http/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static RouteGroupBuilder MapOrderEndpoints(this RouteGroupBuilder group)
    {
        group.MapPost("/", PlaceOrder).WithName("PlaceOrder");
        return group;
    }

    private static async Task<IResult> PlaceOrder(
        PlaceOrderRequest request,
        IMediator mediator,
        CancellationToken ct)
    {
        var command = request.ToCommand();
        var orderId = await mediator.Send(command, ct);
        return Results.Created($"/api/v1/orders/{orderId.Value}", new { id = orderId.Value });
    }
}
```

### DI Wiring
- Use `IServiceCollection` extension methods in `Configuration/DependencyInjection.cs`.
- Register MediatR, FluentValidation, EF, and adapters here.

```csharp
// Configuration/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(PlaceOrderHandler).Assembly));
        services.AddValidatorsFromAssembly(typeof(PlaceOrderCommandValidator).Assembly);
        services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
        return services;
    }

    public static IServiceCollection AddInfrastructure(this IServiceCollection services,
                                                        IConfiguration config)
    {
        services.AddDbContext<OrderDbContext>(opt =>
            opt.UseNpgsql(config.GetConnectionString("OrdersDb")));
        services.AddScoped<IOrderRepository, EfOrderRepository>();
        services.AddScoped<IOrderEventPublisher, MassTransitOrderPublisher>();
        return services;
    }
}
```

---

## 6. Testing

### Unit tests
- xUnit + NSubstitute + FluentAssertions — zero DI container, zero I/O.

```csharp
public class PlaceOrderHandlerTests
{
    private readonly IOrderRepository _repository = Substitute.For<IOrderRepository>();
    private readonly IOrderEventPublisher _publisher = Substitute.For<IOrderEventPublisher>();
    private readonly PlaceOrderHandler _sut;

    public PlaceOrderHandlerTests() => _sut = new(_repository, _publisher);

    [Fact]
    public async Task ShouldPublishEventWhenOrderIsPlaced()
    {
        var command = new PlaceOrderCommand([new OrderLineDto("SKU-1", 2)]);
        await _sut.Handle(command, CancellationToken.None);
        await _publisher.Received(1).PublishAsync(Arg.Any<OrderPlaced>(), Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task ShouldThrowWhenNoLinesProvided()
    {
        var command = new PlaceOrderCommand([]);
        var act = () => _sut.Handle(command, CancellationToken.None);
        await act.Should().ThrowAsync<DomainException>();
    }
}
```

### Integration tests
- Testcontainers.NET for Postgres.
- `WebApplicationFactory<Program>` for full HTTP slice tests.

### Architecture tests (ArchUnitNET)
```csharp
[Fact]
public void DomainShouldNotDependOnInfrastructure()
{
    var architecture = new ArchLoader()
        .LoadAssemblies(typeof(Order).Assembly, typeof(EfOrderRepository).Assembly)
        .Build();

    IArchRule rule = Classes().That().ResideInNamespace(".*Domain.*")
        .Should().NotDependOnAny(Classes().That().ResideInNamespace(".*Infrastructure.*"));

    rule.Check(architecture);
}
```

---

## 7. Observability Setup

```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.AddAspNetCoreInstrumentation().AddOtlpExporter())
    .WithMetrics(m => m.AddAspNetCoreInstrumentation().AddPrometheusExporter());

builder.Host.UseSerilog((ctx, cfg) =>
    cfg.ReadFrom.Configuration(ctx.Configuration)
       .Enrich.FromLogContext()
       .WriteTo.Console(new JsonFormatter()));
```

- Expose Prometheus metrics at `/metrics` via `app.MapPrometheusScrapingEndpoint()`.
- Include `traceId` and `spanId` in Serilog via `Serilog.Enrichers.OpenTelemetry`.

---

## 8. C#-Specific "Never Do"

- ❌ Add EF Core attributes (`[Column]`, `[Key]`, `[ForeignKey]`) to domain model classes.
- ❌ Use `public` setters on domain entities — use methods that encode intent.
- ❌ Inject `DbContext` directly into application handlers — go through the repository interface.
- ❌ Use `AutoMapper` across layer boundaries involving domain objects.
- ❌ Use `async void` — always `async Task`.
- ❌ Use `IQueryable<T>` outside the persistence layer — it leaks EF semantics.
- ❌ Throw generic `Exception` — create typed exceptions in `Domain/Exceptions/`.
- ❌ Use nullable reference types without proper null guards (enable `<Nullable>enable</Nullable>` in all projects).

---

*Read [CLAUDE.md](./CLAUDE.md) for language-agnostic rules. This file extends those rules for C# / .NET.*
*Last updated: 2026 — engineering commons.*
