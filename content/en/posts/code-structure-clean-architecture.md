---
title: "Clean Architecture in .NET: Dependencies Pointing the Right Way"
date: 2026-04-08
draft: false
tags: ["architecture", "dotnet", "clean-architecture"]
categories: ["Architecture"]
series: ["Code Structure"]
description: "Clean Architecture is not about four projects and a circle diagram. It is about one rule: dependencies point inward. Here is how to build it in .NET without the dogma."
---

Clean Architecture has a branding problem. Many .NET codebases that advertise it on the README are closer to N-Layered with renamed folders, and others grow enough interfaces, mappers, and DTO hops that a simple product listing becomes a multi-file expedition. Both outcomes are understandable: the pattern is often introduced without its original framing, and teams fill the gap with ceremony. The actual idea, the one Robert Martin formalized in 2012 (building on earlier work like Alistair Cockburn's Hexagonal Architecture from 2005 and Jeffrey Palermo's Onion Architecture from 2008), is much smaller and much more useful: **your business rules should not depend on your framework, your database, or your HTTP stack**. Everything else is implementation detail.

If you have read the previous articles in this series, you already know the two patterns that came before: [N-Layered Architecture](/en/posts/code-structure-n-layered/) with its physically separated projects, and [UI / Repositories / Services](/en/posts/code-structure-ui-repos-services/) with its pragmatic single-project layering. Clean Architecture is what you reach for when those patterns start to leak and you need the compiler to hold the line between your domain and the outside world.

## Why this pattern exists

Picture a mature .NET codebase. The team has been shipping for three years. EF Core queries live inside service classes. Controllers build domain objects by hand from request bodies. A `UserService` references `HttpContext` to read the current user. A domain rule about discount eligibility lives half in a stored procedure, half in an `if` inside a controller, and half in a JavaScript file on the frontend. When a new payment provider shows up, nobody can change the `OrderService` without breaking two features, because the business logic is tangled with Stripe-specific calls.

This is the pain Clean Architecture fixes. Its contract:

1. **Your domain model knows nothing** about EF Core, ASP.NET, MediatR, or Stripe.
2. **Your application layer** orchestrates use cases using only abstractions.
3. **Infrastructure plugs in from the outside** and can be swapped without touching the domain.

The payoff is not theoretical. It is the ability to upgrade EF Core, change message brokers, or replace a payment provider without opening your domain project. It is also the ability to write fast unit tests against your business rules without spinning up a database.

## Overview: the layers and the rule

Before the code, here are the bricks of Clean Architecture as we will use them in .NET:

{{< mermaid >}}
graph TD
    A[Api / Presentation<br/>Controllers, Minimal APIs, SignalR] --> B[Application<br/>Use cases, commands, queries, ports]
    B --> C[Domain<br/>Entities, value objects, domain services, invariants]
    D[Infrastructure<br/>EF Core, HTTP clients, file system, message bus] --> B
    D --> C
    A --> D
{{< /mermaid >}}

The arrows are the only thing that matters. **Everything points toward Domain.** Domain depends on nothing. Application depends only on Domain. Infrastructure implements interfaces declared in Application (or Domain). The Api project wires everything up at startup. If you get the arrows right, you have Clean Architecture. If you do not, you have four projects that share the cost of the split without sharing the benefit.

> 💡 **Info** : The original diagram has four concentric circles (Entities, Use Cases, Interface Adapters, Frameworks). In practice, most .NET teams collapse this to four csproj files: `Domain`, `Application`, `Infrastructure`, `Api`. That mapping is good enough and I will use it throughout this article.

## Zoom: Domain, the heart

The Domain project holds your business concepts and their invariants. It references nothing. Not EF Core, not MediatR, not `Microsoft.Extensions.*`. Just `netstandard2.1` or `net10.0` and your own types.

```csharp
// Domain/Orders/Order.cs
namespace Shop.Domain.Orders;

public sealed class Order
{
    private readonly List<OrderLine> _lines = new();

    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();
    public Money Total => new(_lines.Sum(l => l.Subtotal.Amount), Currency.Eur);

    private Order(OrderId id, CustomerId customerId)
    {
        Id = id;
        CustomerId = customerId;
        Status = OrderStatus.Draft;
    }

    public static Order Create(CustomerId customerId)
        => new(OrderId.New(), customerId);

    public void AddLine(ProductId productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Cannot modify a submitted order.");
        if (quantity <= 0)
            throw new DomainException("Quantity must be positive.");

        _lines.Add(new OrderLine(productId, quantity, unitPrice));
    }

    public void Submit()
    {
        if (_lines.Count == 0)
            throw new DomainException("An empty order cannot be submitted.");
        Status = OrderStatus.Submitted;
    }
}
```

Notice what is **not** there: no `[Table]` attribute, no `DbContext`, no `virtual` keyword for lazy loading, no navigation property that assumes EF Core. The entity enforces its own invariants. Breaking a rule throws a domain exception, not an HTTP 400.

> ✅ **Good practice** : Make constructors private or internal and expose factory methods (`Order.Create(...)`). This forces all callers through your invariant checks. There is no way to get a broken `Order` from the outside.

> ❌ **Never do this** : Do not put `[Column]` or `[Required]` attributes on domain entities to "save time". The moment you do, your Domain project gains a hard dependency on an ORM, and the invariant that Domain references nothing stops holding. The fluent API inside Infrastructure gives you the same mapping without leaking the framework into your entities.

## Zoom: Application, the use cases

The Application layer describes what the system *does*, expressed as use cases. This is where commands and queries live, where transactions are coordinated, and where you declare the *ports* (interfaces) that Infrastructure will plug into.

```csharp
// Application/Orders/SubmitOrder/SubmitOrderCommand.cs
public sealed record SubmitOrderCommand(Guid OrderId) : IRequest<Result>;

// Application/Orders/SubmitOrder/SubmitOrderHandler.cs
public sealed class SubmitOrderHandler : IRequestHandler<SubmitOrderCommand, Result>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;
    private readonly IPaymentGateway _payments;

    public SubmitOrderHandler(
        IOrderRepository orders,
        IUnitOfWork uow,
        IPaymentGateway payments)
    {
        _orders = orders;
        _uow = uow;
        _payments = payments;
    }

    public async Task<Result> Handle(SubmitOrderCommand cmd, CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(new OrderId(cmd.OrderId), ct);
        if (order is null)
            return Result.NotFound($"Order {cmd.OrderId} not found.");

        order.Submit();

        var charge = await _payments.ChargeAsync(order.CustomerId, order.Total, ct);
        if (!charge.Success)
            return Result.Failure(charge.Error);

        await _uow.SaveChangesAsync(ct);
        return Result.Success();
    }
}
```

The interfaces `IOrderRepository`, `IUnitOfWork`, and `IPaymentGateway` live in the Application project, right next to the use case that needs them. They describe *what* the application needs, not *how* it is implemented.

```csharp
// Application/Abstractions/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}
```

> 💡 **Info** : This is the **Dependency Inversion Principle** made physical. The high-level policy (Application) owns the abstraction. The low-level detail (Infrastructure) implements it. The arrow points from Infrastructure to Application, not the other way around.

> ⚠️ **It works, but...** : You will see teams put all their interfaces in a separate `Application.Contracts` project "for reuse". Ninety percent of the time that project is imported only by Infrastructure and adds zero value. Keep interfaces with their use cases until you have a real second consumer.

## Zoom: Infrastructure, the plugs

Infrastructure is where EF Core, HTTP clients, file writers, and message bus code finally appear. It references Application (to implement the ports) and Domain (to map to entities). Nothing in Application or Domain references Infrastructure.

```csharp
// Infrastructure/Persistence/OrderRepository.cs
internal sealed class OrderRepository : IOrderRepository
{
    private readonly ShopDbContext _db;

    public OrderRepository(ShopDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct)
        => _db.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct)
        => await _db.Orders.AddAsync(order, ct);
}
```

The EF Core configuration that knows about tables and columns also lives here, not on the entity:

```csharp
// Infrastructure/Persistence/Configurations/OrderConfiguration.cs
internal sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.ToTable("Orders");
        b.HasKey(o => o.Id);
        b.Property(o => o.Id)
            .HasConversion(id => id.Value, value => new OrderId(value));
        b.Property(o => o.Status).HasConversion<string>();
        b.OwnsMany(o => o.Lines, lines =>
        {
            lines.ToTable("OrderLines");
            lines.WithOwner().HasForeignKey("OrderId");
        });
    }
}
```

> ✅ **Good practice** : Mark your Infrastructure implementations `internal`. The only way the outside world should get an `IOrderRepository` is through DI. If a controller can `new OrderRepository(...)`, something is wrong.

## Zoom: Api, the composition root

The Api project is where everything gets wired. It references Application and Infrastructure, registers the services, and exposes endpoints. It stays thin: parse, delegate, map the result.

```csharp
// Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplication();      // MediatR, validators, pipeline behaviors
builder.Services.AddInfrastructure(     // DbContext, repositories, HTTP clients
    builder.Configuration);

var app = builder.Build();
app.MapOrderEndpoints();
app.Run();

// Api/Endpoints/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/orders");

        g.MapPost("/{id:guid}/submit", async (
            Guid id,
            ISender mediator,
            CancellationToken ct) =>
        {
            var result = await mediator.Send(new SubmitOrderCommand(id), ct);
            return result.IsSuccess
                ? Results.NoContent()
                : result.ToProblemDetails();
        });
    }
}
```

No business logic here. The endpoint parses the route, dispatches the command, and translates the result. If you need a different transport tomorrow, a gRPC service or a background worker, you write a new composition root and reuse Application and Domain untouched.

> 💡 **Info** : `AddApplication` and `AddInfrastructure` are extension methods that live in their respective projects. That keeps each layer in charge of its own registrations, and the Api project does not need to know what a `DbContext` is.

## The rule the compiler must enforce

The point of splitting into four projects is that the compiler checks the arrows for you. Your csproj graph should look like this:

```
Api           -> Application, Infrastructure
Infrastructure-> Application, Domain
Application   -> Domain
Domain        -> (nothing)
```

If Domain gains a reference to anything, you have a bug in the pattern. A good practice is to add an **architecture test** so the rule becomes executable:

```csharp
[Fact]
public void Domain_should_not_depend_on_any_other_project()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .ShouldNot()
        .HaveDependencyOnAny("Shop.Application", "Shop.Infrastructure", "Shop.Api")
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}
```

> ✅ **Good practice** : Add one architecture test per layer. It takes five minutes with NetArchTest or ArchUnitNET and it catches the accidental `using Shop.Infrastructure;` before it quietly becomes load-bearing.

## When Clean Architecture is the wrong call

Clean Architecture is overhead. Four projects, a dependency injection dance, handlers, ports, mappers. For a thirty-endpoint CRUD app where the "business rules" are "save this and return it", the structure costs more than it buys. In those cases, [UI / Repositories / Services](/en/posts/code-structure-ui-repos-services/) is honestly better and shipping it will make you faster.

Reach for Clean Architecture when at least two of the following are true:

- You have real business rules, not just CRUD. Invariants, state machines, calculations, cross-aggregate consistency.
- The app has to survive multiple frameworks, storage engines, or transports over its lifetime.
- Multiple teams or developers need enforced boundaries.
- You are going to write a non-trivial amount of unit tests against the domain and you do not want to drag a database into them.

If none of these apply, you are paying the tax without getting the benefit.

## Wrap-up

You now understand what Clean Architecture really is: one rule about dependency direction, enforced by project references and occasionally by architecture tests. You can set up the four projects, put your invariants in the Domain, keep your use cases in Application, plug Infrastructure from the outside, and keep the Api as a thin composition root. You can also tell when your codebase does not need this level of structure and pick a lighter option.

Ready to level up your next project or share it with your team? See you in the next one, Vertical Slicing is where we go next.

## Related articles

- [N-Layered Architecture in .NET: The Foundation You Need to Master](/en/posts/code-structure-n-layered/)
- [UI / Repositories / Services: The Pragmatic .NET Layering](/en/posts/code-structure-ui-repos-services/)

## References

- [Common web application architectures, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
- [Ports and Adapters pattern, Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/patterns/ports-and-adapters)
- [Dependency injection in ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)
- [Entity Framework Core fluent configuration, Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/modeling/)
- [NetArchTest on GitHub](https://github.com/BenMorris/NetArchTest)
