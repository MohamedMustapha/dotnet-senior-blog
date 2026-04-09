---
title: "Data Access in .NET: Repository Pattern, or Not?"
date: 2026-04-11
draft: false
tags: ["data-access", "repository", "efcore", "dotnet"]
categories: ["Architecture"]
series: ["Layer Focused"]
description: "The Repository pattern was the right answer for data access in 2004. EF Core changed the shape of the problem. Here is when a repository still earns its place, when it just duplicates the DbContext, and how to decide on your next project."
---

Ask ten .NET developers whether to wrap `DbContext` in a repository and you will get ten answers, all of them confident and half of them contradictory. "Always", because testability. "Never", because `DbSet<T>` already is a repository. "Depends", because everything depends. The honest answer is closer to the third, but "depends" is only useful if you know *what* it depends on.

This article lays out the original problem the Repository pattern solved, what changed when EF Core arrived, and the concrete decision framework for today's .NET applications.

## Why the Repository pattern exists

The Repository pattern was formalized by Martin Fowler in *Patterns of Enterprise Application Architecture* in 2002, and reinforced by Eric Evans in *Domain-Driven Design* in 2003. In their world, "data access" meant raw ADO.NET: `SqlConnection`, `SqlCommand`, `DataReader`, manual mapping from columns to properties, manual transaction management, and SQL strings scattered across the codebase. The Repository pattern gave you a place to put all of that: one class per aggregate, one interface the domain could depend on, and one implementation that knew the database.

It solved four real problems at once:

1. **Persistence ignorance.** The domain model did not know what a `SqlCommand` was.
2. **Centralized SQL.** Query logic lived in one place, not sprinkled in every service.
3. **Unit testability.** You could mock `IOrderRepository` in a test instead of standing up a database.
4. **A plausible translation boundary.** Column names, table names, and JOINs stayed inside the repository, while the domain stayed clean.

In 2004, wrapping ADO.NET in repositories was the right call. The pattern became gospel, and for a decade it was copy-pasted into every .NET enterprise template.

Then EF Core happened. `DbContext` is, by design, a **Unit of Work**. `DbSet<T>` is, by design, a **Repository**. `SaveChangesAsync()` is, by design, the commit of that Unit of Work. When you write `_db.Orders.Where(...).ToListAsync()`, you are already using a repository abstraction, you just did not name it that. Wrapping it in a second repository to say "we use the Repository pattern" adds ceremony without adding what the original pattern was trying to add.

## Overview: the three positions you can take

Three reasonable positions exist on this spectrum, and each has contexts where it wins.

{{< mermaid >}}
graph TD
    A[Data access need] --> B{Position}
    B -->|1| C[Use DbContext directly]
    B -->|2| D[Thin repository per aggregate]
    B -->|3| E[Full abstract repository<br/>with generic interface]
    C --> F[Handlers call _db.Orders]
    D --> G[IOrderRepository<br/>IOrderReader split]
    E --> H[IRepository&lt;T&gt; with Add, Update, List, Find]
{{< /mermaid >}}

Position 1 trusts EF Core as the data access layer. Position 2 keeps a thin, aggregate-scoped repository where it genuinely helps. Position 3 is the "generic repository" pattern that 2010-era tutorials made famous and that most senior engineers now regret.

The rest of the article walks through each position, shows code, and explains where the line sits.

## Zoom: position 1, use DbContext directly

In a CQRS-flavored application layer (see [Application Layer: CQS and CQRS](/en/posts/layer-focused-cqs-cqrs/)), the command handler holds a `DbContext` and writes queries inline:

```csharp
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

        order.Submit();

        var charge = await _payments.ChargeAsync(order.CustomerId, order.Total, ct);
        if (!charge.Success)
            throw new PaymentFailedException(charge.Error);

        await _db.SaveChangesAsync(ct);
        return new SubmitOrderResult(order.Id, order.Status.ToString());
    }
}
```

No repository interface. No mock of `IOrderRepository`. Just the handler, the `DbContext`, and domain methods on the entity.

**What you gain:**

- Less code per handler. No interface, no implementation class, no test double to wire up.
- Full access to EF Core's features: `Include`, `AsNoTracking`, compiled queries, `ExecuteUpdate`, interceptors. None of them have to be re-exposed through a repository interface.
- Query shapes can be optimized per handler. The `SubmitOrderHandler` includes lines; a different handler can project into a DTO without going through the domain.

**What you give up:**

- A mockable `IOrderRepository` interface. You test handlers against a real database (with `Testcontainers` or SQLite in-memory), not against a hand-written mock.
- A central home for reusable query logic. When two handlers need "orders placed in the last 30 days", you either duplicate the `Where` clause or extract an extension method.

This is the position most modern .NET codebases end up in, especially when combined with Vertical Slicing (see [Vertical Slicing in .NET](/en/posts/code-structure-vertical-slicing/)) or a Clean Architecture where the application layer depends on an `EF Core` abstraction package.

> 💡 **Info** — `DbContext` implements the Unit of Work pattern, and `DbSet<T>` implements the Repository pattern. This is explicitly documented by the EF Core team. If you already use `DbContext`, you already have both patterns; wrapping them in hand-written equivalents is a stylistic choice, not an architectural upgrade.

> ✅ **Good practice** — For integration tests, use a real database in a container (PostgreSQL or SQL Server) via Testcontainers. The test runs against the same provider as production, so LINQ translation quirks and migration issues show up in CI, not in a 2 AM incident.

## Zoom: position 2, a thin repository per aggregate

Sometimes you genuinely want a named place to put query logic. Position 2 keeps the `DbContext` inside a small, aggregate-scoped repository with methods that answer domain questions, not CRUD stubs:

```csharp
public interface IOrderRepository
{
    Task<Order?> FindForSubmissionAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<Order>> FindStaleAsync(TimeSpan olderThan, CancellationToken ct);
    void Add(Order order);
}

public sealed class OrderRepository : IOrderRepository
{
    private readonly ShopDbContext _db;

    public OrderRepository(ShopDbContext db) => _db = db;

    public Task<Order?> FindForSubmissionAsync(Guid id, CancellationToken ct) =>
        _db.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<Order>> FindStaleAsync(TimeSpan olderThan, CancellationToken ct)
    {
        var cutoff = DateTime.UtcNow - olderThan;
        return await _db.Orders
            .Where(o => o.Status == OrderStatus.Pending && o.PlacedAt < cutoff)
            .ToListAsync(ct);
    }

    public void Add(Order order) => _db.Orders.Add(order);
}
```

Notice the shape of the interface. Methods are **named after domain concepts** (`FindForSubmission`, `FindStale`), not after CRUD operations (`GetById`, `GetAll`, `Update`). Each method encodes a specific way the domain wants to load or save an aggregate, including the right `Include` calls and the right predicates.

`SaveChangesAsync` is deliberately *not* on the repository. That would turn every repository into its own Unit of Work, and two repositories in the same handler would each try to commit. The handler (or a pipeline behavior) owns `SaveChangesAsync`.

**When this pays off:**

- The application has a rich domain layer where loading an aggregate requires non-trivial `Include` chains, `AsSplitQuery`, or careful ordering. Hiding that in one place keeps handlers clean.
- Multiple handlers load the same aggregate the same way. Extracting to a repository method removes duplication without forcing an abstract shape.
- The team uses **architecture tests** (see [Architecture Testing in .NET](/en/posts/testing-architecture-testing/)) to enforce that the application layer depends only on interfaces, not on `DbContext` directly.

**When it does not:**

- The application is a CRUD app with 90% trivial queries. Every handler gets a bespoke repository method that is used exactly once. The repository becomes a pass-through layer with no value.
- The team uses Vertical Slicing and wants the query shape to live next to the handler. A repository drags the query away from where it matters.

> ✅ **Good practice** — If you write a repository, make the method names describe the **intent**, not the SQL. `FindForSubmission` beats `GetByIdWithLines`. The first name survives a refactor that changes which tables are joined; the second does not.

## Zoom: position 3, the generic repository, and why to avoid it

The "generic repository" pattern became famous around 2010 through MVC tutorials and the original unit-of-work-plus-generic-repository scaffolds. It looks like this:

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(object id, CancellationToken ct);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct);
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct);
    Task AddAsync(T entity, CancellationToken ct);
    void Update(T entity);
    void Delete(T entity);
}
```

It looks elegant. One interface, one implementation, all entities inherit for free. The problem is that it re-invents `DbSet<T>` with fewer features and a worse API.

- `GetAllAsync` invites loading an entire table into memory.
- `FindAsync(Expression<Func<T, bool>>)` is a leaky abstraction: the predicate has to be translatable by EF Core, so the caller has to know they are dealing with EF Core anyway.
- `Update(T)` is ambiguous: is it `Attach + State = Modified`, or `Update` (which marks every property as modified)?
- There is no clean way to express `Include`, `AsNoTracking`, or projections, so you end up with `GetByIdWithLinesAsync`, `GetByIdNoTrackingAsync`, and a dozen variants.
- You still need a Unit of Work somewhere to call `SaveChanges`, so the abstraction does not fully hide the `DbContext` lifecycle.

After a year, every method on every entity has been added, the interface is two hundred lines long, and the team is debugging "why does `Update` sometimes lose fields". The right move is usually to delete the generic repository and go back to position 1 or 2.

> ❌ **Never do this** — A generic `IRepository<T>` with `Expression<Func<T, bool>>` predicates is not an abstraction over data access. It is `DbSet<T>` with holes. If your goal is persistence ignorance, this does not deliver it: the caller still has to write expressions that only EF Core can translate. If your goal is testability, position 1 with Testcontainers gives you higher confidence at a lower code cost.

## Zoom: the testability question

The original argument for "always repository" was "so we can mock it in tests". That argument was strong when tests against a real database meant installing SQL Server and writing a reset script by hand. It is weaker today.

- **Unit tests** that mock a repository to verify handler logic still have value, and they are cheap to write. They prove that *given a set of domain objects, the handler produces the right outcome*.
- **Integration tests** with `WebApplicationFactory` and Testcontainers prove that *given real SQL, real LINQ translation, and real migrations, the endpoint produces the right HTTP response*. They catch a class of bugs mocks never see: N+1 queries, broken `Include` chains, translation failures, migration drift.

See [Integration Testing with TestContainers](/en/posts/testing-integration-testing-testcontainers/) and [API Testing with WebApplicationFactory](/en/posts/testing-webapplicationfactory/) for the concrete setup.

In practice, the teams that skip repositories in favor of direct `DbContext` use are also the teams that invest in integration tests. The two decisions travel together: you are not "losing testability", you are *moving* it to a layer that catches more real bugs.

> ⚠️ **It works, but...** — Mocking `DbContext` directly (with `Mock<DbSet<T>>` or similar) is possible and popular in tutorials, but every real codebase that tried it eventually hit a LINQ expression the mock could not translate. If you want mock-friendly handlers, use position 2 (thin per-aggregate repositories) and mock the interface, not the DbContext.

## The decision framework

Here is how I decide in practice, on a new or existing .NET project:

1. **Start at position 1** (direct `DbContext`). It is the smallest amount of code and the most features. It fits Vertical Slicing and soft CQRS naturally.
2. **Promote to position 2** (thin per-aggregate repository) when you notice:
   - The same `Include` chain is duplicated across three or more handlers.
   - The application layer is an enforced boundary (Clean Architecture, architecture tests) that should not reference EF Core directly.
   - You need a named home for complex loading strategies, especially around aggregate consistency.
3. **Never go to position 3** (generic `IRepository<T>`). If you inherit a codebase that uses it, treat it as technical debt and factor it out gradually into position 1 or 2 as you touch each module.

The choice is not ideological. It is about where the complexity actually lives in your app.

## Wrap-up

You now know why the Repository pattern existed (raw ADO.NET, 2002, persistence ignorance), what EF Core changed (`DbContext` *is* a Unit of Work and `DbSet<T>` *is* a repository), and how to pick between the three reasonable positions on a modern .NET project. You can ship handlers that use `DbContext` directly and still test them with Testcontainers. You can wrap an aggregate in a thin repository when the named intent genuinely helps. You can refuse the generic `IRepository<T>` without losing sleep. Match the abstraction to the shape of the problem, not to the scaffolding habits of a decade ago.

Ready to level up your next project or share it with your team? See you in the next one, a++ 👋

## Related articles

- [Endpoints in .NET: Controllers vs Minimal API, the Honest Comparison](/en/posts/layer-focused-controllers-vs-minimal-api/)
- [Application Layer in .NET: CQS and CQRS Without the Hype](/en/posts/layer-focused-cqs-cqrs/)
- [Vertical Slicing in .NET: Organize by Feature, Not by Layer](/en/posts/code-structure-vertical-slicing/)
- [Clean Architecture in .NET: Dependencies Pointing the Right Way](/en/posts/code-structure-clean-architecture/)

## References

- [Repository pattern, Martin Fowler](https://martinfowler.com/eaaCatalog/repository.html)
- [Unit of Work pattern, Martin Fowler](https://martinfowler.com/eaaCatalog/unitOfWork.html)
- [DbContext and Unit of Work, Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/miscellaneous/testing/)
- [Testing code that uses EF Core, Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/testing/)
- [Infrastructure persistence layer design, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)
