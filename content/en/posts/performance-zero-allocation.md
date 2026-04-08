---
title: "Zero Allocation in .NET: When the GC Becomes the Bottleneck"
date: 2026-04-08
draft: false
tags: ["performance", "dotnet", "memory"]
categories: ["Performance"]
series: ["Performance"]
description: "Span, Memory, stackalloc, ArrayPool, ValueTask, and the patterns that let hot-path .NET code run without producing garbage. Not for every method, but decisive where it matters."
---

For 95% of .NET code, the garbage collector is a quiet helper that nobody thinks about. Allocations happen, memory is reclaimed, and the program continues. For the other 5%, the hot paths of a high-throughput system, the garbage collector is the bottleneck, and every byte allocated per request becomes a request per second the system cannot handle. The difference between these two worlds is not code quality, it is frequency: at 100,000 requests per second, a single 1 KB allocation per request becomes 100 MB per second of heap pressure, and the GC starts running continuously to keep up.

Zero-allocation programming is the set of techniques that let performance-sensitive code avoid producing garbage on the hot path. It is not a style to apply everywhere. It is a toolbox to reach for when a [stress test](/posts/load-testing-stress/) or a [soak test](/posts/load-testing-soak/) shows that GC pause times, gen0 collection frequency, or heap pressure are limiting throughput. Used at the right places, it can double or triple a system's capacity without changing anything else.

## Why zero allocation matters

The .NET garbage collector is generational. Objects start in gen0, survive into gen1 if they live long enough, and reach gen2 if they survive two collections. Collecting gen0 is cheap (a few hundred microseconds), gen1 is more expensive, gen2 is the one that produces visible application pauses and can take tens of milliseconds. A well-behaved system keeps most allocations in gen0, where collection is nearly free.

The problem is that "nearly free" is not free. Every gen0 collection stops the managed threads (in server GC mode, briefly), measures the roots, compacts the young generation, and resumes. At 100,000 requests per second, if each request allocates 2 KB, gen0 fills in milliseconds, and the GC runs several times per second. Each run introduces jitter, latency spikes, and contention with real work.

Zero-allocation code changes this equation. Instead of allocating for every operation, it reuses buffers, uses the stack for temporary data, and keeps the managed heap quiet. The goals are concrete:

1. **Stable tail latency**, because fewer GC pauses mean fewer latency spikes at p99 and p99.9.
2. **Higher throughput**, because the CPU spends less time collecting and more time running application code.
3. **Lower memory pressure**, because the working set stays bounded and the system can pack more instances per host.
4. **Predictable behavior under load**, because the GC is no longer one of the moving parts whose cost scales with traffic.

## Overview: the allocation pyramid

{{< mermaid >}}
graph TD
    A[Value types on the stack<br/>free] --> B[Span&lt;T&gt; over stackalloc<br/>free]
    B --> C[ArrayPool&lt;T&gt;<br/>reused, not allocated]
    C --> D[Pooled objects<br/>ObjectPool&lt;T&gt;]
    D --> E[Gen0 heap<br/>cheap, but not free]
    E --> F[LOH / Gen2<br/>expensive, avoid]
{{< /mermaid >}}

Not every allocation is equal. The pyramid above orders the options from cheapest to most expensive. The guiding principle is simple: on a hot path, try to stay as high on the pyramid as possible. If the data fits on the stack, put it on the stack. If it does not, rent from a pool. If neither works, at least keep the allocation in gen0 and out of the Large Object Heap.

This article covers four techniques in that pyramid, each of which applies to a specific situation.

## Zoom: `Span<T>` and `stackalloc`

`Span<T>` was introduced in .NET Core 2.1 as the canonical abstraction over contiguous memory. It can point at a managed array, at a native pointer, at a portion of a string, or at stack-allocated memory, with the same API. Combined with `stackalloc`, it enables zero-allocation buffers for short-lived operations.

```csharp
public static bool IsValidIban(ReadOnlySpan<char> iban)
{
    if (iban.Length < 15 || iban.Length > 34) return false;

    // Stack-allocated buffer, no heap allocation at all.
    Span<char> rearranged = stackalloc char[iban.Length];
    iban[4..].CopyTo(rearranged);
    iban[..4].CopyTo(rearranged[^4..]);

    // Convert to digits, validated mod 97.
    Span<byte> digits = stackalloc byte[rearranged.Length * 2];
    int digitCount = 0;
    foreach (char c in rearranged)
    {
        if (char.IsDigit(c))
            digits[digitCount++] = (byte)(c - '0');
        else if (c is >= 'A' and <= 'Z')
        {
            int value = c - 'A' + 10;
            digits[digitCount++] = (byte)(value / 10);
            digits[digitCount++] = (byte)(value % 10);
        }
        else return false;
    }

    int remainder = 0;
    for (int i = 0; i < digitCount; i++)
        remainder = (remainder * 10 + digits[i]) % 97;

    return remainder == 1;
}
```

This method validates an IBAN without allocating a single byte on the heap. The `stackalloc` buffers live in the current stack frame and are reclaimed automatically when the method returns. The caller passes a `ReadOnlySpan<char>`, which can come from a string, a parsed request body, or another span, at no allocation cost.

> 💡 **Info** : `stackalloc` is safe inside a method that does not store the resulting span in a field or return it. The compiler enforces this via the `ref struct` rules of `Span<T>`. The stack buffer size should stay under roughly 1 KB to avoid risking a `StackOverflowException`. For larger buffers, use `ArrayPool<T>`.

> ✅ **Good practice** : Accept `ReadOnlySpan<char>` or `ReadOnlySpan<byte>` as method parameters instead of `string` or `byte[]`. Callers can pass slices of existing data without copying, and the method gains zero-allocation behavior by default.

## Zoom: `ArrayPool<T>` for rented buffers

When the required buffer is larger than a stack allocation should handle (say, 4 KB or more), `ArrayPool<T>.Shared` provides a managed pool of reusable arrays. Renting an array from the pool is much cheaper than allocating a new one, and returning it makes it available for the next caller.

```csharp
public static async Task<int> ReadAllToCountAsync(Stream input, CancellationToken ct)
{
    // Rent a 16 KB buffer from the shared pool. Zero heap allocation for this buffer.
    byte[] buffer = ArrayPool<byte>.Shared.Rent(16 * 1024);
    try
    {
        int total = 0;
        int read;
        while ((read = await input.ReadAsync(buffer, ct)) > 0)
            total += read;
        return total;
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

The `try/finally` is non-negotiable. Renting without returning leaks a buffer from the pool, which silently reduces its effectiveness. The standard pattern is always `rent → try → use → finally → return`.

> ⚠️ **It works, but...** : If the buffer might contain sensitive data (tokens, personally identifiable information), call `ArrayPool<T>.Shared.Return(buffer, clearArray: true)` to zero the memory before it goes back to the pool. Otherwise the next rental sees the old contents. The cost of clearing a 16 KB buffer is negligible compared to the security consequences of not clearing it.

## Zoom: `ValueTask` for the common case of "already complete"

Every `async` method that returns `Task` allocates at least one `Task` object, plus a state machine box if the method actually yields. For methods that frequently return synchronously (the cached value, the empty collection, the early return on a guard clause), that allocation is pure waste.

`ValueTask<T>` was added in .NET Core 2.0 specifically for this case. It is a value type that can represent either a completed result inline (zero allocation) or an underlying task (normal allocation). Used correctly, it eliminates allocations for the 80% of calls that complete synchronously.

```csharp
public sealed class PriceCache
{
    private readonly IDistributedCache _cache;
    private readonly IPriceRepository _repo;
    private readonly ConcurrentDictionary<string, decimal> _local = new();

    public ValueTask<decimal> GetPriceAsync(string sku, CancellationToken ct)
    {
        // Hot path: already in local cache, no async work, no allocation.
        if (_local.TryGetValue(sku, out var price))
            return new ValueTask<decimal>(price);

        // Cold path: go to the slower cache, real await, real allocation.
        return new ValueTask<decimal>(FetchAsync(sku, ct));
    }

    private async Task<decimal> FetchAsync(string sku, CancellationToken ct)
    {
        var bytes = await _cache.GetAsync(sku, ct);
        if (bytes is not null)
        {
            var cached = BitConverter.ToDecimal(bytes);
            _local[sku] = cached;
            return cached;
        }

        var fresh = await _repo.GetPriceAsync(sku, ct);
        _local[sku] = fresh;
        return fresh;
    }
}
```

In a typical pricing service where the local cache hits 95% of the time, this pattern eliminates 95% of the `Task<decimal>` allocations. At 100,000 requests per second, that is 95,000 saved allocations per second, compounded by the allocations the state machine box would have produced.

> ❌ **Never do this** : Do not `await` a `ValueTask` twice, or store it in a field, or call `.Result` on a not-yet-completed one. `ValueTask` is optimized for single-await consumption, and misuse can corrupt the underlying object or cause hangs. The safe pattern is `await ValueTaskMethod();` once, at the call site.

## Zoom: pooled objects with `ObjectPool<T>`

For objects that are more complex than a buffer (a `StringBuilder`, a custom parser state, a request context), `Microsoft.Extensions.ObjectPool` provides a pool that applications can use directly. It is the same mechanism ASP.NET Core uses internally for things like `StringBuilder` reuse in the pipeline.

```csharp
public sealed class ReportFormatter
{
    private readonly ObjectPool<StringBuilder> _builderPool;

    public ReportFormatter(ObjectPoolProvider provider)
    {
        _builderPool = provider.Create(
            new StringBuilderPooledObjectPolicy { MaximumRetainedCapacity = 16 * 1024 });
    }

    public string Format(Order order)
    {
        var sb = _builderPool.Get();
        try
        {
            sb.Append("Order ").Append(order.Id).Append(": ");
            foreach (var line in order.Lines)
                sb.Append(line.ProductName).Append(' ').Append(line.Quantity).Append(", ");

            return sb.ToString();
        }
        finally
        {
            _builderPool.Return(sb);  // policy.Return clears the builder
        }
    }
}
```

The same `rent → try → finally → return` pattern as `ArrayPool`, with a dedicated policy that bounds the retained capacity. The `MaximumRetainedCapacity` setting matters: a pool that keeps arbitrarily large `StringBuilder` instances defeats its own purpose by retaining memory for the worst-case request forever.

> 💡 **Info** : `ObjectPoolProvider` is registered by default in ASP.NET Core via `services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>()` (which ASP.NET Core does automatically). For console applications or workers, register it explicitly.

## Zoom: measuring the gain

Zero-allocation code only matters if it actually saves allocations. The only reliable way to verify this is BenchmarkDotNet with the `[MemoryDiagnoser]` attribute. It reports the allocations per operation, broken down by generation, alongside the runtime.

```csharp
[MemoryDiagnoser]
public class IbanValidationBench
{
    private static readonly string Iban = "FR7630006000011234567890189";

    [Benchmark(Baseline = true)]
    public bool Naive()
    {
        var sb = new StringBuilder();
        sb.Append(Iban.AsSpan(4));
        sb.Append(Iban.AsSpan(0, 4));
        var rearranged = sb.ToString();
        return Validate(rearranged);
    }

    [Benchmark]
    public bool ZeroAlloc() => IsValidIban(Iban.AsSpan());

    private static bool Validate(string s) { /* ... */ return true; }
}
```

A typical BenchmarkDotNet output for this comparison looks like:

```
|     Method |      Mean |   Allocated |
|----------- |----------:|------------:|
|      Naive | 412.7 ns  |       216 B |
|  ZeroAlloc |  89.3 ns  |         0 B |
```

Four to five times faster, zero bytes allocated, and the difference is directly attributable to the GC pressure that is no longer happening. Without the benchmark, the optimization is speculation. With it, the optimization is a measured gain worth shipping.

> ✅ **Good practice** : Run `[MemoryDiagnoser]` benchmarks as part of the repository, committed alongside the code they measure. When someone refactors the hot path six months later, the benchmark tells them immediately whether allocations crept back in.

## Zoom: when zero allocation is the wrong goal

Zero-allocation code is harder to read, harder to debug, and easier to get wrong. Applying it to a method that runs twice a minute is pure negative return on investment. Reach for it when:

- A [stress test](/posts/load-testing-stress/) shows GC time dominating the hot path.
- A [soak test](/posts/load-testing-soak/) shows heap pressure climbing, with a large fraction of time spent in collections.
- A BenchmarkDotNet profile shows an inner loop allocating per iteration on a path called thousands of times per second.
- Latency percentiles show a long tail that aligns with gen2 collection events in the GC logs.

Do *not* reach for it when:

- The method is not on a hot path. CRUD endpoints, admin operations, and background jobs rarely need it.
- Readability is the bottleneck. Code that one engineer understands today is often more valuable than code that runs 2% faster and nobody can modify.
- The allocations are unavoidable by design (serializing to JSON, rendering a full HTML page). Optimize the allocations that are actually optional.

## Wrap-up

Zero-allocation .NET is a precision tool, not a lifestyle. You can reach for `Span<T>` and `stackalloc` on short-lived buffers, `ArrayPool<T>` on larger ones, `ValueTask<T>` on async methods that often complete synchronously, and `ObjectPool<T>` on complex reusable objects. You can measure every change with `[MemoryDiagnoser]` benchmarks so the gains are real and do not regress silently. You can apply these techniques where a [stress test](/posts/load-testing-stress/) or a [soak test](/posts/load-testing-soak/) proves the GC is the bottleneck, and leave the rest of the codebase alone.

Ready to level up your next project or share it with your team? See you in the next one, AOT Compilation is where we go next.

## Related articles

- [Load Testing for .NET: An Overview of the Four Types That Matter](/posts/load-testing-overview/)
- [Stress Testing in .NET: Finding the Breaking Point and Its Shape](/posts/load-testing-stress/)
- [Soak Testing in .NET: The Bugs That Only Appear After Hours](/posts/load-testing-soak/)
- [Unit Testing in .NET: Fast, Focused, and Actually Useful](/posts/testing-unit-testing/)

## References

- [Memory and span-related types, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/)
- [ArrayPool&lt;T&gt;, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)
- [ValueTask&lt;T&gt; guidance, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.valuetask-1)
- [Object pool design pattern, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/performance/objectpool)
- [BenchmarkDotNet documentation](https://benchmarkdotnet.org/)
- [Garbage collection fundamentals, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)
