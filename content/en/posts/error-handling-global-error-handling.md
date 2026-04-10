---
title: "Global Error Handling in ASP.NET Core"
date: 2026-04-10
draft: false
tags: ["error-handling", "dotnet", "middleware"]
categories: ["Error Handling"]
series: ["Error Handling"]
description: "Every exception that escapes a controller or handler must land somewhere. ASP.NET Core gives you IExceptionHandler, ProblemDetails, and the status code pages middleware. Here is how to wire them into a single, predictable error pipeline."
---

Unhandled exceptions in an ASP.NET Core application do not vanish. If nothing catches them, the framework returns a blank 500 response with no body, which is correct from a security standpoint and useless from a debugging standpoint. Clients get no error code to report, logs get a stack trace with no structured context, and the operations team cannot tell whether the 500 was a transient database timeout or a permanent logic bug. Global error handling is the layer that turns every unhandled exception into a structured response for the client and a structured log entry for the team, in one place, without scattering try-catch blocks across every controller.

This article covers the three mechanisms ASP.NET Core provides for this: the `IExceptionHandler` interface (the modern path since .NET 8), the `ProblemDetails` service, and the status code pages middleware. You do not need all three at once, but you need to understand how they compose.

## Why a single global handler

Before ASP.NET Core 8, the standard approach was exception-handling middleware: either the built-in `UseExceptionHandler()` with a lambda, or a custom middleware that wrapped `await _next(context)` in a try-catch. Both worked, but both required you to write the full pipeline yourself: check the exception type, pick the status code, serialize the response body, log the error, and handle edge cases like response already started.

`IExceptionHandler`, introduced in .NET 8, provides a structured hook into the built-in exception handler. You implement the interface, register it in DI, and ASP.NET Core calls it when an unhandled exception reaches the pipeline. Multiple handlers can be registered and are called in order until one marks the exception as handled. This replaces the lambda-based approach with testable, composable classes.

The goal is always the same: exactly one place in the codebase decides what to return when things go wrong.

## Overview: the error pipeline

{{< mermaid >}}
flowchart LR
    A[Request] --> B[Middleware pipeline]
    B --> C[Endpoint / Controller]
    C -->|Exception thrown| D[UseExceptionHandler]
    D --> E[IExceptionHandler 1:<br/>DomainExceptionHandler]
    E -->|Handled| F[ProblemDetails 4xx response]
    E -->|Not handled| G[IExceptionHandler 2:<br/>FallbackExceptionHandler]
    G --> H[ProblemDetails 500 response]
{{< /mermaid >}}

The exception bubbles up from the endpoint, hits `UseExceptionHandler()`, which invokes each registered `IExceptionHandler` in order. The first one that returns `true` ends the chain. If none handle it, the built-in handler returns the default response.

## Zoom: IExceptionHandler for domain exceptions

```csharp
public sealed class DomainExceptionHandler : IExceptionHandler
{
    private readonly ILogger<DomainExceptionHandler> _logger;

    public DomainExceptionHandler(ILogger<DomainExceptionHandler> logger) => _logger = logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is not DomainException domainException)
            return false;

        _logger.LogWarning(exception, "Domain error: {Message}", domainException.Message);

        var statusCode = domainException switch
        {
            OrderNotFoundException => StatusCodes.Status404NotFound,
            OrderAlreadyShippedException => StatusCodes.Status409Conflict,
            _ => StatusCodes.Status422UnprocessableEntity
        };

        httpContext.Response.StatusCode = statusCode;
        await httpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = statusCode,
            Title = domainException.GetType().Name,
            Detail = domainException.Message,
            Instance = httpContext.Request.Path
        }, cancellationToken);

        return true;
    }
}
```

This handler only catches [custom exceptions](/en/posts/error-handling-custom-exceptions/) that derive from `DomainException`. It maps each type to an HTTP status code, writes a `ProblemDetails` response (RFC 9457), and returns `true` to signal "handled". For any other exception type it returns `false`, and the next handler in the chain takes over.

> 💡 **Info** — `ProblemDetails` is the RFC 9457 standard for machine-readable error responses in HTTP APIs. ASP.NET Core has built-in support via `Microsoft.AspNetCore.Http.ProblemDetails` and `Results.Problem()`.

> ✅ **Good practice** — Log domain exceptions at `Warning`, not `Error`. They are expected conditions (not found, conflict, validation failure), not crashes. Reserve `Error` for exceptions that indicate an actual bug or infrastructure failure.

## Zoom: a fallback handler for everything else

```csharp
public sealed class FallbackExceptionHandler : IExceptionHandler
{
    private readonly ILogger<FallbackExceptionHandler> _logger;
    private readonly IHostEnvironment _env;

    public FallbackExceptionHandler(ILogger<FallbackExceptionHandler> logger, IHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "Unhandled exception on {Method} {Path}",
            httpContext.Request.Method, httpContext.Request.Path);

        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;

        var problem = new ProblemDetails
        {
            Status = 500,
            Title = "Internal Server Error",
            Instance = httpContext.Request.Path
        };

        if (_env.IsDevelopment())
            problem.Detail = exception.ToString();

        await httpContext.Response.WriteAsJsonAsync(problem, cancellationToken);
        return true;
    }
}
```

This handler catches everything the domain handler did not. It logs at `Error`, returns a generic 500, and only includes the exception detail in development. In production, the response body never exposes the stack trace.

> ❌ **Never do** — Do not include stack traces, connection strings, or internal type names in production error responses. A `ProblemDetails` body with `Detail = exception.ToString()` is a security leak. Attackers use stack traces to map your internal architecture.

## Zoom: wiring it all together

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
builder.Services.AddExceptionHandler<FallbackExceptionHandler>();

var app = builder.Build();

app.UseExceptionHandler();
app.UseStatusCodePages();

app.MapControllers();
app.Run();
```

The registration order matters: `DomainExceptionHandler` is called first. If it returns `false`, `FallbackExceptionHandler` runs. `AddProblemDetails()` registers the `IProblemDetailsService` that other parts of the framework use to generate consistent `ProblemDetails` responses (including model validation errors from controllers).

`UseStatusCodePages()` handles the case where a middleware sets a 4xx/5xx status code without writing a body. Without it, a `return NotFound()` from a controller returns `404` with no JSON body. With it, the middleware generates a `ProblemDetails` body for the status code.

> 💡 **Info** — `AddProblemDetails()` is available since .NET 7. It wires up the `IProblemDetailsService` so that framework-generated errors (model validation, authorization failures, not-found routes) also return RFC 9457 responses instead of plain text.

## Zoom: what about exception filters?

MVC exception filters (`IExceptionFilter`, `IAsyncExceptionFilter`) predate the middleware approach and still work in ASP.NET Core. They run inside the MVC pipeline, after model binding and action execution. They do not cover exceptions thrown in middleware, in Minimal API handlers, or in the request pipeline before MVC kicks in.

For a new project, `IExceptionHandler` is the right choice. It sits in the middleware pipeline, covers every exception regardless of source, and is testable through DI. Exception filters remain useful if you have an existing MVC codebase with filter-based error handling and no reason to migrate.

> ⚠️ **Works, but...** — Mixing exception filters and `IExceptionHandler` creates two error-handling paths that developers have to remember. If you adopt `IExceptionHandler`, remove the exception filters to avoid confusion about which one runs when.

## Zoom: structured logging in the error handler

The error handler is the single place where every unhandled exception passes through. That makes it the right place to attach structured log properties:

```csharp
using (LogContext.PushProperty("TraceId", httpContext.TraceIdentifier))
using (LogContext.PushProperty("UserId", httpContext.User.FindFirst("sub")?.Value))
{
    _logger.LogError(exception, "Unhandled exception on {Method} {Path}",
        httpContext.Request.Method, httpContext.Request.Path);
}
```

`TraceId` and `UserId` end up in the structured log entry, which means the operations team can search for "all errors for user X in the last hour" or "all errors with this trace ID". This is the minimum context for a useful incident investigation.

## Wrap-up

Global error handling in ASP.NET Core is three pieces: `IExceptionHandler` implementations registered in order of specificity, `AddProblemDetails()` for consistent RFC 9457 responses, and `UseStatusCodePages()` for empty-body status codes. Domain exceptions map to 4xx with a typed handler; everything else maps to 500 with a fallback handler. The response never leaks internals in production, and the structured log carries enough context to diagnose without reproducing.

The next article in this series covers the alternative path: returning errors as values instead of throwing them.

## Related articles

- [Custom Exceptions in .NET](/en/posts/error-handling-custom-exceptions/)
- [Application Layer: CQS and CQRS](/en/posts/layer-focused-cqs-cqrs/)

## References

- [Microsoft: Handle errors in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)
- [Microsoft: IExceptionHandler](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.diagnostics.iexceptionhandler)
- [Microsoft: Using ProblemDetails](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling#problem-details)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
