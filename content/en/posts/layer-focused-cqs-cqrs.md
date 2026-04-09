---
title: "Application Layer in .NET: CQS and CQRS Without the Hype"
date: 2026-04-10
draft: false
tags: ["application-layer", "cqrs", "cqs", "dotnet"]
categories: ["Architecture"]
series: ["Layer Focused"]
description: "CQS is a method-level design rule. CQRS is an architectural pattern built on top of it. Mixing them up is how teams end up with two databases for a three-screen CRUD app. Here is the honest breakdown, with working .NET code."
---

CQS and CQRS are two of the most misquoted acronyms in .NET. Teams say "we do CQRS" when they mean "we have a `MediatR` pipeline", or "CQRS is overkill for us" when they actually mean "we do not need event sourcing". The two ideas are related, but they live at different levels: CQS is a rule about methods, CQRS is a rule about entire read and write paths. Knowing which one you are reaching for is the difference between a clean application layer and two years of regret.

This article unpacks both, with realistic ASP.NET Core code, and shows where the line sits in practice.

## Why these ideas exist

**CQS (Command Query Separation)** was coined by Bertrand Meyer in the 1980s in *Object-Oriented Software Construction*. His rule is simple: a method should either perform an action (a command, which changes state and returns nothing) or answer a question (a query, which returns data and does not change state), but not both. The point is not religious purity, it is to make reasoning about code easier. If you know a method is a query, you can call it a hundred times in a debugger without worrying about side effects.

**CQRS (Command Query Responsibility Segregation)** was introduced by Greg Young around 2010, building on Udi Dahan's ideas and his own experience with event sourcing. Young took CQS and pushed it up one level: instead of separating commands and queries at the method level, separate them at the *architectural* level. Writes and reads become two different paths through the application, potentially with two different models, two different stores, and two different teams. CQRS is a response to a specific pain: trying to serve a rich write model and a dozen different read views from the same set of entities turns your domain classes into a bag of `IsXxx`, `HasYyy`, and "include this only on the summary screen" properties.

The two ideas share a lineage, but they solve problems at very different scales. CQS is something you should probably do in every class you write. CQRS is something you should reach for when the complexity of your reads genuinely diverges from the complexity of your writes.

## Overview: where each one lives

{{< mermaid >}}
graph TD
    subgraph CQS["CQS: method-level rule"]
        A[OrderService] --> B["PlaceOrder cmd : void"]
        A --> C["GetTotal query : decimal"]
    end
    subgraph CQRS["CQRS: architectural split"]
        D[API] --> E[Command side]
        D --> F[Query side]
        E --> G[Domain model]
        G --> H[(Write store)]
        F --> I[(Read store<br/>or projections)]
    end
{{< /mermaid >}}

CQS lives inside a class. CQRS lives across your codebase. You can do CQS without CQRS. You can also do CQRS without event sourcing, without two databases, and without a message bus. Losing track of these distinctions is how "let us add CQRS" becomes a six-month project.

## Zoom: CQS in a service, end to end

Start with a plain service that violates CQS, then fix it. Here is the kind of method you have almost certainly written or reviewed:

```csharp
public sealed class OrderService
{
    private readonly ShopDbContext _db;

    public OrderService(ShopDbContext db) => _db = db;

    // Violates CQS: changes state AND returns data.
    public async Task<Order> PlaceOrderAsync(PlaceOrderInput input, CancellationToken ct)
    {
        var order = new Order(input.CustomerId, input.Lines);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);
        return order;
    }
}
```

The method places an order *and* returns the full entity. The caller now has a tracked `Order` with lazy-loadable navigation properties, which may or may not be safely used outside this transaction. Two unrelated concerns travel on the same return value. Tests that want to assert "the order was placed" end up also asserting "the returned entity has the expected shape".

The CQS version splits it:

```csharp
public sealed class OrderService
{
    private readonly ShopDbContext _db;

    public OrderService(ShopDbContext db) => _db = db;

    // Command: changes state, returns only what the caller strictly needs (an id).
    public async Task<Guid> PlaceOrderAsync(PlaceOrderInput input, CancellationToken ct)
    {
        var order = new Order(input.CustomerId, input.Lines);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);
        return order.Id;
    }

    // Query: returns data, never mutates.
    public async Task<OrderSummary> GetOrderSummaryAsync(Guid id, CancellationToken ct)
    {
        return await _db.Orders
            .AsNoTracking()
            .Where(o => o.Id == id)
            .Select(o => new OrderSummary(o.Id, o.Status.ToString(), o.Total))
            .FirstAsync(ct);
    }
}
```

Two methods, two intents. The command returns an id (a purist would return `void`, but an id is the minimum information the caller needs to route to the next screen and is still trivially "not part of the domain state"). The query is `AsNoTracking`, projects into a DTO, and has no business logic.

> 💡 **Info** — Returning an id from a command is the pragmatic compromise every serious .NET codebase makes. Strict Meyer-style CQS would return `void` and make the caller issue a follow-up query. In practice, that costs a round trip for no real benefit. Returning `Guid` (or `Result<Guid>`) is fine.

> ✅ **Good practice** — Write queries with `AsNoTracking()` and projections to DTOs by default. The change tracker is a feature you pay for on every load, and queries almost never need it.

## Zoom: CQRS with MediatR, the honest version

CQRS at the architectural level means: the command path and the query path are two different shapes, even if they share the same database. In .NET, the most common way to express this is with MediatR (or any in-process mediator), sending `ICommand<T>` and `IQuery<T>` objects from the API layer to handlers.

Here is a command handler, the write path, going through the domain model:

```csharp
// Application/Orders/Commands/SubmitOrderCommand.cs
public sealed record SubmitOrderCommand(Guid OrderId) : IRequest<SubmitOrderResult>;
public sealed record SubmitOrderResult(Guid OrderId, string Status);

// Application/Orders/Commands/SubmitOrderHandler.cs
public sealed class SubmitOrderHandler : IRequestHandler<SubmitOrderCommand, SubmitOrderResult>
{
    private readonly ShopDbContext _db;
    private readonly IPaymentGateway _payments;

    public SubmitOrderHandler(ShopDbContext db, IPaymentGateway payments)
    {
        _db = db;
        _payments = payments;
    }

    public async Task<SubmitOrderResult> Handle(SubmitOrderCommand cmd, CancellationToken ct)
    {
        var order = await _db.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == cmd.OrderId, ct)
            ?? throw new NotFoundException($"Order {cmd.OrderId} not found.");

        order.Submit(); // enforces invariants inside the aggregate

        var charge = await _payments.ChargeAsync(order.CustomerId, order.Total, ct);
        if (!charge.Success)
            throw new PaymentFailedException(charge.Error);

        await _db.SaveChangesAsync(ct);
        return new SubmitOrderResult(order.Id, order.Status.ToString());
    }
}
```

And here is a query handler, the read path, bypassing the domain entirely and projecting straight from the DbContext into the shape the UI actually wants:

```csharp
// Application/Orders/Queries/GetOrderListQuery.cs
public sealed record GetOrderListQuery(int Page, int PageSize, string? Status)
    : IRequest<PagedResult<OrderListItem>>;

public sealed record OrderListItem(
    Guid Id, string CustomerName, decimal Total, string Status, DateTime PlacedAt);

// Application/Orders/Queries/GetOrderListHandler.cs
public sealed class GetOrderListHandler
    : IRequestHandler<GetOrderListQuery, PagedResult<OrderListItem>>
{
    private readonly ShopDbContext _db;

    public GetOrderListHandler(ShopDbContext db) => _db = db;

    public async Task<PagedResult<OrderListItem>> Handle(
        GetOrderListQuery q, CancellationToken ct)
    {
        var query = _db.Orders.AsNoTracking();

        if (!string.IsNullOrWhiteSpace(q.Status))
            query = query.Where(o => o.Status.ToString() == q.Status);

        var total = await query.CountAsync(ct);

        var items = await query
            .OrderByDescending(o => o.PlacedAt)
            .Skip((q.Page - 1) * q.PageSize)
            .Take(q.PageSize)
            .Select(o => new OrderListItem(
                o.Id, o.Customer.Name, o.Total, o.Status.ToString(), o.PlacedAt))
            .ToListAsync(ct);

        return new PagedResult<OrderListItem>(items, total, q.Page, q.PageSize);
    }
}
```

Same `DbContext`. One database. Two very different shapes of code. The command goes through aggregates to protect invariants. The query goes around aggregates to deliver pixels. This is CQRS at its most useful: a clean architectural split *without* two databases, *without* event sourcing, *without* eventual consistency headaches.

> 💡 **Info** — Single database, split code paths. This is sometimes called "soft CQRS" or "CQRS lite", and for most business applications it is the version that earns its keep. For the full story on how CQRS evolved and when the heavier variants make sense, see [CQRS Explained for .NET Developers](/en/posts/cqrs-explained/).

## Zoom: where the endpoint layer fits

Commands and queries need to be dispatched from somewhere. Endpoints (whether Controllers or Minimal APIs) become thin dispatchers: they bind the request, call `mediator.Send`, and return the result.

```csharp
// Endpoint, Minimal API style
orders.MapPost("/{id:guid}/submit", async (
    Guid id, ISender mediator, CancellationToken ct) =>
{
    var result = await mediator.Send(new SubmitOrderCommand(id), ct);
    return TypedResults.Ok(result);
});

orders.MapGet("/", async (
    int page, int pageSize, string? status,
    ISender mediator, CancellationToken ct) =>
{
    var result = await mediator.Send(new GetOrderListQuery(page, pageSize, status), ct);
    return TypedResults.Ok(result);
});
```

No business logic in the endpoint. Binding, dispatch, return. For the endpoint style tradeoff (and why the two examples above use Minimal APIs), see [Endpoints in .NET: Controllers vs Minimal API](/en/posts/layer-focused-controllers-vs-minimal-api/).

> ✅ **Good practice** — Add a MediatR pipeline behavior for cross-cutting concerns (validation, logging, transactions). One behavior added to the DI container applies to every command and every query, which means you do not sprinkle try/catch or `using var tx = ...` in every handler.

```csharp
public sealed class TransactionBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ShopDbContext _db;

    public TransactionBehavior(ShopDbContext db) => _db = db;

    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        // Only wrap commands in a transaction, not queries.
        if (request is not ICommand) return await next();

        await using var tx = await _db.Database.BeginTransactionAsync(ct);
        var response = await next();
        await tx.CommitAsync(ct);
        return response;
    }
}
```

## Zoom: when CQRS earns its complexity

A soft CQRS setup (split handlers, shared DB) is almost always a good idea in any non-trivial application. The harder variants are a different conversation:

- **Separate read store**: when the read model is genuinely different (denormalized, full-text indexed, geographically distributed), and keeping it in sync with the write store is worth the operational cost.
- **Event sourcing**: when you need to reconstruct history, support temporal queries, or audit every state change. This is a huge commitment. It changes how you design tests, migrations, and deployments.
- **Separate command and query APIs**: when different teams own reads and writes, or when you need to scale the two independently.

Each of these layers solves a real problem, and each adds real cost. The right question is not "should we do CQRS", it is "which layer of CQRS does the problem actually need".

> ⚠️ **It works, but...** — Adding a separate read database and a message bus to a system with 3 aggregates and 12 endpoints is a classic over-engineering pattern. The split handlers inside one DB give you most of the benefit (clean read paths, no domain pollution, easier query tuning) with almost none of the cost.

> ❌ **Never do this** — Do not force your queries to go through your domain aggregates "for consistency". A query that loads an `Order` aggregate, walks its lines, and reshapes them into a DTO in C# is slower, more memory-hungry, and harder to maintain than a single `Select` projection. Reads and writes are allowed to have different shapes on purpose.

## Wrap-up

You now know the difference between CQS and CQRS: CQS is the method-level rule that says a method should either do or ask, never both; CQRS is the architectural pattern that takes that rule and lifts it to whole command and query paths in your application. You can write clean CQS in any service class today. You can adopt soft CQRS with MediatR and one database on any greenfield .NET project. You can reach for the heavier variants (separate stores, event sourcing) only when the problem demands it, not because a blog post said so.

Ready to level up your next project or share it with your team? See you in the next one, a++ 👋

## Related articles

- [Endpoints in .NET: Controllers vs Minimal API, the Honest Comparison](/en/posts/layer-focused-controllers-vs-minimal-api/)
- [CQRS Explained for .NET Developers](/en/posts/cqrs-explained/)
- [Vertical Slicing in .NET: Organize by Feature, Not by Layer](/en/posts/code-structure-vertical-slicing/)

## References

- [Command Query Separation, Bertrand Meyer (original book reference)](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)
- [CQRS Documents by Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
- [MediatR on GitHub](https://github.com/jbogard/MediatR)
- [Entity Framework Core, Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/)
- [CQRS pattern, Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
