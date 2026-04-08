---
title: "Soak Testing in .NET: The Bugs That Only Appear After Hours"
date: 2026-04-08
draft: false
tags: ["load-testing", "soak", "performance", "dotnet"]
categories: ["Testing"]
series: ["Load Testing"]
description: "Memory leaks, connection pool exhaustion, log growth, cache drift. The bugs that a 10-minute baseline will never catch, and that a soak test catches every time."
---

A system can pass every unit test, every [integration test](/posts/testing-integration-testing-testcontainers/), every [API test](/posts/testing-webapplicationfactory/), every [Playwright E2E](/posts/testing-e2e-playwright/), and the [baseline load test](/posts/load-testing-baseline/), and still fall over at 4 AM on the third day after deployment. The bugs that cause this have a common signature: they only appear after hours of sustained operation. Memory that leaks one kilobyte per request. A connection pool that creeps from 40 to 99 over the course of a weekend. A log file that reaches the disk quota on day six. A cache that drifts out of sync because an invalidation event is occasionally lost under load.

None of these show up in a 10-minute baseline. All of them show up in a soak test. That is the whole value proposition of soak testing: run the system at moderate, sustained load for long enough that time-dependent bugs have a chance to surface.

## Why soak tests exist

The traditional story about sudden production incidents is wrong. Most production incidents are not sudden. They are slow failures that look sudden because nobody was watching the gradient. A 2% daily growth in memory usage is invisible on a graph that spans one hour, and unmissable on a graph that spans seven days. A background job that leaks one thread per run is fine at 1 run per hour and catastrophic at 10 runs per minute. These are the failure modes that a soak test is designed to catch.

Concretely, soak tests answer four questions that no other test type answers:

1. **Does memory stay stable under sustained load?** A true memory leak produces a monotonically rising working set. A garbage collection that keeps up with allocations produces a sawtooth pattern that stays bounded. The difference is only visible over time.
2. **Do connection pools stay healthy?** Database pools, HTTP client pools, gRPC channels, message broker connections, all of them have a max size. An occasional leak of one connection per hour is invisible at minute ten and fatal at hour eighteen.
3. **Does disk usage stay bounded?** Logs, temporary files, dead-letter queues, failed job tables. Any of these can grow without bound if rotation, pruning, or cleanup is broken.
4. **Do caches, queues, and background state stay consistent?** Cache invalidation under concurrent writes, queue depth under varying consumer speed, scheduled jobs that do not clean up after themselves, all of these drift over time and only reveal themselves after hours.

## Overview: the shape of a soak run

{{< mermaid >}}
graph TD
    A[Moderate load<br/>50-70% of baseline] --> B[Duration<br/>4 to 24 hours]
    B --> C[Continuous metrics]
    C --> D1[Working set<br/>growth rate]
    C --> D2[GC heap<br/>gen0/1/2 sizes]
    C --> D3[Pool wait times<br/>DB, HTTP, threads]
    C --> D4[Disk usage<br/>log files, tmp]
    C --> D5[Latency<br/>drift over time]
{{< /mermaid >}}

A soak test is not a peak-throughput test. The load is kept deliberately moderate, usually 50 to 70 percent of what the baseline establishes as normal, so that the system has headroom for real work and the test stresses duration, not intensity. The duration is the variable: four hours for a first run, overnight for a pre-release validation, multi-day for a platform-level change (EF Core upgrade, runtime upgrade, infrastructure migration).

The output of a soak test is not a single number. It is a set of time-series graphs showing how metrics evolve over the run. A run that reports "p95 was 120 ms average" and nothing else is a failed soak test, because the average tells you nothing about whether the latency climbed from 90 ms to 160 ms over the window, which is the actual question.

## Zoom: k6 soak configuration

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 30 },     // warmup
    { duration: '8h', target: 30 },     // soak at 30 VUs (~60% of baseline)
    { duration: '1m', target: 0 },      // cooldown
  ],
  thresholds: {
    // Notice the sliding window: the soak fails if *any* hour degrades.
    'http_req_duration': ['p(95)<400'],
    'http_req_failed': ['rate<0.01'],
  },
  // Stream results to Prometheus so drift is visible live.
  ext: {
    loadimpact: { projectID: 0 },
  },
};

const BASE = __ENV.BASE_URL || 'https://shop.preprod.internal';

export default function () {
  group('catalog', () => {
    http.get(`${BASE}/api/products?page=1&size=20`);
  });

  if (Math.random() < 0.2) {
    group('write', () => {
      http.post(`${BASE}/api/cart`, JSON.stringify({
        productId: `SKU-${Math.floor(Math.random() * 1000)}`,
        quantity: 1,
      }), { headers: { 'Content-Type': 'application/json' } });
    });
  }

  sleep(2);
}
```

Eight hours, thirty virtual users, moderate load. The `sleep(2)` between requests is deliberate: a soak is not meant to maximize throughput, it is meant to keep the system under continuous, realistic pressure for a long time.

> ✅ **Good practice** : Run the soak with results streaming live to Grafana (or any dashboard). The most useful moment in a soak is not the end, it is the point where you notice the slope changing. Waiting for the final report defeats the purpose.

## Zoom: what to watch during the run

The load generator captures request-level metrics. The real signal lives on the application side. For a .NET system, the minimum dashboard during a soak run shows:

**Working set and GC heap over time.** The `process.runtime.dotnet.gc.heap.size` metric, broken down by generation, plotted against wall-clock time. A healthy system shows a stable or sawtooth pattern. A leak shows a rising trend that never resets, even after gen2 collections.

**Database connection pool metrics.** The `pool_wait_time` and `pool_in_use` counters from Npgsql, SqlClient, or whichever provider is in use. A pool that starts at 10 in-use and creeps to 90 over six hours has a connection leak somewhere, and the soak is the test that catches it.

**Thread pool queue length.** `System.Runtime` counters expose `threadpool-queue-length` and `threadpool-thread-count`. A queue that grows without bound means work is arriving faster than threads can handle it, usually because of a sync-over-async pattern that is only visible under sustained load.

**Request latency distribution, over time, not averaged.** A Grafana heatmap of `http_server_request_duration` per endpoint tells you whether p95 is stable or drifting upward. The drift, if it exists, is the bug.

**Disk usage on the host.** A simple `df` check reported every minute catches log rotation failures, temporary file leaks, and dead-letter queue bloat before they take the process down.

```csharp
// Program.cs: expose the metrics soak tests need
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics
            .AddMeter("Microsoft.AspNetCore.Hosting")
            .AddMeter("Microsoft.AspNetCore.Http.Connections")
            .AddMeter("System.Net.Http")
            .AddMeter("Microsoft.EntityFrameworkCore")
            .AddRuntimeInstrumentation()   // GC, thread pool, lock contention
            .AddProcessInstrumentation()   // CPU, memory, handles
            .AddPrometheusExporter();
    });
```

> 💡 **Info** : `AddRuntimeInstrumentation` comes from the `OpenTelemetry.Instrumentation.Runtime` NuGet package and is the single most useful line a .NET team can add to a soak-testable system. It exposes GC heap sizes, thread pool queue length, and lock contention with zero custom code.

## Zoom: reading the results

A soak test produces three typical outcomes.

**Flat and stable.** All metrics stay within their starting band for the whole duration. Latency sawtooths, GC recovers, pools stay stable, disk usage stays flat. The soak passes, and the team has evidence that the system can run for as long as the test duration.

**Gradual drift.** Latency climbs slowly, memory trends upward, or one pool grows. This is the diagnostic case the soak exists to catch. The team looks at the slope and asks, "at this rate, when do we hit the limit?". A linear leak of 50 MB per hour, on a 16 GB machine, gives you about two weeks. A sublinear drift may still be acceptable. A super-linear drift is a red alert, because it will not simply take twice as long to fail at twice the load, it will fail much faster.

**Cliff edge.** Everything looks fine for six hours, then a pool exhausts, a circuit breaker trips, or the process OOMs. The timing of the cliff is useful information: it tells you where the hidden limit is and gives the team a concrete target to fix.

> ⚠️ **It works, but...** : A soak that shows no drift over 8 hours is not a proof that the system can run for 8 days. Duration coverage grows non-linearly: weekly cron jobs, monthly batch runs, and seasonal load patterns will only be stressed by longer runs. Soak is a confidence signal, not a guarantee.

> ❌ **Never do this** : Do not run a soak test and only report the final number. Latency p95 averaged over 8 hours hides the entire story. The story is in the time-series graph. If the report does not include a graph, the report is incomplete.

## Zoom: when to run a soak

Soak tests are expensive in elapsed time, even though they are cheap in compute. Three cadences cover most teams:

**Before every platform upgrade.** A .NET runtime upgrade, an EF Core major version bump, a Kubernetes cluster migration, a change of database engine version: any of these warrants a full overnight soak before rolling out to production. This is where the highest-value bugs hide.

**Weekly, scheduled.** A once-a-week 8-hour soak, running Saturday night into Sunday morning, catches the regressions that accumulated during the week and establishes a rolling baseline for long-duration behavior.

**On suspicion.** When a production incident has "degraded slowly over hours" in its post-mortem, the follow-up is almost always a soak test designed to reproduce the degradation in pre-prod, with the offending component instrumented harder than usual.

## When soak is the wrong tool

Soak tests are the right answer for time-dependent failure modes. They are the wrong answer for:

- **Peak throughput questions**: that is a stress test.
- **Burst handling**: that is a spike test.
- **Logic correctness under concurrency**: that is an integration test with parallel workers, or a race-condition hunt, not a soak.
- **Finding the breaking point**: stress tests find it, soak tests do not push hard enough to reach it.

Running a soak to answer a stress question means waiting eight hours for a conclusion a one-hour stress test would have delivered.

## Wrap-up

A soak test is what reveals the bugs that live in the gap between a healthy ten-minute run and a multi-day production deployment. You can set one up in k6 or NBomber in an afternoon, keep the load moderate (50 to 70 percent of baseline), run it for four to twenty-four hours against a realistic pre-prod environment, and watch time-series metrics live rather than waiting for a final report. You can catch leaking connection pools, drifting caches, growing log files, and linear memory leaks before they become a production incident, and you can distinguish gradual drift, cliff-edge failure, and flat stable behavior from the shape of the graph.

Ready to level up your next project or share it with your team? See you in the next one, Stress Testing is where we go next.

## Related articles

- [Load Testing for .NET: An Overview of the Four Types That Matter](/posts/load-testing-overview/)
- [Baseline Load Testing in .NET: Knowing What Normal Looks Like](/posts/load-testing-baseline/)
- [API Testing with WebApplicationFactory in ASP.NET Core](/posts/testing-webapplicationfactory/)

## References

- [k6 stages and scenarios](https://grafana.com/docs/k6/latest/using-k6/scenarios/)
- [OpenTelemetry Runtime Instrumentation for .NET](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/tree/main/src/OpenTelemetry.Instrumentation.Runtime)
- [ASP.NET Core metrics, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/log-mon/metrics/metrics)
- [.NET garbage collection fundamentals, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)
