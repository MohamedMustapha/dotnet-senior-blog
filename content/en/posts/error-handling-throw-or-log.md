---
title: "Where to Throw, Where to Log"
date: 2026-04-10
draft: false
tags: ["error-handling", "dotnet", "logging"]
categories: ["Error Handling"]
series: ["Error Handling"]
description: "Throw where you detect. Log where you handle. Never do both at the same spot. A decision framework for when to throw, when to return a Result, when to log, and where each responsibility sits in the architecture."
---

The most common error-handling bug in .NET codebases is not a missing catch or a wrong status code. It is a try-catch that logs the exception and then rethrows it, which produces three identical log entries for the same error as it bubbles through three layers. The second most common bug is a catch that swallows the exception, logs a generic "error occurred", and returns null, which hides the root cause and makes the caller think the operation succeeded. Both bugs come from the same confusion: not knowing which layer is responsible for what.

This article lays out the decision framework: which layer detects the error, which layer handles it, which layer logs it, and what each layer does with it. It builds on the three previous articles in this series, [custom exceptions](/en/posts/error-handling-custom-exceptions/), the [global error handler](/en/posts/error-handling-global-error-handling/), and the [Result pattern](/en/posts/error-handling-result-pattern/), and ties them into a single coherent strategy.

## Why the confusion exists

.NET makes it easy to throw, easy to catch, and easy to log. The three operations look the same from every layer, so without a convention, every developer adds a try-catch wherever they feel uneasy. The result is a codebase where exceptions are caught four times, logged three times, and handled zero times, because nobody was sure whose job it was.

The convention is simple: **throw where you detect, handle where you can act, log where you handle**. If a layer detects an error and cannot do anything useful about it, it lets the exception propagate. If a layer can translate the error into a meaningful response (an HTTP status, a retry, a fallback value), it catches, handles, and logs in one place.

## Overview: the responsibility map

{{< mermaid >}}
graph TD
    A[Domain / Entity] -->|Throws DomainException| B[Application Layer / Handler]
    B -->|Returns Result or lets exception propagate| C[API Layer / Controller]
    C -->|Maps Result to HTTP or lets exception propagate| D[Global Error Handler]
    D -->|Logs and returns ProblemDetails| E[Client]

    style A fill:#f9f,stroke:#333
    style D fill:#ff9,stroke:#333
{{< /mermaid >}}

Two layers have explicit responsibilities:
- **Domain** (pink): detects business rule violations and throws or returns a failure.
- **Global error handler** (yellow): catches everything that nobody else handled, logs it, and returns a safe response.

Everything in between is a relay: it either translates the error into a different form (Result to HTTP, domain exception to Result) or lets it pass through untouched.

## Rule 1: throw where you detect

The method that discovers the violation is the method that throws. It does not log. It does not catch. It throws and lets the caller decide.

```csharp
public void Ship()
{
    if (Status == OrderStatus.Shipped)
        throw new OrderAlreadyShippedException(Id);

    if (Lines.Count == 0)
        throw new InvalidOperationException("Cannot ship an order with no lines.");

    Status = OrderStatus.Shipped;
    ShippedAt = DateTime.UtcNow;
}
```

The entity does not have access to `ILogger`. It does not know whether the caller is an API controller, a background job, or a unit test. Its job is to protect its own invariants. Throwing is how it does that.

> ✅ **Good practice** — Domain methods throw domain exceptions for business rule violations and framework exceptions for programming errors (`ArgumentNullException`, `InvalidOperationException`). They never log.

## Rule 2: handlers translate, they do not catch-and-rethrow

The application layer (the command/query handler) calls domain methods and infrastructure. If it uses the [Result pattern](/en/posts/error-handling-result-pattern/), it catches domain exceptions at the boundary and translates them into Result failures:

```csharp
public async Task<Result<Unit>> Handle(ShipOrderCommand command, CancellationToken ct)
{
    var order = await _db.Orders.FindAsync([command.OrderId], ct);
    if (order is null)
        return Error.NotFound($"Order {command.OrderId} was not found.");

    try
    {
        order.Ship();
    }
    catch (OrderAlreadyShippedException)
    {
        return Error.Conflict($"Order {command.OrderId} has already been shipped.");
    }

    await _db.SaveChangesAsync(ct);
    return Unit.Value;
}
```

This is the only catch in the handler, and it translates a specific domain exception into a Result. It does not catch `Exception`. It does not log. Infrastructure exceptions (`DbException`, `HttpRequestException`) propagate up to the global handler.

If you do not use the Result pattern, the handler does not catch at all. The domain exception propagates to the global error handler, which maps it to a 4xx response.

> ❌ **Never do** — Do not write `catch (Exception ex) { _logger.LogError(ex, "..."); throw; }` in a handler. This produces a log entry at the handler level and another one at the global handler level for the same exception. The global handler is the single logging point for unhandled exceptions.

## Rule 3: infrastructure adapters wrap and rethrow

When calling an external service, the adapter catches the infrastructure exception and wraps it in a [custom exception](/en/posts/error-handling-custom-exceptions/) that makes sense to the application layer:

```csharp
public async Task<PaymentResult> ChargeAsync(Guid orderId, decimal amount, CancellationToken ct)
{
    try
    {
        var response = await _httpClient.PostAsJsonAsync("/charges", new { orderId, amount }, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<PaymentResult>(ct)
            ?? throw new PaymentGatewayException("EMPTY_RESPONSE", "Gateway returned empty body", null!);
    }
    catch (HttpRequestException ex)
    {
        throw new PaymentGatewayException("HTTP_ERROR", ex.Message, ex);
    }
    catch (JsonException ex)
    {
        throw new PaymentGatewayException("PARSE_ERROR", "Failed to parse gateway response", ex);
    }
}
```

The adapter does not log. It wraps the infrastructure exception in a domain-meaningful type and passes the original as `innerException`. The global handler logs the full chain.

> 💡 **Info** — Keeping the original exception as `innerException` preserves the entire stack trace. When the global handler logs `PaymentGatewayException`, the `innerException` chain shows exactly which HTTP call failed, what the response was, and where in the code it happened.

## Rule 4: the global handler is the single logging point

The [global error handler](/en/posts/error-handling-global-error-handling/) is the one place where every unhandled exception is logged. It logs once, with full context, and returns a structured response:

```csharp
public async ValueTask<bool> TryHandleAsync(HttpContext httpContext, Exception exception, CancellationToken ct)
{
    var level = exception is DomainException ? LogLevel.Warning : LogLevel.Error;

    _logger.Log(level, exception, "Exception on {Method} {Path}",
        httpContext.Request.Method, httpContext.Request.Path);

    // ... write ProblemDetails response
    return true;
}
```

Domain exceptions log at `Warning` because they are expected conditions. Infrastructure exceptions log at `Error` because they indicate a real problem. The handler does not re-throw: it is the end of the line.

## Rule 5: log at the point of decision, not at the point of detection

The corollary of "throw where you detect, log where you handle" is that logging happens where a decision is made. Three examples:

**Retry policy decides to retry.** The retry middleware logs at `Warning` because it is handling the error by retrying, not propagating it:

```csharp
retryPolicy.Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)),
        onRetry: (ex, delay, attempt, _) =>
            _logger.LogWarning(ex, "Retry {Attempt} after {Delay}s", attempt, delay.TotalSeconds));
```

**Fallback decides to return a default.** The fallback logs at `Warning` and returns the default value:

```csharp
public async Task<ExchangeRate> GetRateAsync(string currency, CancellationToken ct)
{
    try
    {
        return await _rateService.GetAsync(currency, ct);
    }
    catch (HttpRequestException ex)
    {
        _logger.LogWarning(ex, "Rate service unavailable, using cached rate for {Currency}", currency);
        return _cache.GetLastKnownRate(currency);
    }
}
```

**Background job decides to skip the item.** The job logs at `Warning` and continues with the next item:

```csharp
foreach (var order in pendingOrders)
{
    try
    {
        await ProcessOrderAsync(order, ct);
    }
    catch (Exception ex)
    {
        _logger.LogWarning(ex, "Failed to process order {OrderId}, skipping", order.Id);
    }
}
```

In all three cases, the catch-and-log is at the point where the decision is made. Not earlier, not later.

> ⚠️ **Works, but...** — Catching `Exception` in a background job loop is a pragmatic choice to keep the job running. But log every skip, and monitor the skip rate. If the skip rate grows, the job is hiding a systemic failure behind individual retries.

## The decision matrix

| Layer | Detects error | Throws | Returns Result | Catches | Logs |
|---|---|---|---|---|---|
| Domain entity | Yes | Yes | No | No | No |
| Application handler | Sometimes | Rarely | Yes (if using Result) | Only to translate | No |
| Infrastructure adapter | Yes | Yes (wrapped) | No | Only to wrap | No |
| Retry/fallback policy | No | No | No | Yes | Yes (Warning) |
| API controller | No | No | No | No (maps Result) | No |
| Global error handler | No | No | No | Yes (everything) | Yes (Error or Warning) |

The table has one logger per row. No row has both "Throws: Yes" and "Logs: Yes". That is the convention that stops duplicate log entries.

## Wrap-up

The entire error strategy comes down to one principle: separate detection from handling. The layer that finds the problem throws or returns a failure. The layer that can act on the problem catches, decides what to do, and logs. The global handler is the safety net for everything nobody else handled. When every layer follows this, exceptions propagate cleanly, log entries are unique, and the post-mortem has exactly one stack trace per incident, not three.

## Related articles

- [Custom Exceptions in .NET](/en/posts/error-handling-custom-exceptions/)
- [Global Error Handling in ASP.NET Core](/en/posts/error-handling-global-error-handling/)
- [The Result Pattern](/en/posts/error-handling-result-pattern/)
- [Application Layer: CQS and CQRS](/en/posts/layer-focused-cqs-cqrs/)

## References

- [Microsoft: Best practices for exceptions](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions)
- [Microsoft: Logging in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging)
- [Serilog: Structured Logging](https://serilog.net/)
