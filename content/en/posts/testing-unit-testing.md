---
title: "Unit Testing in .NET: Fast, Focused, and Actually Useful"
date: 2026-04-08
draft: false
tags: ["testing", "dotnet", "xunit"]
categories: ["Testing"]
series: ["Testing"]
description: "Unit tests that catch real bugs, run in milliseconds, and survive refactors. xUnit, the AAA pattern, mocking trade-offs, and what to never test."
---

Unit tests are the cheapest test you can write and the first ones that rot when nobody maintains them. A test suite full of tests that break every time you rename a variable, that mock everything into meaninglessness, and that take four seconds to run a single assertion brings less value than having no tests at all. The goal of this article is to help you write the other kind: fast, focused, and the kind you actually rely on during a refactor.

The .NET unit testing story is mature. xUnit.net was started by James Newkirk in 2007 after he co-created NUnit, as a rewrite that removed a decade of accumulated habits. It became the default test framework in ASP.NET Core templates around 2016, and .NET 10 ships with xUnit v3 as the current major version. Around it, FluentAssertions (for readable asserts), NSubstitute or Moq (for mocks), and Bogus (for test data) make up the standard toolkit.

## Why unit tests exist

Unit tests solve four concrete problems that no other layer of the test pyramid addresses as efficiently.

**1. They protect against regression over time.** This is the primary reason they exist. A team ships a pricing engine in month one. In month two, a volume discount tier is added. In month four, a loyalty multiplier interacts with it. In month nine, a new joiner refactors a helper and unknowingly breaks the interaction between tiers and loyalty. Without unit tests, that regression reaches production, and the team discovers it from a customer complaint. With unit tests, the change never merges: the test that pinned the month-two behavior fails in under a second, on the laptop of the person who made the change.

**2. They detect god classes and god methods early.** A method that is hard to unit-test is almost never a testing problem, it is a design problem. When a single test requires fifteen mocks, four pages of arrange code, and a dozen assertions to cover one call, the test is telling you that the method under test is doing too many things at once. The correct response is not to write the giant test. It is to split the method. Unit tests act as an early warning system for god classes and god methods, long before they show up in a code-quality report.

**3. They test the logic of methods, and nothing else.** Unit tests are the right tool when the question is "does this piece of logic compute the right result for a given input". Database queries, HTTP pipelines, middleware, serialization, authentication: those are not logic of a method, they are behaviors of infrastructure. They belong in integration tests, API tests, or E2E tests, not here. Keeping the scope to pure logic is what makes unit tests fast, stable, and trustworthy.

**4. Cherry on top for domain-driven designs.** When the business logic lives inside a well-designed, non-anemic aggregate (that is, an entity that enforces its own invariants instead of exposing public setters for a service to manipulate), unit tests become exceptionally clean. The aggregate contains the rules, the tests contain the scenarios, and there is nothing else to wire up. This is the strongest argument for keeping logic inside the domain instead of scattering it across services, mappers, and validators. A dedicated article on DDD and aggregates will go deeper into this point.

## What should be tested

A reasonable default for every method that contains real logic: **the happy path, the edge cases, and the failure cases**. Those three categories cover almost every bug worth catching.

- **Happy path**: the normal, successful execution with valid inputs. One test per method, minimum.
- **Edge cases**: boundaries where behavior flips. Quantity of 0, 1, exactly the discount threshold, an empty collection, a null optional field, the maximum allowed value, the first day of a month, a leap year.
- **Failure cases**: what happens when an invariant is violated. A negative quantity, an already-submitted order being submitted again, a refund that exceeds the original amount. The test asserts that the right exception (or `Result.Failure`) comes back, not a half-corrupted state.

On top of that baseline, two more rules earn their place in any serious team.

**Every production bug should add one test.** When a bug is found in production, the fix is incomplete until a test exists that would have caught it. This is the only durable way to make regression protection accumulate. A test suite that grows one test per incident becomes, over years, a map of everything that has ever gone wrong, and the team inherits that knowledge for free.

**Guard rails and authorization deserve their own tests.** Defensive programming is not complete until it is verified. For every role-sensitive operation, write the pair: "as an admin, the action is allowed" and "as a regular user, the action is denied". Same for tenant isolation, ownership checks, and rate limits. These are the rules that get silently broken during a refactor and discovered during an audit.

**For CRUD-heavy applications**, the same categorization still applies, but the focus shifts:

- **Write operations** with business logic: validation rules, cross-field invariants, state transitions. Test these at the unit level, with the aggregate or service as the SUT.
- **Read operations** with transforms: projection from entity to DTO, aggregation, computed fields, formatting. Test the transformation itself.
- **Pure pass-through CRUD** (controller to repository to database, no logic in between) does not need a unit test. It needs an integration test that proves the round trip works.

Unit tests deliver all of the above, but only if they stay scoped correctly. The moment a "unit test" spins up a database, a web host, or the file system, it stops being a unit test and becomes a slow integration test. That is a different tool, with a different job.

## Overview: the pieces

Before the code, here are the tools a .NET unit test suite actually uses in 2026:

{{< mermaid >}}
graph TD
    A[Test project<br/>xUnit v3] --> B[Assertions<br/>FluentAssertions or Shouldly]
    A --> C[Mocks / Fakes<br/>NSubstitute or Moq]
    A --> D[Test data<br/>Bogus, AutoFixture]
    A --> E[SUT<br/>System Under Test]
    B --> E
    C --> E
    D --> E
{{< /mermaid >}}

The **SUT** is the class you are testing. Everything else is scaffolding. Your job is to keep the ratio high: minimal scaffolding, maximum SUT.

## Zoom: the AAA pattern

Every good unit test has three sections: **Arrange**, **Act**, **Assert**. Separated visually, they read like prose.

```csharp
using FluentAssertions;
using Xunit;

public class PriceCalculatorTests
{
    [Fact]
    public void Calculate_applies_volume_discount_above_10_items()
    {
        // Arrange
        var calculator = new PriceCalculator();
        var order = new Order(
            customerId: CustomerId.New(),
            lines: [new OrderLine("SKU-42", quantity: 12, unitPrice: 10m)]);

        // Act
        var total = calculator.Calculate(order);

        // Assert
        total.Amount.Should().Be(108m); // 10% off above 10 items
    }
}
```

Three things make this test good: the name describes the *behavior*, not the method; the arrange is minimal; the assert checks one outcome. If someone changes `PriceCalculator` internals tomorrow, this test still passes as long as the rule holds.

> 💡 **Info** : The `[Fact]` attribute marks a parameterless test. For multiple inputs, use `[Theory]` with `[InlineData]` or `[MemberData]`. It is not syntactic sugar, it is precisely what parameterized tests exist for.

> ✅ **Good practice** : Name tests as `MethodName_state_expectedOutcome` or in plain sentences like `applies_volume_discount_above_10_items`. Your test runner output is documentation for future-you.

## Zoom: Theory for input tables

When a method has multiple input branches, a theory beats ten copy-pasted facts:

```csharp
[Theory]
[InlineData(1,  10.00,   0,  10.00)]   // no discount
[InlineData(10, 10.00,   0, 100.00)]   // threshold exactly
[InlineData(11, 10.00,  10,  99.00)]   // 10% off
[InlineData(50, 10.00,  15, 425.00)]   // 15% tier
public void Calculate_applies_tiered_discount(
    int quantity, decimal unitPrice, int expectedDiscountPct, decimal expectedTotal)
{
    var calculator = new PriceCalculator();
    var order = new Order(
        CustomerId.New(),
        [new OrderLine("SKU-1", quantity, unitPrice)]);

    var total = calculator.Calculate(order);

    total.Amount.Should().Be(expectedTotal);
}
```

One test method, four test cases, four rows in the runner. Adding a new tier is one line.

## Zoom: mocking, carefully

Mocking is the technique most often applied incorrectly in unit testing. The rule is simple: mock **boundaries**, not **behavior**. A boundary is an interface your SUT calls out to (repository, HTTP client, time provider). Everything else should be real.

```csharp
[Fact]
public async Task Submit_charges_customer_and_marks_order_submitted()
{
    // Arrange
    var payments = Substitute.For<IPaymentGateway>();
    payments.ChargeAsync(Arg.Any<CustomerId>(), Arg.Any<Money>(), Arg.Any<CancellationToken>())
        .Returns(new ChargeResult(Success: true));

    var repo = Substitute.For<IOrderRepository>();
    var order = Order.Create(CustomerId.New());
    order.AddLine(new ProductId(1), 2, new Money(50m));
    repo.GetByIdAsync(order.Id, Arg.Any<CancellationToken>()).Returns(order);

    var handler = new SubmitOrderHandler(repo, payments, new FakeUnitOfWork());

    // Act
    var result = await handler.Handle(new SubmitOrderCommand(order.Id.Value), default);

    // Assert
    result.IsSuccess.Should().BeTrue();
    order.Status.Should().Be(OrderStatus.Submitted);
    await payments.Received(1).ChargeAsync(order.CustomerId, order.Total, Arg.Any<CancellationToken>());
}
```

The `Order` domain entity is **real**, not mocked. Only `IPaymentGateway` and `IOrderRepository` are substituted because they talk to the outside world.

> ⚠️ **It works, but...** : If you find yourself mocking your own domain classes (`Order`, `Invoice`, `Customer`), step back. Either the class is a boundary in disguise (extract an interface) or the test is testing the mock, not the SUT.

> ❌ **Never do this** : Do not write tests that assert `mock.Received(1).InternalHelperMethod()`. You are pinning the implementation, not the behavior. A refactor that keeps the same public contract will break your tests for no reason.

## Zoom: what not to unit-test

Unit tests are the wrong tool for:

- **Database queries**: an EF Core `Where` expression is not unit-testable in a meaningful way. The bug hides in the generated SQL. Test it with a real database using integration tests.
- **HTTP pipeline, middleware, filters**: spin up the real pipeline with `WebApplicationFactory` instead.
- **Serialization round-trips**: use an end-to-end assertion on the actual JSON, not a mock.
- **UI rendering**: unit-testing a Blazor component for layout is the wrong level. E2E with Playwright catches the real bugs.

If a test takes more than 50ms to run, it is probably not a unit test. That is fine, it just belongs in a different test project with a different lifecycle.

> ✅ **Good practice** : Split your solution into `MyApp.UnitTests`, `MyApp.IntegrationTests`, and `MyApp.E2ETests`. CI can run unit tests on every commit and the slower suites less often, or in parallel stages.

## Running fast and parallel

xUnit v3 runs test classes in parallel by default. That is great unless your tests share static state (cached singletons, `DateTime.Now`, environment variables). Two rules:

1. **No shared mutable state** between tests. Every test arranges its own world.
2. **Inject a clock** instead of calling `DateTime.UtcNow` directly. In .NET 8+, `TimeProvider` is the canonical abstraction.

```csharp
public sealed class PromotionService(TimeProvider clock)
{
    public bool IsActive(Promotion p) => clock.GetUtcNow() < p.EndsAt;
}

// In tests
var fakeClock = new FakeTimeProvider(DateTimeOffset.Parse("2026-04-08T12:00:00Z"));
var service = new PromotionService(fakeClock);
service.IsActive(new Promotion { EndsAt = DateTimeOffset.Parse("2026-04-09T00:00:00Z") })
    .Should().BeTrue();
```

`FakeTimeProvider` lives in the `Microsoft.Extensions.TimeProvider.Testing` NuGet package. No more `DateTime.UtcNow` in production code.

> 💡 **Info** : `TimeProvider` was introduced in .NET 8. Before that, teams rolled their own `IClock` interface. If you are still on .NET 6/7, keep your own abstraction, the test pattern is identical.

## Wrap-up

You now know how to write unit tests that are genuinely useful: scoped to a single behavior, using the AAA layout, mocking only at boundaries, running in milliseconds, and surviving refactors without rewriting. You can pick xUnit v3 plus FluentAssertions plus NSubstitute as a safe default, use `[Theory]` for input tables, inject `TimeProvider` instead of hitting the system clock, and recognize the cases where a unit test is not the right tool.

Ready to level up your next project or share it with your team? See you in the next one, Integration Testing with TestContainers is where we go next.

## References

- [xUnit.net documentation](https://xunit.net/)
- [Unit testing in .NET, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/testing/)
- [TimeProvider in .NET, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/datetime/timeprovider-overview)
- [NSubstitute documentation](https://nsubstitute.github.io/)
- [FluentAssertions documentation](https://fluentassertions.com/)
