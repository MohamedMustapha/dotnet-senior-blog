---
title: "Docker for .NET Deployment: Dockerfile and Compose in Practice"
date: 2026-04-09
draft: false
tags: ["deployment", "docker", "dotnet"]
categories: ["Deployment"]
series: ["Deployment"]
description: "BuildKit cache mounts, multi-arch builds, docker bake, compose profiles, and the concrete patterns that separate a CI Dockerfile from a hobby one. Complements the hosting-side Docker article."
---

The [Hosting series article on Docker](/posts/hosting-docker/) covered how to run an ASP.NET Core container correctly at runtime: chiseled base image, health probes, signal handling, non-root user. This article looks at the other half of the story, **the build and deployment pipeline itself**. A Dockerfile that is great at runtime can still be terrible in CI if it rebuilds everything from scratch on every commit, produces only `linux/amd64` when half your hosts are `linux/arm64`, or cannot be composed into a multi-service stack for staging.

The goal here is concrete: a production-grade `Dockerfile` that uses BuildKit cache mounts to turn a two-minute image build into a 20-second one, a multi-stage structure that plays nicely with CI, a `docker bake` setup that builds multi-architecture images in a single command, and a `docker compose` file that is actually usable beyond `docker compose up` on a laptop.

## Why the build pipeline matters

A deployment is not "the moment the container runs in production". It is everything between a `git push` and a healthy replica serving traffic, and the Dockerfile is the hinge of that process. Three concrete pain points make this worth the attention:

1. **CI minutes are real money.** A Dockerfile that rebuilds NuGet restore on every commit wastes 60 to 120 seconds per run. Multiplied by 50 commits per day, across branches, that is a significant chunk of the CI budget going to redundant work.
2. **Multi-architecture is no longer optional.** Apple Silicon developers on `arm64`, cloud providers offering cheaper `arm64` instances (Graviton, Ampere, Azure Cobalt), and edge devices all need the same image in multiple architectures. A Dockerfile that only produces `amd64` starts to feel legacy very quickly.
3. **Deployment is often multi-service.** A backend API alone is rarely the whole unit of deployment. There is a worker, a reverse proxy, a background scheduler, a frontend. The composition is part of the deployment artifact, and treating it as an afterthought leads to drift between environments.

## Overview: the build pipeline shape

{{< mermaid >}}
graph LR
    A[git push] --> B[CI runner]
    B --> C[docker buildx<br/>BuildKit]
    C --> D[Cache layer<br/>registry or local]
    C --> E[Multi-arch image<br/>amd64 + arm64]
    E --> F[Container registry]
    F --> G[Deployment target]
{{< /mermaid >}}

Three tools carry most of the weight in a modern .NET container deployment: **BuildKit** (the modern Docker builder, default since Docker 23), **buildx** (the CLI frontend for multi-platform builds), and **bake** (a declarative build orchestrator that replaces ad-hoc shell scripts).

None of these are strictly required, but together they turn a deployment pipeline from a fragile sequence of `docker build` and `docker push` calls into a reproducible, cacheable, multi-target build that a team can reason about.

## Zoom: the CI-friendly Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.9
ARG DOTNET_VERSION=10.0
ARG TARGETARCH

# --- Build stage ---
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build
WORKDIR /src

# Copy csproj files first to maximize layer cache hits on restore.
COPY ["Shop.Api/Shop.Api.csproj", "Shop.Api/"]
COPY ["Shop.Domain/Shop.Domain.csproj", "Shop.Domain/"]
COPY ["Shop.Application/Shop.Application.csproj", "Shop.Application/"]
COPY ["Shop.Infrastructure/Shop.Infrastructure.csproj", "Shop.Infrastructure/"]

# BuildKit cache mount for the NuGet global-packages folder.
# Persists across builds, so restore is near-instant on warm CI runners.
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet restore "Shop.Api/Shop.Api.csproj" \
        -a $TARGETARCH

COPY . .
WORKDIR /src/Shop.Api
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet publish "Shop.Api.csproj" \
        --configuration Release \
        --no-restore \
        --arch $TARGETARCH \
        --output /app/publish \
        /p:UseAppHost=false

# --- Runtime stage ---
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-noble-chiseled AS final
WORKDIR /app
COPY --from=build /app/publish .

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_RUNNING_IN_CONTAINER=true

ENTRYPOINT ["dotnet", "Shop.Api.dll"]
```

Five details that differ from the hosting-side Dockerfile and specifically target the build pipeline:

**`# syntax=docker/dockerfile:1.9` at the top** opts into the latest Dockerfile frontend, which is what enables `--mount=type=cache` and the newer build features. Without it, older Docker versions interpret the file with a more restricted syntax.

**`--mount=type=cache,id=nuget,...`** is the BuildKit cache mount. It persists `/root/.nuget/packages` across builds on the same builder instance, so the second and subsequent builds skip the slow NuGet restore entirely. A cold CI runner still pays the download cost once; a warm one restores in a second. The `id=nuget` shared identifier lets both the `restore` and `publish` steps use the same cache.

**`--platform=$BUILDPLATFORM` on the build stage** keeps compilation on the native host architecture (fast) even when producing cross-architecture output. The alternative, running the full build under emulation, is 3-5x slower on `amd64 → arm64`.

**`-a $TARGETARCH` on `dotnet restore` and `--arch $TARGETARCH` on `dotnet publish`** tells the .NET SDK to produce output for the target architecture even though the build itself runs on the host architecture. This is the `.NET` way of doing cross-compilation and is significantly faster than emulation.

**Final stage has no `--platform` override**, so it inherits the target platform from the `docker buildx build --platform` flag. The end result is a multi-arch manifest where each architecture's runtime matches its target, without emulation overhead.

> 💡 **Info** : BuildKit cache mounts persist per builder instance, not per image. On a CI runner with a persistent workspace (GitHub Actions with cache, GitLab CI with shared runner), the cache survives between jobs. On an ephemeral runner, use a registry-backed cache with `--cache-to type=registry,...` to externalize it.

## Zoom: multi-architecture builds with buildx

A single command produces a multi-arch image and pushes it:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=myregistry.azurecr.io/shop-api:cache \
  --cache-to type=registry,ref=myregistry.azurecr.io/shop-api:cache,mode=max \
  --tag myregistry.azurecr.io/shop-api:1.4.7 \
  --push \
  .
```

The `--platform linux/amd64,linux/arm64` flag tells buildx to build for both architectures in parallel. The `--cache-from` and `--cache-to` flags externalize the BuildKit cache to the container registry, which is the pattern that works on ephemeral CI runners. The `--push` flag pushes the resulting manifest directly; without it, you get a local multi-arch image that cannot be inspected with `docker images`.

The registry then stores a **manifest list**: a single tag (`1.4.7`) that points to two images (one `amd64`, one `arm64`), and any runtime pulling the tag gets the architecture it actually needs. This is transparent to Kubernetes, ACA, Azure Web App, and any modern runtime.

> ✅ **Good practice** : Tag images with both a version and a `cache` alias in the same registry. The version tag (`1.4.7`) is immutable and rolled forward on each release; the `cache` tag is used only by the builder. This keeps the build cache separate from release artifacts and makes garbage collection simpler.

## Zoom: where to place the Dockerfile in a .NET solution

A question that comes up early in every .NET project using Docker: where does the `Dockerfile` live, and what counts as the build context? Two approaches cover the vast majority of cases.

### Approach A: Dockerfile at solution root (recommended for most projects)

```
Shop.sln
Dockerfile
docker-compose.yaml
.dockerignore
src/
├── Shop.Api/
├── Shop.Domain/
├── Shop.Application/
└── Shop.Infrastructure/
```

The build context is the solution root, so every `COPY` instruction in the Dockerfile can reference any project naturally (`COPY ["src/Shop.Domain/Shop.Domain.csproj", "src/Shop.Domain/"]`). This is the simplest layout for a solution with one deployable unit and several class libraries. The `.dockerignore` file sits next to the Dockerfile and excludes everything the build does not need:

```
**/.git
**/bin
**/obj
**/node_modules
**/*.md
**/*.sln.DotSettings
.env*
docker-compose*.yaml
```

A good `.dockerignore` shaves seconds off the build by shrinking the context sent to the Docker daemon, and it prevents leaking files like `.env` or `.git` history into the image layers.

### Approach B: Dockerfile per project (microservices or monorepo)

```
Shop.sln
docker-compose.yaml
src/
├── Shop.Api/
│   └── Dockerfile
├── Shop.Worker/
│   └── Dockerfile
└── Shop.Admin/
    └── Dockerfile
```

Each service has its own Dockerfile, but the build context is still the solution root. In `docker-compose.yaml`, this looks like:

```yaml
services:
  api:
    build:
      context: .
      dockerfile: src/Shop.Api/Dockerfile
```

The `context: .` ensures that shared project references (`Shop.Domain`, `Shop.Application`) are accessible during the build, even though the Dockerfile lives inside `src/Shop.Api/`. This is the standard layout for monorepos or solutions producing multiple container images.

> ⚠️ **It works, but...** : Placing the Dockerfile inside a project folder AND using that folder as the build context (`context: src/Shop.Api/`) breaks `COPY` for any shared project like `Shop.Domain.csproj`, because the build context cannot see files above itself. Always set the context to the solution root and use the `dockerfile:` directive to point at the per-project Dockerfile.

## Zoom: the naive compose vs production-ready compose

Most teams start with a Compose file that works on a laptop and call it done. Here is what that looks like:

```yaml
# compose.yaml — the naive version
services:
  api:
    build: .
    ports:
      - "5000:8080"
    environment:
      ConnectionStrings__Default: "Host=postgres;Database=shop;Username=admin;Password=supersecret"
      ASPNETCORE_ENVIRONMENT: Development
  postgres:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD: supersecret
```

This gets the application running, and there is nothing wrong with it as a starting point. The problems show up when this same file is used for staging or production: secrets are hardcoded in version control, there is no healthcheck so `api` may start before Postgres is ready, no resource limits so a memory leak can take down the host, no log rotation so `/var/lib/docker` fills up over weeks, no restart policy so a crash at 3 AM stays down until someone notices, and the `build:` directive means every deploy rebuilds the image from source instead of pulling a tested artifact from the registry.

The path from that starting point to a production-ready Compose file follows four steps.

### 1. Separate build from run

In production, `image:` replaces `build:`. The CI pipeline builds and pushes the image to the registry; the Compose file on the deployment target only pulls it. This guarantees the same image that passed tests in CI is the one running in production.

```yaml
services:
  api:
    image: myregistry.azurecr.io/shop-api:${VERSION}
```

### 2. Extract secrets to env files

Instead of hardcoding connection strings and passwords, reference variables and supply them at runtime:

```yaml
services:
  api:
    environment:
      ConnectionStrings__Default: "Host=postgres;Database=shop;Username=${DB_USER};Password=${DB_PASSWORD}"
```

The values live in a `.env.production` file that is gitignored:

```env
# .env.production — NEVER committed
DB_USER=shop_app
DB_PASSWORD=r4nd0m-g3n3r4t3d-v4lu3
VERSION=1.4.7
```

### 3. Multiple env files for multiple environments

Different environments get different env files, same Compose file:

```bash
# Staging
docker compose --env-file .env.staging up -d

# Production
docker compose --env-file .env.production up -d
```

No copy-pasting of Compose files per environment. The topology stays identical; only the values change.

### 4. Override files for structural differences

Some differences between dev and prod are not just values but structure: dev needs `build:` and exposed ports for debugging, prod needs `image:` and `deploy:` resource limits. Compose override files handle this cleanly:

```yaml
# compose.yaml — base, shared by all environments
services:
  api:
    image: myregistry.azurecr.io/shop-api:${VERSION:-latest}
    restart: unless-stopped
    environment:
      ASPNETCORE_ENVIRONMENT: ${ASPNETCORE_ENVIRONMENT:-Production}
      ConnectionStrings__Default: ${DB_CONNECTION}
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      retries: 5

volumes:
  pgdata:
```

```yaml
# compose.override.yaml — dev overrides, loaded automatically
services:
  api:
    build: .
    ports:
      - "5000:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: Development
```

```yaml
# compose.prod.yaml — production overrides, loaded explicitly
services:
  api:
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

```bash
# Dev: uses compose.yaml + compose.override.yaml automatically
docker compose up

# Prod: explicit override
docker compose -f compose.yaml -f compose.prod.yaml --env-file .env.production up -d
```

> ✅ **Good practice** : `compose.override.yaml` is loaded automatically by `docker compose up` when no `-f` flag is specified, which makes it the natural place for dev-only settings. Production deployments always use explicit `-f` flags, so the override is never accidentally included.

## Zoom: docker bake for declarative builds

Running the `docker buildx build` command from a Makefile or CI YAML works, but it gets ugly when a repository has multiple images (API, worker, admin UI) with shared base configuration. `docker bake` replaces the shell incantations with an HCL file:

```hcl
# docker-bake.hcl
variable "VERSION" { default = "dev" }
variable "REGISTRY" { default = "myregistry.azurecr.io" }

group "default" {
  targets = ["api", "worker", "admin"]
}

target "_common" {
  platforms = ["linux/amd64", "linux/arm64"]
  cache-from = ["type=registry,ref=${REGISTRY}/shop-cache:latest"]
  cache-to = ["type=registry,ref=${REGISTRY}/shop-cache:latest,mode=max"]
  args = {
    DOTNET_VERSION = "10.0"
  }
}

target "api" {
  inherits = ["_common"]
  context = "."
  dockerfile = "Shop.Api/Dockerfile"
  tags = ["${REGISTRY}/shop-api:${VERSION}"]
}

target "worker" {
  inherits = ["_common"]
  context = "."
  dockerfile = "Shop.Worker/Dockerfile"
  tags = ["${REGISTRY}/shop-worker:${VERSION}"]
}

target "admin" {
  inherits = ["_common"]
  context = "."
  dockerfile = "Shop.Admin/Dockerfile"
  tags = ["${REGISTRY}/shop-admin:${VERSION}"]
}
```

```bash
# Build all three targets for both architectures, with shared cache.
VERSION=1.4.7 docker buildx bake --push
```

One command builds the three images for both architectures, shares the cache across them, and pushes everything. The `_common` target holds shared configuration, and `inherits = ["_common"]` on each image avoids the repetition. A build pipeline that was 150 lines of shell shrinks to 30 lines of HCL plus a single invocation.

> ⚠️ **It works, but...** : `docker bake` is powerful but not yet universal. Some CI providers do not have it installed by default, and some older Docker versions need `docker buildx install` first. Check the CI environment before standardizing on bake, or bake a warmup step into the pipeline.

## Zoom: docker compose for multi-service deployment

`docker compose` is widely used for local development (covered in the [hosting article](/posts/hosting-docker/)), but it is also a legitimate deployment target for small-to-medium systems. A single Linux host with Docker Engine, running a Compose file, can serve real production traffic for internal tools, staging environments, or small SaaS products.

The key is a Compose file that is environment-aware, not hard-coded for "my laptop":

```yaml
# compose.yaml
services:
  api:
    image: myregistry.azurecr.io/shop-api:${VERSION:-latest}
    restart: unless-stopped
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__Default: ${DB_CONNECTION}
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health/live"]
      interval: 10s
      timeout: 2s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  postgres:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      retries: 5

  reverse-proxy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
    depends_on:
      - api

volumes:
  pgdata:
  caddy_data:
```

```bash
# Deploy
VERSION=1.4.7 docker compose up -d

# Update to a new version
VERSION=1.4.8 docker compose up -d  # Compose pulls the new image and recreates only the api
```

Seven details make this a deployment-grade Compose file.

**`${VERSION:-latest}`** substitution drives the image tag from an environment variable, enabling the same file for multiple versions without editing it. `restart: unless-stopped` auto-restarts on failure or reboot. `healthcheck` gives Docker a way to know when the container is actually ready. `deploy.resources.limits` caps CPU and memory. `logging` configuration rotates container logs to prevent disk fill. Environment variables for secrets come from an env file or from the shell, never hardcoded. A reverse proxy (Caddy here, could be Traefik or NGINX) handles TLS termination with automatic Let's Encrypt certificates.

For systems larger than a single host, Compose is the wrong answer and the next article in this series (and the [hosting Kubernetes article](/posts/hosting-kubernetes/)) covers the migration path.

> ✅ **Good practice** : Keep secrets in a `.env` file that is gitignored, and load them with `docker compose --env-file prod.env up -d`. Compose substitutes the variables at launch time, and the `.env` file never reaches version control. For stronger guarantees, use Docker secrets (in Swarm mode) or externalize to a secret store.

## Zoom: compose profiles for environment variants

A single Compose file can describe multiple environment variants using profiles:

```yaml
services:
  api: { ... }
  postgres: { ... }

  # Only starts with --profile debug
  adminer:
    image: adminer:latest
    ports: ["8081:8080"]
    profiles: ["debug"]

  # Only starts with --profile monitoring
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    profiles: ["monitoring"]
```

```bash
docker compose up -d                                        # api + postgres only
docker compose --profile debug up -d                        # + adminer
docker compose --profile monitoring up -d                   # + prometheus
docker compose --profile debug --profile monitoring up -d   # everything
```

Profiles let one file serve several environments: plain production, production-with-observability, dev-with-admin-ui. The alternative of maintaining three separate Compose files leads to drift between them; profiles keep them in sync.

## Wrap-up

Building and deploying .NET containers well in 2026 means a Dockerfile that uses BuildKit cache mounts to keep CI builds fast, the `--platform` flag to produce multi-architecture images without emulation overhead, `docker buildx` or `docker bake` to orchestrate multi-image builds declaratively, and a Compose file that is environment-aware enough to serve as a real deployment artifact for small-to-medium systems. You can cut CI build times in half with cache mounts alone, ship multi-arch images in a single command, and keep your deployment topology in one version-controlled file that is read by both the pipeline and the runtime.

Ready to level up your next project or share it with your team? See you in the next one, Docker Security Best Practices is where we go next.

## Related articles

- [Hosting ASP.NET Core with Docker: A Pragmatic Guide](/posts/hosting-docker/)
- [Hosting ASP.NET Core on Kubernetes: The Essentials for .NET Developers](/posts/hosting-kubernetes/)
- [Integration Testing with TestContainers for .NET](/posts/testing-integration-testing-testcontainers/)

## References

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [BuildKit cache mounts, Docker docs](https://docs.docker.com/build/cache/backends/)
- [Multi-platform builds, Docker docs](https://docs.docker.com/build/building/multi-platform/)
- [docker bake file reference](https://docs.docker.com/build/bake/reference/)
- [Docker Compose specification](https://docs.docker.com/compose/compose-file/)
- [Cross-targeting for .NET CLI, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-publish)
