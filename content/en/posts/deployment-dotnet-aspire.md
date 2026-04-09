---
title: ".NET Aspire: Cloud-Native Orchestration Made Simple"
date: 2026-04-09
draft: false
tags: ["deployment", "aspire", "dotnet"]
categories: ["Deployment"]
series: ["Deployment"]
description: "AppHost, service defaults, resource composition, the Aspire dashboard, and the path from F5 on a laptop to a production deployment on Azure Container Apps, without writing a single line of YAML."
---

For a decade, the gap between "my .NET solution runs on my laptop" and "my .NET solution is deployed to a cloud platform" has been filled with tooling the developer had to assemble themselves: a docker-compose for local orchestration, a separate set of Kubernetes or ACA manifests for deployment, OpenTelemetry wiring per service, a dashboard to watch traces, a way to pass connection strings to containers. Every team reinvented the same scaffolding, slightly differently, and the friction kept .NET microservices more expensive to start than they needed to be.

**.NET Aspire** closes that gap. Released as GA in May 2024, it is Microsoft's opinionated framework for composing, running, and deploying multi-service .NET applications. It is not a new hosting platform. It is a C#-first orchestration layer that sits on top of the hosting you already use ([Docker](/posts/hosting-docker/), [Kubernetes](/posts/hosting-kubernetes/), [ACA](/posts/hosting-azure-container-apps/)), replacing hand-written YAML and shell scripts with a typed AppHost project that describes the entire topology in C#. For a lot of .NET teams, especially those starting new distributed applications, it removes a significant amount of boilerplate without locking anyone into a specific cloud.

This final article of the [Deployment series](/posts/deployment-docker-dockerfile-compose/) covers what Aspire actually is, how to use it as a development and deployment tool, and when it is the right choice.

## Why .NET Aspire exists

Aspire answers a specific observation: every non-trivial .NET application in 2024 looked the same in its orchestration layer. It had an API, a worker, a database, a cache, maybe a message broker. Each service needed OpenTelemetry configured, a connection string wired up, health checks registered, and a way to run locally against the same dependencies. Teams wrote the same twenty boilerplate lines per service, forever, and each team wrote them slightly differently.

The goals of Aspire, stated concretely:

1. **Replace docker-compose with a typed C# model.** The topology of the application (what services run, what they depend on, what they talk to) is described in a regular .NET project called the **AppHost**, with strong typing, IntelliSense, and refactoring support.
2. **Standardize cross-cutting concerns.** OpenTelemetry, health checks, service discovery, resilience policies, and structured logging are packaged into **Service Defaults**, a shared project that every service in the solution references. Add the reference, call one extension method, and you have them all.
3. **Provide a local dashboard.** When you press F5, Aspire starts all the services and opens a local dashboard that shows traces, metrics, logs, and the console output of each process, in one place.
4. **Emit deployment manifests for real targets.** The same AppHost can generate the manifests needed to deploy to Azure Container Apps, Kubernetes, or Docker Compose, without the developer writing them by hand. This is the part that replaces the "I have to maintain three different deployment descriptions" problem.

## Overview: the Aspire project shape

{{< mermaid >}}
graph TD
    A[AppHost project<br/>C# topology] --> B[Shop.Api]
    A --> C[Shop.Worker]
    A --> D[Postgres resource]
    A --> E[Redis resource]
    A --> F[Azure Service Bus]
    B --> D
    B --> E
    C --> D
    C --> F
    G[ServiceDefaults project<br/>OTel, health, resilience] --> B
    G --> C
    H[Aspire Dashboard] --> B
    H --> C
    H --> D
{{< /mermaid >}}

An Aspire solution has a distinctive shape. Two new projects sit alongside the usual service projects:

**AppHost**: a console project that references every service project in the solution and declares, in C#, the resources each one depends on. When you run the AppHost, it launches all the referenced projects, starts the dependencies (Postgres, Redis, whatever), and wires the connection strings between them automatically.

**ServiceDefaults**: a class library that every service project references. It contains the extension methods that wire up OpenTelemetry, health check endpoints, service discovery, and resilience policies in a single call. Instead of copy-pasting 30 lines of telemetry setup into every `Program.cs`, you call `builder.AddServiceDefaults()` and it is done.

The rest of the solution (the API project, the worker project, the domain library) is regular .NET code, unchanged. Aspire does not ask you to restructure your application. It adds orchestration on top.

## Zoom: the AppHost project

```csharp
// Shop.AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

// Managed dependencies. Aspire starts these automatically in dev mode.
var postgres = builder.AddPostgres("db")
    .WithDataVolume()
    .AddDatabase("shopdb");

var redis = builder.AddRedis("cache")
    .WithDataVolume();

var servicebus = builder.AddAzureServiceBus("sb")
    .AddQueue("orders-inbound");

// The API project, with explicit references to its dependencies.
var api = builder.AddProject<Projects.Shop_Api>("shop-api")
    .WithReference(postgres)
    .WithReference(redis)
    .WithReference(servicebus)
    .WithExternalHttpEndpoints()
    .WithReplicas(2);

// The worker project.
builder.AddProject<Projects.Shop_Worker>("shop-worker")
    .WithReference(postgres)
    .WithReference(servicebus);

builder.Build().Run();
```

Twelve lines of C# describe the entire topology of a distributed application. Five things worth noticing.

**`AddPostgres("db")` with `WithDataVolume()`** does not just spin up a container. It declares Postgres as a managed resource in the AppHost, persists its data across runs via a Docker volume, and exposes its connection string to any project that calls `WithReference(postgres)`. The `AddDatabase("shopdb")` call creates the database inside the Postgres instance automatically.

**`AddAzureServiceBus("sb")`** is an interesting case. In development mode, Aspire runs an emulator (based on a container) that speaks the Service Bus protocol. In production, the same AppHost descriptor maps to a real Azure Service Bus namespace. The application code does not change between the two; Aspire resolves the difference at deployment time.

**`WithReference(postgres)`** is the magic. It takes the connection string that Aspire constructs for the managed Postgres and injects it into the referenced project as an environment variable, following the same naming convention ASP.NET Core uses (`ConnectionStrings__db`). The project then reads it from `IConfiguration` without any extra glue.

**`WithExternalHttpEndpoints()`** marks the project as externally reachable. In local dev, Aspire assigns a random port and shows it in the dashboard. In production, it maps to an ingress rule on the target platform.

**`WithReplicas(2)`** declares how many instances of the project should run. In local dev, Aspire launches two copies and load-balances between them. In production, the number translates into replica count on Kubernetes or ACA.

> 💡 **Info** : Aspire's catalog of `Add*` methods covers most common dependencies out of the box: Postgres, SQL Server, MySQL, Redis, MongoDB, RabbitMQ, Kafka, Azure Service Bus, Azure Storage, Azure Cosmos DB, Azure Key Vault, and more. The full list is in the `Aspire.Hosting.*` NuGet packages. Third-party integrations (Dapr, NATS, Elastic) are available as community packages.

## Zoom: the ServiceDefaults project

Every service in the solution references a shared `ServiceDefaults` project that provides the common cross-cutting setup:

```csharp
// Shop.ServiceDefaults/Extensions.cs
public static class Extensions
{
    public static IHostApplicationBuilder AddServiceDefaults(this IHostApplicationBuilder builder)
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();
        builder.Services.AddServiceDiscovery();

        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();
            http.AddServiceDiscovery();
        });

        return builder;
    }

    public static IHostApplicationBuilder ConfigureOpenTelemetry(this IHostApplicationBuilder builder)
    {
        builder.Logging.AddOpenTelemetry(logging =>
        {
            logging.IncludeFormattedMessage = true;
            logging.IncludeScopes = true;
        });

        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics =>
            {
                metrics.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation()
                       .AddRuntimeInstrumentation();
            })
            .WithTracing(tracing =>
            {
                tracing.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation();
            });

        builder.AddOpenTelemetryExporters();
        return builder;
    }
}
```

And in each service's `Program.cs`:

```csharp
// Shop.Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

builder.Services.AddDbContext<ShopDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("shopdb")));

var app = builder.Build();

app.MapDefaultEndpoints();  // /health/live, /health/ready
app.MapOrdersEndpoints();
app.Run();
```

Two calls: `AddServiceDefaults()` in `ConfigureServices` and `MapDefaultEndpoints()` in the pipeline. Every service now has OpenTelemetry wired to the dashboard, health check endpoints, service discovery via DNS, and resilient HTTP clients with retry and circuit breaker. No copy-paste. No drift. If the team decides to add a new telemetry exporter or a new resilience policy, it happens in one place.

> ✅ **Good practice** : Keep `ServiceDefaults` under strict review. It is the blast radius for every service's startup behavior. Changes to it affect every service at once, which is exactly what makes it valuable and exactly what makes it dangerous. Treat it like a shared library with its own release notes.

## Zoom: the Aspire Dashboard

When you press F5 on the AppHost, Aspire starts the dashboard on a local port and opens it in the browser. The dashboard shows:

- **Resources**: every service and dependency, with their status, ports, environment variables, and container logs.
- **Console logs**: a unified view of stdout/stderr from every running process, with filtering by service and log level.
- **Structured logs**: the ILogger entries, indexed and searchable.
- **Traces**: OpenTelemetry spans, with distributed tracing across services. A single request that hits the API, queries Postgres, publishes to Service Bus, and triggers the worker shows as a single trace with all the spans.
- **Metrics**: the runtime counters (GC, thread pool, HTTP request duration) and any custom metrics the application emits.

This is, for many teams, the most visible benefit of adopting Aspire. Getting the same level of local observability without Aspire requires running Jaeger, Prometheus, Grafana, and a log aggregator in a compose file, configuring each one, and making sure every service exports to the right endpoint. Aspire does all of it by default, in process, with zero configuration.

> 💡 **Info** : The Aspire Dashboard is a standalone application. It can also run against any OpenTelemetry-compatible workload (including non-Aspire apps) via the standalone image at `mcr.microsoft.com/dotnet/aspire-dashboard`. Some teams adopt it as their local observability stack even when they are not using the rest of Aspire.

## Zoom: deploying an Aspire app

Aspire is not a hosting platform. It generates manifests or resources for a real hosting platform. The canonical deployment path uses the **Azure Developer CLI** (`azd`) to deploy an Aspire solution to Azure Container Apps with a single command.

```bash
# Once, at the root of the solution
azd init                     # interactive wizard, detects the AppHost
azd auth login               # authenticates with Azure

# Every deployment after that
azd up                       # provisions Azure resources and deploys
```

Under the hood, `azd up` does three things:

1. **Provisions infrastructure.** From the AppHost description, `azd` generates a Bicep template that creates the required Azure resources: a Container Apps Environment, a Log Analytics workspace, a Service Bus namespace (because the AppHost references one), a Postgres Flexible Server, and so on.
2. **Builds the container images** for each service project in the solution, using the standard .NET SDK container publish (`dotnet publish -t:PublishContainer`), and pushes them to an Azure Container Registry that `azd` also provisions.
3. **Deploys the Container Apps** with the right environment variables, secrets, ingress configuration, and replica counts, derived from the AppHost.

The whole round trip, from `git clone` to a running production-like environment on Azure, is typically under 10 minutes on a fresh account.

For teams targeting Kubernetes instead, Aspire can emit a manifest via `aspire publish`:

```bash
aspire publish --publisher kubernetes --output ./deploy/k8s
```

This generates Kubernetes manifests for every service in the AppHost, which can then be further customized with Kustomize (covered in the [Kubernetes primer](/posts/deployment-kubernetes-primer/)) or packaged with Helm. The generated output is a starting point, not the final artifact, but it captures the dependency graph and the environment wiring, which is the tedious part.

> ⚠️ **It works, but...** : `azd up` is excellent for development, demos, and proof of concept environments. For production, most teams move to a proper CI/CD pipeline with separate build, test, and deploy stages, using the Aspire manifest as an input to their existing deployment tooling rather than calling `azd up` from a workstation.

## Zoom: when Aspire is the right choice

Aspire is particularly well-suited for:

- **New distributed .NET applications** where the team wants a fast on-ramp to multi-service development without assembling the scaffolding from scratch.
- **Existing solutions that struggle with cross-cutting concerns.** If the team has five services and each has a slightly different OpenTelemetry setup, moving them all under a shared `ServiceDefaults` is a net win.
- **Teams that want local observability without running a parallel compose stack** for Jaeger, Prometheus, and friends.
- **Azure-first .NET shops.** The `azd` deployment path is the smoothest experience on Azure. It works elsewhere, but the rough edges are fewer on Azure.
- **Demos, workshops, and internal tools** where fast F5-to-running is more important than deployment flexibility.

It is *not* the right choice when:

- **The solution is a single service.** Aspire's value comes from orchestrating multiple services. For a single API, the AppHost is overhead without benefit.
- **The team has a mature deployment pipeline.** If there is already a working Kubernetes + Helm + GitOps setup, introducing Aspire as the authoring layer may create friction rather than reduce it.
- **Non-.NET services are part of the topology.** Aspire can reference containers or executables of any language, but its strongest integration is with .NET projects. A polyglot system with heavy Python, Go, or Node.js services may fit better in a compose-first or Kubernetes-first workflow.
- **The target is not Azure and not Kubernetes.** Aspire can generate compose files, but its strongest deployment paths are ACA and K8s. For bare VMs, [IIS](/posts/hosting-iis/), or plain Docker hosts, the benefit is smaller.

## Wrap-up

.NET Aspire replaces the "every team reinvents the same scaffolding" pattern with a typed, C#-first orchestration layer that describes the full topology in the AppHost project, standardizes observability and resilience via ServiceDefaults, provides a local dashboard for free, and generates deployment manifests for Azure Container Apps, Kubernetes, or Docker Compose. You can start a new distributed .NET application with two extra projects and a handful of lines of code, get traces and metrics on the dashboard without wiring anything, and deploy to ACA with `azd up` in minutes. You can also recognize when an existing solution would not benefit from the migration and stick with the hosting and deployment tools already in place.

Ready to level up your next project or share it with your team? See you in the next one, a++ 👋

## Related articles

- [Hosting ASP.NET Core with Docker: A Pragmatic Guide](/posts/hosting-docker/)
- [Hosting ASP.NET Core on Kubernetes: The Essentials for .NET Developers](/posts/hosting-kubernetes/)
- [Hosting ASP.NET Core on Azure Container Apps](/posts/hosting-azure-container-apps/)
- [Docker for .NET Deployment: Dockerfile and Compose in Practice](/posts/deployment-docker-dockerfile-compose/)
- [Kubernetes Primer for .NET Developers: From kubectl to Helm](/posts/deployment-kubernetes-primer/)

## References

- [.NET Aspire documentation, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/aspire/)
- [.NET Aspire overview](https://learn.microsoft.com/en-us/dotnet/aspire/get-started/aspire-overview)
- [Azure Developer CLI documentation](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/)
- [Aspire Dashboard standalone](https://learn.microsoft.com/en-us/dotnet/aspire/fundamentals/dashboard/standalone)
- [.NET Aspire on GitHub](https://github.com/dotnet/aspire)
