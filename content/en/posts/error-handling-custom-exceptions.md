---
title: "Custom Exceptions in .NET"
date: 2026-04-10
draft: false
tags: ["error-handling", "dotnet", "exceptions"]
categories: ["Error Handling"]
series: ["Error Handling"]
description: "Throwing System.Exception with a string message is not error handling, it is error reporting. Custom exceptions give callers a type to catch, a payload to log, and a contract to rely on."
---

Every .NET codebase has a layer of error handling, even if that layer is `throw new Exception("something went wrong")`. The problem with that default is not that it crashes the application, it is that every caller has to catch `Exception`, parse the message string, and guess what actually happened. Custom exceptions replace guessing with a type system: the caller catches `OrderAlreadyShippedException`, not `Exception`, and the type tells them exactly what went wrong, what data is available, and what they can do about it.

This article walks through when a custom exception earns its place, how to write one that carries useful context, and the patterns that keep a growing exception hierarchy from turning into a maintenance burden.

## Why custom exceptions exist

The .NET runtime already ships hundreds of exception types: `ArgumentNullException`, `InvalidOperationException`, `UnauthorizedAccessException`, `TimeoutException`, and many more. These cover infrastructure failures and programming errors. What they do not cover is your domain: an order that was already shipped, a payment that failed validation, a user that exceeded their rate limit, a document that failed a compliance check. Those scenarios are specific to your application, and the framework cannot anticipate them.

Before custom exceptions, teams handled domain errors in one of three ways: returning magic values (`-1`, `null`, `false`), throwing `InvalidOperationException` with a message string, or returning a tuple of `(bool success, string error)`. All three share the same flaw: the caller cannot distinguish errors by type. They have to read a string, check a boolean, or inspect a value, which means every error-handling path is a conditional branch on a weakly-typed signal. Custom exceptions solve this by making the error a type, and types are the one thing the compiler can check.

The practice dates back to .NET 1.0 and the Framework Design Guidelines by Krzysztof Cwalina and Brad Abrams, published in 2005, which established the convention: derive from `Exception` (not `ApplicationException`), include the three standard constructors, and make the exception serializable. The serialization requirement has relaxed since .NET Core dropped binary serialization by default, but the three-constructor convention remains.

## Overview: when to create a custom exception

{{< mermaid >}}
flowchart TD
    A[Error scenario] --> B{Can a framework exception express it?}
    B -->|Yes| C[Use ArgumentNullException,<br/>InvalidOperationException, etc.]
    B -->|No| D{Does a caller need to catch<br/>this specifically?}
    D -->|No| E[Throw InvalidOperationException<br/>with a clear message]
    D -->|Yes| F[Create a custom exception]
    F --> G[Carry domain context as properties]
{{< /mermaid >}}

The decision gate is whether a caller needs to catch this error by type. If the only handler is a global error middleware that logs and returns 500, a custom exception adds ceremony without value. If a command handler, a retry policy, or an API controller needs to react differently to this specific failure, the custom exception is how they know what happened.

## Zoom: the anatomy of a custom exception

```csharp
public sealed class OrderAlreadyShippedException : Exception
{
    public OrderAlreadyShippedException(Guid orderId)
        : base($"Order {orderId} has already been shipped and cannot be modified.")
    {
        OrderId = orderId;
    }

    public OrderAlreadyShippedException(Guid orderId, Exception innerException)
        : base($"Order {orderId} has already been shipped and cannot be modified.", innerException)
    {
        OrderId = orderId;
    }

    public Guid OrderId { get; }
}
```

Three things to note:

1. **Sealed.** Unless you expect derived exception types (you usually do not), sealing the class prevents an uncontrolled hierarchy.
2. **Domain payload.** `OrderId` is a typed property, not buried inside the message string. The caller can log it, return it in a ProblemDetails response, or use it to look up the order without parsing text.
3. **No parameterless constructor.** If an `OrderAlreadyShippedException` must always carry an `OrderId`, the compiler enforces it. There is no way to create one without specifying which order.

> 💡 **Info** — The three-constructor convention (`()`, `(string message)`, `(string message, Exception inner)`) comes from the Framework Design Guidelines. In practice, you only need the constructors that make sense for your domain. A parameterless constructor on `OrderAlreadyShippedException` has no meaning, so do not add it just to follow the convention.

> ✅ **Good practice** — Put domain context in typed properties, not in the message string. The message is for humans reading logs. The properties are for code making decisions.

## Zoom: where to throw

Custom exceptions are thrown from the domain or application layer, where the business rule is enforced:

```csharp
public sealed class ShipOrderHandler : IRequestHandler<ShipOrderCommand>
{
    private readonly ShopDbContext _db;

    public ShipOrderHandler(ShopDbContext db) => _db = db;

    public async Task Handle(ShipOrderCommand command, CancellationToken ct)
    {
        var order = await _db.Orders.FindAsync([command.OrderId], ct)
            ?? throw new OrderNotFoundException(command.OrderId);

        if (order.Status == OrderStatus.Shipped)
            throw new OrderAlreadyShippedException(command.OrderId);

        order.Ship();
        await _db.SaveChangesAsync(ct);
    }
}
```

The handler does not catch these exceptions. It throws them. The catching happens upstream: in the API layer for HTTP responses, in a middleware for logging, or in a retry policy for transient errors. This separation keeps the domain clean of HTTP concerns and the API layer clean of business logic.

> ⚠️ **Works, but...** — Throwing and catching exceptions inside the same method is a code smell. If the method knows the error condition, it can return a result instead of throwing. Exceptions are for crossing layer boundaries, not for control flow within a single method.

## Zoom: a domain exception base class

When several custom exceptions share the same shape (all carry an entity ID, all map to the same HTTP status), a thin base class cuts the repetition:

```csharp
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    protected DomainException(string message, Exception innerException) : base(message, innerException) { }
}

public sealed class OrderNotFoundException : DomainException
{
    public OrderNotFoundException(Guid orderId)
        : base($"Order {orderId} was not found.")
    {
        OrderId = orderId;
    }

    public Guid OrderId { get; }
}

public sealed class OrderAlreadyShippedException : DomainException
{
    public OrderAlreadyShippedException(Guid orderId)
        : base($"Order {orderId} has already been shipped and cannot be modified.")
    {
        OrderId = orderId;
    }

    public Guid OrderId { get; }
}
```

Now the global error handler can catch `DomainException` as a category and map it to a 4xx response, while infrastructure exceptions (`DbException`, `HttpRequestException`) map to 5xx. The hierarchy stays shallow: one abstract base, N sealed leaves. Two levels is the maximum that stays manageable.

> ❌ **Never do** — Do not build a deep exception hierarchy (`DomainException → EntityException → OrderException → OrderAlreadyShippedException`). Each level adds a catch clause somebody might depend on and nobody will maintain. Flat and sealed is the goal.

## Zoom: exceptions for infrastructure boundaries

Not all custom exceptions are domain exceptions. When you wrap an external service, a custom exception encapsulates the failure:

```csharp
public sealed class PaymentGatewayException : Exception
{
    public PaymentGatewayException(string gatewayCode, string gatewayMessage, Exception inner)
        : base($"Payment gateway returned error {gatewayCode}: {gatewayMessage}", inner)
    {
        GatewayCode = gatewayCode;
        GatewayMessage = gatewayMessage;
    }

    public string GatewayCode { get; }
    public string GatewayMessage { get; }
}
```

The `inner` exception is the raw `HttpRequestException` or `JsonException` from the HTTP call. The wrapper gives the caller a type to catch and properties to log, without leaking the implementation detail of which HTTP client or serializer was used.

> ✅ **Good practice** — Always pass the original exception as `innerException`. The stack trace of the root cause is what you need in the incident. Swallowing it makes debugging harder for everyone.

## Zoom: what not to make a custom exception

Not every error needs a dedicated type. Three cases where a framework exception is enough:

1. **Programming errors.** `ArgumentNullException`, `ArgumentOutOfRangeException`, `InvalidOperationException` already communicate "the caller made a mistake". Adding `OrderIdCannotBeEmptyException` just duplicates what `ArgumentException("orderId")` already says.
2. **Errors that nobody catches specifically.** If the only handler is the global middleware, `InvalidOperationException("Order has no lines")` is fine. The middleware logs the message and returns 500 regardless of the type.
3. **Validation errors.** A list of field-level validation failures is not an exception, it is a result. Throwing `ValidationException` with a list of errors and catching it in a filter to return 400 works but leads to control-flow-by-exception. The [Result pattern](/en/posts/error-handling-result-pattern/) is a cleaner fit.

## Wrap-up

Custom exceptions are a communication tool between layers. You create one when a caller needs to catch a specific domain failure, carry typed context for logging and responses, and keep the domain free of infrastructure concerns. You seal them, keep the hierarchy flat, and always attach the inner exception when wrapping external failures. For everything else, the framework exceptions are enough.

The next article covers what happens to those exceptions when they reach the API boundary: global error handling in ASP.NET Core.

## Related articles

- [Application Layer: CQS and CQRS](/en/posts/layer-focused-cqs-cqrs/)

## References

- [Microsoft: Exception design guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/exceptions)
- [Microsoft: Creating and throwing exceptions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/exceptions/creating-and-throwing-exceptions)
- [Microsoft: Best practices for exceptions](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions)
