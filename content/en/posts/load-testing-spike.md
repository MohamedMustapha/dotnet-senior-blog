---
title: "Spike Testing in .NET: Surviving the Sudden Burst"
date: 2026-04-08
draft: false
tags: ["load-testing", "spike", "performance", "dotnet"]
categories: ["Testing"]
series: ["Load Testing"]
description: "Black Friday, viral content, cold cache, autoscale lag. Spike tests prove whether a system can survive a sudden burst from low to very high load, not just a gradual ramp."
---

A system can pass a [baseline](/posts/load-testing-baseline/), hold up in a [soak](/posts/load-testing-soak/), recover cleanly from a [stress test](/posts/load-testing-stress/), and still fail its most visible public moment: the exact second traffic goes from quiet to overwhelming, without warning. Black Friday at midnight, a viral tweet pointing at a landing page, a marketing email delivered to five hundred thousand inboxes at once, a partner integration whose cron job fires on the hour. These are the moments the team remembers, and they are not what a gradual stress ramp prepares for.

Spike testing is the last of the four test types introduced in the [overview article](/posts/load-testing-overview/). It answers a single, specific question: **when traffic goes from near-zero to very high in under ten seconds, does the system stay up, degrade gracefully, or collapse**.

## Why spike tests exist

A stress test with a smooth ramp gives a system every chance to adapt: CPU caches warm up, the JIT compiles hot paths, the database connection pool grows to meet demand, the autoscaler reacts and provisions new instances. A spike gives the system none of that. It starts quiet, and fifteen seconds later it is overwhelmed. The systems that die in spikes are the ones that needed the ramp.

Concretely, spikes expose four distinct weaknesses that no other test type stresses as hard:

1. **Cold cache penalty.** Distributed caches are fine until every node misses at once. The database gets hit by the full traffic, amplified by a thundering herd of concurrent misses, and collapses before the cache has time to rehydrate.
2. **Autoscale lag.** Kubernetes horizontal pod autoscalers, Azure Container Apps, AWS ECS, and every other autoscaler has a reaction time. That time is usually measured in minutes. A spike lasting ninety seconds is over before any new instance comes online.
3. **Connection pool startup cost.** Database drivers, HTTP clients, and message broker connections take time to establish. An application that starts with a pool of 10 connections and needs 200 will spend the first thirty seconds of the spike timing out while the pool grows.
4. **JIT compilation and warmup.** .NET JITs methods on first call. Tier-0 methods get re-JITted at tier-1 after they prove hot. A spike hits the system before the hot paths are tier-1 compiled, which can double the latency of the first thousand requests.

None of these are visible in a steady-state test. All of them are visible in a spike test, and all of them are fixable, usually with configuration changes and warmup strategies that cost very little.

## Overview: the shape of a spike run

{{< mermaid >}}
graph LR
    A[Idle or very low<br/>5-10 VUs] --> B[Sudden jump<br/>10 -> 500 VUs<br/>in under 30s]
    B --> C[Hold at peak<br/>2-5 min]
    C --> D[Drop back<br/>to idle]
    D --> E[Observe<br/>second spike<br/>if needed]
{{< /mermaid >}}

A spike test has four phases, each with a specific purpose.

**The idle phase** establishes that the system is quiet. Low or zero traffic, for a minute or two. This is the state the spike will interrupt.

**The jump** is the defining characteristic of the test. The ramp happens in seconds, not minutes. A spike is meant to catch the system unprepared. If the ramp is gradual, the test is a stress test, not a spike.

**The peak hold** keeps the high load for two to five minutes. Long enough for the autoscaler (if any) to react, the JIT to warm up, the cache to rehydrate, and the connection pools to grow. This phase answers the question "does the system recover while still under load".

**The drop** returns to idle. Optionally, a second spike follows a minute later, to test whether the system is actually ready for the next burst or if it is still recovering from the first.

## Zoom: a spike test with k6

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m',  target: 10 },    // idle
    { duration: '10s', target: 500 },   // the spike: 10 -> 500 in 10s
    { duration: '3m',  target: 500 },   // hold at peak
    { duration: '10s', target: 10 },    // drop back
    { duration: '30s', target: 10 },    // recovery observation
    { duration: '10s', target: 500 },   // second spike (optional)
    { duration: '1m',  target: 500 },
    { duration: '10s', target: 0 },
  ],
  thresholds: {
    // Spike tests have looser thresholds: the goal is "still up", not "baseline latency".
    'http_req_duration': ['p(95)<2000'],
    'http_req_failed': ['rate<0.10'],
  },
};

const BASE = __ENV.BASE_URL || 'https://shop.preprod.internal';

export default function () {
  http.get(`${BASE}/api/products/featured`);
  sleep(0.1);  // tight loop: spikes maximize pressure
}
```

Ten to five hundred virtual users in ten seconds, held for three minutes, dropped, held at low, then spiked again. The thresholds are deliberately looser than a baseline or a stress test, because the question is not "did performance stay at baseline" but "did the system stay available through the spike and the second spike".

> ✅ **Good practice** : Run the spike test against a system that has been idle for at least ten minutes before the test starts. A spike against a warm system is not a spike, it is a stress test. Coldness is the whole point.

## Zoom: the same test with NBomber

```csharp
using NBomber.CSharp;
using NBomber.Http;
using NBomber.Http.CSharp;

using var httpClient = new HttpClient { BaseAddress = new Uri("https://shop.preprod.internal") };

var scenario = Scenario.Create("spike_hot_path", async context =>
{
    var request = Http.CreateRequest("GET", "/api/products/featured");
    return await Http.Send(httpClient, request);
})
.WithLoadSimulations(
    // Idle
    Simulation.KeepConstant(copies: 10, during: TimeSpan.FromMinutes(1)),
    // The spike: ramp 10 -> 500 in 10 seconds
    Simulation.RampingConstant(copies: 500, during: TimeSpan.FromSeconds(10)),
    // Hold
    Simulation.KeepConstant(copies: 500, during: TimeSpan.FromMinutes(3)),
    // Drop
    Simulation.RampingConstant(copies: 10, during: TimeSpan.FromSeconds(10)),
    // Recovery
    Simulation.KeepConstant(copies: 10, during: TimeSpan.FromSeconds(30)),
    // Second spike
    Simulation.RampingConstant(copies: 500, during: TimeSpan.FromSeconds(10)),
    Simulation.KeepConstant(copies: 500, during: TimeSpan.FromMinutes(1)),
    Simulation.RampingConstant(copies: 0,  during: TimeSpan.FromSeconds(10))
);

NBomberRunner.RegisterScenarios(scenario)
    .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
    .WithReportFolder("./reports/spike")
    .Run();
```

`RampingConstant` with a 10-second duration from 10 to 500 virtual users is the NBomber equivalent of k6's spike stage. Everything else is a matter of phase sequencing, which NBomber expresses as an ordered list of `LoadSimulation` entries.

## Zoom: what to watch during a spike

Five signals matter during a spike, and all of them need sub-second resolution in the dashboard to be readable at all.

**Time-to-first-response after the spike begins.** How many seconds pass between the load jumping and the first `200 OK` being served under the new load. This is often the single most useful number: it captures JIT warmup, connection pool growth, and cache rehydration in one metric.

**Connection pool growth curve.** For Npgsql or SqlClient, plot `pool_in_use` over time. During a spike, the pool should grow quickly to match demand. If it plateaus early, the pool has reached its configured maximum and the team has found the first bottleneck.

**Database query latency distribution.** During a spike with a cold cache, the database is the first place to feel the pain. Plot the per-second p95 of query duration. Look for the moment it peaks, then returns to baseline. The delta is the cold-cache cost.

**Autoscaler events.** If the system runs on Kubernetes or a container orchestrator with autoscaling, log the pod count over time. Compare the scale-up moment to the start of the spike. The gap is the autoscale lag, and it is almost always longer than teams expect.

**Error rate per endpoint.** During a spike, certain endpoints fail before others. Plot error rate per endpoint to identify which one broke first. That is your next fix target.

```csharp
// Program.cs: expose the minimal metrics needed for a spike test
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddMeter("Microsoft.AspNetCore.Hosting")
        .AddMeter("Microsoft.EntityFrameworkCore")  // query duration
        .AddMeter("Npgsql")                          // pool_in_use
        .AddRuntimeInstrumentation()                 // GC, thread pool
        .AddPrometheusExporter());
```

> 💡 **Info** : Grafana's default time resolution is 15 or 30 seconds, which is too coarse for a 90-second spike. Set the scrape interval to 1 second and the dashboard refresh to 1 second during spike tests. Otherwise the graph will show two points on the entire spike and nothing will be diagnosable.

## Zoom: the four common spike failures

**Cold cache thundering herd.** Every request hits the cache, every cache lookup misses, every miss hits the database, and the database sees 500 concurrent identical queries. The fix is request coalescing or a lock around cache rehydration, so only the first miss triggers a database query while the others wait.

**Connection pool exhaustion.** The default Npgsql pool caps at 100 connections. An instance handling 400 concurrent requests during a spike will block 300 of them waiting for a connection. The fix is either a larger pool (if the database can handle it) or a concurrency limiter in front of the endpoint (to shed load rather than queue it).

**Autoscaler lag.** The autoscaler is configured to add pods when CPU exceeds 70%. The spike drives CPU to 100% in 10 seconds, the autoscaler reacts in 60 seconds, and the first new pod is ready in another 45 seconds. The first 90 seconds of the spike run with half the needed capacity. The fix is pre-warming: run more idle capacity, or use predictive autoscaling, or pre-scale before an expected event (midnight sale).

**JIT warmup cost.** The first thousand requests after a cold start are served by tier-0 JIT-compiled code, which is slower than tier-1. In a spike, those first thousand requests happen in a few seconds, and their latency is two to three times baseline. The fix is ReadyToRun (R2R) compilation, AOT, or a warmup endpoint that the orchestrator calls before declaring the pod healthy.

> ⚠️ **It works, but...** : A spike test that triggers none of these failures on the first run is usually a sign that the target system is not configured the way production will be. Check that the cache is actually empty, the database pool is at its production default, and the replica count matches production minimum. Otherwise the test is confirming the wrong thing.

> ❌ **Never do this** : Do not run spike tests immediately after another load test. The system is warm, the pools are full, the caches are populated. A spike against a warm system tells you nothing. Either wait ten minutes for idle, or restart the target.

## Zoom: when to run a spike

Spike tests are less routine than baselines but more targeted. Three triggers:

**Before an expected traffic event.** A product launch, a marketing campaign, a known external integration going live. If the team knows a spike is coming in production, rehearse it in pre-prod first.

**After a deployment topology change.** New autoscaling rules, a different instance type, a new cache backend, a database migration. Any of these can change spike behavior without showing up in a baseline or a stress test.

**When a production incident says "traffic jumped and we fell over".** The follow-up is always a spike test in pre-prod, with the exact traffic shape from the incident, and the exact infrastructure configuration from the incident. The goal is to reproduce the failure, fix it, and prove the fix works.

## Wrap-up

A spike test is the only test that measures how a system survives a sudden jump from quiet to overwhelmed. You can set one up in k6 or NBomber in an afternoon, start from an idle state (not a warm one), jump from low to high in under thirty seconds, hold at peak for a few minutes, optionally trigger a second spike to test recovery readiness, and watch time-to-first-response, pool growth, autoscale lag, and cold-cache cost with sub-second dashboard resolution. You can walk out knowing which of the four common spike failures your system would hit, and you can plan the fixes (request coalescing, larger pools, pre-warming, ReadyToRun compilation) before the next marketing campaign makes them urgent.

Ready to level up your next project or share it with your team? See you in the next one, a++ 👋

## Related articles

- [Load Testing for .NET: An Overview of the Four Types That Matter](/posts/load-testing-overview/)
- [Baseline Load Testing in .NET: Knowing What Normal Looks Like](/posts/load-testing-baseline/)
- [Soak Testing in .NET: The Bugs That Only Appear After Hours](/posts/load-testing-soak/)
- [Stress Testing in .NET: Finding the Breaking Point and Its Shape](/posts/load-testing-stress/)

## References

- [k6 spike testing guide](https://grafana.com/docs/k6/latest/testing-guides/test-types/spike-testing/)
- [NBomber load simulations](https://nbomber.com/docs/using-nbomber/load-simulations/)
- [ReadyToRun compilation, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/deploying/ready-to-run)
- [Horizontal Pod Autoscaler in Kubernetes](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Google SRE book: Handling Overload](https://sre.google/sre-book/handling-overload/)
