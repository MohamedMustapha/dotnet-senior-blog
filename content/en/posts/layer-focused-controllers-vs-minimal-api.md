---
title: "Endpoints in .NET: Controllers vs Minimal API, the Honest Comparison"
date: 2026-04-09
draft: false
tags: ["endpoints", "controllers", "minimal-api", "dotnet"]
categories: ["Architecture"]
series: ["Layer Focused"]
description: "Controllers or Minimal APIs? Both ship HTTP endpoints in ASP.NET Core, both are fully supported, and the defaults each one gives you are different. Here is what actually matters when you pick one over the other."
---

Every ASP.NET Core project starts with the same decision. Do you scaffold a `Controllers/` folder with `[ApiController]` classes, or do you map endpoints in `Program.cs` with `app.MapPost(...)` and a lambda? The two styles do the same job, they both live in the same framework, and both are fully supported long term. The differences show up in how much ceremony they impose, how they compose filters and middleware, and how they scale with team size and codebase maturity.

This article walks through both styles head to head, with realistic code, and explains when each one earns its place.

## Why this choice exists at all

Controllers have been in ASP.NET since MVC 1 in 2009. Web API controllers arrived in 2012, and the two were unified under ASP.NET Core's single controller model in 2016. For a decade, if you wrote an HTTP endpoint in .NET, you wrote a controller. The model binds, the filters compose, the attributes describe metadata, and the conventions are deep.

Minimal APIs landed in .NET 6 in November 2021 as a response to a very specific observation: for small services and simple endpoints, the ceremony of a controller class, a base class, routing attributes, and an action method is a lot of typing for one line of real work. The team led by Damian Edwards and David Fowler started from the question "what is the absolute minimum code to map a URL to a handler?" and built from there.

For the first year, Minimal APIs were visibly missing pieces: no endpoint filters, no typed results, weak OpenAPI, no support for `[FromServices]` shortcuts. .NET 7 and .NET 8 closed most of those gaps. By .NET 9, the gap is narrow enough that the choice is genuinely a taste and architecture decision, not a capability one.

## Overview: the two mental models

Before the code, here is how each style lays out the same endpoint in your head.

{{< mermaid >}}
graph LR
    subgraph Controllers
        direction TB
        C1[Program.cs<br/>AddControllers] --> C2[OrdersController]
        C2 --> C3[Action method]
        C3 --> C4[Filter pipeline]
        C4 --> C5[Handler logic]
    end
    subgraph Minimal["Minimal API"]
        direction TB
        M1[Program.cs] --> M2[MapPost lambda]
        M2 --> M3[Endpoint filters]
        M3 --> M4[Handler logic]
    end
{{< /mermaid >}}

Same request, same response, same middleware, same dependency injection. The difference is where the endpoint is declared and how metadata is attached to it. Controllers lean on attributes and conventions. Minimal APIs lean on fluent chaining.

## Zoom: the same endpoint, written both ways

Let us implement a classic endpoint, `POST /orders`, in both styles, with validation, authorization, typed results, and OpenAPI metadata. This is what you would actually ship to production.

### Controller version

```csharp
// Controllers/OrdersController.cs
[ApiController]
[Route("orders")]
[Authorize]
public sealed class OrdersController : ControllerBase
{
    private readonly ISender _mediator;

    public OrdersController(ISender mediator) => _mediator = mediator;

    [HttpPost]
    [ProducesResponseType(typeof(CreateOrderResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<CreateOrderResponse>> Create(
        [FromBody] CreateOrderCommand command,
        CancellationToken ct)
    {
        var response = await _mediator.Send(command, ct);
        return CreatedAtAction(nameof(GetById), new { id = response.OrderId }, response);
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(OrderDetailsResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDetailsResponse>> GetById(
        Guid id, CancellationToken ct)
    {
        var response = await _mediator.Send(new GetOrderDetailsQuery(id), ct);
        return Ok(response);
    }
}
```

Wired in `Program.cs` with two lines:

```csharp
builder.Services.AddControllers();
// ...
app.MapControllers();
```

### Minimal API version

```csharp
// Features/Orders/OrdersEndpoints.cs
public static class OrdersEndpoints
{
    public static IEndpointRouteBuilder MapOrders(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapPost("/", async (
                CreateOrderCommand command,
                ISender mediator,
                CancellationToken ct) =>
            {
                var response = await mediator.Send(command, ct);
                return TypedResults.Created($"/orders/{response.OrderId}", response);
            })
            .WithName("CreateOrder")
            .Produces<CreateOrderResponse>(StatusCodes.Status201Created)
            .ProducesValidationProblem();

        group.MapGet("/{id:guid}", async Task<Results<Ok<OrderDetailsResponse>, NotFound>> (
                Guid id,
                ISender mediator,
                CancellationToken ct) =>
            {
                var response = await mediator.Send(new GetOrderDetailsQuery(id), ct);
                return TypedResults.Ok(response);
            })
            .WithName("GetOrderById");

        return app;
    }
}
```

Wired in `Program.cs` with one line:

```csharp
app.MapOrders();
```

Two styles, same behaviour, same routing table, same OpenAPI document. The controller version is more declarative (attributes do the talking), the Minimal version is more explicit (the chain tells you exactly what applies to what).

> 💡 **Info** — `TypedResults` (the typed variant of `Results`) is the recommended return type in Minimal APIs since .NET 7. It lets you express the response in the signature, which both improves testability and gives the OpenAPI generator enough information to describe each branch without extra attributes.

## Zoom: filters, the moment both styles diverge

Cross-cutting concerns (logging, validation, tenant resolution, idempotency) are where the two models feel most different.

Controllers use **action filters**, which have existed since MVC 1 and form a rich pipeline: authorization filters, resource filters, action filters, exception filters, result filters. They can short-circuit the pipeline, mutate the action arguments, and wrap the result. They are powerful and well understood.

```csharp
public sealed class IdempotencyFilter : IAsyncActionFilter
{
    private readonly IIdempotencyStore _store;

    public IdempotencyFilter(IIdempotencyStore store) => _store = store;

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var key = context.HttpContext.Request.Headers["Idempotency-Key"].ToString();
        if (string.IsNullOrWhiteSpace(key))
        {
            context.Result = new BadRequestObjectResult("Idempotency-Key header required.");
            return;
        }

        if (await _store.HasSeenAsync(key))
        {
            context.Result = new StatusCodeResult(StatusCodes.Status409Conflict);
            return;
        }

        await next();
        await _store.MarkAsync(key);
    }
}

// Usage
[ServiceFilter(typeof(IdempotencyFilter))]
[HttpPost]
public async Task<ActionResult<CreateOrderResponse>> Create(...) { }
```

Minimal APIs use **endpoint filters**, introduced in .NET 7. They are simpler (one interface, one delegate), they compose by chaining `.AddEndpointFilter(...)`, and they run after model binding but before the handler.

```csharp
public sealed class IdempotencyFilter : IEndpointFilter
{
    private readonly IIdempotencyStore _store;

    public IdempotencyFilter(IIdempotencyStore store) => _store = store;

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var key = ctx.HttpContext.Request.Headers["Idempotency-Key"].ToString();
        if (string.IsNullOrWhiteSpace(key))
            return TypedResults.BadRequest("Idempotency-Key header required.");

        if (await _store.HasSeenAsync(key))
            return TypedResults.Conflict();

        var result = await next(ctx);
        await _store.MarkAsync(key);
        return result;
    }
}

// Usage
group.MapPost("/", Handler)
    .AddEndpointFilter<IdempotencyFilter>();
```

The endpoint filter is smaller, but it also does less. There is no separation between authorization, resource, action, and exception stages. For a big app with dozens of cross-cutting rules at different stages, the richer controller pipeline can be an advantage. For most apps, a handful of endpoint filters is simpler to reason about.

> ✅ **Good practice** — In Minimal APIs, attach filters at the **route group** level, not the individual endpoint. One `.AddEndpointFilter<IdempotencyFilter>()` on the `/orders` group applies it to every order endpoint, which is both less noisy and less error-prone than remembering to add it to each `MapPost`.

## Zoom: model binding and validation

Controllers have a long history with model binding. `[FromBody]`, `[FromQuery]`, `[FromRoute]`, `[FromForm]`, `[FromHeader]`, plus implicit binding based on type and source. Combined with `[ApiController]`, you get automatic 400 responses on invalid models and automatic `ProblemDetails` formatting.

Minimal APIs bind by convention: a parameter of a complex type is read from the body (unless decorated with `[FromServices]`), primitives are read from the route or query, `IFormFile` comes from the form, services are resolved from DI automatically. You can still use the `[From...]` attributes when the convention is ambiguous.

Validation is where both styles leave a gap. Neither integrates with `FluentValidation` out of the box. The canonical answer in both worlds today is to push validation into a **MediatR pipeline behavior**, or into an **endpoint filter** for Minimal APIs. That way the validator lives with the command, and the endpoint definition stays clean.

> ⚠️ **It works, but...** — Data annotations (`[Required]`, `[StringLength]`) technically work in Minimal APIs via `ValidationFilter` or similar middleware, but they leak validation rules into your request records and do not compose well with conditional rules. For anything beyond toy validation, use FluentValidation in a filter.

## Zoom: testing, OpenAPI, and AOT

For **unit testing**, controllers have a small advantage: an action method is just a method on a class. You new up the controller, call the method, assert on the `ActionResult`. Minimal API handlers are usually lambdas inside `MapPost`, which makes them harder to test directly. The fix is to extract the handler into a named static method or a MediatR handler, which is a good practice either way.

For **integration testing**, both styles work identically with `WebApplicationFactory`. The HTTP surface is what you test, and it does not care how the endpoints were declared.

For **OpenAPI**, .NET 9 introduced the new `Microsoft.AspNetCore.OpenApi` package which replaces Swashbuckle as the default. It works equally well with both models. Minimal APIs get slightly richer metadata for free when you use `TypedResults`, because the result type is part of the signature.

For **AOT (Ahead of Time) compilation**, Minimal APIs are the recommended path. The ASP.NET Core team has invested heavily in making `RequestDelegateFactory` source-generated and trim-safe. Controllers rely on more reflection at startup and have a harder time becoming AOT-friendly.

> 💡 **Info** — For the deep dive on why AOT matters and what it buys you in .NET, see [AOT Compilation in .NET](/en/posts/performance-aot-compilation/).

## The honest decision matrix

Neither style is objectively better. Here is how I choose in practice:

**Pick Minimal APIs when:**

- The project is a small or medium service, a microservice, a function-like workload.
- You want AOT compilation, fast cold starts, or a small container image.
- You use a feature-organized codebase (each feature maps its own endpoints) and want the endpoint declaration to sit next to the handler.
- Your team prefers explicit wiring over convention-based wiring.

**Pick Controllers when:**

- The app has dozens of endpoints sharing rich filter pipelines and metadata attributes.
- The team has deep MVC conventions already (model binding, view-model mapping, complex filter ordering).
- You need the full action filter taxonomy (authorization, resource, action, result, exception) with short-circuit behaviour between stages.
- You rely on tooling or libraries that still expect `ControllerBase` (some older analyzers, legacy scaffolders, a few third-party extensions).

**Pick "both in the same project" when:**

- A greenfield area ships with Minimal APIs, a legacy area keeps its controllers. They coexist peacefully. `AddControllers()` and `MapControllers()` do not interfere with `MapPost(...)`. The only rule is to be consistent within a feature, not within the whole solution.

> ❌ **Never do this** — Do not copy-paste entire controllers into Minimal API lambdas or vice versa as a "migration". A style change is not a feature, it adds risk for no user-visible value. Migrate a module when you are already touching it for another reason, and leave the rest alone. Consistency inside a module matters more than consistency across the whole codebase.

## Wrap-up

You now know how Controllers and Minimal APIs actually compare in modern ASP.NET Core: controllers give you a rich filter pipeline, deep conventions, and a decade of tooling; Minimal APIs give you less ceremony, first-class AOT support, typed results, and endpoint filters that are simpler to compose. You can write the same endpoint in both styles and ship the same HTTP surface, and you can mix the two in one project when it makes sense. Pick the one that fits the shape of your codebase, not the one that is newer or older.

Ready to level up your next project or share it with your team? See you in the next one, a++ 👋

## References

- [Minimal APIs overview, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/overview)
- [Choose between controller-based APIs and Minimal APIs, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/apis)
- [Filters in Minimal API apps, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/min-api-filters)
- [Handle errors in Minimal APIs, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/handle-errors)
- [OpenAPI support in ASP.NET Core API apps, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/openapi/overview)
- [Native AOT deployment, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/native-aot)
