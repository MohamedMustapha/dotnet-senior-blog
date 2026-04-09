---
title: "Hosting ASP.NET Core with Docker: A Pragmatic Guide"
date: 2026-04-08
draft: false
tags: ["hosting", "docker", "dotnet"]
categories: ["Hosting"]
series: ["Hosting"]
description: "Multi-stage Dockerfiles, chiseled images, non-root containers, health checks, signal handling. Docker hosting for ASP.NET Core done right, without the 12 MB trim battle."
---

Containers changed .NET hosting more than any other technology of the last decade. Before Docker, shipping a .NET application meant producing an MSI, a Web Deploy package, or a ZIP file, and hoping the target environment had the right runtime installed. After Docker, shipping a .NET application means producing an image, and that image contains everything needed to run: the runtime, the application, the trimmed dependencies, nothing else. The image runs identically on a developer laptop, a CI agent, a pre-prod cluster, and a production host.

This article is not about "Docker in general". It is about hosting an ASP.NET Core application on Docker correctly in 2026, with the Microsoft base images that actually make sense, a multi-stage Dockerfile that produces a small and secure image, and the handful of configuration details that separate a working container from a production-ready one. If a previous article in this series covered [IIS](/posts/hosting-iis/) as the Windows-first option, this one covers the cross-platform default.

## Why Docker hosting

The advantages of containerizing a .NET application are well-known, but worth stating plainly because "we always did it this way" is a surprisingly common reason to still not be on Docker:

1. **Deterministic deployment.** The image built in CI is bit-for-bit identical to the image that runs in production. No "it worked on my machine", no "the base image was patched between builds", no "the runtime version drifted".
2. **Decoupling from the host OS.** The host needs a container runtime (containerd, Docker Engine, or a compatible alternative) and nothing else. No .NET Hosting Bundle, no IIS, no machine-wide dependency.
3. **A single deployment target.** The same image runs on a developer laptop, Kubernetes, Azure Container Apps, AWS ECS, a bare Docker host. The orchestrator changes; the image does not.
4. **Fast, scriptable operations.** Rolling updates, rollbacks, and blue/green deployments become simple orchestrator primitives instead of custom scripts.

For a new .NET project in 2026, the default hosting strategy is a container. The question is not whether to use Docker; it is how to build the image well.

## Overview: the image pipeline

{{< mermaid >}}
graph LR
    A[Source code] --> B[SDK image<br/>build stage]
    B --> C[Restore + Publish]
    C --> D[Runtime image<br/>final stage]
    D --> E[Application binary]
    D --> F[Metadata:<br/>ports, user, entrypoint]
    E --> G[Final image<br/>80-120 MB]
    F --> G
{{< /mermaid >}}

Every .NET Docker image worth shipping is built in two stages. The **build stage** uses a large SDK image (`mcr.microsoft.com/dotnet/sdk`) that contains the compiler, NuGet, and the tooling needed to produce a publish output. The **runtime stage** uses a much smaller image (`mcr.microsoft.com/dotnet/aspnet` or its chiseled variant) that contains only what is needed at runtime. The published output from the build stage is copied into the runtime stage, and the runtime stage is what ships.

This two-stage pattern is not optional. A single-stage image based on the SDK would be 700+ MB, which is fine for a developer playground and entirely wrong for production.

## Zoom: the canonical multi-stage Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.9
ARG DOTNET_VERSION=10.0

# --- Build stage ---
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build
WORKDIR /src

# Copy only the csproj first, restore, then copy the rest.
# This lets Docker cache the restore layer when nothing in csproj changes.
COPY ["Shop.Api/Shop.Api.csproj", "Shop.Api/"]
COPY ["Shop.Domain/Shop.Domain.csproj", "Shop.Domain/"]
COPY ["Shop.Application/Shop.Application.csproj", "Shop.Application/"]
COPY ["Shop.Infrastructure/Shop.Infrastructure.csproj", "Shop.Infrastructure/"]
RUN dotnet restore "Shop.Api/Shop.Api.csproj"

COPY . .
WORKDIR /src/Shop.Api
RUN dotnet publish "Shop.Api.csproj" \
    --configuration Release \
    --no-restore \
    --output /app/publish \
    /p:UseAppHost=false

# --- Runtime stage ---
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-noble-chiseled AS final
WORKDIR /app

# Copy the published output from the build stage.
COPY --from=build /app/publish .

# Non-root user is already set by the chiseled image.
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_RUNNING_IN_CONTAINER=true

ENTRYPOINT ["dotnet", "Shop.Api.dll"]
```

Three details make this a production-grade Dockerfile instead of a toy one.

**Layer caching on `csproj` first.** Copying only the `.csproj` files before the rest of the source lets Docker skip the (slow) `dotnet restore` step on subsequent builds when only application code has changed, not the dependencies. On a large solution, this cuts build times by an order of magnitude.

**Chiseled base image.** The `-noble-chiseled` suffix refers to Ubuntu 24.04 "Noble" chiseled images, which Microsoft publishes alongside the full runtime images. Chiseled images are built from Canonical's chisel tool, which slices Ubuntu packages to include only the files actually needed. A chiseled ASP.NET Core runtime image is around 100 MB instead of 220 MB for the full image, with no shell, no package manager, and a smaller attack surface.

**Non-root user by default.** Chiseled images run as a non-root user (`$APP_UID`, UID 64198) out of the box, which is a security posture that used to require an explicit `USER` directive. Running as root inside a container is a common mistake and a real risk, and the chiseled images solve it for you.

> 💡 **Info** : The full tag list for Microsoft's .NET base images lives at [mcr.microsoft.com/dotnet/aspnet](https://mcr.microsoft.com/en-us/artifact/mar/dotnet/aspnet/tags). Pin to a specific version (e.g., `10.0.0-noble-chiseled`) in production; use the major version tag (`10.0`) only in development.

## Zoom: the chiseled vs full image decision

Microsoft ships three relevant variants of the ASP.NET Core runtime image:

**Full image** (`aspnet:10.0`): Debian-based, with a shell, `apt`, and the common Linux userland. Around 220 MB. Use this when you need to install additional packages at build time or debug the container with a shell.

**Alpine image** (`aspnet:10.0-alpine`): Alpine Linux base, around 100 MB. Smaller than Debian, uses `musl` libc instead of glibc. Some native libraries that assume glibc will not work; most .NET code does. Lowest size for a conventional image.

**Chiseled image** (`aspnet:10.0-noble-chiseled`): Ubuntu chiseled, around 100 MB, no shell, no package manager, non-root by default. The most secure option and the one most production systems should default to.

The trade-off is debuggability. A chiseled image has no shell, which means `docker exec -it container bash` will not work. For production, this is a feature, not a bug: you should not be debugging from inside a running container, you should be collecting logs, metrics, and traces. For local development where you actually need a shell, switch to the full image temporarily.

> ✅ **Good practice** : Use the chiseled image by default and switch to the full image only when a specific scenario requires it (native dependency, debugging). Do not standardize on the full image "just in case".

## Zoom: health checks that actually work

An orchestrator (Docker Compose, Kubernetes, Azure Container Apps) uses health checks to decide whether a container is ready to receive traffic and whether it should be restarted. A broken or missing health check is how teams discover, in production, that their "zero downtime" rollout was not.

ASP.NET Core provides built-in health check support that pairs cleanly with container orchestration:

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddDbContextCheck<ShopDbContext>("database", tags: ["ready"])
    .AddCheck<RedisHealthCheck>("redis", tags: ["ready"]);

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Name == "self",
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
});
```

Two endpoints, two different purposes.

**`/health/live`** is the **liveness** check. It answers "is the process alive enough to respond to HTTP". If it fails, the orchestrator kills and restarts the container. It should *not* check database connectivity, because a transient database outage should not trigger a container restart storm.

**`/health/ready`** is the **readiness** check. It answers "is this instance ready to take traffic". If it fails, the orchestrator removes the instance from the load balancer until it recovers. This check *should* verify database and cache dependencies, because an instance that cannot talk to its database should not be serving requests.

In the Dockerfile, add the `HEALTHCHECK` directive only when running on plain Docker or Docker Compose. Kubernetes ignores the Dockerfile directive and uses its own `livenessProbe` and `readinessProbe`.

```dockerfile
HEALTHCHECK --interval=10s --timeout=2s --start-period=15s --retries=3 \
  CMD curl --fail http://localhost:8080/health/live || exit 1
```

> ⚠️ **It works, but...** : `curl` is not installed in the chiseled image. For Dockerfile-level health checks on chiseled images, either add the ASP.NET Core health checks library's ability to self-check via its own process, or switch the base image to one that includes a health check tool.

## Zoom: signal handling and graceful shutdown

When Docker (or any orchestrator) wants to stop a container, it sends `SIGTERM` to the process, waits up to 30 seconds (the default stop grace period), and then sends `SIGKILL` if the process has not exited. ASP.NET Core handles `SIGTERM` correctly out of the box: it stops accepting new connections, drains in-flight requests, flushes logs, and exits cleanly. For this to work, two details matter.

**The process must be PID 1 in the container.** The `ENTRYPOINT ["dotnet", "Shop.Api.dll"]` form runs the process directly as PID 1, which is what you want. The shell form (`ENTRYPOINT dotnet Shop.Api.dll` without the JSON array) runs it as a child of `/bin/sh`, which does not forward signals and breaks graceful shutdown.

**The grace period must be long enough for in-flight requests to complete.** For a web API, the default 30 seconds is usually fine. For long-running operations (file uploads, long-polling, WebSocket connections), configure the orchestrator to give more time, or implement a circuit breaker that stops accepting the long operations well before shutdown.

```csharp
// Program.cs: extend the graceful shutdown window to 45 seconds
builder.Services.Configure<HostOptions>(options =>
{
    options.ShutdownTimeout = TimeSpan.FromSeconds(45);
});
```

## Zoom: docker-compose for local development

A docker-compose file is the fastest path to a realistic local environment that mirrors production dependencies. It pairs especially well with the integration tests covered in the [TestContainers article](/posts/testing-integration-testing-testcontainers/), where production-identical images run inside the test process.

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Shop.Api/Dockerfile
    environment:
      ConnectionStrings__Default: "Host=postgres;Database=shop;Username=shop;Password=shop"
      Redis__Endpoint: "redis:6379"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    ports:
      - "8080:8080"

  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: shop
      POSTGRES_USER: shop
      POSTGRES_PASSWORD: shop
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shop"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

Three details worth knowing. The `depends_on` with `condition: service_healthy` means Compose will wait for Postgres to pass its health check before starting the API, avoiding the race condition where the app starts before the database is ready. The `volumes:` declaration for `pgdata` persists the database between `docker compose up` and `docker compose down`; use `docker compose down -v` to reset. The `ports: "8080:8080"` exposes the API to the host, which is what you want locally but should never end up in a production Compose file.

## Zoom: what not to put in the image

A production container image should contain **only** the application and its runtime dependencies. Things that should *never* be inside the image:

- **Secrets.** Connection strings, API keys, certificates, JWT signing keys. These belong in environment variables injected at runtime, or in a secret store (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault, Kubernetes Secrets).
- **Build tools.** The compiler, NuGet, debuggers. The multi-stage pattern keeps these in the build stage.
- **Test projects and test data.** Tests run in CI before the image is built; they do not belong in the deployed image.
- **Development configuration files.** `appsettings.Development.json` should either be excluded or copied only in non-production images.
- **Source code.** The runtime stage should copy the *publish output*, not the source. Shipping source to production is a common mistake and a security liability.

> ❌ **Never do this** : Do not bake secrets into the image at build time, even as environment variables in the Dockerfile. Anyone who pulls the image (including an attacker with read access to the registry) can recover them. Secrets belong at runtime, never at build time.

## Wrap-up

Hosting an ASP.NET Core application on Docker correctly in 2026 means a two-stage Dockerfile with aggressive layer caching, a chiseled base image for security and size, separate liveness and readiness health check endpoints, signal handling through PID 1 for graceful shutdown, and a docker-compose file that matches production dependencies for local development. You can ship a ~100 MB image, run as non-root, expose the right health endpoints for whatever orchestrator comes next, and keep secrets out of the image entirely.

Ready to level up your next project or share it with your team? See you in the next one, Hosting on Kubernetes is where we go next.

## Related articles

- [Hosting ASP.NET Core on IIS: The Classic, Demystified](/posts/hosting-iis/)
- [Integration Testing with TestContainers for .NET](/posts/testing-integration-testing-testcontainers/)
- [AOT Compilation in .NET: Startup, Size, and Trade-offs](/posts/performance-aot-compilation/)

## References

- [.NET Docker images on MCR](https://mcr.microsoft.com/en-us/artifact/mar/dotnet/aspnet/tags)
- [Chiseled Ubuntu images for .NET, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/docker/container-images)
- [Health checks in ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)
- [Docker Compose specification](https://docs.docker.com/compose/compose-file/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
