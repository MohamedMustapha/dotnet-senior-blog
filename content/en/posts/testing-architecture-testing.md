---
title: "Architecture Testing in .NET: Rules the Compiler Cannot Enforce"
date: 2026-04-08
draft: false
tags: ["testing", "architecture", "dotnet"]
categories: ["Testing"]
series: ["Testing"]
description: "Turn your architecture decisions into executable rules. NetArchTest and ArchUnitNET let you fail the build when someone leaks Infrastructure into Domain, drops an async suffix, or skips a layer."
---

Every .NET codebase has rules that live only in README files, onboarding docs, or the tribal memory of the senior engineer. "Domain never references Infrastructure." "Handlers end with `Handler`." "No `using System.Data;` inside the Application layer." The compiler does not check any of them. Six months later, someone adds a `using Shop.Infrastructure;` inside a domain class because IntelliSense suggested it, the build passes, and the rule silently dies.

Architecture tests turn these rules into executable assertions. They are unit tests over your assembly graph: "assert that no type in `Shop.Domain` depends on `Shop.Infrastructure`", "assert that every handler ends with `Handler`", "assert that every type in `Application.Orders.Commands` implements `IRequest`". The first widely-used library was ArchUnit on the JVM (2017). The .NET port, NetArchTest by Ben Morris, landed in 2018, followed by ArchUnitNET in 2020 with a richer fluent API. Both are mature, open source, and work with any test runner.

If you have followed this series from [unit testing](/posts/testing-unit-testing/) through [E2E with Playwright](/posts/testing-e2e-playwright/), you already have the correctness story covered. Architecture tests cover a different axis: **structural drift over time**.

## Why this pattern exists

Picture a team that adopted [Clean Architecture](/posts/code-structure-clean-architecture/) eighteen months ago. On day one, the rules were written on a whiteboard and everyone knew them. By month three, a new joiner added `[Table("orders")]` to a domain entity because "it was quicker". By month six, a shortcut during an incident added a `using Microsoft.EntityFrameworkCore;` inside `Domain/Orders/Order.cs`. By month twelve, the "Clean Architecture" project looks Clean on the diagram and is layered spaghetti in the code.

None of these changes caused a bug on the day they were merged. They caused slow decay. What the team actually needs:

1. **Executable invariants**: a test that fails the build when a rule is broken, not a line in a review checklist.
2. **One source of truth**: the rule lives in code, next to the tests, readable by every engineer.
3. **Low ceremony**: writing a new rule takes five minutes, not a sprint of yak-shaving.

NetArchTest and ArchUnitNET both deliver.

## Overview: what you can enforce

Architecture tests cover three broad categories:

{{< mermaid >}}
graph TD
    A[Architecture tests] --> B[Dependency rules<br/>who can reference whom]
    A --> C[Naming rules<br/>suffixes, prefixes, namespaces]
    A --> D[Structural rules<br/>sealed, public, abstract, interfaces]
{{< /mermaid >}}

Dependency rules are the most valuable: they protect the shape of your application. Naming and structural rules are cheaper but add up to real consistency across a large codebase.

> 💡 **Info** : Architecture tests run as regular xUnit / NUnit tests. No extra tooling, no SonarQube plugin, no custom Roslyn analyzer unless you want one. The assertions execute in milliseconds against your compiled assemblies.

## Zoom: dependency rules with NetArchTest

NetArchTest has a fluent API that reads like prose. The classic "Domain references nothing" rule:

```csharp
using NetArchTest.Rules;
using Xunit;

public class DomainDependencyTests
{
    [Fact]
    public void Domain_should_not_depend_on_Application_Infrastructure_or_Api()
    {
        var result = Types.InAssembly(typeof(Shop.Domain.Orders.Order).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny(
                "Shop.Application",
                "Shop.Infrastructure",
                "Shop.Api")
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            "Domain must be independent. Offending types: "
            + string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>()));
    }

    [Fact]
    public void Domain_should_not_depend_on_EntityFramework()
    {
        var result = Types.InAssembly(typeof(Shop.Domain.Orders.Order).Assembly)
            .ShouldNot()
            .HaveDependencyOn("Microsoft.EntityFrameworkCore")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }
}
```

The assertion message includes the failing type names when it fires, so the developer who broke the rule knows exactly which file to open.

> ✅ **Good practice** : Add one dependency test per layer, not one giant test that checks everything. When a failure appears, the test name tells you which boundary was crossed.

## Zoom: naming and structural rules

Consistent naming is mostly a code review job, but architecture tests catch the drift:

```csharp
[Fact]
public void Every_IRequestHandler_should_be_named_with_Handler_suffix()
{
    var result = Types.InAssembly(typeof(SubmitOrderHandler).Assembly)
        .That()
        .ImplementInterface(typeof(IRequestHandler<,>))
        .Should()
        .HaveNameEndingWith("Handler")
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}

[Fact]
public void Every_command_should_be_sealed_record()
{
    var result = Types.InAssembly(typeof(SubmitOrderCommand).Assembly)
        .That()
        .ImplementInterface(typeof(IRequest<>))
        .And()
        .ResideInNamespace("Shop.Application")
        .And()
        .HaveNameEndingWith("Command")
        .Should()
        .BeSealed()
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}
```

These rules are cheap, they catch real drift, and they educate new joiners faster than any style guide.

> ⚠️ **It works, but...** : Do not over-constrain. If you write 200 naming tests, every refactor becomes a negotiation. Pick the ten rules that genuinely matter to your team and skip the rest. Taste is part of the job.

## Zoom: ArchUnitNET for richer assertions

For more complex rules, ArchUnitNET has a more expressive API:

```csharp
using ArchUnitNET.Domain;
using ArchUnitNET.Fluent;
using ArchUnitNET.Loader;
using ArchUnitNET.xUnit;
using static ArchUnitNET.Fluent.ArchRuleDefinition;

public class ApplicationArchitectureTests
{
    private static readonly Architecture Architecture =
        new ArchLoader()
            .LoadAssemblies(typeof(SubmitOrderHandler).Assembly,
                            typeof(Order).Assembly)
            .Build();

    [Fact]
    public void Handlers_should_only_be_used_by_the_mediator()
    {
        IObjectProvider<Class> handlers = Classes()
            .That().ImplementInterface(typeof(IRequestHandler<,>))
            .As("Handlers");

        IObjectProvider<IType> mediator = Types()
            .That().ResideInNamespace("MediatR", true)
            .As("MediatR");

        Classes()
            .That().Are(handlers)
            .Should()
            .OnlyBeAccessedBy(mediator)
            .Check(Architecture);
    }
}
```

`OnlyBeAccessedBy` is the kind of rule that is awkward to express by hand and trivial with ArchUnitNET. The rule says "handlers are a private implementation detail, only MediatR should reach them", which is exactly how you want a CQRS codebase to behave.

> 💡 **Info** : ArchUnitNET loads the assembly once into an in-memory model, so every rule runs against the same graph. For a codebase with 50 architecture tests, it is noticeably faster than NetArchTest, which re-walks types per assertion.

## Zoom: what to enforce first

Start with the three rules that protect the most value:

**1. Dependency direction** (Domain → nothing, Application → Domain, Infrastructure → Application + Domain). This is the only rule that stops a Clean Architecture project from collapsing into layered spaghetti.

**2. No framework leakage into the domain**. Domain should not reference `Microsoft.EntityFrameworkCore`, `Microsoft.AspNetCore.*`, `System.Data.*`, `Stripe.*`. A single test with a list of forbidden namespaces.

**3. Naming of shared vocabulary**. If your team says "commands end with `Command`", "queries end with `Query`", "handlers end with `Handler`", enforce it. When the words in the code match the words in the meetings, onboarding accelerates measurably.

Everything else is bonus. Add rules when a real incident taught you the lesson, not preemptively.

> ✅ **Good practice** : Put architecture tests in a dedicated project, `Shop.ArchitectureTests`, that references every other assembly. They run in CI alongside unit tests and fail the build on any violation.

> ❌ **Never do this** : Do not `Skip` an architecture test when someone breaks the rule "to unblock a release". A skipped architecture test is a rule that no longer exists. Either fix the code, or delete the test and openly acknowledge that the rule is gone.

## When architecture tests are the wrong tool

Architecture tests work well for rules about assembly graphs, type names, and type structure. They are **not** the right tool for:

- **Logical correctness**: "the discount is 15% above 50 items". That is a unit test.
- **Runtime behavior**: "the handler commits the transaction". That is an integration test.
- **Performance**: "this query runs in under 100ms". That is a benchmark or a load test.
- **Security**: "this endpoint requires the `admin` role". That is a WebApplicationFactory test.

If you catch yourself writing `Types.InAssembly(...).Should().HaveMethodBody("...")`, stop and write a real test instead.

## Wrap-up

You now know how to turn architectural decisions into executable invariants: pick NetArchTest for simple fluent rules or ArchUnitNET for richer graph assertions, start with dependency direction and framework leakage tests, add naming rules that mirror your team's shared vocabulary, put them all in a dedicated architecture test project, and refuse to skip them under pressure. You can make your codebase resist the slow structural drift that kills every long-lived project.

Ready to level up your next project or share it with your team? See you in the next one, a++ 👋

## Related articles

- [Clean Architecture in .NET: Dependencies Pointing the Right Way](/posts/code-structure-clean-architecture/)
- [Unit Testing in .NET: Fast, Focused, and Actually Useful](/posts/testing-unit-testing/)
- [Integration Testing with TestContainers for .NET](/posts/testing-integration-testing-testcontainers/)
- [API Testing with WebApplicationFactory in ASP.NET Core](/posts/testing-webapplicationfactory/)
- [End-to-End Testing with Playwright for .NET](/posts/testing-e2e-playwright/)

## References

- [NetArchTest on GitHub](https://github.com/BenMorris/NetArchTest)
- [ArchUnitNET on GitHub](https://github.com/TNG/ArchUnitNET)
- [ArchUnit (Java original)](https://www.archunit.org/)
- [Clean Architecture, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures#clean-architecture)
