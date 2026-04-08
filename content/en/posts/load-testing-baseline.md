---
title: "Baseline Load Testing in .NET: Knowing What Normal Looks Like"
date: 2026-04-08
draft: false
tags: ["load-testing", "baseline", "performance", "dotnet"]
categories: ["Testing"]
series: ["Load Testing"]
description: "A baseline test establishes the reference metrics every other load test is compared against. Here is how to run one cleanly in .NET with k6 and NBomber, and what to do with the numbers."
---

The first load test a team should run is almost never the most impressive one. It is the boring one: the system under the traffic it is expected to handle every day, for long enough to produce stable numbers, and nothing more. That is the **baseline**. Without it, every other load test is meaningless, because there is no reference to compare against. A p95 of 300 ms means nothing unless you know whether it is better or worse than last week.

The [overview article](/posts/load-testing-overview/) in this series covered why load testing exists and what metrics matter. This article zooms into the first of the four test types and explains how to actually set up, run, and use a baseline in a .NET project.

## Why baseline tests exist

A team ships a feature. The feature involves an EF Core query that looks innocent. The next deployment goes live. Two weeks later, a customer complains that the dashboard is slow. The team checks Grafana, sees that latency is indeed higher than usual, and asks the only question that matters: "higher than *what*, exactly?". Without a baseline, the answer is "higher than my memory of what it felt like last month", which is not an answer.

Baseline tests solve four concrete problems:

1. **They establish a reference.** A stable number recorded under a known traffic profile, saved with the commit hash and the deployment date. Every subsequent run can be compared against it.
2. **They catch regressions before production notices.** A pull request that doubles the database round trips of the checkout flow will fail its baseline comparison in CI, not at 2 AM on Monday.
3. **They validate the sizing assumptions.** If the baseline p95 is close to the SLO at expected traffic, production has no margin, and the team knows it before the incident.
4. **They anchor every other load test.** Soak, stress, and spike tests are always relative to the baseline. Without it, "the system degraded under stress" is a sentence with no denominator.

## Overview: the shape of a baseline run

{{< mermaid >}}
graph LR
    A[Fresh pre-prod<br/>environment] --> B[Warmup<br/>1-2 min]
    B --> C[Steady-state<br/>5-10 min at<br/>expected RPS]
    C --> D[Metrics capture]
    D --> E[Store reference<br/>with commit hash]
    E --> F[Compare with<br/>previous baseline]
{{< /mermaid >}}

A baseline run has four phases. **Warmup** exists because JIT compilation, cache population, and connection pool priming all distort the first minute of any .NET test. **Steady-state** is where the numbers are actually captured, long enough to average out noise from GC and background jobs. **Capture** produces a structured artifact, not just a log dump. **Storage and comparison** is the part most teams skip and regret.

The traffic profile during steady-state should mirror production as closely as possible. If production does 70% reads, 20% writes, and 10% search, the baseline does the same. A baseline that hits only `GET /orders` is not a baseline, it is a microbenchmark with delusions.

## Zoom: a realistic baseline with k6

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Trend, Counter } from 'k6/metrics';

const checkoutLatency = new Trend('checkout_flow_duration');
const ordersCreated = new Counter('orders_created');

export const options = {
  stages: [
    { duration: '1m', target: 50 },    // warmup ramp
    { duration: '10m', target: 50 },   // steady state
    { duration: '30s', target: 0 },    // cooldown
  ],
  thresholds: {
    'http_req_duration{group:::catalog}': ['p(95)<200'],
    'http_req_duration{group:::checkout}': ['p(95)<500'],
    'http_req_failed': ['rate<0.005'],  // <0.5% error rate
    'checkout_flow_duration': ['p(95)<1200'],
  },
};

const BASE = __ENV.BASE_URL || 'https://shop.preprod.internal';

export default function () {
  // 70% read path
  group('catalog', () => {
    const r = http.get(`${BASE}/api/products?page=1&size=20`);
    check(r, { 'catalog ok': (res) => res.status === 200 });
  });

  // 20% write path: full checkout flow
  if (Math.random() < 0.2) {
    group('checkout', () => {
      const start = Date.now();

      const cart = http.post(`${BASE}/api/cart`, JSON.stringify({
        productId: 'SKU-42',
        quantity: 1,
      }), { headers: { 'Content-Type': 'application/json' } });

      const submit = http.post(`${BASE}/api/orders/${cart.json('id')}/submit`);

      checkoutLatency.add(Date.now() - start);
      if (submit.status === 204) ordersCreated.add(1);
    });
  }

  // 10% search path
  if (Math.random() < 0.1) {
    group('search', () => {
      http.get(`${BASE}/api/search?q=jean`);
    });
  }

  sleep(1);
}
```

Three traffic paths, weighted to match production. A warmup ramp, a 10-minute steady state, and a cooldown. Thresholds that fail the run in CI if any of them break. Custom metrics that track the business transaction (the full checkout flow), not only the individual endpoints. This is what a serious baseline looks like.

> ✅ **Good practice** : Tag requests with `group()` so k6 reports metrics per path. A global p95 that mixes reads and writes is almost always useless. Per-group p95 tells you where the latency lives.

## Zoom: the same baseline with NBomber

For teams that prefer to keep everything in C#:

```csharp
using NBomber.CSharp;
using NBomber.Http;
using NBomber.Http.CSharp;

using var httpClient = new HttpClient { BaseAddress = new Uri("https://shop.preprod.internal") };

var catalogScenario = Scenario.Create("catalog", async context =>
{
    var request = Http.CreateRequest("GET", "/api/products?page=1&size=20")
        .WithHeader("Accept", "application/json");
    return await Http.Send(httpClient, request);
})
.WithWeight(70)
.WithLoadSimulations(
    Simulation.RampingConstant(copies: 50, during: TimeSpan.FromMinutes(1)),
    Simulation.KeepConstant(copies: 50, during: TimeSpan.FromMinutes(10)));

var checkoutScenario = Scenario.Create("checkout", async context =>
{
    var addToCart = Http.CreateRequest("POST", "/api/cart")
        .WithJsonBody(new { productId = "SKU-42", quantity = 1 });
    var cartResponse = await Http.Send(httpClient, addToCart);
    if (!cartResponse.IsError)
    {
        var cartId = cartResponse.Payload.Value.RootElement.GetProperty("id").GetString();
        var submit = Http.CreateRequest("POST", $"/api/orders/{cartId}/submit");
        return await Http.Send(httpClient, submit);
    }
    return cartResponse;
})
.WithWeight(20)
.WithLoadSimulations(
    Simulation.KeepConstant(copies: 50, during: TimeSpan.FromMinutes(10)));

NBomberRunner
    .RegisterScenarios(catalogScenario, checkoutScenario)
    .WithReportFormats(ReportFormat.Html, ReportFormat.Csv, ReportFormat.Md)
    .WithReportFolder("./reports/baseline")
    .Run();
```

Same idea, expressed in C#. The `WithWeight` option lets NBomber distribute virtual users across scenarios in the expected ratio. Reports land in `./reports/baseline/` and can be committed, archived, or pushed to a storage bucket for historical comparison.

## Zoom: what to capture, and where to store it

A baseline is not useful as a pile of CSV files. It is useful as a structured record that can be diffed. At minimum, every baseline run should store:

- **Commit hash and branch** of the application under test
- **Deployment timestamp** and the environment identifier
- **k6 or NBomber version** and the scenario source file hash
- **Per-group metrics**: p50, p95, p99, p99.9 latency; RPS; error rate by status code
- **Runtime signals**: CPU, memory, GC pause times, thread pool queue length, database pool usage
- **Pass / fail status** against the configured thresholds

A simple convention that works well: write a JSON summary to an S3 / blob bucket after each run, keyed by `<env>/<yyyy-mm-dd>/<commit-hash>.json`. A later job diffs the most recent run against the previous one and posts the delta as a comment on the pull request. This turns baseline testing into a living regression signal instead of a one-off exercise.

> 💡 **Info** : k6 supports pushing results directly to Prometheus (`k6 run --out experimental-prometheus-rw`) and to Grafana Cloud. NBomber writes HTML, CSV, and Markdown reports natively and can plug into InfluxDB. Either path is enough to build the historical comparison.

## Zoom: baseline against what, exactly

A question worth asking explicitly: what traffic level does "baseline" mean for your system? Three common definitions, each valid in context:

- **Average daily peak.** The busiest hour of a typical weekday. Safest starting point for most teams, because it matches what the system actually handles on a normal day.
- **Weekly peak.** The traffic at the busiest hour of the busiest day of the week. Useful for systems with predictable weekly patterns (e.g., Monday morning dashboards, Friday evening e-commerce).
- **Target SLO load.** The traffic level the system is contracted to sustain, regardless of whether current production reaches it. Used when the SLO is above current real traffic and the team needs to prove the headroom exists.

Pick one, write it down, and stick to it. Moving the baseline target silently between runs is how teams accidentally ship "improvements" that only look like improvements because the comparison shifted underneath them.

> ❌ **Never do this** : Do not record the baseline from a cold system on a quiet Sunday morning and compare it against a test run on a warm system under normal load. The two are not comparable. Warmup matters, steady state matters, consistency of the reference environment matters. A baseline that moves every run is not a baseline.

## Zoom: when to run it

Three cadences cover most teams:

**Nightly, in CI.** A scheduled job runs the baseline against pre-prod every night, stores the result, and notifies on regression. This is the highest-value automation most teams can add.

**Before every significant release.** Even with nightly runs, a dedicated pre-release run catches the issues that show up on the specific code path of the upcoming version.

**On demand, before merging a performance-sensitive PR.** Teams that practice this have a `dotnet run --project LoadTests.Baseline` or a `k6 run baseline.js` target that a developer can trigger locally against a shared pre-prod, before asking for review.

> ✅ **Good practice** : Store the baseline reference artifact alongside release notes. When a customer reports "it used to be faster", the team can pull the baseline from the last known good release and prove, or disprove, the claim with data.

## Wrap-up

A baseline test is the cheapest load test and the one that pays off fastest. Running it gives the team a reference point against which every subsequent change, deployment, and [soak / stress / spike test](/posts/load-testing-overview/) can be compared. You can set one up in k6 or NBomber in an afternoon, tag the traffic by business path so per-group metrics reflect real user flows, store structured artifacts with commit hashes, and schedule nightly runs against pre-prod to catch regression before production does.

Ready to level up your next project or share it with your team? See you in the next one, Soak Testing is where we go next.

## Related articles

- [Load Testing for .NET: An Overview of the Four Types That Matter](/posts/load-testing-overview/)
- [Integration Testing with TestContainers for .NET](/posts/testing-integration-testing-testcontainers/)
- [API Testing with WebApplicationFactory in ASP.NET Core](/posts/testing-webapplicationfactory/)

## References

- [k6 documentation](https://grafana.com/docs/k6/latest/)
- [k6 thresholds and metrics](https://grafana.com/docs/k6/latest/using-k6/thresholds/)
- [NBomber documentation](https://nbomber.com/)
- [ASP.NET Core metrics, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/log-mon/metrics/metrics)
