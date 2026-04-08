---
title: "Stress Testing in .NET: Finding the Breaking Point and Its Shape"
date: 2026-04-08
draft: false
tags: ["load-testing", "stress", "performance", "dotnet"]
categories: ["Testing"]
series: ["Load Testing"]
description: "Stress tests do not ask 'can the system handle normal load'. They ask 'when does it break, how does it break, and how does it recover'. Three questions that matter more than a peak RPS number."
---

A baseline tells you what normal looks like. A [soak](/posts/load-testing-soak/) tells you whether the system holds up over time. Neither answers the question that production will eventually force you to care about: **when does it break, and how**. That is the job of a stress test. Not to prove the system can handle arbitrary load (no system can), but to characterize the exact shape of its failure so the team can design around it.

The [overview article](/posts/load-testing-overview/) introduced the four test types. The [baseline article](/posts/load-testing-baseline/) covered the reference run. This article covers the one that deliberately breaks the system, learns something from the breakage, and walks away with a concrete capacity plan.

## Why stress tests exist

Every system has a point past which more traffic makes things worse instead of better. Adding one more request per second starts queuing work faster than the workers can process it. Latency rises, then climbs steeply, then the error rate begins to grow. Eventually something gives: a connection pool saturates, a thread pool starves, a circuit breaker opens, or the process runs out of memory and restarts. The team that learns this in production pays for the lesson with an outage. The team that learns it in a stress test pays for the same lesson with a spreadsheet.

Stress tests answer four questions that no other test type answers:

1. **Where is the breaking point?** The load (in RPS or concurrent users) at which latency explodes, error rate spikes, or the process fails. The number itself is useful for capacity planning.
2. **What is the shape of the failure?** Linear degradation, exponential degradation, cliff-edge collapse, and cascading failure all demand different remediations. The shape is more actionable than the raw number.
3. **Which component gives first?** Is it the database connection pool, the thread pool, the memory, the downstream API, the rate limiter? The first bottleneck is the one worth fixing.
4. **Does the system recover?** Once the stress is removed, does the system return to healthy latency and throughput, or does it stay degraded and require a restart? Recovery behavior matters as much as breaking point.

Without a stress test, capacity planning is guesswork. With one, the team has a number, a shape, and a recovery profile.

## Overview: the shape of a stress run

{{< mermaid >}}
graph LR
    A[Baseline load<br/>50 VUs] --> B[Ramp up<br/>+50 VUs every 2 min]
    B --> C[Observe<br/>breaking point]
    C --> D[Hold past break<br/>1-2 min]
    D --> E[Ramp down<br/>observe recovery]
{{< /mermaid >}}

A stress test is a controlled ramp, not a sudden burst. The system starts at baseline load, ramps up in measured steps, and the test captures the point at which the pre-defined service level objective is breached. That point is the breaking point. The ramp continues for a short while past it to characterize the failure mode, then ramps down to observe recovery.

Three rules shape a useful stress run:

**Ramp, do not jump.** A sudden burst is a spike test, which is a different question. A stress test wants to see the slope of degradation, which requires a gradual, measured ramp.

**Define failure before the run.** "The system is broken" is not an objective statement. Decide in advance: for example, breaking point is reached when p95 exceeds 1 second or error rate exceeds 5%. Without this, the team will argue about the results after the fact.

**Always ramp back down.** Observing how the system recovers (or does not) is half the value of the test. A stress test that cuts traffic at peak and reports "we hit 5000 RPS" has learned nothing about whether production could actually sustain it.

## Zoom: a stress run with k6

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  stages: [
    { duration: '2m',  target: 50 },    // baseline hold
    { duration: '2m',  target: 100 },   // +50 VUs
    { duration: '2m',  target: 150 },
    { duration: '2m',  target: 200 },
    { duration: '2m',  target: 300 },
    { duration: '2m',  target: 400 },
    { duration: '2m',  target: 500 },
    { duration: '2m',  target: 500 },   // hold at peak
    { duration: '3m',  target: 0 },     // ramp down, observe recovery
  ],
  thresholds: {
    // These thresholds are the failure definition.
    // A violated threshold fails the run, which is expected past breaking point.
    'http_req_duration{group:::hot}': ['p(95)<1000'],
    'http_req_failed': ['rate<0.05'],
  },
};

const BASE = __ENV.BASE_URL || 'https://shop.preprod.internal';

export default function () {
  group('hot', () => {
    http.get(`${BASE}/api/products/featured`);
  });

  if (Math.random() < 0.3) {
    group('write', () => {
      http.post(`${BASE}/api/cart`, JSON.stringify({
        productId: 'SKU-1',
        quantity: 1,
      }), { headers: { 'Content-Type': 'application/json' } });
    });
  }

  sleep(0.5);
}
```

A 500-VU ceiling, reached in six steps of +50 to +100 VUs each. Each step holds for two minutes, which is long enough for the system to stabilize at that load level before the next step. The ramp-down is short and deliberate: three minutes from peak to zero, which is where the recovery behavior is captured.

> ✅ **Good practice** : Pick the step size so that the entire ramp takes 15 to 25 minutes. Shorter runs miss steady-state behavior at each level. Longer runs burn budget and make the result hard to interpret.

## Zoom: the same stress run with NBomber

```csharp
using NBomber.CSharp;
using NBomber.Http;
using NBomber.Http.CSharp;

using var httpClient = new HttpClient { BaseAddress = new Uri("https://shop.preprod.internal") };

var scenario = Scenario.Create("hot_path", async context =>
{
    var request = Http.CreateRequest("GET", "/api/products/featured");
    return await Http.Send(httpClient, request);
})
.WithLoadSimulations(
    Simulation.KeepConstant(copies: 50,  during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 100, during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 200, during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 300, during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 400, during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 500, during: TimeSpan.FromMinutes(2)),
    Simulation.KeepConstant(copies: 500, during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 0,   during: TimeSpan.FromMinutes(3))
);

NBomberRunner.RegisterScenarios(scenario)
    .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
    .WithReportFolder("./reports/stress")
    .Run();
```

Same stair-step profile, expressed as a list of `LoadSimulation` stages. NBomber's HTML report plots latency and throughput per step, which is exactly the shape a stress test is meant to produce.

## Zoom: identifying the breaking point

The breaking point is not always obvious from a single graph. It is the intersection of three signals.

**Latency p95 curve.** Plot p95 latency against VU count. In a healthy system, the curve is nearly flat, then begins to rise, then rises steeply. The breaking point is where the rise becomes super-linear, usually visible as an inflection point on the graph.

**Error rate curve.** Plot error rate against VU count. In most .NET systems, the error rate stays near zero until the breaking point, then rises fast. If the error rate starts rising before the latency does, the bottleneck is a hard limit (a connection pool, a rate limiter, a circuit breaker). If latency rises first, the bottleneck is a soft limit (CPU, memory, thread pool queuing).

**Throughput curve.** Plot successful RPS against VU count. In a healthy system, throughput grows with VUs, then plateaus at the system's maximum. In a failing system, throughput peaks, then *drops* as the system spends more time handling failures than real work. The drop is the most actionable signal: it means the system is doing worse under more load, not just handling less well.

The intersection of these three curves gives a defensible number: "the system supports 320 RPS before p95 exceeds 1 second and error rate exceeds 1%". That number is usable in capacity planning, in contract negotiations, and in deployment sizing.

## Zoom: the shape of failure

The curve itself matters as much as the number. Four failure shapes are common.

**Linear degradation.** Latency rises smoothly, error rate stays near zero, throughput plateaus cleanly. The best shape possible, because it means the team can scale out linearly to match demand and predict behavior past the breaking point. Usually indicates a CPU-bound system with well-tuned pools.

**Knee curve.** Latency is flat, then bends upward sharply at a specific load level. Indicates a hard resource limit: a connection pool reaching max, a cache miss storm, a thread pool saturating. The fix is usually a single configuration change, once the resource is identified.

**Cliff edge.** Latency is flat, everything looks fine, then the system collapses within 30 seconds: errors spike, throughput drops to zero. Indicates a cascading failure: a circuit breaker that opens and starves a dependent service, a deadlock that propagates across requests, an OOM that restarts the process. Cliff-edge failures are the most dangerous because there is no warning before the outage.

**Death spiral.** Latency climbs, then throughput drops, then latency climbs more because retries pile up on an already-overloaded system. The system gets worse the more traffic it receives, even if the traffic stops growing. The fix is usually backpressure or load shedding, not more capacity.

> 💡 **Info** : The `.NET` runtime has a built-in concurrency limiter (`Microsoft.AspNetCore.RateLimiting`, available since .NET 7) specifically designed to prevent death spirals. Adding a queue-based rate limiter in front of sensitive endpoints turns a death spiral into controlled rejection, which is much easier to reason about.

## Zoom: recovery

Once the ramp-down begins, the question changes from "how bad did it get" to "does the system come back". Three outcomes.

**Clean recovery.** Within seconds of the load dropping, latency returns to baseline, error rate returns to zero, throughput matches demand. This is the expected outcome, and it confirms that the system can shed load without side effects.

**Slow recovery.** Latency takes minutes to return to baseline even after the load drops. Usually indicates that something is still draining: a queue that accumulated backlog, a connection pool that is slowly releasing stuck connections, a cache that is rebuilding from cold after an invalidation storm. The recovery time is itself a metric, and it is often where the hidden cost of the failure lives.

**No recovery.** Latency stays elevated, or the system keeps returning errors, even at zero load. Indicates permanent damage: a leaked thread that holds a lock, a deadlocked async state machine, a circuit breaker stuck open, a cache that cannot rehydrate. The process needs a restart to return to health, which is information the team needs before the same failure happens in production.

> ⚠️ **It works, but...** : A stress test that only measures peak RPS without measuring recovery is reporting half the story. The team can hit a peak that production cannot, if recovery after that peak is impossible. Capacity planning must account for the margin needed to avoid the peak, not just the peak itself.

> ❌ **Never do this** : Do not run stress tests against production without a strict blast radius and a pre-agreed abort condition. Stress tests are designed to break things, and the production database is not the place to discover what breaks.

## Wrap-up

A stress test is the only test that produces a capacity number the team can actually defend. You can set one up in k6 or NBomber in an afternoon, use a stair-step ramp of 15 to 25 minutes, define the failure condition before the run to avoid arguing about results afterward, capture latency, error rate, and throughput curves side by side, identify the shape of the failure, and always include a ramp-down phase to measure recovery. You can walk out of a stress test with a defensible number for capacity planning, a named first bottleneck to fix, and confidence about how the system will behave when production traffic spikes past expected limits.

Ready to level up your next project or share it with your team? See you in the next one, Spike Testing is where we go next.

## Related articles

- [Load Testing for .NET: An Overview of the Four Types That Matter](/posts/load-testing-overview/)
- [Baseline Load Testing in .NET: Knowing What Normal Looks Like](/posts/load-testing-baseline/)
- [Soak Testing in .NET: The Bugs That Only Appear After Hours](/posts/load-testing-soak/)

## References

- [k6 stress testing patterns](https://grafana.com/docs/k6/latest/testing-guides/test-types/stress-testing/)
- [NBomber load simulations](https://nbomber.com/docs/using-nbomber/load-simulations/)
- [ASP.NET Core rate limiting middleware, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)
- [Google SRE book: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
