---
title: "EF Core: Read Optimization"
date: 2026-04-09
draft: false
tags: ["database", "efcore", "performance", "dotnet"]
categories: ["Database"]
series: ["Database"]
description: "AsNoTracking, projections, split queries, compiled queries, and the usual N+1 and cartesian explosion traps. A practical tour of making EF Core reads fast without leaving LINQ."
---

Most EF Core performance problems are read problems, and most of them look the same: a page that loaded in 40 milliseconds with 20 rows in dev takes 3 seconds in staging with 20 000 rows. The developer reaches for Dapper, migrates five queries, and the rest of the codebase stays on LINQ-to-EF. The honest answer is that EF Core has been fast enough for 95% of reads since version 6, as long as you know which knobs to turn. This article is about those knobs: change tracking, projections, split queries, compiled queries, and the two traps (N+1 and cartesian explosion) that account for most "slow query" tickets.

## Why EF Core reads can be slow by default

EF Core's default mode is built for writing: every entity you load goes into the change tracker, every navigation property can be lazy-loaded, every query hydrates full entities. That default is correct for command handlers that load an aggregate, mutate it, and save. It is wasteful for queries that only read. The change tracker takes memory and CPU to snapshot every loaded entity. The hydration builds a full object graph when you only needed four columns. The included navigations join tables you do not need for that specific page. And when a single `.Include()` crosses two collection navigations, the result set explodes into a cartesian product that is slow to transfer and slow to materialize.

Fixing a slow read means deciding, per query, which of those defaults to turn off. The LINQ stays the same shape; you add one or two method calls.

## Overview: the read optimization toolbox

{{< mermaid >}}
graph TD
    A[Slow query] --> B{Diagnosis}
    B --> C[N+1 pattern]
    B --> D[Cartesian explosion]
    B --> E[Tracking overhead]
    B --> F[Over-fetching columns]
    B --> G[Query compilation cost]
    C --> H[.Include or projection]
    D --> I[AsSplitQuery]
    E --> J[AsNoTracking]
    F --> K[Select projection to DTO]
    G --> L[EF.CompileAsyncQuery]
{{< /mermaid >}}

Five levers, five distinct problems. You do not apply them all at once: you diagnose first, then pick the one that matches.

## Zoom: AsNoTracking, the cheapest win

Any query that does not mutate the result should skip the change tracker:

```csharp
var customers = await _db.Customers
    .AsNoTracking()
    .Where(c => c.LoyaltyTier == "gold")
    .ToListAsync(ct);
```

`AsNoTracking()` tells EF Core not to snapshot the loaded entities, which saves memory and CPU proportional to the result size. For a read-only list endpoint, it is usually a 20-40% speedup on the .NET side with no change to the SQL. If you have many such queries, set it as the context default:

```csharp
services.AddDbContext<ShopDbContext>(options =>
{
    options.UseNpgsql(conn)
           .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
});
```

Commands that need to mutate then opt back in explicitly with `.AsTracking()`. This inverts the default in favor of reads, which is the right bet for most web applications where reads outnumber writes 10 to 1.

> 💡 **Info** — `AsNoTrackingWithIdentityResolution()` is the middle ground. It skips change tracking but still deduplicates references inside the result, which matters when you load an object graph with shared parents. Use it when a plain `AsNoTracking()` gives you duplicate child rows.

## Zoom: projections, the biggest win

The fastest query is the one that only asks for what the endpoint actually returns. If the API response has 5 fields, fetch 5 columns, not the 30-column entity.

```csharp
public sealed record CustomerListItem(Guid Id, string Email, string LoyaltyTier);

var customers = await _db.Customers
    .Where(c => c.LoyaltyTier == "gold")
    .Select(c => new CustomerListItem(c.Id, c.Email, c.LoyaltyTier))
    .ToListAsync(ct);
```

Three things happen automatically when you project:

1. EF Core generates a `SELECT` with only the projected columns. No unused joins, no `BLOB` columns dragged over the wire.
2. The result type is not an entity, so there is nothing to track. `AsNoTracking()` becomes redundant on a projected query.
3. EF Core can translate a projection across a navigation without needing an `.Include()`.

That third point is the one most developers miss. You do not need `.Include(c => c.Orders)` if you only want the latest order date:

```csharp
var customers = await _db.Customers
    .Select(c => new
    {
        c.Id,
        c.Email,
        LastOrderDate = c.Orders.Max(o => (DateTime?)o.CreatedAt)
    })
    .ToListAsync(ct);
```

EF Core translates that into a `LEFT JOIN LATERAL` (or an equivalent subquery depending on the provider) and returns one row per customer. No N+1, no extra round trip.

> ✅ **Good practice** — For any query feeding a list view, project to a DTO. `.Include()` is for commands that load an aggregate, not for reads that feed a UI.

## Zoom: the cartesian explosion

```csharp
var orders = await _db.Orders
    .Include(o => o.Lines)
    .Include(o => o.Payments)
    .Where(o => o.CreatedAt > since)
    .ToListAsync(ct);
```

This looks innocent and is a classic pitfall. EF Core generates a single SQL query with two `LEFT JOIN`s. An order with 5 lines and 2 payments produces 10 rows in the result set, one for every combination. Scale that to 1 000 orders and you are transferring tens of thousands of rows over the wire to hydrate what the domain sees as 1 000 objects. EF Core warns you at startup in the logs when it detects this, and the fix is one call:

```csharp
var orders = await _db.Orders
    .AsSplitQuery()
    .Include(o => o.Lines)
    .Include(o => o.Payments)
    .Where(o => o.CreatedAt > since)
    .ToListAsync(ct);
```

`AsSplitQuery()` makes EF Core emit three separate `SELECT`s (one for orders, one for lines, one for payments) and stitch them in memory. You pay three round trips instead of one, but you transfer `N + L + P` rows instead of `N * L * P`. For any collection that has more than a handful of children, split queries win.

You can also set the default at the context level:

```csharp
options.UseNpgsql(conn, npgsql => npgsql.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
```

> ⚠️ **Works, but...** — Single-query mode is fine for 1:1 navigations and for small 1:N collections. The moment you chain two `.Include()` over collections, you are in cartesian explosion territory. When in doubt, switch to split query.

## Zoom: the N+1 trap

The classic N+1 comes from iterating loaded entities and accessing a navigation on each one:

```csharp
var orders = await _db.Orders.Where(o => o.CreatedAt > since).ToListAsync(ct);
foreach (var order in orders)
{
    var customer = order.Customer; // lazy load, one SQL per order
    Console.WriteLine($"{order.Id} - {customer.Email}");
}
```

Lazy loading fires a query per iteration. The fix is either an explicit `.Include(o => o.Customer)`, or, better, a projection that pulls the customer email directly:

```csharp
var orderSummaries = await _db.Orders
    .Where(o => o.CreatedAt > since)
    .Select(o => new { o.Id, CustomerEmail = o.Customer.Email })
    .ToListAsync(ct);
```

> ❌ **Never do** — Do not enable lazy loading globally on a CRUD API. It turns every forgotten `.Include()` into a silent N+1 that only shows up under production traffic. EF Core ships with lazy loading off by default for exactly this reason. If you need it, enable it per context, and audit every query that touches a navigation.

## Zoom: compiled queries for hot paths

EF Core parses and translates the LINQ expression tree on every call. For most queries the cost is negligible compared to the database round trip, but for a hot path called thousands of times per second, the translation starts to show up in the profiler. Compiled queries cache the translation:

```csharp
private static readonly Func<ShopDbContext, Guid, CancellationToken, Task<Customer?>> GetCustomerById =
    EF.CompileAsyncQuery((ShopDbContext db, Guid id, CancellationToken ct) =>
        db.Customers.AsNoTracking().FirstOrDefault(c => c.Id == id));

public Task<Customer?> FindAsync(Guid id, CancellationToken ct) => GetCustomerById(_db, id, ct);
```

`EF.CompileAsyncQuery` takes the LINQ expression, translates it once, and returns a delegate that reuses the compiled plan. On a query called 10 000 times, this typically shaves 20-30% of the .NET-side latency. Use it surgically: the top 5 queries in your telemetry, not every repository method.

> 💡 **Info** — Since EF Core 6, query compilation results are also cached internally per distinct LINQ expression shape. `EF.CompileAsyncQuery` still wins on raw throughput, but the gap is narrower than it used to be.

## Zoom: reading the generated SQL

Every optimization decision depends on reading the actual SQL. Enable the logger and log SQL at the debug level in development:

```csharp
options.UseNpgsql(conn)
       .LogTo(Console.WriteLine, LogLevel.Information)
       .EnableSensitiveDataLogging();
```

In production, keep EF Core's category at `Warning` to catch the cartesian explosion and multiple-collection warnings, but do not log every statement. Instead, rely on OpenTelemetry instrumentation, which emits each database query as a span with the SQL attached.

```csharp
services.AddOpenTelemetry()
        .WithTracing(t => t.AddEntityFrameworkCoreInstrumentation(o => o.SetDbStatementForText = true));
```

That gives you a timeline of every query each request executes, which is usually enough to spot the one that is taking 400 ms when the other 19 are taking 2 ms each.

## Wrap-up

The five moves cover most read performance problems. `AsNoTracking` for every read-only query. Project to a DTO for any list endpoint. `AsSplitQuery` the moment you chain `.Include()` over collections. Hunt lazy-loading N+1s by reading the generated SQL, not by inspecting the C#. And compile the three or four queries that actually show up in the hot path. Once those are in place, the remaining 5% of slow queries are the ones that genuinely need raw SQL, which is the topic of the next article.

## Related articles

- [EF Core: Configuration and Seeding](/en/posts/database-efcore-config-seed/)
- [EF Core: Migrations in Practice](/en/posts/database-efcore-migrations/)

## References

- [EF Core: Performance](https://learn.microsoft.com/en-us/ef/core/performance/)
- [EF Core: Efficient Querying](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying)
- [EF Core: Split Queries](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries)
- [EF Core: Compiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics#compiled-queries)
- [EF Core: Tracking vs No-Tracking](https://learn.microsoft.com/en-us/ef/core/querying/tracking)
