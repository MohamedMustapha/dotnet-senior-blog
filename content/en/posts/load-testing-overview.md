---
title: "Load Testing for .NET: An Overview of the Four Types That Matter"
date: 2026-04-08
draft: false
tags: ["load-testing", "performance", "dotnet"]
categories: ["Testing"]
series: ["Load Testing"]
description: "Baseline, soak, stress, spike: four load test types, four questions about production readiness. A pragmatic overview for .NET teams, with tools and metrics that actually matter."
---

Unit tests, integration tests, API tests, and end-to-end tests all share one quiet assumption: they run one user at a time. That assumption is comfortable, productive, and completely blind to the question production will inevitably ask, "what happens when a thousand users arrive at once". Load testing exists to answer that question before the answer is a 3 AM phone call.

The [Testing series](/posts/testing-unit-testing/) covered correctness across the pyramid, from unit tests through [integration tests with TestContainers](/posts/testing-integration-testing-testcontainers/), [API tests with WebApplicationFactory](/posts/testing-webapplicationfactory/), and [end-to-end tests with Playwright](/posts/testing-e2e-playwright/). Load testing is a different axis entirely. It does not ask "does the logic work", it asks "does the logic hold up under concurrency, sustained traffic, sudden bursts, and beyond its designed capacity". Four questions, four test types, four articles in this series. This article is the map.

## Why load testing exists

The traditional excuse for skipping load tests was "we will scale when we need to". That works until a marketing campaign, a viral moment, or an integration with a newly popular partner sends ten times the traffic in thirty seconds. At that point, the team discovers, all at once, that the database connection pool is capped at 100, that the cache does not rebuild gracefully under concurrent misses, that a log framework is holding a lock that serializes every request, and that the autoscaler takes four minutes to react to a burst that lasts two.

Load testing surfaces all of this *before* the incident. More concretely, it answers four specific questions that production will ask:

1. **What does "normal" look like?** Without a reference point, there is no way to detect that a deployment made things worse.
2. **Does the system degrade gracefully over hours or days?** Memory leaks, connection exhaustion, log rotation bugs, cache staleness: these only appear after sustained operation.
3. **Where does the system break, and how does it break?** Understanding the failure mode matters as much as knowing the breaking point.
4. **How does the system react to sudden bursts?** Autoscaling, backpressure, queue depth, and cold caches all behave differently under a gradual ramp-up than under a spike.

Each question has a dedicated load test type. None of them replaces the others.

## Overview: the four types

{{< mermaid >}}
graph TD
    A[Load testing] --> B[Baseline<br/>Establish normal<br/>steady-state]
    A --> C[Soak<br/>Long duration<br/>moderate load]
    A --> D[Stress<br/>Beyond capacity<br/>find the break]
    A --> E[Spike<br/>Sudden burst<br/>from low to high]
    B --> B1[Reference<br/>for regression]
    C --> C1[Leaks, pool exhaustion,<br/>log growth, cache drift]
    D --> D1[Breaking point,<br/>capacity planning]
    E --> E1[Autoscale response,<br/>cold cache, backpressure]
{{< /mermaid >}}

**Baseline** runs the system under the traffic it is expected to handle every day, for long enough to produce stable numbers. The output is a set of reference metrics: requests per second, latency percentiles, error rate, CPU, memory, database pool usage. Every subsequent load test is compared against this reference.

**Soak** runs the same moderate load for hours, often overnight, sometimes for days. Its job is not to measure peak throughput, it is to verify that the system does not degrade over time. Memory leaks, connection pool exhaustion, log file growth, cache invalidation drift, and background task pile-ups all show up here and nowhere else.

**Stress** pushes the system past its designed capacity and keeps pushing until something gives. The goal is not to prove the system can handle infinite load, it is to characterize the *failure mode*: does latency grow linearly, then explode? Does the error rate climb before latency does? Does the system recover cleanly when the stress is removed?

**Spike** starts from a quiet state and ramps to a very high load within seconds. This is the test that exposes autoscaling lag, cold cache penalties, connection burst handling, and the warmup cost of JIT-compiled code paths. A system that handles a gradual ramp-up perfectly can still collapse under a spike.

Each of these has its own article in this series. The rest of this overview covers the shared vocabulary and the toolchain choices that apply to all four.

## Zoom: the metrics that matter

Every load test, regardless of type, should report the same set of numbers. If any of these is missing, the test is incomplete.

**Throughput** measured in requests per second (RPS). The raw count of work the system handles in a unit of time. High RPS is only meaningful paired with the next metric.

**Latency percentiles**: p50, p95, p99, and p99.9. The average latency is almost never useful, because a system where 90% of requests take 20 ms and 10% take 2 seconds has the same average as a system where every request takes 220 ms, and the two are not the same to a user. Report percentiles, always.

**Error rate**, broken down by status code. A test that holds a p95 under 100 ms while silently serving 4% of requests as 500s is not a passing test, it is a misleading one.

**Saturation signals** from the .NET runtime and the infrastructure: CPU, memory, GC pause times (gen0/1/2), thread pool queue length, database connection pool wait time, HTTP client connection count. These tell you *why* the latency rose, which is the actionable half of the information.

**Correlation with business transactions**, not just HTTP endpoints. A test that reports "POST /orders p95 is 300 ms" is less useful than one that reports "the checkout flow (add to cart, apply discount, submit order, confirm payment) p95 is 1.2 seconds". The user experience is the composition of the individual endpoints, not any single one.

> 💡 **Info** : In modern .NET (8+), `System.Diagnostics.Metrics` and the built-in `http.server.request.duration` histogram expose these numbers natively. Feeding them to Prometheus and Grafana is a couple of lines of configuration and is the foundation for everything in this series.

## Zoom: tools landscape in 2026

Two .NET-friendly tools cover 90% of real use cases, and the choice between them is mostly about where the test code lives.

**k6** (Grafana Labs) is the current industry standard. Tests are written in JavaScript, run by a Go-based runner, and scale to hundreds of thousands of virtual users from a single machine. k6 integrates cleanly with Grafana for visualization, with Prometheus as a metrics sink, and with most CI systems. It is language-agnostic, which is a strength if your team ships more than one backend stack, and a neutral point if you ship only .NET.

```javascript
// k6: a baseline test, 50 virtual users for 5 minutes
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 50,
  duration: '5m',
  thresholds: {
    http_req_duration: ['p(95)<300'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('https://shop.test/api/orders');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

**NBomber** is the .NET-native option. Tests are written in C# or F#, live in a regular .NET project, share types with the application under test, and run from `dotnet test` or a console host. The advantage is that the load test suite is code the team already knows how to read, review, and refactor.

```csharp
// NBomber: same baseline, written in C#
using NBomber.CSharp;
using NBomber.Http;
using NBomber.Http.CSharp;

var scenario = Scenario.Create("get_orders", async context =>
{
    var response = await Http.CreateRequest("GET", "https://shop.test/api/orders")
        .WithHeader("Accept", "application/json")
        .SendAsync(httpClient, context);

    return response;
})
.WithLoadSimulations(
    Simulation.KeepConstant(copies: 50, during: TimeSpan.FromMinutes(5)));

NBomberRunner.RegisterScenarios(scenario).Run();
```

Both are production-grade. For .NET teams that prefer to keep everything in C#, NBomber is the lower-friction choice. For teams that want the larger community and ecosystem, k6 is the safer bet. JMeter, Gatling, Artillery, and Locust all exist and have legitimate use cases, but for a greenfield .NET project in 2026, k6 or NBomber is the default recommendation.

> ✅ **Good practice** : Write the load test code in the same repository as the application, next to the integration tests. Load tests are part of the codebase, not a separate folder on someone's laptop.

## Zoom: where load tests run

A load test against a developer laptop is almost always meaningless. The network, the local database, the shared CPU with every IDE and browser open, and the lack of realistic infrastructure all distort the result. The useful environments are:

- **A dedicated pre-prod environment** that mirrors production sizing and topology. This is the default target for baseline, soak, and spike tests.
- **A clone of production**, stood up for a scheduled test window. More expensive, more accurate, reserved for stress tests and capacity planning exercises.
- **Production itself, with a controlled subset of traffic**, for advanced teams practicing continuous load testing. This requires observability maturity that most teams do not have, and it is not the starting point.

For most teams, the right answer is a pre-prod environment provisioned from the same Infrastructure-as-Code as production, with the same database size class, the same cache, and the same dependencies spun up through [TestContainers](/posts/testing-integration-testing-testcontainers/) where a real managed service is not available.

> ⚠️ **It works, but...** : Running load tests against a free-tier cloud database or a small dev container will produce numbers that look terrible compared to production, or worse, numbers that look great and are completely wrong. Pay attention to the sizing of the target, not only the sizing of the load generator.

## Zoom: what load tests do not catch

Load tests are not a replacement for any other layer of the test pyramid. They do not catch:

- **Logic bugs**: the calculator can be wrong and still handle 10,000 RPS. That is a [unit test](/posts/testing-unit-testing/) problem.
- **Authorization holes**: a broken role check is fast. Fast and wrong is worse than slow and correct. That is a [WebApplicationFactory test](/posts/testing-webapplicationfactory/) problem.
- **Data migration correctness**: load tests against a broken migration will simply fail with broken data. Run migrations in a real database first, via [integration tests with TestContainers](/posts/testing-integration-testing-testcontainers/).
- **UI-level race conditions**: those belong in [Playwright E2E tests](/posts/testing-e2e-playwright/).

Load tests sit on top of a correct system, not instead of one. Running them before the rest of the pyramid is green is a waste of the load generator's time and a source of false confidence.

## Wrap-up

You now have a map of the four load test types that matter for a .NET system: baseline to establish what normal looks like, soak to verify the system holds up over time, stress to find the breaking point and its shape, and spike to validate autoscaling and burst handling. You can pick k6 or NBomber as a default runner, capture throughput, latency percentiles, error rate, and saturation signals for every test, and run against a pre-prod environment that actually mirrors production.

Ready to level up your next project or share it with your team? See you in the next one, Baseline Testing is where we go next.

## Related articles

- [Unit Testing in .NET: Fast, Focused, and Actually Useful](/posts/testing-unit-testing/)
- [Integration Testing with TestContainers for .NET](/posts/testing-integration-testing-testcontainers/)
- [API Testing with WebApplicationFactory in ASP.NET Core](/posts/testing-webapplicationfactory/)
- [End-to-End Testing with Playwright for .NET](/posts/testing-e2e-playwright/)

## References

- [k6 documentation](https://grafana.com/docs/k6/latest/)
- [NBomber documentation](https://nbomber.com/)
- [ASP.NET Core metrics, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/log-mon/metrics/metrics)
- [.NET diagnostics and performance, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/)
- [Google SRE book: Handling Overload](https://sre.google/sre-book/handling-overload/)
