---
title: "End-to-End Testing with Playwright for .NET"
date: 2026-04-08
draft: false
tags: ["testing", "e2e", "playwright", "dotnet"]
categories: ["Testing"]
series: ["Testing"]
description: "Full browser-driven end-to-end tests in .NET with Playwright: reliable selectors, parallel execution, auto-wait, and the discipline that keeps the suite green."
---

End-to-end tests have a bad reputation earned the hard way. A decade of flaky Selenium suites, implicit waits that never quite wait long enough, XPath selectors that break every UI refresh, and CI runs that fail "sometimes" convinced many teams that E2E was not worth it. They were right about Selenium. They were wrong about E2E.

Playwright changed the equation. Microsoft released it in 2020 as a modern successor to Puppeteer, and the .NET bindings followed in early 2021. It bundles Chromium, Firefox, and WebKit, auto-waits for elements to be actionable, isolates each test in a fresh browser context, and ships with a code generator that records your actions into a test file. If you have read the previous articles on [unit testing](/posts/testing-unit-testing/), [integration testing with TestContainers](/posts/testing-integration-testing-testcontainers/), and [API testing with WebApplicationFactory](/posts/testing-webapplicationfactory/), you already have the fast, cheap, in-process layers. Playwright is the top of the pyramid: slower, but the only thing that proves your app actually works the way a user will use it.

## Why this pattern exists

Picture a team that ships a checkout flow. Unit tests are green. API tests are green. A manual QA pass reveals that the "Pay" button is disabled when the form has validation errors, so users never see the error message the API returns. A second pass reveals a race condition where the cart total updates after the submit handler fires, so users pay the old price. Neither bug is catchable at any layer below the browser. Both ship.

What the team actually needs:

1. **A real browser** rendering the real page, so DOM events, CSS, and JavaScript all behave exactly as a user sees them.
2. **Reliable selectors** that survive minor UI changes, so one div reorder does not break 40 tests.
3. **Fast parallel execution** and isolation, so the suite runs in minutes, not hours, and one flaky test does not poison the rest.

Playwright delivers all three.

## Overview: the pieces

{{< mermaid >}}
graph TD
    A[Playwright test] --> B[Microsoft.Playwright.NUnit<br/>or MSTest / xUnit wrapper]
    B --> C[Browser<br/>Chromium / Firefox / WebKit]
    C --> D[Your running app<br/>Kestrel on localhost:5000]
    D --> E[(Postgres from TestContainers)]
    A --> F[Page Object<br/>CheckoutPage]
    F --> C
{{< /mermaid >}}

The test drives a real browser. The browser talks to your running app, which in turn talks to a real database. The Page Object pattern keeps selectors in one place so UI refactors touch one file, not the whole suite.

> 💡 **Info** : Playwright for .NET ships with its own test runner via the `Microsoft.Playwright.NUnit` / `Microsoft.Playwright.MSTest` packages. These give you parallel execution, fresh browser contexts per test, and trace recording out of the box. You can also use raw `PlaywrightSharp` inside xUnit, but the NUnit adapter is more mature.

## Zoom: installing and the first test

Install the package and the browsers in one step:

```bash
dotnet add package Microsoft.Playwright.NUnit
dotnet build
pwsh bin/Debug/net10.0/playwright.ps1 install
```

The `install` script downloads Chromium, Firefox, and WebKit (roughly 500 MB). Commit the invocation to your CI script so agents get the browsers on first run.

```csharp
using Microsoft.Playwright;
using Microsoft.Playwright.NUnit;
using NUnit.Framework;

[Parallelizable(ParallelScope.Self)]
public class CheckoutTests : PageTest
{
    [Test]
    public async Task User_can_submit_an_order_from_the_cart()
    {
        await Page.GotoAsync("http://localhost:5000");
        await Page.GetByRole(AriaRole.Link, new() { Name = "Catalog" }).ClickAsync();
        await Page.GetByRole(AriaRole.Button, new() { Name = "Add to cart" }).First.ClickAsync();
        await Page.GetByRole(AriaRole.Link, new() { Name = "Cart" }).ClickAsync();
        await Page.GetByRole(AriaRole.Button, new() { Name = "Checkout" }).ClickAsync();

        await Expect(Page.GetByText("Order confirmed")).ToBeVisibleAsync();
    }
}
```

`PageTest` gives you a fresh `Page` per test and auto-disposes everything at the end. No boilerplate.

> ✅ **Good practice** : Prefer `GetByRole`, `GetByLabel`, `GetByPlaceholder`, and `GetByText` over CSS or XPath selectors. They match how users and assistive tech perceive the page, and they survive CSS class renames.

## Zoom: locators and auto-wait

Playwright's killer feature is **auto-wait**. Every action (`ClickAsync`, `FillAsync`) waits for the element to be visible, enabled, and stable before acting. Every assertion (`ToBeVisibleAsync`, `ToHaveTextAsync`) retries until the condition holds or a timeout expires. You almost never write explicit waits.

```csharp
// Playwright waits for the button to exist, be enabled, and be visible.
await Page.GetByRole(AriaRole.Button, new() { Name = "Checkout" }).ClickAsync();

// Playwright retries the assertion for up to 5 seconds by default.
await Expect(Page.GetByTestId("order-total")).ToHaveTextAsync("€199.98");
```

Compare this to the Selenium world where you wrote `Thread.Sleep(2000)` because the element loaded from an API. Those sleeps are gone.

> ❌ **Never do this** : Do not add `Task.Delay` in Playwright tests. If a test fails intermittently, the answer is almost always a better locator (use `getByTestId` on a stable attribute) or a better assertion (let Playwright retry), not a longer sleep.

## Zoom: the Page Object pattern

Keep selectors out of tests. One class per page or component, reused across tests:

```csharp
public sealed class CheckoutPage
{
    private readonly IPage _page;
    public CheckoutPage(IPage page) => _page = page;

    public ILocator CheckoutButton =>
        _page.GetByRole(AriaRole.Button, new() { Name = "Checkout" });

    public ILocator OrderTotal => _page.GetByTestId("order-total");
    public ILocator Confirmation => _page.GetByText("Order confirmed");

    public Task GotoAsync() => _page.GotoAsync("/cart");
    public Task SubmitAsync() => CheckoutButton.ClickAsync();
}

[Test]
public async Task Checkout_shows_confirmation()
{
    var checkout = new CheckoutPage(Page);
    await checkout.GotoAsync();
    await checkout.SubmitAsync();

    await Expect(checkout.Confirmation).ToBeVisibleAsync();
}
```

When the "Checkout" button becomes "Place order" next quarter, you change one line in `CheckoutPage.cs` and 40 tests keep passing.

> 💡 **Info** : Add `data-testid` attributes in your Razor or React components for things that have no natural accessible name. `GetByTestId("cart-line-1-qty")` is rock solid and survives every kind of UI refactor.

## Zoom: hosting the app under test

You have two options for where "the app" runs during the test:

**1. Start it in-process** : use `WebApplicationFactory` (covered in the [previous article](/posts/testing-webapplicationfactory/)) to boot the app on a real Kestrel port inside the test process.

```csharp
public sealed class AppFixture : IDisposable
{
    private readonly WebApplication _app;
    public string BaseUrl { get; }

    public AppFixture()
    {
        var builder = WebApplication.CreateBuilder();
        // ... same services as Program.cs ...
        _app = builder.Build();
        _app.Urls.Add("http://127.0.0.1:0"); // random free port
        _app.StartAsync().GetAwaiter().GetResult();
        BaseUrl = _app.Urls.First();
    }

    public void Dispose() => _app.StopAsync().GetAwaiter().GetResult();
}
```

Pros: no external dependency, test owns the lifecycle. Cons: you need the real Kestrel, not the in-memory `TestServer`, because Playwright drives a real browser that needs a real socket.

**2. Run it as a separate process** : a CI job starts `dotnet run` in the background, waits for the health endpoint, then runs the Playwright suite. More realistic, closer to prod, but more moving parts.

Option 1 is the sweet spot for most teams.

> ⚠️ **It works, but...** : Pairing Playwright with TestContainers for the database works, and it is the most honest E2E you can get without a full staging environment. The trade-off is startup time: a cold run (pulling Postgres and browser binaries) can take 30 seconds. A warm run is fast.

## Zoom: traces, videos, and debugging

When a test fails in CI, Playwright's trace viewer is worth its weight in gold. Enable it for failing tests only:

```csharp
[SetUp]
public async Task BeforeEach()
{
    await Context.Tracing.StartAsync(new() { Screenshots = true, Snapshots = true });
}

[TearDown]
public async Task AfterEach()
{
    var failed = TestContext.CurrentContext.Result.Outcome != ResultState.Success;
    await Context.Tracing.StopAsync(new()
    {
        Path = failed ? $"traces/{TestContext.CurrentContext.Test.Name}.zip" : null
    });
}
```

Upload the `traces/` folder as a CI artifact. Open the zip with `pwsh bin/Debug/net10.0/playwright.ps1 show-trace trace.zip` and you see the exact DOM snapshot at each action, with network calls and console logs.

> ✅ **Good practice** : Keep the E2E suite small and high-value. Ten tests that cover the critical flows (sign up, add to cart, checkout, view order history, cancel) are worth a hundred tests that click around low-value pages. Flakiness scales with suite size.

## When not to use Playwright

E2E is the slowest, most expensive layer of your test pyramid. Use it for:

- **User-visible critical paths** that must not break.
- **Flows that touch multiple systems** (auth + API + database + UI).
- **Regression after a big refactor** of the frontend or the deployment topology.

Do *not* use it for:

- **Business rules**: those belong in unit tests.
- **API contracts**: those belong in `WebApplicationFactory` tests.
- **Database behavior**: those belong in TestContainers integration tests.

Follow the pyramid: lots of unit tests, fewer integration tests, very few E2E tests. The ratio keeps the suite fast and trustworthy.

## Wrap-up

You now know how to write end-to-end tests that actually survive: install Playwright and its browsers, use role and label-based locators so your tests match how users and accessibility tools see the page, lean on auto-wait instead of manual sleeps, organize selectors through the Page Object pattern, host the app under test with `WebApplicationFactory` on a real Kestrel port, and capture traces for failed runs in CI. You can keep the suite tight, focused on critical paths, and run it in minutes rather than hours.

Ready to level up your next project or share it with your team? See you in the next one, Architecture Testing is where we go next.

## Related articles

- [Unit Testing in .NET: Fast, Focused, and Actually Useful](/posts/testing-unit-testing/)
- [Integration Testing with TestContainers for .NET](/posts/testing-integration-testing-testcontainers/)
- [API Testing with WebApplicationFactory in ASP.NET Core](/posts/testing-webapplicationfactory/)

## References

- [Playwright for .NET, official docs](https://playwright.dev/dotnet/)
- [Playwright locators guide](https://playwright.dev/dotnet/docs/locators)
- [Playwright auto-waiting](https://playwright.dev/dotnet/docs/actionability)
- [Playwright trace viewer](https://playwright.dev/dotnet/docs/trace-viewer)
