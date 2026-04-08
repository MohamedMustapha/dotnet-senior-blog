---
title: "Integration Testing with TestContainers for .NET"
date: 2026-04-08
draft: false
tags: ["testing", "integration", "testcontainers", "dotnet"]
categories: ["Testing"]
series: ["Testing"]
description: "Real databases, real Redis, real brokers, real Keycloak, provisioned per test run. TestContainers for .NET turns fake integration tests into genuine ones, without shared infrastructure."
---

Integration testing has long been the most compromised discipline in .NET delivery. Not because engineers did not care, but because the available tools forced a choice between fragile shared infrastructure and tests that quietly stopped being integration tests at all. If you have read the previous article on [Unit Testing in .NET](/posts/testing-unit-testing/), you already know that mocking a `DbContext` cannot catch the bugs that live in the generated SQL. What the industry needed was a way to run integration tests against the real services they claim to integrate with, reproducibly, without coordinating a shared environment.

TestContainers provides exactly that. The original Java library was released in 2015 by Richard North, and the .NET port landed in 2017 as Testcontainers for .NET. It is now the reference standard, maintained under the `testcontainers` GitHub organization, and .NET 10 treats it as a first-class integration testing tool. The principle is straightforward: your test code starts a real Postgres, Redis, RabbitMQ, Keycloak, or any other service inside an ephemeral Docker container, waits for it to become ready, exposes its connection details, and tears it down when the test fixture disposes.

## Why this pattern exists

For most of the history of .NET, writing an honest integration test required accepting five structural problems, none of which had a clean solution.

**1. A shared dev or integration infrastructure.** The database, the identity provider, the message broker, the object store: all of them lived on a central environment that every developer and every CI job pointed to. Running two test suites in parallel was a genuine risk: one engineer's fixture data would collide with another's, a cleanup script would wipe a row someone else depended on, and a flaky test would suddenly look like a real regression. Teams defended themselves with locking schemes, naming conventions, and implicit social contracts that broke the moment a new joiner arrived.

**2. CI/CD required network access to these shared services.** The build agents needed routes to the dev database, credentials rotated by hand, and firewall rules maintained by another team. Every new pipeline meant a ticket. Every shared-infrastructure outage blocked every build. The test suite was only as available as the least reliable service it talked to.

**3. The setup was extraordinarily easy to break.** A single `ALTER TABLE` applied by one engineer during debugging, a role change in Keycloak, an SSL certificate expiration on the SMTP relay, a stale Redis snapshot: any of these would silently invalidate the test suite for everyone. Mornings began with the question "is CI red because of my change, or because someone touched the test environment?".

**4. It required continuous cleanup and maintenance from the developers themselves.** Seed scripts drifted out of sync with migrations. Test users accumulated in the identity provider. Orphaned rows piled up in join tables. Someone on the team ended up being the unofficial custodian of the integration environment, and that person's time was never accounted for in sprint planning.

**5. And most importantly: any dependency that did not have an in-memory NuGet package was mocked.** This is the most damaging consequence, and the one nobody wants to admit. If your service talked to SQL Server, you had `Microsoft.EntityFrameworkCore.InMemory` and pretended that counted, even though it silently ignores foreign keys, case sensitivity, and every SQL-specific feature. If it talked to Keycloak, you mocked `IAuthenticationService`. If it talked to MinIO, you mocked `IAmazonS3`. If it talked to RabbitMQ, you mocked `IBus`. The suites were labelled "integration tests" and were, in practice, **fake integration tests**: they exercised your code against a fiction you had written yourself. The day the real dependency behaved differently, the tests were silent.

TestContainers dismantles all five points at once. It replaces shared infrastructure with per-run containers, removes the CI dependency on external services (all the agent needs is Docker), makes setup reproducible from code instead of from a wiki page, moves cleanup from "developer discipline" to "container disposal", and, crucially, removes the last excuse for mocking a dependency that has a Docker image: Postgres with `pg_trgm`, Keycloak with a full realm, MinIO for S3, RabbitMQ, Kafka, Mongo, Elasticsearch. If the tool has an image, you test against the real thing.

The rest of this article is about how to do that cleanly.

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

> ✅ **Good practice** : Pin the image tag (`postgres:17-alpine`, not `postgres:latest`). Reproducibility is the point. An unpinned `latest` that shifts under you silently invalidates every run that preceded it.

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

> ❌ **Never do this** : Do not point your integration tests at a shared dev database. Two engineers running the suite concurrently will corrupt each other's state, and the failure will look like a flaky test instead of a shared-resource contention. TestContainers removes the underlying cause entirely.

## Zoom: the scenarios you could not test before

This is where TestContainers shows its full value. Three concrete examples of things that were effectively impossible (or cost you a week of YAML) before, and that now fit in a fixture.

### Postgres-specific behavior: fuzzy search with pg_trgm

You have a search endpoint that finds customers by approximate name using the `pg_trgm` extension. No mock can reproduce the ranking of `similarity()`. The only way to test it is against real Postgres.

```csharp
public sealed class SearchFixture : IAsyncLifetime
{
    public PostgreSqlContainer Postgres { get; } = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine")
        .WithDatabase("search_test")
        .Build();

    public async ValueTask InitializeAsync()
    {
        await Postgres.StartAsync();
        await using var db = CreateDbContext();
        await db.Database.MigrateAsync();
        // Enable the extension, create the GIN index, seed the test data.
        await db.Database.ExecuteSqlRawAsync("CREATE EXTENSION IF NOT EXISTS pg_trgm");
        await db.Database.ExecuteSqlRawAsync(
            "CREATE INDEX idx_customers_name_trgm ON customers USING gin (name gin_trgm_ops)");
        db.Customers.AddRange(
            new Customer("Jean Dupont"),
            new Customer("Jeanne Dupond"),
            new Customer("John Doe"));
        await db.SaveChangesAsync();
    }

    public ShopDbContext CreateDbContext() => new(new DbContextOptionsBuilder<ShopDbContext>()
        .UseNpgsql(Postgres.GetConnectionString()).Options);

    public ValueTask DisposeAsync() => Postgres.DisposeAsync();
}

[Fact]
public async Task Search_returns_fuzzy_matches_ranked_by_similarity()
{
    await using var db = _fixture.CreateDbContext();
    var repo = new CustomerRepository(db);

    var hits = await repo.SearchAsync("Jen Dupon", limit: 5);

    hits.Should().HaveCountGreaterThan(0);
    hits[0].Name.Should().BeOneOf("Jean Dupont", "Jeanne Dupond");
}
```

The test proves the extension is installed, the index is used, and the SQL you wrote ranks the results the way a real user expects. A mock of the repository would validate none of this, because the behavior under test lives inside Postgres, not inside your C# code.

### Keycloak with a real realm, users, roles, and clients

Role-based authorization is notoriously annoying to test. "Does `/admin/users` reject a non-admin?" used to require a shared Keycloak, a hand-curated realm, and a convention nobody documented. With TestContainers you import a realm JSON at container startup and you get the whole thing: users, passwords, roles, clients, client scopes, mappers.

```csharp
public sealed class KeycloakFixture : IAsyncLifetime
{
    public IContainer Keycloak { get; } = new ContainerBuilder()
        .WithImage("quay.io/keycloak/keycloak:26.0")
        .WithPortBinding(8080, true)
        .WithEnvironment("KC_BOOTSTRAP_ADMIN_USERNAME", "admin")
        .WithEnvironment("KC_BOOTSTRAP_ADMIN_PASSWORD", "admin")
        .WithResourceMapping(
            new FileInfo("test-realm.json"),
            "/opt/keycloak/data/import/test-realm.json")
        .WithCommand("start-dev", "--import-realm")
        .WithWaitStrategy(Wait.ForUnixContainer()
            .UntilHttpRequestIsSucceeded(r => r.ForPath("/realms/test").ForPort(8080)))
        .Build();

    public string BaseUrl =>
        $"http://{Keycloak.Hostname}:{Keycloak.GetMappedPublicPort(8080)}";

    public ValueTask InitializeAsync() => new(Keycloak.StartAsync());
    public ValueTask DisposeAsync() => Keycloak.DisposeAsync();
}
```

`test-realm.json` lives next to the fixture. It contains `alice` (role `admin`), `bob` (role `user`), a confidential client, scopes, everything your production realm has, pinned as test data. Every run gets a clean Keycloak with the exact same state.

```csharp
[Fact]
public async Task Admin_endpoint_rejects_non_admin_user()
{
    var token = await GetTokenAsync("bob", "bob-password"); // plain user
    _client.DefaultRequestHeaders.Authorization = new("Bearer", token);

    var response = await _client.GetAsync("/admin/users");

    response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
}
```

The test goes through real Keycloak, a real JWT, the real ASP.NET Core authorization pipeline, against the real policy. Nothing is mocked. When your role mapping changes in production, this test tells you before the deploy.

### MinIO for S3-compatible storage

Your code uses `AmazonS3Client` to upload invoices, generate presigned URLs, and set bucket policies. You want to verify the presigned URL actually downloads the file and expires when it should.

```csharp
public sealed class MinioFixture : IAsyncLifetime
{
    public IContainer Minio { get; } = new ContainerBuilder()
        .WithImage("minio/minio:latest")
        .WithPortBinding(9000, true)
        .WithEnvironment("MINIO_ROOT_USER", "minioadmin")
        .WithEnvironment("MINIO_ROOT_PASSWORD", "minioadmin")
        .WithCommand("server", "/data")
        .WithWaitStrategy(Wait.ForUnixContainer().UntilPortIsAvailable(9000))
        .Build();

    public AmazonS3Client CreateClient() => new(
        new BasicAWSCredentials("minioadmin", "minioadmin"),
        new AmazonS3Config
        {
            ServiceURL = $"http://{Minio.Hostname}:{Minio.GetMappedPublicPort(9000)}",
            ForcePathStyle = true,
        });

    public ValueTask InitializeAsync() => new(Minio.StartAsync());
    public ValueTask DisposeAsync() => Minio.DisposeAsync();
}
```

From here you test real multipart uploads, real presigned URLs, real expiry behavior. The exact same client code runs in production against AWS S3, and in tests against MinIO, because both speak the S3 protocol.

> 💡 **Info** : The pattern generalizes. If a tool has an official Docker image, you can drive it from a fixture: RabbitMQ, Kafka, Mongo, Elasticsearch, Vault, Mailhog. The `Testcontainers.*` NuGet packages provide pre-built builders for the common ones, and `ContainerBuilder` handles everything else.

## Zoom: composing multiple services

Real apps need more than one dependency. Postgres plus Keycloak plus MinIO plus Redis is a common shape. Compose them in one fixture and start them in parallel:

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
