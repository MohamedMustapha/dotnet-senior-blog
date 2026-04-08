---
title: "Integration Testing with TestContainers for .NET"
date: 2026-04-08
draft: false
tags: ["testing", "integration", "testcontainers", "dotnet"]
categories: ["Testing"]
series: ["Testing"]
description: "Real databases, real Redis, real brokers, spun up per test run in seconds. TestContainers for .NET kills the 'works on my machine' for integration tests."
---

Integration tests used to be the worst part of a .NET project. A shared dev SQL Server that three teams fought over. A docker-compose file that everyone ran locally "the right way" until it drifted. A CI pipeline with a hardcoded connection string that only worked on Tuesdays. The result: nobody trusted the tests, and the team fell back to mocking the database and pretending. If you have read the previous article on [Unit Testing in .NET](/posts/testing-unit-testing/), you already know why that fallback is a bad idea: EF Core bugs live in the generated SQL, and you cannot catch them by mocking `DbContext`.

TestContainers fixes this. The original Java library was released in 2015 by Richard North, and the .NET port landed in 2017 as Testcontainers for .NET. It is now the official standard, maintained under the `testcontainers` GitHub organization, and .NET 10 treats it as a first-class integration testing tool. The idea is simple: your test code starts a real Postgres / Redis / RabbitMQ / whatever in a throwaway Docker container, waits for it to be ready, hands you a connection string, and tears it down when the test fixture disposes.

## Why this pattern exists

Picture a team whose integration tests run against a shared SQL Server instance on a dev VM. One test leaves a row behind. Another test assumes the row is not there. Tuesday morning, all hell breaks loose. Someone patches the test with `DELETE FROM Orders WHERE ...` and the cycle repeats. Six months later, half the suite is disabled.

What the team actually needs:

1. **A real database**, so EF Core migrations, indexes, and queries run against the thing that will run in production.
2. **Isolation per test run**, so nobody leaves state for anyone else.
3. **A one-command developer experience**, so a new hire clones the repo, runs `dotnet test`, and everything works.

TestContainers delivers all three by outsourcing the hard part to Docker.

## Overview: how it plugs in

Before the code, here is how TestContainers sits in a .NET test project:

{{< mermaid >}}
graph TD
    A[Test fixture] --> B[Testcontainers library]
    B --> C[Docker daemon]
    C --> D[Postgres container]
    C --> E[Redis container]
    A --> F[Your SUT<br/>e.g. Repository + DbContext]
    F --> D
    F --> E
{{< /mermaid >}}

The test fixture owns the container lifecycle. The SUT gets a real connection string and has no idea it is talking to a container that will be gone in 20 seconds.

> 💡 **Info** : TestContainers needs Docker running on the machine (Docker Desktop on Windows/macOS, or rootless Docker on Linux). In CI, GitHub Actions and Azure DevOps both provide Docker-in-Docker runners out of the box.

## Zoom: a Postgres fixture with xUnit

Here is the minimum setup to spin up Postgres, apply EF Core migrations, and make it available to tests:

```csharp
using Testcontainers.PostgreSql;
using Microsoft.EntityFrameworkCore;
using Xunit;

public sealed class PostgresFixture : IAsyncLifetime
{
    public PostgreSqlContainer Container { get; } = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine")
        .WithDatabase("shop_test")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public ShopDbContext CreateDbContext()
    {
        var options = new DbContextOptionsBuilder<ShopDbContext>()
            .UseNpgsql(Container.GetConnectionString())
            .Options;
        return new ShopDbContext(options);
    }

    public async ValueTask InitializeAsync()
    {
        await Container.StartAsync();
        await using var db = CreateDbContext();
        await db.Database.MigrateAsync();
    }

    public ValueTask DisposeAsync() => Container.DisposeAsync();
}
```

`IAsyncLifetime` is xUnit's hook for async setup and teardown. `StartAsync()` pulls the image (cached after the first run) and waits for Postgres to be ready. Then EF Core applies your real migrations against it.

> ✅ **Good practice** : Pin the image tag (`postgres:17-alpine`, not `postgres:latest`). CI needs to be reproducible. A "latest" that changes under you is a ticking bug.

## Zoom: a test that uses the fixture

```csharp
[Collection("postgres")]
public class OrderRepositoryTests
{
    private readonly PostgresFixture _fixture;

    public OrderRepositoryTests(PostgresFixture fixture) => _fixture = fixture;

    [Fact]
    public async Task AddAsync_persists_order_with_lines()
    {
        // Arrange
        await using var db = _fixture.CreateDbContext();
        var repo = new OrderRepository(db);
        var order = Order.Create(CustomerId.New());
        order.AddLine(new ProductId(1), 2, new Money(49.99m));

        // Act
        await repo.AddAsync(order, default);
        await db.SaveChangesAsync();

        // Assert
        await using var verify = _fixture.CreateDbContext();
        var loaded = await verify.Orders.Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == order.Id);

        loaded.Should().NotBeNull();
        loaded!.Lines.Should().HaveCount(1);
        loaded.Lines.First().Subtotal.Amount.Should().Be(99.98m);
    }
}

[CollectionDefinition("postgres")]
public class PostgresCollection : ICollectionFixture<PostgresFixture> { }
```

`[Collection("postgres")]` tells xUnit to share the same fixture across all tests in the collection. One container, many tests, fast.

> 💡 **Info** : xUnit v3 still uses collection fixtures for shared expensive resources. The collection guarantees tests inside it do not run in parallel, which is exactly what you want when they share a database.

## Zoom: cleaning between tests

Sharing a container across tests means tests can see each other's data. Two common strategies:

**1. Respawn (fastest)** : the [Respawn](https://github.com/jbogard/Respawn) library (also by Jimmy Bogard) deletes all rows between tests, keeping the schema:

```csharp
public async Task ResetDatabaseAsync()
{
    await using var conn = new NpgsqlConnection(Container.GetConnectionString());
    await conn.OpenAsync();
    var respawner = await Respawner.CreateAsync(conn,
        new RespawnerOptions { DbAdapter = DbAdapter.Postgres });
    await respawner.ResetAsync(conn);
}
```

Call `ResetDatabaseAsync` in a test constructor or an `IAsyncLifetime` on the test class.

**2. Transaction rollback** : begin a transaction at the start of each test, let the test run, rollback at the end. Faster than Respawn but cannot test code that commits its own transaction.

> ⚠️ **It works, but...** : An in-memory provider like `Microsoft.EntityFrameworkCore.InMemory` is tempting because it is fast, but it silently ignores foreign keys, constraints, and SQL-specific behavior. It is fine for testing services with trivial EF logic and dangerous for anything that touches a real query. Prefer a real Postgres container.

> ❌ **Never do this** : Do not point your integration tests at a shared dev database. You will discover, painfully, that two engineers running the suite at the same time corrupt each other's state. TestContainers removes the excuse.

## Zoom: composing multiple services

Real apps need more than a database. Redis for caching, RabbitMQ for messages, MinIO for S3-compatible storage. TestContainers composes these in the same fixture:

```csharp
public sealed class AppServicesFixture : IAsyncLifetime
{
    public PostgreSqlContainer Postgres { get; } = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine").Build();

    public RedisContainer Redis { get; } = new RedisBuilder()
        .WithImage("redis:7-alpine").Build();

    public async ValueTask InitializeAsync()
    {
        await Task.WhenAll(Postgres.StartAsync(), Redis.StartAsync());
    }

    public async ValueTask DisposeAsync()
    {
        await Postgres.DisposeAsync();
        await Redis.DisposeAsync();
    }
}
```

`Task.WhenAll` starts them in parallel, saving seconds per test run. The first run pulls images; subsequent runs reuse the Docker image cache and start in under two seconds each.

> ✅ **Good practice** : Put the fixture in a shared testing project and reference it from `IntegrationTests`, `ApiTests`, and `E2ETests`. One source of truth for what your app depends on.

## When this is overkill

Not every project needs TestContainers. A service that has no database and talks only to stateless HTTP APIs can test everything with unit tests plus `WebApplicationFactory`. A prototype that will be rewritten in two months probably does not need the setup cost.

Reach for TestContainers when:

- You have real EF Core queries whose generated SQL matters.
- Your tests must prove that a migration applies cleanly.
- You depend on Redis, a message broker, or an S3-compatible store whose real behavior matters.
- You have more than one developer and want "clone and test" to actually work on day one.

## Wrap-up

You now know how to stand up real databases and dependencies for your integration tests using TestContainers: pick a container builder, wire an xUnit fixture with `IAsyncLifetime`, apply EF Core migrations against it, share it across a test collection, and reset state between tests with Respawn or a transaction. You can compose Postgres, Redis, and other services in the same fixture and give your team a "clone and `dotnet test`" experience that actually works.

Ready to level up your next project or share it with your team? See you in the next one, API Testing with WebApplicationFactory is where we go next.

## Related articles

- [Unit Testing in .NET: Fast, Focused, and Actually Useful](/posts/testing-unit-testing/)

## References

- [Testcontainers for .NET, official docs](https://dotnet.testcontainers.org/)
- [Integration tests in ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests)
- [EF Core testing, Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/testing/)
- [Respawn on GitHub](https://github.com/jbogard/Respawn)
