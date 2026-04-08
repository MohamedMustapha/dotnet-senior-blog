---
title: "CQRS Explained for .NET Developers"
date: 2026-04-08
draft: false
tags: ["cqrs", "architecture", "dotnet", "mediatr"]
categories: ["Architecture"]
series: ["Code Architecture"]
description: "A pragmatic tour of CQRS for ASP.NET Core: where it comes from, what it actually is, and how to apply it without over-engineering."
---

CQRS is one of those acronyms that gets thrown around in architecture discussions, often confused with event sourcing, separate databases, or "you must use MediatR". Let's clear the fog and see what it really means for an ASP.NET Core codebase.

## 1. Context: why CQRS exists

Back in the late 1980s, Bertrand Meyer formalized the **Command Query Separation (CQS)** principle in *Object-Oriented Software Construction*: a method should either perform an action (command) or return data (query), but never both. Simple, elegant, and mostly ignored in practice.

Fast-forward to 2010. Greg Young took that idea up one level: instead of separating methods on the same object, separate the entire **model**. One model optimized for writes, one model optimized for reads. He called it **Command Query Responsibility Segregation**, CQRS.

Why did this become a thing? Because the classic "fat service" pattern stops scaling:

```csharp
public class OrderService
{
    public async Task<OrderDto> CreateOrder(CreateOrderDto dto) { /* ... */ }
    public async Task<OrderDto> GetOrderById(Guid id) { /* ... */ }
    public async Task<List<OrderSummaryDto>> SearchOrders(OrderFilter f) { /* ... */ }
    public async Task CancelOrder(Guid id) { /* ... */ }
    public async Task<OrderReportDto> GetMonthlyReport(int y, int m) { /* ... */ }
    // ... 20 more methods
}
```

Reads and writes have completely different needs. Writes care about invariants, validation, transactions, domain rules. Reads care about shape, projection, performance, caching. Mixing them in one service turns every change into a merge conflict and every DTO into a compromise.

CQRS fixes this by giving you two lanes.

## 2. Overview: the building blocks

Here are the moving parts of a CQRS-style application:

- **Commands**: intent to change state. `CreateOrderCommand`, `CancelOrderCommand`. They return nothing useful, or just an ID.
- **Queries**: intent to read state. `GetOrderByIdQuery`. They never mutate anything.
- **Handlers**: one handler per command or query. Single responsibility.
- **Dispatcher**: routes a command or query to its handler. Can be MediatR, or a hand-rolled dispatcher, or nothing at all.

{{< mermaid >}}
flowchart LR
    Client([Client]) --> API[ASP.NET Core API]
    API -->|Command| CBus[Command Dispatcher]
    API -->|Query| QBus[Query Dispatcher]
    CBus --> CH[CreateOrderHandler]
    CH --> WriteDB[(Write Model)]
    QBus --> QH[GetOrderByIdHandler]
    QH --> ReadDB[(Read Model)]
    WriteDB -.optional sync.-> ReadDB
{{< /mermaid >}}

> 💡 **Info** : The dashed arrow is optional. CQRS does not require two databases. A single EF Core `DbContext` hitting the same SQL Server is perfectly valid CQRS.

## 3. Technical deep dive

### 3.1 The contracts

Start with tiny marker interfaces. With MediatR 12.x, `IRequest<T>` already does the job, but let's keep our own aliases for clarity:

```csharp
using MediatR;

public interface ICommand : IRequest { }
public interface ICommand<TResponse> : IRequest<TResponse> { }
public interface IQuery<TResponse> : IRequest<TResponse> { }

public interface ICommandHandler<TCommand> : IRequestHandler<TCommand>
    where TCommand : ICommand { }

public interface ICommandHandler<TCommand, TResponse> : IRequestHandler<TCommand, TResponse>
    where TCommand : ICommand<TResponse> { }

public interface IQueryHandler<TQuery, TResponse> : IRequestHandler<TQuery, TResponse>
    where TQuery : IQuery<TResponse> { }
```

> ✅ **Good practice** : These aliases cost nothing and make intent readable at call sites. `IQueryHandler<GetOrderByIdQuery, OrderDto>` reads better than `IRequestHandler<...>`.

### 3.2 A real command: creating an order

```csharp
public sealed record CreateOrderCommand(
    Guid CustomerId,
    IReadOnlyList<OrderLineInput> Lines,
    string ShippingAddress) : ICommand<Guid>;

public sealed record OrderLineInput(Guid ProductId, int Quantity);

public sealed class CreateOrderCommandHandler
    : ICommandHandler<CreateOrderCommand, Guid>
{
    private readonly AppDbContext _db;
    private readonly IPricingService _pricing;
    private readonly ILogger<CreateOrderCommandHandler> _logger;

    public CreateOrderCommandHandler(
        AppDbContext db,
        IPricingService pricing,
        ILogger<CreateOrderCommandHandler> logger)
    {
        _db = db;
        _pricing = pricing;
        _logger = logger;
    }

    public async Task<Guid> Handle(
        CreateOrderCommand request,
        CancellationToken ct)
    {
        var customer = await _db.Customers
            .FirstOrDefaultAsync(c => c.Id == request.CustomerId, ct)
            ?? throw new CustomerNotFoundException(request.CustomerId);

        if (request.Lines.Count == 0)
            throw new DomainException("An order must contain at least one line.");

        var productIds = request.Lines.Select(l => l.ProductId).ToArray();
        var products = await _db.Products
            .Where(p => productIds.Contains(p.Id))
            .ToDictionaryAsync(p => p.Id, ct);

        var order = Order.Create(customer.Id, request.ShippingAddress);
        foreach (var line in request.Lines)
        {
            if (!products.TryGetValue(line.ProductId, out var product))
                throw new ProductNotFoundException(line.ProductId);

            var price = await _pricing.GetUnitPriceAsync(product, customer, ct);
            order.AddLine(product.Id, line.Quantity, price);
        }

        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);

        _logger.LogInformation("Order {OrderId} created for customer {CustomerId}",
            order.Id, customer.Id);

        return order.Id;
    }
}
```

Notice: the handler owns the full use case. Validation, domain rules, persistence, logging. Nothing leaks to a "service layer".

> ⚠️ **Works but...** : You might be tempted to return the full `OrderDto` from the command. Don't. Return just the ID and let the caller issue a query if it needs the full representation. Mixing write and read concerns in one handler is exactly what CQRS tries to avoid.

### 3.3 A real query

Queries should skip the domain model entirely and project straight to a DTO:

```csharp
public sealed record GetOrderByIdQuery(Guid OrderId) : IQuery<OrderDetailsDto?>;

public sealed record OrderDetailsDto(
    Guid Id,
    string CustomerName,
    string Status,
    decimal Total,
    DateTime CreatedAt,
    IReadOnlyList<OrderLineDto> Lines);

public sealed record OrderLineDto(
    string ProductName,
    int Quantity,
    decimal UnitPrice,
    decimal LineTotal);

public sealed class GetOrderByIdQueryHandler
    : IQueryHandler<GetOrderByIdQuery, OrderDetailsDto?>
{
    private readonly AppDbContext _db;

    public GetOrderByIdQueryHandler(AppDbContext db) => _db = db;

    public Task<OrderDetailsDto?> Handle(
        GetOrderByIdQuery request,
        CancellationToken ct)
    {
        return _db.Orders
            .AsNoTracking()
            .Where(o => o.Id == request.OrderId)
            .Select(o => new OrderDetailsDto(
                o.Id,
                o.Customer.FullName,
                o.Status.ToString(),
                o.Lines.Sum(l => l.UnitPrice * l.Quantity),
                o.CreatedAt,
                o.Lines.Select(l => new OrderLineDto(
                    l.Product.Name,
                    l.Quantity,
                    l.UnitPrice,
                    l.UnitPrice * l.Quantity)).ToList()))
            .FirstOrDefaultAsync(ct);
    }
}
```

> ✅ **Good practice** : `AsNoTracking` on the read path is almost always correct. You are projecting to a DTO, there is nothing to track.

> ❌ **Never do** : Don't load the full `Order` aggregate, map it to a DTO in memory, then return it. That wastes a change tracker, pulls columns you don't need, and couples your read model to your write model. Project directly in the query.

### 3.4 Wiring it up in ASP.NET Core

Register MediatR and dispatch from minimal APIs:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<CreateOrderCommandHandler>());

builder.Services.AddScoped<IPricingService, PricingService>();

var app = builder.Build();

app.MapPost("/orders", async (
    CreateOrderCommand command,
    IMediator mediator,
    CancellationToken ct) =>
{
    var id = await mediator.Send(command, ct);
    return Results.Created($"/orders/{id}", new { id });
});

app.MapGet("/orders/{id:guid}", async (
    Guid id,
    IMediator mediator,
    CancellationToken ct) =>
{
    var result = await mediator.Send(new GetOrderByIdQuery(id), ct);
    return result is null ? Results.NotFound() : Results.Ok(result);
});

app.Run();
```

> 💡 **Info** : Available from MediatR 12+: `RegisterServicesFromAssemblyContaining<T>` replaced the older `AddMediatR(typeof(T))` overload.

### 3.5 What CQRS is NOT

This is where most teams get lost:

- **CQRS does not require two databases.** You can do CQRS against a single SQL Server. The separation is in the code, not the infrastructure.
- **CQRS does not require event sourcing.** Event sourcing is a completely separate pattern that pairs well with CQRS, but neither implies the other.
- **CQRS does not require MediatR.** You can dispatch commands and queries through a hand-rolled interface, a source generator, or plain DI. MediatR is just a convenient library.
- **CQRS is not "DDD".** You can apply CQRS to a boring CRUD app and still get value from the read/write split.

> ⚠️ **Works but...** : If your app is a pure CRUD form over data with no business rules, CQRS probably buys you nothing. The overhead of two types per operation is only worth it when reads and writes genuinely diverge.

## 4. Wrap-up

CQRS is a structural discipline: one model for writing, one model for reading, and no handler doing both jobs. Born from Meyer's CQS, generalized by Greg Young, it scales because it stops forcing a single mental model to serve two very different workloads.

You now know where CQRS comes from, the difference between CQS and CQRS, how to model commands and queries with MediatR in ASP.NET Core, and which myths to ignore. You can start refactoring a fat service into focused handlers today, and you can do it without touching your database topology.

Ready to push this into your next feature branch or share it with your team?

## References

- [Microsoft Learn: CQRS pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Microsoft Learn: ASP.NET Core minimal APIs overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/overview)
- [Microsoft Learn: EF Core no-tracking queries](https://learn.microsoft.com/en-us/ef/core/querying/tracking)
- [MediatR GitHub repository](https://github.com/jbogard/MediatR)
