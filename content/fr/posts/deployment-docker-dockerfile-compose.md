---
title: "Docker pour le déploiement .NET : Dockerfile et Compose en pratique"
date: 2026-04-09
draft: false
tags: ["deployment", "docker", "dotnet"]
categories: ["Deployment"]
series: ["Deployment"]
description: "BuildKit cache mounts, builds multi-arch, docker bake, compose profiles, et les patterns concrets qui séparent un Dockerfile de CI d'un Dockerfile de tutoriel. Complément à l'article sur l'hébergement Docker."
---

Hello tous le monde, aujourd'hui on va explorer **le build et le déploiement Docker pour .NET**, la moitié de l'histoire qui n'est pas couverte par l'article sur l'hébergement : le pipeline lui-même.

L'[article de la série Hosting sur Docker](/fr/posts/hosting-docker/) a couvert comment faire tourner un container ASP.NET Core correctement au runtime : image de base chiseled, probes de santé, gestion des signaux, utilisateur non-root. Cet article regarde l'autre moitié, **le pipeline de build et de déploiement lui-même**. Un Dockerfile excellent au runtime peut rester terrible en CI s'il rebuilde tout depuis zéro à chaque commit, ne produit que du `linux/amd64` alors que la moitié des hôtes sont en `linux/arm64`, ou ne peut pas être composé dans une stack multi-services pour la staging.

L'objectif est concret : un `Dockerfile` prêt pour la prod qui utilise les BuildKit cache mounts pour transformer un build d'image de deux minutes en vingt secondes, une structure multi-stage qui joue bien avec la CI, un setup `docker bake` qui build des images multi-architectures en une seule commande, et un fichier `docker compose` réellement utilisable au-delà de `docker compose up` sur un poste de dev.

## Le contexte : pourquoi le pipeline de build compte

Un déploiement, ce n'est pas "le moment où le container tourne en prod". C'est tout ce qui se passe entre un `git push` et un replica sain qui sert du trafic, et le Dockerfile est la charnière de ce processus. Trois douleurs concrètes justifient l'attention :

1. **Les minutes de CI sont de l'argent réel.** Un Dockerfile qui refait le restore NuGet à chaque commit perd 60 à 120 secondes par run. Multiplié par 50 commits par jour, sur plusieurs branches, cela devient une part significative du budget CI qui part en travail redondant.
2. **Le multi-architecture n'est plus optionnel.** Les développeurs sur Apple Silicon en `arm64`, les providers cloud qui proposent des instances `arm64` moins chères (Graviton, Ampere, Azure Cobalt), et les appareils edge ont tous besoin de la même image dans plusieurs architectures. Un Dockerfile qui ne produit que du `amd64` commence à paraître daté très vite.
3. **Le déploiement est souvent multi-services.** Une API backend seule est rarement toute l'unité de déploiement. Il y a un worker, un reverse proxy, un scheduler de fond, un frontend. La composition fait partie de l'artefact de déploiement, et la traiter comme un détail mène à de la dérive entre les environnements.

## Vue d'ensemble : la forme du pipeline de build

{{< mermaid >}}
graph LR
    A[git push] --> B[Runner CI]
    B --> C[docker buildx<br/>BuildKit]
    C --> D[Layer de cache<br/>registry ou local]
    C --> E[Image multi-arch<br/>amd64 + arm64]
    E --> F[Registry container]
    F --> G[Cible de déploiement]
{{< /mermaid >}}

Trois outils portent l'essentiel du travail dans un déploiement container .NET moderne : **BuildKit** (le builder Docker moderne, défaut depuis Docker 23), **buildx** (le frontend CLI pour les builds multi-plateformes), et **bake** (un orchestrateur de build déclaratif qui remplace les scripts shell ad hoc).

Aucun n'est strictement obligatoire, mais ensemble ils transforment un pipeline de déploiement d'une séquence fragile d'appels `docker build` et `docker push` en un build reproductible, cacheable, et multi-cibles sur lequel une équipe peut raisonner.

## Zoom : le Dockerfile compatible CI

```dockerfile
# syntax=docker/dockerfile:1.9
ARG DOTNET_VERSION=10.0
ARG TARGETARCH

# --- Stage de build ---
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build
WORKDIR /src

# Copie des csproj d'abord pour maximiser les hits de cache sur le restore.
COPY ["Shop.Api/Shop.Api.csproj", "Shop.Api/"]
COPY ["Shop.Domain/Shop.Domain.csproj", "Shop.Domain/"]
COPY ["Shop.Application/Shop.Application.csproj", "Shop.Application/"]
COPY ["Shop.Infrastructure/Shop.Infrastructure.csproj", "Shop.Infrastructure/"]

# Cache mount BuildKit pour le dossier global-packages NuGet.
# Persiste entre les builds, donc le restore est quasi instantané sur un runner CI chaud.
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

# --- Stage runtime ---
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-noble-chiseled AS final
WORKDIR /app
COPY --from=build /app/publish .

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_RUNNING_IN_CONTAINER=true

ENTRYPOINT ["dotnet", "Shop.Api.dll"]
```

Cinq détails qui diffèrent du Dockerfile vu côté hébergement et qui ciblent spécifiquement le pipeline de build :

**`# syntax=docker/dockerfile:1.9` en tête** active la dernière frontend Dockerfile, ce qui autorise `--mount=type=cache` et les fonctionnalités de build plus récentes. Sans cette ligne, les versions plus anciennes de Docker interprètent le fichier avec une syntaxe plus restreinte.

**`--mount=type=cache,id=nuget,...`** est le cache mount BuildKit. Il persiste `/root/.nuget/packages` entre les builds sur la même instance de builder, pour que le deuxième build (et les suivants) saute entièrement le restore NuGet lent. Un runner CI froid paie encore le coût de téléchargement une fois ; un runner chaud restore en une seconde. L'identifiant partagé `id=nuget` permet aux étapes `restore` et `publish` d'utiliser le même cache.

**`--platform=$BUILDPLATFORM` sur le stage de build** garde la compilation sur l'architecture native de l'hôte (rapide) même quand la sortie vise une autre architecture. L'alternative, lancer tout le build sous émulation, est 3 à 5 fois plus lente sur `amd64 → arm64`.

**`-a $TARGETARCH` sur `dotnet restore` et `--arch $TARGETARCH` sur `dotnet publish`** disent au SDK .NET de produire la sortie pour l'architecture cible alors que le build lui-même tourne sur l'architecture hôte. C'est la façon `.NET` de faire du cross-compile et c'est significativement plus rapide que l'émulation.

**Le stage final n'a pas de `--platform` explicite**, il hérite donc de la plateforme cible depuis le flag `docker buildx build --platform`. Le résultat est un manifeste multi-arch où le runtime de chaque architecture correspond à sa cible, sans overhead d'émulation.

> 💡 **Info** : Les cache mounts BuildKit persistent par instance de builder, pas par image. Sur un runner CI avec un workspace persistant (GitHub Actions avec cache, GitLab CI avec runner partagé), le cache survit entre les jobs. Sur un runner éphémère, utiliser un cache basé sur le registry avec `--cache-to type=registry,...` pour l'externaliser.

## Zoom : builds multi-architectures avec buildx

Une seule commande produit une image multi-arch et la pousse :

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=myregistry.azurecr.io/shop-api:cache \
  --cache-to type=registry,ref=myregistry.azurecr.io/shop-api:cache,mode=max \
  --tag myregistry.azurecr.io/shop-api:1.4.7 \
  --push \
  .
```

Le flag `--platform linux/amd64,linux/arm64` dit à buildx de builder les deux architectures en parallèle. Les flags `--cache-from` et `--cache-to` externalisent le cache BuildKit vers le registry container, ce qui est le pattern qui marche sur des runners CI éphémères. Le flag `--push` pousse directement le manifeste résultant ; sans lui, on obtient une image multi-arch locale qui ne peut pas être inspectée avec `docker images`.

Le registry stocke ensuite un **manifest list** : un seul tag (`1.4.7`) qui pointe vers deux images (une `amd64`, une `arm64`), et n'importe quel runtime qui tire le tag récupère l'architecture dont il a réellement besoin. C'est transparent pour Kubernetes, ACA, Azure Web App, et tout runtime moderne.

> ✅ **Bonne pratique** : Tagger les images avec à la fois une version et un alias `cache` dans le même registry. Le tag de version (`1.4.7`) est immuable et avance à chaque release ; le tag `cache` n'est utilisé que par le builder. Cela garde le cache de build séparé des artefacts de release et simplifie le garbage collection.

## Zoom : où placer le Dockerfile dans une solution .NET

Une question qui revient tôt dans tout projet .NET utilisant Docker : où vit le `Dockerfile`, et qu'est-ce qui constitue le build context ? Deux approches couvrent la grande majorité des cas.

### Approche A : Dockerfile à la racine de la solution (recommandé pour la plupart des projets)

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

Le build context est la racine de la solution, ce qui permet à chaque instruction `COPY` du Dockerfile de référencer n'importe quel projet naturellement (`COPY ["src/Shop.Domain/Shop.Domain.csproj", "src/Shop.Domain/"]`). C'est la disposition la plus simple pour une solution avec une seule unité déployable et plusieurs bibliothèques de classes. Le fichier `.dockerignore` se place à côté du Dockerfile et exclut tout ce dont le build n'a pas besoin :

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

Un bon `.dockerignore` fait gagner des secondes sur le build en réduisant le contexte envoyé au daemon Docker, et il empêche de laisser fuiter des fichiers comme `.env` ou l'historique `.git` dans les layers de l'image.

### Approche B : Dockerfile par projet (microservices ou monorepo)

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

Chaque service a son propre Dockerfile, mais le build context reste la racine de la solution. Dans `docker-compose.yaml`, cela donne :

```yaml
services:
  api:
    build:
      context: .
      dockerfile: src/Shop.Api/Dockerfile
```

Le `context: .` garantit que les références de projets partagés (`Shop.Domain`, `Shop.Application`) sont accessibles pendant le build, même si le Dockerfile se trouve dans `src/Shop.Api/`. C'est la disposition standard pour les monorepos ou les solutions produisant plusieurs images container.

> ⚠️ **Ça marche, mais...** : Placer le Dockerfile dans le dossier d'un projet ET utiliser ce dossier comme build context (`context: src/Shop.Api/`) casse le `COPY` pour tout projet partagé comme `Shop.Domain.csproj`, car le build context ne peut pas voir les fichiers au-dessus de lui. Il faut toujours définir le context à la racine de la solution et utiliser la directive `dockerfile:` pour pointer vers le Dockerfile du projet.

## Zoom : le compose naïf vs le compose prêt pour la prod

La plupart des équipes commencent avec un fichier Compose qui marche sur un poste de dev et s'arrêtent là. Voici à quoi cela ressemble :

```yaml
# compose.yaml — la version naïve
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

Cela fait tourner l'application, et il n'y a rien de mal à commencer ainsi. Les problèmes apparaissent quand ce même fichier est utilisé pour la staging ou la prod : les secrets sont codés en dur dans le contrôle de version, il n'y a pas de healthcheck donc `api` peut démarrer avant que Postgres ne soit prêt, pas de limites de ressources donc une fuite mémoire peut faire tomber l'hôte, pas de rotation de logs donc `/var/lib/docker` se remplit en quelques semaines, pas de politique de redémarrage donc un crash à 3h du matin reste down jusqu'à ce que quelqu'un le remarque, et la directive `build:` signifie que chaque déploiement rebuild l'image depuis les sources au lieu de tirer un artefact testé depuis le registry.

Le chemin de ce point de départ vers un Compose prêt pour la prod suit quatre étapes.

### 1. Séparer le build du run

En production, `image:` remplace `build:`. Le pipeline CI build et pousse l'image vers le registry ; le fichier Compose sur la cible de déploiement ne fait que la tirer. Cela garantit que la même image qui a passé les tests en CI est celle qui tourne en production.

```yaml
services:
  api:
    image: myregistry.azurecr.io/shop-api:${VERSION}
```

### 2. Extraire les secrets dans des fichiers env

Plutôt que de coder en dur les chaînes de connexion et mots de passe, il suffit de référencer des variables et de les fournir au runtime :

```yaml
services:
  api:
    environment:
      ConnectionStrings__Default: "Host=postgres;Database=shop;Username=${DB_USER};Password=${DB_PASSWORD}"
```

Les valeurs vivent dans un fichier `.env.production` qui est gitignoré :

```env
# .env.production — JAMAIS commité
DB_USER=shop_app
DB_PASSWORD=r4nd0m-g3n3r4t3d-v4lu3
VERSION=1.4.7
```

### 3. Plusieurs fichiers env pour plusieurs environnements

Chaque environnement reçoit son propre fichier env, le même fichier Compose :

```bash
# Staging
docker compose --env-file .env.staging up -d

# Production
docker compose --env-file .env.production up -d
```

Pas de copier-coller de fichiers Compose par environnement. La topologie reste identique ; seules les valeurs changent.

### 4. Fichiers override pour les différences structurelles

Certaines différences entre le dev et la prod ne sont pas juste des valeurs mais de la structure : le dev a besoin de `build:` et de ports exposés pour le debugging, la prod a besoin de `image:` et de limites de ressources via `deploy:`. Les fichiers override de Compose gèrent cela proprement :

```yaml
# compose.yaml — base, partagé par tous les environnements
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
# compose.override.yaml — overrides dev, chargé automatiquement
services:
  api:
    build: .
    ports:
      - "5000:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: Development
```

```yaml
# compose.prod.yaml — overrides production, chargé explicitement
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
# Dev : utilise compose.yaml + compose.override.yaml automatiquement
docker compose up

# Prod : override explicite
docker compose -f compose.yaml -f compose.prod.yaml --env-file .env.production up -d
```

> ✅ **Bonne pratique** : `compose.override.yaml` est chargé automatiquement par `docker compose up` quand aucun flag `-f` n'est spécifié, ce qui en fait l'endroit naturel pour les réglages dev-only. Les déploiements en production utilisent toujours des flags `-f` explicites, donc l'override n'est jamais inclus par accident.

## Zoom : docker bake pour des builds déclaratifs

Lancer la commande `docker buildx build` depuis un Makefile ou un YAML CI marche, mais devient vite moche quand un repository a plusieurs images (API, worker, UI admin) avec une configuration de base partagée. `docker bake` remplace les incantations shell par un fichier HCL :

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
# Build les trois targets pour les deux architectures, avec cache partagé.
VERSION=1.4.7 docker buildx bake --push
```

Une seule commande build les trois images pour les deux architectures, partage le cache entre elles, et pousse tout. Le target `_common` contient la configuration partagée, et `inherits = ["_common"]` sur chaque image évite la répétition. Un pipeline de build qui faisait 150 lignes de shell se réduit à 30 lignes de HCL plus une seule invocation.

> ⚠️ **Ça marche, mais...** : `docker bake` est puissant mais pas encore universel. Certains providers CI ne l'ont pas installé par défaut, et certaines versions de Docker plus anciennes ont besoin d'un `docker buildx install` d'abord. Vérifier l'environnement CI avant de standardiser sur bake, ou l'installer dans une étape de préparation du pipeline.

## Zoom : docker compose pour le déploiement multi-services

`docker compose` est largement utilisé pour le dev local (couvert dans l'[article hosting](/fr/posts/hosting-docker/)), mais c'est aussi une cible de déploiement légitime pour des systèmes petits à moyens. Un seul hôte Linux avec Docker Engine, qui fait tourner un fichier Compose, peut servir du vrai trafic de prod pour des outils internes, des environnements de staging, ou de petits produits SaaS.

La clé est un fichier Compose conscient de son environnement, pas codé en dur pour "mon poste" :

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
# Déploiement
VERSION=1.4.7 docker compose up -d

# Mise à jour vers une nouvelle version
VERSION=1.4.8 docker compose up -d  # Compose tire la nouvelle image et recrée uniquement l'api
```

Sept détails font de ce Compose un Compose de déploiement.

**`${VERSION:-latest}`** pour la substitution pilote le tag d'image depuis une variable d'environnement, ce qui permet d'utiliser le même fichier pour plusieurs versions sans l'éditer. `restart: unless-stopped` redémarre automatiquement en cas d'échec ou de reboot. `healthcheck` donne à Docker une façon de savoir quand le container est réellement prêt. `deploy.resources.limits` plafonne CPU et mémoire. La configuration `logging` tourne les logs de container pour éviter le remplissage du disque. Les variables d'environnement pour les secrets viennent d'un fichier env ou du shell, jamais codées en dur. Un reverse proxy (Caddy ici, pourrait être Traefik ou NGINX) gère la terminaison TLS avec des certificats Let's Encrypt automatiques.

Pour des systèmes plus grands qu'un seul hôte, Compose n'est pas la bonne réponse et les prochains articles de la série (et l'[article hosting Kubernetes](/fr/posts/hosting-kubernetes/)) couvrent le chemin de migration.

> ✅ **Bonne pratique** : Garder les secrets dans un fichier `.env` gitignoré, et les charger avec `docker compose --env-file prod.env up -d`. Compose substitue les variables au lancement, et le fichier `.env` ne rejoint jamais le contrôle de version. Pour des garanties plus fortes, utiliser les Docker secrets (en mode Swarm) ou externaliser vers un store de secrets.

## Zoom : compose profiles pour les variantes d'environnement

Un seul fichier Compose peut décrire plusieurs variantes d'environnement via les profils :

```yaml
services:
  api: { ... }
  postgres: { ... }

  # Démarre uniquement avec --profile debug
  adminer:
    image: adminer:latest
    ports: ["8081:8080"]
    profiles: ["debug"]

  # Démarre uniquement avec --profile monitoring
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    profiles: ["monitoring"]
```

```bash
docker compose up -d                                        # api + postgres seulement
docker compose --profile debug up -d                        # + adminer
docker compose --profile monitoring up -d                   # + prometheus
docker compose --profile debug --profile monitoring up -d   # tout
```

Les profils permettent à un seul fichier de servir plusieurs environnements : prod classique, prod-avec-observabilité, dev-avec-ui-admin. L'alternative de maintenir trois fichiers Compose séparés mène à de la dérive entre eux ; les profils les gardent en phase.

## Wrap-up

Construire et déployer des containers .NET proprement en 2026 signifie un Dockerfile qui utilise les cache mounts BuildKit pour garder les builds CI rapides, le flag `--platform` pour produire des images multi-architectures sans overhead d'émulation, `docker buildx` ou `docker bake` pour orchestrer des builds multi-images de façon déclarative, et un fichier Compose assez conscient de son environnement pour servir d'artefact de déploiement réel sur des systèmes petits à moyens. Tu peux réduire les temps de build CI de moitié juste avec les cache mounts, livrer des images multi-arch en une seule commande, et garder la topologie de déploiement dans un seul fichier versionné lu à la fois par le pipeline et le runtime.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Héberger ASP.NET Core avec Docker : un guide pragmatique](/fr/posts/hosting-docker/)
- [Héberger ASP.NET Core sur Kubernetes : l'essentiel pour les devs .NET](/fr/posts/hosting-kubernetes/)
- [Tests d'Intégration avec TestContainers pour .NET](/fr/posts/testing-integration-testing-testcontainers/)

## Références

- [Référence Dockerfile](https://docs.docker.com/reference/dockerfile/)
- [Cache mounts BuildKit, docs Docker](https://docs.docker.com/build/cache/backends/)
- [Builds multi-plateformes, docs Docker](https://docs.docker.com/build/building/multi-platform/)
- [Référence docker bake](https://docs.docker.com/build/bake/reference/)
- [Spécification Docker Compose](https://docs.docker.com/compose/compose-file/)
- [Cross-targeting pour la CLI .NET, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/core/tools/dotnet-publish)
