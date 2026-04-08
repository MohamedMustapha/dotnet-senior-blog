---
title: "Héberger ASP.NET Core avec Docker : un guide pragmatique"
date: 2026-04-08
draft: false
tags: ["hosting", "docker", "dotnet"]
categories: ["Hosting"]
series: ["Hosting"]
description: "Dockerfile multi-stage, images chiseled, containers non-root, health checks, gestion des signaux. L'hébergement Docker pour ASP.NET Core fait correctement, sans la bataille à 12 Mo."
---

Hello tous le monde, aujourd'hui on va explorer l'**hébergement d'ASP.NET Core avec Docker**, en visant un résultat propre et production-ready plutôt qu'un Dockerfile de tutoriel.

Les containers ont changé l'hébergement .NET plus que n'importe quelle autre technologie de la dernière décennie. Avant Docker, livrer une application .NET signifiait produire un MSI, un package Web Deploy ou un ZIP, et espérer que l'environnement cible avait le bon runtime installé. Après Docker, livrer une application .NET signifie produire une image, et cette image contient tout ce qu'il faut pour tourner : le runtime, l'application, les dépendances trimmées, rien d'autre. L'image tourne à l'identique sur un poste de dev, un agent de CI, un cluster de pré-prod, et un hôte de prod.

Cet article ne traite pas de "Docker en général". Il traite de l'hébergement correct d'une application ASP.NET Core sur Docker en 2026, avec les images de base Microsoft qui ont réellement du sens, un Dockerfile multi-stage qui produit une image petite et sécurisée, et la poignée de détails de configuration qui séparent un container qui marche d'un container prêt pour la prod. Là où un article précédent de la série a couvert [IIS](/fr/posts/hosting-iis/) comme option Windows-first, celui-ci couvre le défaut cross-platform.

## Le contexte : pourquoi héberger sur Docker

Les avantages de containeriser une application .NET sont bien connus, mais valent la peine d'être rappelés clairement parce que "on a toujours fait comme ça" est une raison étonnamment courante de ne pas être encore sur Docker :

1. **Déploiement déterministe.** L'image construite en CI est bit pour bit identique à l'image qui tourne en prod. Plus de "ça marche sur ma machine", plus de "l'image de base a été patchée entre deux builds", plus de "la version du runtime a bougé".
2. **Découplage de l'OS hôte.** L'hôte a besoin d'un runtime de container (containerd, Docker Engine, ou une alternative compatible) et de rien d'autre. Pas de .NET Hosting Bundle, pas d'IIS, pas de dépendance installée sur la machine.
3. **Une seule cible de déploiement.** La même image tourne sur un poste de dev, sur Kubernetes, Azure Container Apps, AWS ECS, ou un hôte Docker nu. L'orchestrateur change ; l'image ne change pas.
4. **Des opérations rapides et scriptables.** Les rolling updates, les rollbacks, et les déploiements blue/green deviennent des primitives simples de l'orchestrateur plutôt que des scripts custom.

Pour un nouveau projet .NET en 2026, la stratégie d'hébergement par défaut est un container. La question n'est pas "faut-il utiliser Docker", c'est "comment construire l'image proprement".

## Vue d'ensemble : le pipeline d'image

{{< mermaid >}}
graph LR
    A[Code source] --> B[Image SDK<br/>stage de build]
    B --> C[Restore + Publish]
    C --> D[Image runtime<br/>stage final]
    D --> E[Binaire applicatif]
    D --> F[Métadonnées :<br/>ports, user, entrypoint]
    E --> G[Image finale<br/>80-120 Mo]
    F --> G
{{< /mermaid >}}

Toute image Docker .NET qui vaut la peine d'être livrée est construite en deux stages. Le **stage de build** utilise une grosse image SDK (`mcr.microsoft.com/dotnet/sdk`) qui contient le compilateur, NuGet, et l'outillage nécessaire pour produire une sortie de publish. Le **stage runtime** utilise une image beaucoup plus petite (`mcr.microsoft.com/dotnet/aspnet` ou sa variante chiseled) qui ne contient que ce qui est nécessaire à l'exécution. La sortie publiée du stage de build est copiée dans le stage runtime, et le stage runtime est celui qui part en prod.

Ce pattern à deux stages n'est pas optionnel. Une image à un seul stage basée sur le SDK ferait 700 Mo et plus, ce qui est acceptable pour une playground de dev et complètement inadapté à la prod.

## Zoom : le Dockerfile multi-stage canonique

```dockerfile
# syntax=docker/dockerfile:1.9
ARG DOTNET_VERSION=10.0

# --- Stage de build ---
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build
WORKDIR /src

# Copie uniquement les csproj d'abord, restore, puis copie le reste.
# Permet à Docker de cacher le layer restore quand rien dans csproj n'a changé.
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

# --- Stage runtime ---
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-noble-chiseled AS final
WORKDIR /app

# Copie la sortie publiée depuis le stage de build.
COPY --from=build /app/publish .

# L'utilisateur non-root est déjà positionné par l'image chiseled.
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_RUNNING_IN_CONTAINER=true

ENTRYPOINT ["dotnet", "Shop.Api.dll"]
```

Trois détails font de ce Dockerfile un Dockerfile prêt pour la prod plutôt qu'un Dockerfile de tutoriel.

**Le layer cache sur `csproj` d'abord.** Copier uniquement les fichiers `.csproj` avant le reste du code source permet à Docker de sauter l'étape (lente) `dotnet restore` sur les builds suivants quand seul le code applicatif a changé, pas les dépendances. Sur une grosse solution, cela réduit les temps de build d'un ordre de grandeur.

**L'image de base chiseled.** Le suffixe `-noble-chiseled` fait référence aux images Ubuntu 24.04 "Noble" chiseled, que Microsoft publie à côté des images runtime complètes. Les images chiseled sont construites avec l'outil chisel de Canonical, qui découpe les packages Ubuntu pour n'inclure que les fichiers réellement nécessaires. Une image runtime ASP.NET Core chiseled fait environ 100 Mo contre 220 Mo pour l'image complète, sans shell, sans gestionnaire de paquets, et avec une surface d'attaque plus petite.

**Utilisateur non-root par défaut.** Les images chiseled tournent sous un utilisateur non-root (`$APP_UID`, UID 64198) nativement, ce qui est une posture de sécurité qui nécessitait autrefois une directive `USER` explicite. Tourner en root à l'intérieur d'un container est une erreur courante et un vrai risque, et les images chiseled résolvent ce point pour le développeur.

> 💡 **Info** : La liste complète des tags des images de base .NET de Microsoft vit sur [mcr.microsoft.com/dotnet/aspnet](https://mcr.microsoft.com/en-us/artifact/mar/dotnet/aspnet/tags). Pinner sur une version spécifique (ex. `10.0.0-noble-chiseled`) en prod ; utiliser le tag de version majeure (`10.0`) uniquement en dev.

## Zoom : la décision chiseled vs full image

Microsoft publie trois variantes pertinentes de l'image runtime ASP.NET Core :

**Image complète** (`aspnet:10.0`) : basée sur Debian, avec un shell, `apt`, et l'userland Linux courant. Environ 220 Mo. À utiliser quand il faut installer des packages supplémentaires au build ou debugger le container avec un shell.

**Image Alpine** (`aspnet:10.0-alpine`) : base Alpine Linux, environ 100 Mo. Plus petite que Debian, utilise `musl` libc au lieu de glibc. Certaines bibliothèques natives qui présupposent glibc ne fonctionneront pas ; la plupart du code .NET fonctionne. Taille la plus petite pour une image conventionnelle.

**Image chiseled** (`aspnet:10.0-noble-chiseled`) : Ubuntu chiseled, environ 100 Mo, pas de shell, pas de gestionnaire de paquets, non-root par défaut. L'option la plus sécurisée et celle vers laquelle la plupart des systèmes de prod devraient tendre par défaut.

Le compromis, c'est la debuggabilité. Une image chiseled n'a pas de shell, ce qui veut dire que `docker exec -it container bash` ne marchera pas. Pour la prod, c'est une fonctionnalité, pas un bug : le debug ne devrait pas se faire depuis l'intérieur d'un container qui tourne, mais via la collecte de logs, de métriques et de traces. Pour le dev local où un shell est réellement nécessaire, basculer temporairement sur l'image complète.

> ✅ **Bonne pratique** : Utiliser l'image chiseled par défaut et basculer sur l'image complète uniquement quand un scénario précis le demande (dépendance native, debug). Ne pas standardiser sur l'image complète "au cas où".

## Zoom : des health checks qui marchent vraiment

Un orchestrateur (Docker Compose, Kubernetes, Azure Container Apps) utilise les health checks pour décider si un container est prêt à recevoir du trafic et s'il doit être redémarré. Un health check absent ou cassé, c'est comme ça que les équipes découvrent, en prod, que leur rollout "zero downtime" n'en était pas un.

ASP.NET Core fournit un support natif pour les health checks, qui s'appaire proprement avec l'orchestration de containers :

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

Deux endpoints, deux rôles différents.

**`/health/live`** est le check de **liveness**. Il répond à "le process est-il assez vivant pour répondre en HTTP". S'il échoue, l'orchestrateur tue et redémarre le container. Il ne doit *pas* vérifier la connectivité à la base, parce qu'une panne transitoire de la base ne doit pas déclencher une tempête de redémarrages de containers.

**`/health/ready`** est le check de **readiness**. Il répond à "cette instance est-elle prête à prendre du trafic". S'il échoue, l'orchestrateur retire l'instance du load balancer jusqu'à ce qu'elle récupère. Ce check *doit* vérifier la base et les dépendances de cache, parce qu'une instance qui ne peut pas parler à sa base ne doit pas servir de requêtes.

Dans le Dockerfile, ajouter la directive `HEALTHCHECK` uniquement quand on tourne sur Docker nu ou Docker Compose. Kubernetes ignore la directive du Dockerfile et utilise ses propres `livenessProbe` et `readinessProbe`.

```dockerfile
HEALTHCHECK --interval=10s --timeout=2s --start-period=15s --retries=3 \
  CMD curl --fail http://localhost:8080/health/live || exit 1
```

> ⚠️ **Ça marche, mais...** : `curl` n'est pas installé dans l'image chiseled. Pour des health checks au niveau Dockerfile sur des images chiseled, soit ajouter la capacité de la bibliothèque ASP.NET Core health checks à se vérifier via son propre process, soit basculer l'image de base sur une qui contient un outil de health check.

## Zoom : gestion des signaux et arrêt gracieux

Quand Docker (ou n'importe quel orchestrateur) veut arrêter un container, il envoie `SIGTERM` au process, attend jusqu'à 30 secondes (la période de grâce par défaut), puis envoie `SIGKILL` si le process n'est pas sorti. ASP.NET Core gère `SIGTERM` correctement nativement : il cesse d'accepter de nouvelles connexions, draine les requêtes en vol, flush les logs, et sort proprement. Pour que cela fonctionne, deux détails comptent.

**Le process doit être PID 1 dans le container.** La forme `ENTRYPOINT ["dotnet", "Shop.Api.dll"]` lance le process directement comme PID 1, ce qui est l'objectif. La forme shell (`ENTRYPOINT dotnet Shop.Api.dll` sans le tableau JSON) le lance comme enfant de `/bin/sh`, qui ne forwarde pas les signaux et casse l'arrêt gracieux.

**La période de grâce doit être assez longue pour que les requêtes en vol terminent.** Pour une API web, les 30 secondes par défaut suffisent en général. Pour des opérations longues (uploads de fichiers, long-polling, connexions WebSocket), configurer l'orchestrateur pour donner plus de temps, ou implémenter un disjoncteur qui cesse d'accepter les opérations longues bien avant l'arrêt.

```csharp
// Program.cs : étendre la fenêtre d'arrêt gracieux à 45 secondes
builder.Services.Configure<HostOptions>(options =>
{
    options.ShutdownTimeout = TimeSpan.FromSeconds(45);
});
```

## Zoom : docker-compose pour le dev local

Un fichier docker-compose est le chemin le plus rapide vers un environnement local réaliste qui reflète les dépendances de prod. Il s'appaire particulièrement bien avec les tests d'intégration couverts dans l'[article TestContainers](/fr/posts/testing-integration-testing-testcontainers/), où des images identiques à la prod tournent à l'intérieur du process de test.

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

Trois détails à connaître. Le `depends_on` avec `condition: service_healthy` veut dire que Compose attendra que Postgres passe son health check avant de démarrer l'API, évitant la race condition où l'application démarre avant que la base ne soit prête. La déclaration `volumes:` pour `pgdata` persiste la base entre `docker compose up` et `docker compose down` ; utiliser `docker compose down -v` pour reset. Le `ports: "8080:8080"` expose l'API sur l'hôte, ce qui est l'objectif en local mais ne doit jamais se retrouver dans un fichier Compose de prod.

## Zoom : ce qu'il ne faut pas mettre dans l'image

Une image container de prod doit contenir **uniquement** l'application et ses dépendances runtime. Les choses qui ne doivent *jamais* se retrouver dans l'image :

- **Les secrets.** Chaînes de connexion, clés d'API, certificats, clés de signature JWT. Ces éléments ont leur place dans des variables d'environnement injectées au runtime, ou dans un store de secrets (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault, Kubernetes Secrets).
- **Les outils de build.** Le compilateur, NuGet, les debuggers. Le pattern multi-stage les garde dans le stage de build.
- **Les projets de test et les données de test.** Les tests tournent en CI avant que l'image ne soit construite ; ils n'ont pas leur place dans l'image déployée.
- **Les fichiers de configuration de dev.** `appsettings.Development.json` doit être soit exclu, soit copié uniquement dans les images non-prod.
- **Le code source.** Le stage runtime doit copier la *sortie de publish*, pas le code source. Livrer le code source en prod est une erreur courante et une responsabilité de sécurité.

> ❌ **Ne jamais faire** : Ne jamais figer des secrets dans l'image au build, même comme variables d'environnement dans le Dockerfile. Toute personne qui tire l'image (y compris un attaquant avec un accès en lecture au registry) peut les récupérer. Les secrets se gèrent au runtime, jamais au build.

## Wrap-up

Héberger correctement une application ASP.NET Core sur Docker en 2026, c'est un Dockerfile à deux stages avec un layer cache agressif, une image de base chiseled pour la sécurité et la taille, des endpoints de health check liveness et readiness séparés, une gestion des signaux via PID 1 pour un arrêt gracieux, et un fichier docker-compose qui reflète les dépendances de prod pour le dev local. Tu peux livrer une image d'environ 100 Mo, tourner en non-root, exposer les bons endpoints de santé pour l'orchestrateur qui viendra ensuite, et garder les secrets complètement hors de l'image.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Héberger ASP.NET Core sur IIS : le classique, démystifié](/fr/posts/hosting-iis/)
- [Tests d'Intégration avec TestContainers pour .NET](/fr/posts/testing-integration-testing-testcontainers/)
- [La Compilation AOT en .NET : démarrage, taille, et les compromis dont personne ne parle](/fr/posts/performance-aot-compilation/)

## Références

- [Images Docker .NET sur MCR](https://mcr.microsoft.com/en-us/artifact/mar/dotnet/aspnet/tags)
- [Images Ubuntu chiseled pour .NET, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/core/docker/container-images)
- [Health checks dans ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/host-and-deploy/health-checks)
- [Spécification Docker Compose](https://docs.docker.com/compose/compose-file/)
- [Référence Dockerfile](https://docs.docker.com/reference/dockerfile/)
