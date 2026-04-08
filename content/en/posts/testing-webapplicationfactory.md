---
title: "API Testing with WebApplicationFactory in ASP.NET Core"
date: 2026-04-08
draft: false
tags: ["testing", "api", "webapplicationfactory", "dotnet"]
categories: ["Testing"]
series: ["Testing"]
description: "Test your real ASP.NET Core pipeline end to end, in-process, without Kestrel or a real HTTP port. WebApplicationFactory sits between unit tests and full E2E, and for most API testing needs, it is where the best return on effort lives."
---

Between [unit tests](/posts/testing-unit-testing/) and full browser-driven [end-to-end tests](/posts/testing-e2e-playwright/) sits a very productive middle layer: tests that spin up your *real* ASP.NET Core pipeline (routing, model binding, middleware, filters, DI, authentication) in the same process as the test, and drive it through an in-memory `HttpClient`. No Kestrel, no socket, no browser. Just your app, running for real, in milliseconds.

`WebApplicationFactory<TEntryPoint>` was introduced in ASP.NET Core 2.1 in 2018 as part of the `Microsoft.AspNetCore.Mvc.Testing` package. It replaced a decade of hand-rolled solutions (`TestServer` directly, custom host builders, startup tricks) with one clean primitive. With .NET 6's minimal APIs and the top-level `Program.cs`, the story got even simpler. If you have read the previous article on [integration testing with TestContainers](/posts/testing-integration-testing-testcontainers/), you already have the database story. This article covers the HTTP story on top of it.

## Why this pattern exists

Picture a team whose API has 40 endpoints. They have great unit tests for handlers and repositories, but the bugs they keep finding in production are:

- A filter that accidentally skipped authorization on one route.
- A model binder that silently coerced a null enum to `0`.
- A Problem Details response that changed shape after a middleware upgrade.
- A route collision between `/orders/{id}` and `/orders/export`.

None of these bugs live in a single class. They live in the *pipeline*: the interaction between routing, filters, DI, and serialization. Unit tests cannot see them. Running the app in CI and curling it works but is slow and fragile. What the team actually needs:

1. **The real pipeline**, not a simulation, so routing and filters behave as they will in production.
2. **Fast startup**, so a test run covers 200 endpoints in under a minute.
3. **Service override hooks**, so one test can swap a dependency (the payment gateway, the clock) without touching production code.

`WebApplicationFactory` gives you all three.

## Overview: how it plugs in

{{< mermaid >}}
graph TD
    A[Test] --> B[WebApplicationFactory&lt;Program&gt;]
    B --> C[TestServer<br/>in-memory]
    C --> D[Your Program.cs<br/>DI, middleware, endpoints]
    D --> E[HttpClient]
    A --> E
    D --> F[(Postgres from TestContainers)]
{{< /mermaid >}}

The factory boots your `Program.cs` with a `TestServer` instead of Kestrel. The `HttpClient` it hands back talks to the pipeline directly, skipping the network. Everything you care about (routing, filters, auth, serialization) runs for real.

> 💡 **Info** : `WebApplicationFactory<TEntryPoint>` uses a type argument that points at any type in your startup assembly. The convention is `WebApplicationFactory<Program>`. If you use top-level statements, you need to add `public partial class Program { }` at the bottom of `Program.cs` so the test project can reference the type.

## Zoom: the minimum test

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using System.Net.Http.Json;
using Xunit;

public class OrderEndpointsTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrderEndpointsTests(WebApplicationFactory<Program> factory)
        => _client = factory.CreateClient();

    [Fact]
    public async Task GET_orders_returns_200_with_list()
    {
        var response = await _client.GetAsync("/orders");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var orders = await response.Content.ReadFromJsonAsync<List<OrderDto>>();
        orders.Should().NotBeNull();
    }
}
```

Five lines of setup, the rest is an actual HTTP assertion. `factory.CreateClient()` returns an `HttpClient` pre-wired to the test server. No ports, no hostname, no real socket.

> ✅ **Good practice** : Assert on status codes and response bodies, not on internal state. A good API test should be replaceable with a curl command that proves the same behavior. Internal coupling makes tests brittle.

## Zoom: overriding services

The most valuable feature of `WebApplicationFactory` is `ConfigureWebHost`, where you can replace any service registered in production. Stripe payment gateway? Swap for a fake. System clock? Inject a `FakeTimeProvider`.

```csharp
public class TestAppFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<IPaymentGateway>();
            services.AddSingleton<IPaymentGateway, FakePaymentGateway>();

            services.RemoveAll<TimeProvider>();
            services.AddSingleton<TimeProvider>(new FakeTimeProvider(
                DateTimeOffset.Parse("2026-04-08T12:00:00Z")));
        });
    }
}

public sealed class FakePaymentGateway : IPaymentGateway
{
    public Task<ChargeResult> ChargeAsync(CustomerId c, Money m, CancellationToken ct)
        => Task.FromResult(new ChargeResult(Success: true));
}
```

Then consume it in tests:

```csharp
public class SubmitOrderTests : IClassFixture<TestAppFactory>
{
    private readonly TestAppFactory _factory;
    public SubmitOrderTests(TestAppFactory factory) => _factory = factory;

    [Fact]
    public async Task POST_submit_charges_and_returns_204()
    {
        var client = _factory.CreateClient();
        var response = await client.PostAsync($"/orders/{Guid.NewGuid()}/submit", null);

        response.StatusCode.Should().Be(HttpStatusCode.NoContent);
    }
}
```

> 💡 **Info** : `services.RemoveAll<T>()` comes from `Microsoft.Extensions.DependencyInjection.Extensions`. It is the idiomatic way to override a registration instead of appending a second one.

> ❌ **Never do this** : Do not use `Mock.Setup(...)` to fake behavior inside `ConfigureServices`. Mocks belong in unit tests. For integration tests, a small hand-written `Fake*` class is easier to read and survives refactors better.

## Zoom: combining with TestContainers

The real power is combining `WebApplicationFactory` with a Postgres container. Your tests drive the real pipeline against the real database. This is where 80% of the bugs actually hide.

```csharp
public class ApiWithDbFixture : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine").Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<DbContextOptions<ShopDbContext>>();
            services.AddDbContext<ShopDbContext>(o =>
                o.UseNpgsql(_db.GetConnectionString()));
        });
    }

    public async ValueTask InitializeAsync()
    {
        await _db.StartAsync();
        using var scope = Services.CreateScope();
        var ctx = scope.ServiceProvider.GetRequiredService<ShopDbContext>();
        await ctx.Database.MigrateAsync();
    }

    public new async ValueTask DisposeAsync()
    {
        await _db.DisposeAsync();
        await base.DisposeAsync();
    }
}
```

One fixture. Real pipeline. Real database. Tests that POST a command, commit a row, and verify the GET endpoint returns it. If an EF Core mapping is wrong, a filter forgets to run, or a middleware mangles the response body, this kind of test catches it.

> ✅ **Good practice** : Seed your test data through the API whenever possible, not by inserting rows into the database directly. Tests that do `POST /orders` then `GET /orders/{id}` prove the whole flow works end to end. Tests that bypass the API prove only the pieces you remembered to exercise.

## Zoom: authentication in tests

Real APIs are protected. You have two clean options:

**1. Test authentication handler** : register a fake scheme that authenticates every request as a test user.

```csharp
services.AddAuthentication("Test")
    .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });

services.PostConfigure<AuthenticationOptions>(o =>
{
    o.DefaultAuthenticateScheme = "Test";
    o.DefaultChallengeScheme = "Test";
});
```

`TestAuthHandler` just builds a `ClaimsPrincipal` from a configured test user. Simple, fast, deterministic.

**2. Real JWT flow** : have the test call your actual `/token` endpoint with a test account, grab the token, attach it to the `HttpClient`. Slower but proves the auth flow works too.

> ⚠️ **It works, but...** : Option 1 is fine for most tests, but keep at least one test per auth-protected route that goes through option 2. Otherwise, the day your real JWT setup breaks, no test will notice.

## When not to use it

`WebApplicationFactory` is effective, but it remains in-process. If your production system depends on behavior that only shows up with real network, multiple processes, or a real reverse proxy (sticky sessions, SignalR over WebSockets through Nginx, client certificate auth), you will need a full E2E setup on top. That is the topic of the next article.

## Wrap-up

You now know how to drive your real ASP.NET Core pipeline from tests: create a `WebApplicationFactory<Program>`, use `ConfigureWebHost` to swap production dependencies for fakes, combine it with a Postgres container for full integration coverage, and add a test authentication handler so your protected routes are reachable. You can write tests that post a command and verify state via a second request, proving the whole pipeline from routing to persistence works end to end.

Ready to level up your next project or share it with your team? See you in the next one, End-to-End testing with Playwright is where we go next.

## Related articles

- [Unit Testing in .NET: Fast, Focused, and Actually Useful](/posts/testing-unit-testing/)
- [Integration Testing with TestContainers for .NET](/posts/testing-integration-testing-testcontainers/)

## References

- [Integration tests in ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests)
- [WebApplicationFactory&lt;TEntryPoint&gt;, API docs](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.testing.webapplicationfactory-1)
- [Test authentication in ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests#mock-authentication)
