---
title: "Dapper: When and How"
date: 2026-04-09
draft: false
tags: ["database", "dapper", "dotnet"]
categories: ["Database"]
series: ["Database"]
description: "Dapper is not a replacement for EF Core, it is a different tool with a different contract. When to reach for it, how to use it well, and how to let it coexist with EF Core in the same codebase."
---

Dapper has been around since 2011, written by the Stack Overflow team to handle the SQL workload that Entity Framework of the time could not. It is still maintained, still fast, and still the right answer for a specific set of problems. The mistake most teams make is treating it as a choice between Dapper and EF Core, as if you have to pick one for the whole project. In practice, the two coexist cleanly: EF Core for writes and for the 90% of reads that LINQ handles well, Dapper for the reporting queries, the dashboards, the bulk exports, and the hot paths where the SQL is fundamentally not a LINQ query.

This article covers what Dapper actually is, where it wins, how to use it without rebuilding a mini-ORM around it, and how to run it in the same codebase as EF Core.

## Why Dapper exists

Dapper is a set of extension methods on `IDbConnection`. That is the entire library. You open a connection, you call `connection.QueryAsync<T>("SELECT ...", parameters)`, and Dapper maps the result set to `T` by matching column names to property names. There is no model, no change tracker, no migrations, no LINQ provider, no relationship mapping. You write the SQL, Dapper runs it and hydrates the objects.

That minimalism is the point. EF Core solves the problem "I have a domain model and I want the database to follow it". Dapper solves the problem "I have a SQL query and I want it to return typed objects". Those are different problems, and a team that understands the distinction stops arguing about which one to use.

The performance gap between Dapper and EF Core has narrowed significantly since EF Core 6. On a single-entity query, EF Core with `AsNoTracking()` and a projection is typically within 10-15% of Dapper. The gap widens on queries that involve unusual SQL shapes: window functions, recursive CTEs, `UNION ALL` across unrelated tables, or hand-tuned `JOIN` orders. Those are the queries where Dapper earns its keep, not because it is faster on trivial cases but because writing them in LINQ is either impossible or produces translated SQL you would not ship.

## Overview: where Dapper fits

{{< mermaid >}}
graph TD
    A[Data access need] --> B{Shape}
    B -->|Domain write: load aggregate, mutate, save| C[EF Core]
    B -->|Typed list/detail read from an entity| C
    B -->|Reporting query, CTE, window function| D[Dapper]
    B -->|Bulk export, read-heavy endpoint| D
    B -->|Multi-table stitching with hand-tuned SQL| D
    C --> E[ShopDbContext]
    D --> F[IDbConnection + SQL files]
{{< /mermaid >}}

The split is by query shape, not by feature area. The same module can have EF Core for the command handlers and Dapper for the read model that feeds the dashboard.

## Zoom: a clean Dapper setup

```csharp
public sealed class OrderReports
{
    private readonly string _connectionString;

    public OrderReports(IConfiguration config)
    {
        _connectionString = config.GetConnectionString("Shop")
            ?? throw new InvalidOperationException("Missing connection string");
    }

    public async Task<IReadOnlyList<MonthlyRevenueRow>> GetMonthlyRevenueAsync(int year, CancellationToken ct)
    {
        const string sql = """
            SELECT
                date_trunc('month', o.created_at) AS Month,
                SUM(o.total_amount)                AS Revenue,
                COUNT(*)                           AS OrderCount
            FROM orders o
            WHERE EXTRACT(YEAR FROM o.created_at) = @Year
              AND o.status = 'completed'
            GROUP BY date_trunc('month', o.created_at)
            ORDER BY Month;
            """;

        await using var conn = new NpgsqlConnection(_connectionString);
        var rows = await conn.QueryAsync<MonthlyRevenueRow>(
            new CommandDefinition(sql, new { Year = year }, cancellationToken: ct));
        return rows.ToList();
    }
}

public sealed record MonthlyRevenueRow(DateTime Month, decimal Revenue, int OrderCount);
```

Three things worth noting. First, `CommandDefinition` is the modern overload that accepts a `CancellationToken`. Use it for every call: cancellation is not free on long-running reports. Second, parameters are passed as an anonymous object, which Dapper turns into parameterized SQL. Never concatenate user input into the SQL string. Third, the SQL is in a raw string literal (`"""`), which keeps it readable and indents naturally.

> 💡 **Info** — Raw string literals arrived in C# 11. For older target frameworks, move the SQL to an embedded `.sql` file and read it with `typeof(OrderReports).Assembly.GetManifestResourceStream(...)`.

## Zoom: reusing the EF Core connection

When EF Core and Dapper coexist in the same request, the common mistake is opening two separate connections. EF Core has one open inside `DbContext`, and Dapper opens a second one, which doubles the connection pool usage and breaks any transactional guarantee you might have wanted between the two. The fix is to ask the `DbContext` for its connection:

```csharp
public sealed class OrderReports
{
    private readonly ShopDbContext _db;

    public OrderReports(ShopDbContext db) => _db = db;

    public async Task<IReadOnlyList<MonthlyRevenueRow>> GetMonthlyRevenueAsync(int year, CancellationToken ct)
    {
        var conn = _db.Database.GetDbConnection();
        if (conn.State != ConnectionState.Open)
            await conn.OpenAsync(ct);

        const string sql = /* same SQL as above */;
        var rows = await conn.QueryAsync<MonthlyRevenueRow>(
            new CommandDefinition(sql, new { Year = year }, cancellationToken: ct));
        return rows.ToList();
    }
}
```

`_db.Database.GetDbConnection()` returns the underlying `DbConnection` that EF Core manages. Dapper can run any query on it, and you stay inside the same transaction if one is active. When the `DbContext` is disposed at the end of the scope, the connection goes back to the pool.

> ✅ **Good practice** — For anything that runs in a request that also touches EF Core, share the connection. For background jobs or standalone reporting endpoints with no EF Core involvement, a dedicated `NpgsqlConnection` is perfectly fine.

## Zoom: multi-row results and splitting

When the query joins multiple tables and you want a parent with children mapped into different CLR types, use `QueryAsync` with a splitOn:

```csharp
const string sql = """
    SELECT o.id, o.reference, o.total_amount,
           c.id, c.email, c.loyalty_tier
    FROM orders o
    INNER JOIN customers c ON c.id = o.customer_id
    WHERE o.created_at > @Since;
    """;

var rows = await conn.QueryAsync<OrderRow, CustomerRow, OrderWithCustomer>(
    sql,
    (order, customer) => new OrderWithCustomer(order, customer),
    new { Since = since },
    splitOn: "id");
```

`splitOn: "id"` tells Dapper where to cut the result row between the two mapped types. Everything from the first column up to (and not including) the first `id` column after it becomes the first type, and the rest becomes the second type. It is the one Dapper detail that catches everyone the first time.

> ⚠️ **Works, but...** — For queries that join four or more tables, the splitOn syntax becomes hard to read. At that point, return flat DTOs and do the stitching in C#, or move the query to a SQL view and map the view to a single DTO.

## Zoom: dynamic parameters and stored procedures

```csharp
var parameters = new DynamicParameters();
parameters.Add("CustomerId", customerId, DbType.Guid);
parameters.Add("Since", since, DbType.DateTime2);
parameters.Add("TotalOut", dbType: DbType.Decimal, direction: ParameterDirection.Output);

await conn.ExecuteAsync(
    "sp_compute_customer_total",
    parameters,
    commandType: CommandType.StoredProcedure);

var total = parameters.Get<decimal>("TotalOut");
```

`DynamicParameters` is the escape hatch for anything beyond plain input parameters: output parameters, return values, stored procedures, typed `DbType` control for edge cases where the default Dapper mapping is wrong.

> ❌ **Never do** — Do not build SQL by string concatenation under the excuse that "it is only for an admin dashboard". The admin dashboard is the first place SQL injection gets noticed, because that is where someone eventually pastes a filter value with a single quote in it. Parameters, always.

## Zoom: when not to use Dapper

Dapper is wrong for three things:

1. **Writing domain aggregates.** It has no change tracker, so you either write all the INSERTs and UPDATEs by hand, or you end up rebuilding EF Core badly. Keep writes on EF Core.
2. **Multi-tenant query filtering.** EF Core global query filters are declarative and hard to forget. In Dapper, the tenant filter is a `WHERE tenant_id = @TenantId` you have to remember on every query. One miss and you are reading another tenant's data.
3. **Projects with junior developers and no SQL review culture.** Dapper trusts the author. It does not protect you from a bad `JOIN` order, a missing index, or a query that looks fine in dev and brings production to its knees. On a team that cannot review SQL, the EF Core guardrails are worth the minor overhead.

## Wrap-up

Dapper and EF Core are complements, not competitors. EF Core for the command side and for typed reads that LINQ expresses cleanly; Dapper for the reporting, the CTE queries, the dashboards, and anything where SQL is the first-class language. Sharing the connection through `_db.Database.GetDbConnection()` lets them coexist in the same request and the same transaction. Once the split is clear, the "EF Core versus Dapper" debate stops being a debate.

The next article in this series is the one that ties everything together: how to diagnose and fix a slow query when the code is already written and production is burning.

## Related articles

- [EF Core: Read Optimization](/en/posts/database-efcore-read-optimization/)
- [Data Access in .NET: Repository Pattern, or Not?](/en/posts/layer-focused-repository-or-not/)
- [Application Layer: CQS and CQRS](/en/posts/layer-focused-cqs-cqrs/)

## References

- [Dapper on GitHub](https://github.com/DapperLib/Dapper)
- [Dapper on NuGet](https://www.nuget.org/packages/Dapper)
- [EF Core: Accessing the Underlying ADO.NET Connection](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connections)
- [Npgsql documentation](https://www.npgsql.org/doc/index.html)
