---
title: "The Result Pattern"
date: 2026-04-10
draft: false
tags: ["error-handling", "dotnet", "result-pattern"]
categories: ["Error Handling"]
series: ["Error Handling"]
description: "Exceptions for expected failures is control flow in disguise. The Result pattern makes success and failure explicit return values that the compiler can check, without the performance cost and the stack unwinding of throw/catch."
---

Exceptions are the right tool for unexpected failures: a database connection that drops, an out-of-memory condition, a null reference. They are the wrong tool for expected failures: a validation error, a business rule violation, a resource that does not exist. The distinction matters because exceptions unwind the stack, allocate a stack trace, and break the normal control flow. When you throw an `OrderNotFoundException` and catch it three frames up to return a 404, you are using the most expensive control flow mechanism in .NET to do something a return value could do for free.

The Result pattern makes the outcome of an operation explicit: the method returns either a success value or a failure value, and the caller must handle both. It is not a new idea: Rust has `Result<T, E>`, Haskell has `Either`, Go returns `(value, error)`. In .NET, the pattern takes the form of a generic struct that carries either the success payload or a typed error, and the caller uses pattern matching to branch on the outcome.

This article walks through why the Result pattern exists in the .NET ecosystem, how to implement it without a library, and the trade-offs between results and exceptions.

## Why the Result pattern exists

The .NET community has debated exceptions-for-flow-control for over a decade. The Framework Design Guidelines say "do not use exceptions for normal flow of control", and then every ASP.NET tutorial throws `NotFoundException` in a handler and catches it in middleware to return 404. The contradiction comes from the fact that .NET did not have a standard result type, so exceptions were the only way to communicate failures across method boundaries without `out` parameters or tuples.

The Result pattern fills that gap. It appeared in the .NET ecosystem around 2017-2018 through libraries like `FluentResults` and `ErrorOr`, and gained traction in CQRS-based architectures where command and query handlers needed to return "this worked, here is the data" or "this failed, here is why" without throwing. MediatR pipelines, in particular, made the pattern popular because behaviors (pipeline steps) could inspect and short-circuit results without a try-catch.

The performance argument is real but secondary. The primary argument is clarity: when a method returns `Result<Order>`, the caller knows at compile time that the operation can fail, and the compiler forces them to handle the failure path. When a method returns `Order` and might throw, the caller has to read the documentation (if it exists) to know which exceptions to catch.

## Overview: the shape of a Result

{{< mermaid >}}
flowchart TD
    A[Handler returns Result&lt;T&gt;] --> B{Success?}
    B -->|Yes| C[Value: T]
    B -->|No| D[Error: typed error object]
    C --> E[Controller returns 200 + body]
    D --> F{Error type}
    F -->|NotFound| G[Controller returns 404]
    F -->|Validation| H[Controller returns 422]
    F -->|Conflict| I[Controller returns 409]
{{< /mermaid >}}

The handler returns a result. The controller pattern-matches on it. No exceptions, no middleware, no global handler needed for expected failures.

## Zoom: a minimal Result type

You do not need a library. A Result type that covers 90% of cases is under 50 lines:

```csharp
public readonly struct Result<TValue>
{
    private readonly TValue? _value;
    private readonly Error? _error;

    private Result(TValue value) { _value = value; IsSuccess = true; }
    private Result(Error error) { _error = error; IsSuccess = false; }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;

    public TValue Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException("Cannot access Value on a failed result.");

    public Error Error => IsFailure
        ? _error!
        : throw new InvalidOperationException("Cannot access Error on a successful result.");

    public static implicit operator Result<TValue>(TValue value) => new(value);
    public static implicit operator Result<TValue>(Error error) => new(error);
}

public sealed record Error(string Code, string Message)
{
    public static readonly Error None = new(string.Empty, string.Empty);
    public static Error NotFound(string message) => new("NotFound", message);
    public static Error Validation(string message) => new("Validation", message);
    public static Error Conflict(string message) => new("Conflict", message);
}
```

The implicit conversions are what make it ergonomic. A handler can `return order;` for success or `return Error.NotFound("Order not found");` for failure, without constructing the `Result` explicitly.

> 💡 **Info** — Using a `struct` rather than a `class` avoids a heap allocation per result. On a hot path that returns thousands of results per second, this matters. On a typical web request that returns one result, it does not, but there is no reason to pay for a heap allocation when the struct works.

## Zoom: using the Result in a handler

```csharp
public sealed class GetOrderHandler : IRequestHandler<GetOrderQuery, Result<OrderDto>>
{
    private readonly ShopDbContext _db;

    public GetOrderHandler(ShopDbContext db) => _db = db;

    public async Task<Result<OrderDto>> Handle(GetOrderQuery query, CancellationToken ct)
    {
        var order = await _db.Orders
            .AsNoTracking()
            .Where(o => o.Id == query.OrderId)
            .Select(o => new OrderDto(o.Id, o.Reference, o.Status.ToString(), o.TotalAmount))
            .FirstOrDefaultAsync(ct);

        if (order is null)
            return Error.NotFound($"Order {query.OrderId} was not found.");

        return order;
    }
}
```

The handler never throws for a missing order. It returns a failure result. The calling code, whether it is a controller, a Minimal API endpoint, or another handler, decides what to do with it.

## Zoom: mapping Result to HTTP in a controller

```csharp
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly ISender _sender;

    public OrdersController(ISender sender) => _sender = sender;

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> Get(Guid id, CancellationToken ct)
    {
        var result = await _sender.Send(new GetOrderQuery(id), ct);

        if (result.IsFailure)
        {
            return result.Error.Code switch
            {
                "NotFound" => NotFound(ToProblem(result.Error, 404)),
                "Validation" => UnprocessableEntity(ToProblem(result.Error, 422)),
                "Conflict" => Conflict(ToProblem(result.Error, 409)),
                _ => StatusCode(500, ToProblem(result.Error, 500))
            };
        }

        return Ok(result.Value);
    }

    private static ProblemDetails ToProblem(Error error, int status) => new()
    {
        Status = status,
        Title = error.Code,
        Detail = error.Message
    };
}
```

The mapping from error code to HTTP status is explicit and local to the API layer. The domain does not know what a 404 is. The controller does not know what the business rule is. Each layer does its job.

> ✅ **Good practice** — If the mapping from `Error.Code` to HTTP status repeats across many controllers, extract it into an extension method or a base controller method. But do not put it in the handler, because the handler might be called from a background job where HTTP status codes have no meaning.

## Zoom: mapping Result in a Minimal API endpoint

```csharp
app.MapGet("api/orders/{id:guid}", async (Guid id, ISender sender, CancellationToken ct) =>
{
    var result = await sender.Send(new GetOrderQuery(id), ct);

    return result.IsSuccess
        ? Results.Ok(result.Value)
        : result.Error.Code switch
        {
            "NotFound" => Results.NotFound(result.Error),
            "Validation" => Results.UnprocessableEntity(result.Error),
            "Conflict" => Results.Conflict(result.Error),
            _ => Results.StatusCode(500)
        };
});
```

Same pattern, same clarity. The Minimal API endpoint is a thin adapter between HTTP and the application layer.

## Zoom: validation errors as a list

A single `Error` works for "order not found". Validation failures are typically a list of field-level errors. Extend the `Error` record to carry them:

```csharp
public sealed record ValidationError : Error
{
    public ValidationError(IReadOnlyList<FieldError> fields)
        : base("Validation", "One or more validation errors occurred.")
    {
        Fields = fields;
    }

    public IReadOnlyList<FieldError> Fields { get; }
}

public sealed record FieldError(string Field, string Message);
```

The handler returns a `ValidationError` with the field list, and the controller maps it to a 422 with a body that lists every field:

```csharp
public async Task<Result<OrderDto>> Handle(CreateOrderCommand command, CancellationToken ct)
{
    var errors = new List<FieldError>();

    if (string.IsNullOrWhiteSpace(command.Reference))
        errors.Add(new FieldError("Reference", "Reference is required."));

    if (command.Lines.Count == 0)
        errors.Add(new FieldError("Lines", "At least one line is required."));

    if (errors.Count > 0)
        return new ValidationError(errors);

    // ... create order
}
```

> ⚠️ **Works, but...** — For simple CRUD validation, FluentValidation with a MediatR pipeline behavior catches validation before the handler runs, which keeps the handler clean. The Result-based validation shown above is more appropriate when the validation depends on data the handler loads from the database.

## Zoom: when to keep throwing exceptions

The Result pattern does not replace exceptions everywhere. It replaces them for expected, domain-level failures. Three cases where exceptions remain the right tool:

1. **Infrastructure failures.** A dead database, a broken network, a disk full. These are unexpected and unrecoverable by the immediate caller. Throwing lets them propagate to the [global error handler](/en/posts/error-handling-global-error-handling/).
2. **Programming errors.** `ArgumentNullException`, `InvalidOperationException` for violated preconditions. These signal a bug, not a runtime condition.
3. **Third-party library boundaries.** If the library throws, catching and wrapping into a Result at the adapter boundary is fine. Trying to prevent the library from throwing is not.

The rule of thumb: if the caller can reasonably recover, return a Result. If the caller cannot, throw an exception.

> ❌ **Never do** — Do not wrap every exception into a Result at every layer. `catch (Exception ex) => Result.Failure(ex.Message)` hides the stack trace, swallows the exception type, and makes debugging harder than the problem it was trying to solve.

## Wrap-up

The Result pattern makes success and failure explicit return values. The handler returns `Result<T>`, the caller pattern-matches, and the compiler enforces that the failure path is handled. It does not replace exceptions for infrastructure crashes or programming errors, but it eliminates the throw/catch overhead and the ambiguity for every expected domain failure. Combined with the [global error handler](/en/posts/error-handling-global-error-handling/) for unexpected failures, it gives you a complete error strategy with clean separation.

The next article ties everything together: where to throw, where to return a result, and where to log.

## Related articles

- [Custom Exceptions in .NET](/en/posts/error-handling-custom-exceptions/)
- [Global Error Handling in ASP.NET Core](/en/posts/error-handling-global-error-handling/)
- [Application Layer: CQS and CQRS](/en/posts/layer-focused-cqs-cqrs/)

## References

- [Microsoft: Exception design guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/exceptions)
- [FluentResults on GitHub](https://github.com/altmann/FluentResults)
- [ErrorOr on GitHub](https://github.com/amantinband/error-or)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
