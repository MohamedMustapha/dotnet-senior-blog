---
title: "AOT Compilation in .NET: Startup, Size, and the Trade-offs Nobody Mentions"
date: 2026-04-08
draft: false
tags: ["performance", "aot", "dotnet"]
categories: ["Performance"]
series: ["Performance"]
description: "Native AOT, ReadyToRun, and the decision framework for when ahead-of-time compilation is worth the constraints. Startup time, binary size, memory footprint, reflection limits."
---

For twenty years, .NET ran through a JIT compiler. Managed code was loaded, verified, compiled to machine instructions on first call, and executed. The JIT produced excellent steady-state performance because it could observe actual runtime behavior and optimize accordingly. The cost was paid at startup: a cold process spent its first few seconds compiling hot paths before reaching peak speed, and the runtime itself had to ship inside every deployment. For a long-running web server, that was a trivial tax. For a serverless function invoked once and destroyed, a CLI tool, a container scaled from zero, or any workload where startup is the dominant cost, it was a problem.

Native AOT in .NET 7 (2022) solved that problem by producing a fully compiled, self-contained native binary at build time, with no JIT at runtime. The result is a .NET application that starts in tens of milliseconds, takes less memory, and ships without a runtime. The constraints, however, are significant, and a team that adopts Native AOT without understanding them will discover the hard way that reflection-heavy libraries, dynamic code generation, and parts of the BCL do not work the same way.

This article covers both sides: what AOT buys, what it costs, when to use it, and when [zero-allocation techniques](/posts/performance-zero-allocation/) or plain JIT are the better choice.

## Why AOT exists

The problem AOT solves is not raw throughput. On long-running workloads, a warm JIT-compiled process is often *faster* than an AOT-compiled one, because the JIT has profile-guided optimization, tiered compilation, and method inlining informed by actual call patterns. What AOT solves is the shape of the latency curve at process start and the size of what ships in the container.

Three concrete pain points drive AOT adoption:

1. **Cold start latency.** A standard .NET 10 web app takes several hundred milliseconds to start, even with ReadyToRun. An AOT-compiled version of the same app starts in 30 to 80 milliseconds. For a Lambda, an Azure Function, a Kubernetes pod that scales from zero, or a CLI tool that runs and exits, that difference is everything.
2. **Binary size.** A self-contained .NET app ships with a runtime that weighs 70 to 100 MB after trimming. A Native AOT binary for the same app can weigh 10 to 20 MB. In a container registry that hosts 500 images, or a CI pipeline that pulls images hundreds of times per day, the difference compounds quickly.
3. **Memory footprint.** A JIT-compiled process needs memory for the runtime, the compiler itself, and the compiled code. An AOT process needs none of that. Peak working set drops by 30 to 50% on typical workloads.

There is also a fourth, less talked-about reason: **deployment simplicity**. A native binary is a single file that runs on the target OS without any runtime installed. No "is this the right .NET version?", no "did the base image get patched?", no "why does this work on my machine and not on the server?". It runs or it does not, and if it runs once, it runs everywhere that shares the same OS and architecture.

## Overview: the AOT landscape

{{< mermaid >}}
graph TD
    A[.NET compilation options] --> B[JIT<br/>default]
    A --> C[ReadyToRun<br/>since .NET Core 3.0]
    A --> D[Native AOT<br/>since .NET 7]
    B --> B1[Best steady-state perf<br/>Slowest startup]
    C --> C1[Pre-JITted methods<br/>Still needs runtime<br/>~30% faster startup]
    D --> D1[No runtime<br/>Smallest size<br/>Fastest startup<br/>Reflection limits]
{{< /mermaid >}}

Three compilation models are available in .NET 10, each with a different trade-off.

**JIT** is the default and the right choice for the vast majority of workloads: web servers that stay up for hours or days, background workers, anything where steady-state performance matters more than startup. The JIT has profile-guided optimization (PGO) since .NET 6, tiered compilation since .NET Core 2.1, and produces code that can, in many benchmarks, beat the equivalent AOT output after warmup.

**ReadyToRun (R2R)** is the middle ground. It pre-compiles methods to native code at build time, then still uses the JIT at runtime for re-optimization and any methods that were not pre-compiled. R2R has been available since .NET Core 3.0 and is enabled by adding `<PublishReadyToRun>true</PublishReadyToRun>` to the csproj. It reduces startup latency by roughly 30% with minimal other changes. It is the low-risk first step for teams whose startup is painful but who cannot afford the AOT constraints.

**Native AOT** is the full commitment. It produces a single native binary with no runtime, no JIT, and no dynamic code generation. The cost is a set of constraints (covered below) that rule out entire categories of libraries. The benefit is the fastest startup, smallest size, and lowest memory footprint .NET can produce.

> 💡 **Info** : Native AOT was released in .NET 7 (2022) as a supported feature for ASP.NET Core minimal APIs, worker services, and console apps. Support expanded in .NET 8, 9, and 10 to cover more libraries and scenarios. In .NET 10, most of ASP.NET Core's common middleware and `System.Text.Json` work correctly under AOT with source generators.

## Zoom: enabling Native AOT

Switching a minimal API to Native AOT is a two-line change in the csproj and a handful of code adjustments:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>
</Project>
```

```csharp
// Program.cs
using System.Text.Json.Serialization;

var builder = WebApplication.CreateSlimBuilder(args);

builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default);
});

var app = builder.Build();

app.MapGet("/products/{id:int}", (int id) =>
    new Product(id, "SKU-42", 19.99m));

app.Run();

public record Product(int Id, string Sku, decimal Price);

[JsonSerializable(typeof(Product))]
internal partial class AppJsonContext : JsonSerializerContext { }
```

Two things to notice. `CreateSlimBuilder` replaces `CreateBuilder` and produces a trimmer-friendly host without the full MVC stack. `JsonSerializerContext` replaces runtime reflection-based JSON serialization with a compile-time source generator, which is the standard AOT-compatible approach.

```bash
dotnet publish -c Release -r linux-x64
```

The output is a single binary in `bin/Release/net10.0/linux-x64/publish/`, weighing typically 12 to 18 MB, with no runtime needed to run it.

> ✅ **Good practice** : Use `CreateSlimBuilder` from day one if AOT is on the roadmap. It makes the migration gradual instead of a single painful flip, and it pushes the team toward patterns that work under both JIT and AOT.

## Zoom: the constraints

This is the section that often gets skipped in AOT marketing. The trade-offs are real and they disqualify AOT for a meaningful portion of real-world codebases.

**No runtime reflection over arbitrary types.** The linker (IL trimmer) removes code it cannot statically prove is used. Any library that uses `Type.GetType(string)`, `Activator.CreateInstance(type)`, or `Assembly.Load` at runtime on types the linker did not see at build time will fail. This affects many popular libraries: older versions of AutoMapper, some IoC containers with runtime type discovery, certain serialization libraries. Newer versions of most of these have adopted source generators, but the migration is not complete across the ecosystem.

**No dynamic code generation.** `System.Reflection.Emit`, `Expression.Compile()`, and any library built on top of them (dynamic proxies, some ORMs, some mocking frameworks) do not work under AOT. Entity Framework Core has AOT support in .NET 8+, but with limitations on some query shapes. Moq and NSubstitute do not work, because they rely on runtime proxy generation.

**Source generators for JSON and others.** `System.Text.Json` reflection-based serialization does not work under AOT. Source-generated serialization does. Same for `Regex` (use source-generated regex via the `[GeneratedRegex]` attribute) and for logging (use source-generated logger methods).

**Invariant globalization.** Native AOT binaries default to invariant mode unless you explicitly ship ICU data. For many workloads this is fine. For any application that formats dates, numbers, or currency according to the user's locale, it is a constraint that has to be addressed.

**Longer build times.** A Native AOT build can take 30 seconds to several minutes per publish, compared to under 10 seconds for a JIT build. For inner-loop development, JIT is still the target; AOT is for the release pipeline.

> ❌ **Never do this** : Do not enable `PublishAot` on a mature codebase and run the publish as the first test. The build will fail in ways that are difficult to map back to the offending library. Instead, add AOT warnings at build time (`<IsAotCompatible>true</IsAotCompatible>`), fix them iteratively, and only switch `PublishAot` on once the warnings are clean.

## Zoom: ReadyToRun as the low-risk alternative

For teams that want startup improvement without AOT's constraints, ReadyToRun is often the right answer. It requires one property in the csproj and no code changes:

```xml
<PropertyGroup>
  <PublishReadyToRun>true</PublishReadyToRun>
  <PublishReadyToRunComposite>true</PublishReadyToRunComposite>
</PropertyGroup>
```

The composite option bundles the framework and application into a single R2R image, improving warmup further. Startup improves by roughly 30%, size increases by 10 to 20% (because the binary now contains both IL and native code), and nothing else changes. The JIT still runs at runtime for tier-1 recompilation, which means steady-state performance is preserved.

ReadyToRun is the sensible first step for any team where startup is a noticed problem but where the codebase depends on reflection, dynamic proxies, or anything else that AOT would reject.

> ⚠️ **It works, but...** : R2R improves cold start on methods it pre-compiled, which is typically the framework and the hot paths declared in the application. It does not help for methods that are never marked cold-compilable, which is a category that grows as the codebase grows. For extreme startup targets (under 100 ms), Native AOT is still the answer.

## Zoom: when AOT is the right call

Native AOT is the right choice for a specific set of workloads. Use it when at least two of the following are true:

- **The process is invoked frequently and short-lived.** Lambdas, Azure Functions, serverless APIs, CLI tools, scheduled jobs. These pay the cold-start tax on every invocation.
- **The container image size matters.** Multi-tenant systems that run thousands of images, CI pipelines that pull images often, edge deployments with bandwidth constraints.
- **The memory footprint per process is a cost driver.** Running hundreds of small instances per host, where every 20 MB saved per process is worth the migration effort.
- **The deployment target has no runtime installed.** Bare Linux containers, minimal distroless images, embedded systems.
- **The codebase is greenfield or small enough to migrate in a sprint.** AOT migration is much easier for a 50-file minimal API than for a 500-file monolith with twenty years of reflection-based patterns.

Do *not* use AOT when:

- The workload is a long-running web server that stays up for days. JIT with PGO will outperform AOT on steady state, and startup time is amortized over the process lifetime.
- The codebase depends heavily on reflection, dynamic proxies, or runtime-generated code. Either migrate those dependencies first, or accept that AOT is not the right tool for this codebase.
- The team has no appetite for breaking changes during migration. AOT exposes warnings and errors that a JIT build tolerates silently, and each warning is real work.

## Zoom: measuring the gain

Any AOT adoption should be backed by before/after measurements, not hope. Three numbers matter:

**Cold start time**, measured from process launch to first successful request. A simple harness that spawns the process, polls a health endpoint, and records the elapsed time. Repeat ten times and report the median.

**Peak resident memory**, measured during a steady 100 RPS run. Use `dotnet-counters` or `ps -o rss` and capture the max.

**Binary size**, measured on the published output directory (`du -sh` on Linux, `Get-ChildItem | Measure-Object -Property Length -Sum` on Windows).

```bash
# Before, JIT
$ time curl -s http://localhost:5000/health
real    0m0.480s    # ~480 ms cold start
$ du -sh ./publish
82M

# After, Native AOT
$ time curl -s http://localhost:5000/health
real    0m0.052s    #  ~52 ms cold start
$ du -sh ./publish
14M
```

Numbers like these justify the migration. Numbers that show only a 10% improvement do not, because the constraints that AOT imposes have a long tail of downstream cost. The decision should be data-driven.

> ✅ **Good practice** : Put the measurement harness in the repo as a dedicated script. When the runtime ships a new version, re-run the measurements. AOT is improving with every .NET release, and a codebase that was "not AOT-ready" in .NET 8 may well be ready in .NET 10 or 11.

## Wrap-up

Native AOT is the right answer for a specific class of .NET workloads where cold start latency, binary size, and memory footprint dominate, and where the codebase can accept the constraints on reflection and dynamic code generation. You can enable it with `<PublishAot>true</PublishAot>` and `CreateSlimBuilder`, adopt source generators for JSON, regex, and logging, fix AOT warnings iteratively before flipping the publish switch, and measure cold start, peak memory, and binary size before and after. You can fall back to ReadyToRun as a lower-risk intermediate, or stick with standard JIT when steady-state throughput is the actual metric that matters.

Ready to level up your next project or share it with your team? See you in the next one, a++ 👋

## Related articles

- [Zero Allocation in .NET: When the GC Becomes the Bottleneck](/posts/performance-zero-allocation/)
- [Load Testing for .NET: An Overview of the Four Types That Matter](/posts/load-testing-overview/)
- [Spike Testing in .NET: Surviving the Sudden Burst](/posts/load-testing-spike/)

## References

- [Native AOT deployment, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)
- [ASP.NET Core support for Native AOT, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/native-aot)
- [ReadyToRun compilation, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/deploying/ready-to-run)
- [System.Text.Json source generation, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/source-generation)
- [Trimming options, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/deploying/trimming/trimming-options)
