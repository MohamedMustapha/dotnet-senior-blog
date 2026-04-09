---
title: "Diagnosis: How to Investigate and Fix Database Performance Issues"
date: 2026-04-09
draft: false
tags: ["database", "performance", "dotnet"]
categories: ["Database"]
series: ["Database"]
description: "A reproducible method for database performance incidents: from symptom to root cause to fix. Execution plans, wait types, missing indexes, and the .NET-side traps that masquerade as database problems."
---

Database performance problems are the incident category where experienced engineers still reach for guesses. "It is probably a missing index." "It must be a bad query plan." "The connection pool is full." Sometimes they are right on the first try; most of the time they replace one guess with another until the symptoms shift. The issue is not knowledge, it is method. A reproducible diagnosis flow beats a good guess every time, because it converges on the real cause in minutes instead of hours, and because it produces evidence you can attach to the post-mortem.

This article lays out that flow: how to triage the symptom, how to confirm the actual bottleneck before touching anything, how to read an execution plan without getting lost, and the .NET-side patterns that look like database problems but are not.

## Why database diagnosis needs a method

The database is a black box with a public interface, and the interface lies about the cause of latency. A slow endpoint measured from the browser could be the database, the network, the serializer, the JSON response size, a `foreach` that awaits a query per iteration, or a garbage collection pause on the application server. A query that runs in 200 ms in SSMS and 2 seconds from the app could be parameter sniffing, a different connection string option, a different collation, or the application opening a second transaction the query has to wait for. Every one of those has a different fix, and picking the wrong hypothesis wastes the first thirty minutes of the incident.

The method exists so that the first thirty minutes produce evidence instead of theories.

## Overview: the diagnosis flow

{{< mermaid >}}
flowchart TD
    A[Symptom: slow endpoint or timeout] --> B[Step 1: Measure where the time goes]
    B --> C{Bottleneck?}
    C -->|.NET side| D[Profile the app: GC, serializer, N+1]
    C -->|Database side| E[Step 2: Capture the query]
    E --> F[Step 3: Execution plan]
    F --> G{Plan issue?}
    G -->|Missing index| H[Add index]
    G -->|Bad estimate| I[Update statistics / rewrite]
    G -->|Plan is fine| J[Step 4: Wait types]
    J --> K{What is it waiting on?}
    K -->|Locks| L[Blocking chain]
    K -->|IO| M[Storage / data size]
    K -->|CPU| N[Hot query or regression]
{{< /mermaid >}}

Four steps, in order, every time. Skipping step 1 is the reason teams spend an hour tuning a query that was not the bottleneck.

## Step 1: measure where the time goes

Before you open SSMS, confirm the slow endpoint is actually slow because of the database. If you have OpenTelemetry instrumentation, the answer is in the trace:

```csharp
services.AddOpenTelemetry()
        .WithTracing(t => t
            .AddAspNetCoreInstrumentation()
            .AddEntityFrameworkCoreInstrumentation(o => o.SetDbStatementForText = true)
            .AddNpgsql()
            .AddOtlpExporter());
```

A request span with its child spans tells you immediately whether the 2-second latency is 1.9 seconds of database time on a single query, 2 seconds of 400 queries at 5 ms each (N+1), or 100 ms of database time and 1.9 seconds of something else entirely. If the sum of the database spans is a small fraction of the request, the bottleneck is on the .NET side and no amount of query tuning will help.

If you do not have tracing yet, the minimum viable measurement is a stopwatch around the database call and a count of queries per request:

```csharp
var sw = Stopwatch.StartNew();
var count = 0;
_db.Database.SetCommandInterceptor(new CountingInterceptor(() => count++));

var result = await handler.HandleAsync(request, ct);
_logger.LogInformation("Request {Route} took {Elapsed} ms with {Count} queries",
    HttpContext.Request.Path, sw.ElapsedMilliseconds, count);
```

"400 queries in one request" is a diagnosis before you even read one of them.

> 💡 **Info** — Since EF Core 7, `DbCommandInterceptor` gives you a clean hook to count commands without changing the LINQ. Keep the interceptor in development only; in production the OpenTelemetry span count is the same data.

## Step 2: capture the exact query

Once you know the database is the bottleneck, you need the exact SQL, with actual parameter values, to reproduce it outside the app. EF Core's `LogTo` or the OpenTelemetry span gives you this. Copy the query verbatim. Do not rewrite it "for clarity": the tuning decisions depend on the literal SQL the driver sent.

```sql
SELECT o.id, o.reference, o.total_amount, c.email
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'completed'
  AND o.created_at > '2026-01-01'
ORDER BY o.created_at DESC
LIMIT 50;
```

Now run it against a copy of production. An exact copy, not dev seed data. Most slow queries are slow because of data shape: the dev database has 500 rows and a hash join; production has 40 million rows and a nested loop join over a missing index. You cannot diagnose that on a 500-row table.

## Step 3: read the execution plan

Every relational database exposes the execution plan. On PostgreSQL:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT ... ;
```

On SQL Server:

```sql
SET STATISTICS IO, TIME ON;
-- or use "Include Actual Execution Plan" in SSMS
SELECT ... ;
```

You are reading the plan for three things, in order:

**1. Where is the time actually spent?** The plan is a tree of operations; each node reports its own cost and its own row count. The expensive node is where you spend your attention. It is often not where you expected.

**2. Are the estimated rows far from the actual rows?** A 100x gap between estimate and actual almost always means the planner made a wrong choice: a nested loop where a hash join would have been better, a sort that spilled to disk, a bitmap that did not help. The fix is usually to update statistics, or to rewrite the predicate in a way the planner can estimate.

**3. Is there a sequential scan on a big table?** A sequential scan on 40 million rows is the classic "missing index" signature. It is not always wrong: if the query returns 30% of the rows, a sequential scan is correct. But if the query returns 50 rows and the plan scans 40 million, you are missing an index.

```sql
-- The fix for the example query above, on PostgreSQL:
CREATE INDEX idx_orders_status_created_at
    ON orders (status, created_at DESC)
    WHERE status = 'completed';
```

A partial index on the rows that match the most common predicate is smaller, faster to maintain, and more selective than a full index. Most dashboards query on one or two statuses; a partial index matches that access pattern exactly.

> ✅ **Good practice** — Add the execution plan (both the "before" and the "after") to the pull request that introduces the index. Future-you reading the commit in six months will want to know why this specific index exists.

> ⚠️ **Works, but...** — Adding an index based on the missing-index hint without reading the plan works often enough to be tempting. It also produces duplicate indexes, write-path slowdowns, and indexes that cover queries nobody runs anymore. Read the plan; add the index you need.

## Step 4: wait types

When the plan is fine and the query is still slow, the database is waiting on something. Every major DBMS exposes wait types:

```sql
-- PostgreSQL
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE state = 'active';

-- SQL Server
SELECT session_id, wait_type, wait_time, blocking_session_id, last_wait_type
FROM sys.dm_exec_requests
WHERE session_id > 50;
```

Three wait categories account for almost every case:

- **Lock waits**: the query is blocked by another transaction. The fix is upstream: shorter transactions, different isolation level, or breaking the blocking query into smaller ones.
- **IO waits**: the query is waiting on storage. The fix is usually data-shape (smaller rows, better compression, archiving old data) or infrastructure (faster disks, more memory so the hot data fits in cache).
- **CPU**: no waits, the query is just computing. The fix is query rewrite or index covering.

> 💡 **Info** — `pg_stat_activity` and `sys.dm_exec_requests` are point-in-time views. For repeat incidents, install a sampler that snapshots them every 5 seconds into a log so the next incident has history.

## Zoom: .NET-side traps that look like database problems

Half of the "database is slow" incidents I have seen in 15 years were not database incidents at all. The classics:

**Connection pool exhaustion.** When all connections are in use, the next query blocks until one frees up. It looks like a slow query in the application log, but the database shows the query as "took 4 ms once it started". The fix is finding the connection leak (usually a `DbContext` held longer than a request, or a missing `await using`), not tuning the query.

**`Task.Run` over a sync EF Core call.** Running a sync query on a thread pool thread looks fine until the thread pool saturates, at which point every query starts queuing on the thread pool before it even reaches the database. Convert to async, always.

**A `foreach` that awaits a query per iteration.** The total latency is `N * query_time`, which feels like a slow query until you realize there are 400 of them. The fix is the projection/include pattern from [EF Core: Read Optimization](/en/posts/database-efcore-read-optimization/).

**Serialization dominating the response.** A 20 MB JSON response takes a second to serialize regardless of the database. The database span in the trace is 20 ms; the response time is 1200 ms. The fix is pagination.

> ❌ **Never do** — Do not "fix" a .NET-side bottleneck by adding a database index. The index will not help and the write path will get slower for everyone.

## Wrap-up

The diagnosis method is four steps, in order: measure where the time goes, capture the exact query, read the execution plan, and if the plan is fine, look at wait types. Add an index when the plan tells you to, not when the incident ticket says "probably missing an index". And always confirm the database is the bottleneck before touching it, because half the "database is slow" incidents are .NET-side problems in disguise. With this flow, incidents stop being guesswork and become a checklist.

## Related articles

- [EF Core: Read Optimization](/en/posts/database-efcore-read-optimization/)
- [Dapper: When and How](/en/posts/database-dapper/)
- [EF Core: Migrations in Practice](/en/posts/database-efcore-migrations/)

## References

- [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL: pg_stat_activity](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ACTIVITY-VIEW)
- [SQL Server: Query Execution Plans](https://learn.microsoft.com/en-us/sql/relational-databases/performance/execution-plans)
- [SQL Server: Wait Statistics](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-os-wait-stats-transact-sql)
- [EF Core: Logging, Events, and Diagnostics](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/)
- [OpenTelemetry .NET: EF Core Instrumentation](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/tree/main/src/OpenTelemetry.Instrumentation.EntityFrameworkCore)
