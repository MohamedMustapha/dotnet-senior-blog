---
title: ".NET Aspire : l'orchestration cloud-native simplifiée"
date: 2026-04-09
draft: false
tags: ["deployment", "aspire", "dotnet"]
categories: ["Deployment"]
series: ["Deployment"]
description: "AppHost, service defaults, composition de ressources, le dashboard Aspire, et le chemin entre F5 sur un poste et un déploiement en prod sur Azure Container Apps, sans écrire une seule ligne de YAML."
---

Hello tous le monde, aujourd'hui on va démystifier **.NET Aspire**, la proposition de Microsoft pour retirer le ciment qu'on écrivait à la main pour orchestrer plusieurs services .NET.

Pendant une décennie, l'écart entre "ma solution .NET tourne sur mon poste" et "ma solution .NET est déployée sur une plateforme cloud" a été comblé par de l'outillage que le développeur devait assembler lui-même : un docker-compose pour l'orchestration locale, un jeu séparé de manifests Kubernetes ou ACA pour le déploiement, du câblage OpenTelemetry par service, un dashboard pour regarder les traces, une façon de passer des chaînes de connexion aux containers. Chaque équipe réinventait le même ciment, un peu différemment, et la friction rendait les microservices .NET plus chers à démarrer qu'ils n'auraient dû l'être.

**.NET Aspire** comble cet écart. Sorti en GA en mai 2024, c'est le framework opinionné de Microsoft pour composer, faire tourner, et déployer des applications .NET multi-services. Ce n'est pas une nouvelle plateforme d'hébergement. C'est une couche d'orchestration C#-first qui se pose au-dessus de l'hébergement déjà utilisé ([Docker](/fr/posts/hosting-docker/), [Kubernetes](/fr/posts/hosting-kubernetes/), [ACA](/fr/posts/hosting-azure-container-apps/)), remplaçant le YAML à la main et les scripts shell par un projet AppHost typé qui décrit toute la topologie en C#. Pour beaucoup d'équipes .NET, surtout celles qui démarrent de nouvelles applications distribuées, cela retire une quantité significative de boilerplate sans verrouiller personne sur un cloud précis.

Ce dernier article de la [série Deployment](/fr/posts/deployment-docker-dockerfile-compose/) couvre ce qu'est réellement Aspire, comment l'utiliser comme outil de dev et de déploiement, et dans quels cas c'est le bon choix.

## Le contexte : pourquoi .NET Aspire existe

Aspire répond à une observation précise : chaque application .NET non triviale en 2024 se ressemblait dans sa couche d'orchestration. Elle avait une API, un worker, une base, un cache, peut-être un broker de messages. Chaque service avait besoin d'OpenTelemetry configuré, d'une chaîne de connexion câblée, de health checks enregistrés, et d'une façon de tourner en local contre les mêmes dépendances. Les équipes écrivaient les mêmes vingt lignes de ciment par service, à vie, et chacune les écrivait un peu différemment.

Les objectifs d'Aspire, formulés concrètement :

1. **Remplacer docker-compose par un modèle C# typé.** La topologie de l'application (quels services tournent, ce dont ils dépendent, ce avec quoi ils parlent) est décrite dans un projet .NET classique appelé le **AppHost**, avec typage fort, IntelliSense, et support du refactoring.
2. **Standardiser les préoccupations transverses.** OpenTelemetry, health checks, service discovery, policies de résilience, et logging structuré sont packagés dans les **Service Defaults**, un projet partagé que chaque service de la solution référence. On ajoute la référence, on appelle une méthode d'extension, et on a tout.
3. **Fournir un dashboard local.** Quand on appuie sur F5, Aspire démarre tous les services et ouvre un dashboard local qui montre les traces, les métriques, les logs, et la sortie console de chaque process, au même endroit.
4. **Émettre des manifests de déploiement pour de vraies cibles.** Le même AppHost peut générer les manifests nécessaires pour déployer sur Azure Container Apps, Kubernetes, ou Docker Compose, sans que le dev les écrive à la main. C'est la partie qui remplace le problème du "je dois maintenir trois descriptions de déploiement différentes".

## Vue d'ensemble : la forme d'un projet Aspire

{{< mermaid >}}
graph TD
    A[Projet AppHost<br/>topologie C#] --> B[Shop.Api]
    A --> C[Shop.Worker]
    A --> D[Ressource Postgres]
    A --> E[Ressource Redis]
    A --> F[Azure Service Bus]
    B --> D
    B --> E
    C --> D
    C --> F
    G[Projet ServiceDefaults<br/>OTel, health, résilience] --> B
    G --> C
    H[Dashboard Aspire] --> B
    H --> C
    H --> D
{{< /mermaid >}}

Une solution Aspire a une forme distinctive. Deux nouveaux projets se placent à côté des projets de service habituels :

**AppHost** : un projet console qui référence chaque projet de service de la solution et déclare, en C#, les ressources dont chacun dépend. Quand on lance le AppHost, il démarre tous les projets référencés, lance les dépendances (Postgres, Redis, quoi que ce soit), et câble les chaînes de connexion automatiquement.

**ServiceDefaults** : une bibliothèque de classes que chaque projet de service référence. Elle contient les méthodes d'extension qui branchent OpenTelemetry, les endpoints de health check, le service discovery, et les policies de résilience en un seul appel. Au lieu de copier-coller 30 lignes de setup de télémétrie dans chaque `Program.cs`, on appelle `builder.AddServiceDefaults()` et c'est fait.

Le reste de la solution (le projet API, le projet worker, la bibliothèque de domaine) est du code .NET classique, inchangé. Aspire ne demande pas de restructurer l'application. Il ajoute l'orchestration par-dessus.

## Zoom : le projet AppHost

```csharp
// Shop.AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

// Dépendances managées. Aspire les démarre automatiquement en mode dev.
var postgres = builder.AddPostgres("db")
    .WithDataVolume()
    .AddDatabase("shopdb");

var redis = builder.AddRedis("cache")
    .WithDataVolume();

var servicebus = builder.AddAzureServiceBus("sb")
    .AddQueue("orders-inbound");

// Le projet API, avec des références explicites à ses dépendances.
var api = builder.AddProject<Projects.Shop_Api>("shop-api")
    .WithReference(postgres)
    .WithReference(redis)
    .WithReference(servicebus)
    .WithExternalHttpEndpoints()
    .WithReplicas(2);

// Le projet worker.
builder.AddProject<Projects.Shop_Worker>("shop-worker")
    .WithReference(postgres)
    .WithReference(servicebus);

builder.Build().Run();
```

Douze lignes de C# décrivent toute la topologie d'une application distribuée. Cinq choses à noter.

**`AddPostgres("db")` avec `WithDataVolume()`** ne se contente pas de démarrer un container. Cela déclare Postgres comme une ressource managée dans le AppHost, persiste ses données entre les runs via un volume Docker, et expose sa chaîne de connexion à tout projet qui appelle `WithReference(postgres)`. L'appel `AddDatabase("shopdb")` crée la base à l'intérieur de l'instance Postgres automatiquement.

**`AddAzureServiceBus("sb")`** est un cas intéressant. En mode dev, Aspire lance un émulateur (basé sur un container) qui parle le protocole Service Bus. En prod, le même descripteur AppHost mappe sur un vrai namespace Azure Service Bus. Le code de l'application ne change pas entre les deux ; Aspire résout la différence au moment du déploiement.

**`WithReference(postgres)`** est la magie. Cela prend la chaîne de connexion qu'Aspire construit pour le Postgres managé et l'injecte dans le projet référencé comme variable d'environnement, en suivant la même convention de nommage qu'ASP.NET Core utilise (`ConnectionStrings__db`). Le projet la lit ensuite depuis `IConfiguration` sans aucune glu supplémentaire.

**`WithExternalHttpEndpoints()`** marque le projet comme joignable de l'extérieur. En dev local, Aspire assigne un port aléatoire et l'affiche dans le dashboard. En prod, cela mappe sur une règle d'ingress sur la plateforme cible.

**`WithReplicas(2)`** déclare combien d'instances du projet doivent tourner. En dev local, Aspire lance deux copies et load balance entre les deux. En prod, le nombre se traduit en nombre de réplicas sur Kubernetes ou ACA.

> 💡 **Info** : Le catalogue de méthodes `Add*` d'Aspire couvre la plupart des dépendances courantes nativement : Postgres, SQL Server, MySQL, Redis, MongoDB, RabbitMQ, Kafka, Azure Service Bus, Azure Storage, Azure Cosmos DB, Azure Key Vault, et d'autres. La liste complète est dans les packages NuGet `Aspire.Hosting.*`. Les intégrations tierces (Dapr, NATS, Elastic) sont disponibles comme packages communautaires.

## Zoom : le projet ServiceDefaults

Chaque service de la solution référence un projet `ServiceDefaults` partagé qui fournit le setup transverse commun :

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

Et dans le `Program.cs` de chaque service :

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

Deux appels : `AddServiceDefaults()` dans `ConfigureServices` et `MapDefaultEndpoints()` dans le pipeline. Chaque service a maintenant OpenTelemetry câblé au dashboard, des endpoints de health check, du service discovery via DNS, et des clients HTTP résilients avec retry et disjoncteur. Pas de copier-coller. Pas de dérive. Si l'équipe décide d'ajouter un nouvel exporter de télémétrie ou une nouvelle policy de résilience, cela se passe à un seul endroit.

> ✅ **Bonne pratique** : Garder `ServiceDefaults` sous revue stricte. C'est le rayon d'impact pour le comportement de démarrage de chaque service. Les changements l'affectent tous d'un coup, ce qui est exactement ce qui le rend précieux et exactement ce qui le rend dangereux. Le traiter comme une bibliothèque partagée avec ses propres release notes.

## Zoom : le Dashboard Aspire

Quand on appuie sur F5 sur le AppHost, Aspire démarre le dashboard sur un port local et l'ouvre dans le navigateur. Le dashboard montre :

- **Ressources** : chaque service et dépendance, avec leur statut, leurs ports, leurs variables d'environnement, et leurs logs container.
- **Logs console** : une vue unifiée de stdout/stderr de chaque process qui tourne, avec filtrage par service et niveau de log.
- **Logs structurés** : les entrées ILogger, indexées et cherchables.
- **Traces** : les spans OpenTelemetry, avec tracing distribué entre services. Une seule requête qui tape l'API, interroge Postgres, publie sur Service Bus et déclenche le worker s'affiche comme une seule trace avec tous les spans.
- **Métriques** : les compteurs runtime (GC, thread pool, durée des requêtes HTTP) et toutes les métriques custom émises par l'application.

C'est, pour beaucoup d'équipes, le bénéfice le plus visible de l'adoption d'Aspire. Obtenir le même niveau d'observabilité locale sans Aspire demande de faire tourner Jaeger, Prometheus, Grafana et un agrégateur de logs dans un fichier compose, de configurer chacun, et de s'assurer que chaque service exporte vers le bon endpoint. Aspire fait tout cela par défaut, in-process, avec zéro configuration.

> 💡 **Info** : Le Dashboard Aspire est une application autonome. Il peut aussi tourner contre n'importe quelle charge compatible OpenTelemetry (y compris des applications non-Aspire) via l'image standalone `mcr.microsoft.com/dotnet/aspire-dashboard`. Certaines équipes l'adoptent comme stack d'observabilité locale même quand elles n'utilisent pas le reste d'Aspire.

## Zoom : déployer une application Aspire

Aspire n'est pas une plateforme d'hébergement. Il génère des manifests ou des ressources pour une vraie plateforme d'hébergement. Le chemin de déploiement canonique utilise l'**Azure Developer CLI** (`azd`) pour déployer une solution Aspire sur Azure Container Apps avec une seule commande.

```bash
# Une fois, à la racine de la solution
azd init                     # wizard interactif, détecte le AppHost
azd auth login               # authentification Azure

# Chaque déploiement ensuite
azd up                       # provisionne les ressources Azure et déploie
```

Sous le capot, `azd up` fait trois choses :

1. **Provisionne l'infrastructure.** À partir de la description du AppHost, `azd` génère un template Bicep qui crée les ressources Azure nécessaires : un Container Apps Environment, un workspace Log Analytics, un namespace Service Bus (parce que le AppHost en référence un), un Postgres Flexible Server, et ainsi de suite.
2. **Construit les images container** pour chaque projet de service de la solution, en utilisant le publish container standard du SDK .NET (`dotnet publish -t:PublishContainer`), et les pousse dans un Azure Container Registry que `azd` provisionne aussi.
3. **Déploie les Container Apps** avec les bonnes variables d'environnement, secrets, configuration d'ingress, et comptes de réplicas, dérivés du AppHost.

L'aller-retour complet, de `git clone` à un environnement de type prod qui tourne sur Azure, prend typiquement moins de 10 minutes sur un compte neuf.

Pour les équipes qui ciblent Kubernetes à la place, Aspire peut émettre un manifest via `aspire publish` :

```bash
aspire publish --publisher kubernetes --output ./deploy/k8s
```

Cela génère des manifests Kubernetes pour chaque service du AppHost, qui peuvent ensuite être personnalisés avec Kustomize (couvert dans le [primer Kubernetes](/fr/posts/deployment-kubernetes-primer/)) ou packagés avec Helm. La sortie générée est un point de départ, pas l'artefact final, mais elle capture le graphe de dépendances et le câblage d'environnement, qui est la partie fastidieuse.

> ⚠️ **Ça marche, mais...** : `azd up` est excellent pour le dev, les démos, et les environnements de proof of concept. Pour la prod, la plupart des équipes passent à un vrai pipeline CI/CD avec des étages build, test, et deploy séparés, en utilisant le manifest Aspire comme entrée de leur outillage de déploiement existant plutôt que d'appeler `azd up` depuis un poste de travail.

## Zoom : quand Aspire est le bon choix

Aspire est particulièrement bien adapté pour :

- **Les nouvelles applications distribuées .NET** où l'équipe veut une rampe d'accès rapide au dev multi-services sans assembler le ciment depuis zéro.
- **Les solutions existantes qui luttent avec les préoccupations transverses.** Si l'équipe a cinq services et que chacun a un setup OpenTelemetry légèrement différent, les déplacer tous sous un `ServiceDefaults` partagé est un gain net.
- **Les équipes qui veulent de l'observabilité locale sans faire tourner une stack compose parallèle** pour Jaeger, Prometheus, et compagnie.
- **Les boutiques .NET Azure-first.** Le chemin de déploiement `azd` est l'expérience la plus fluide sur Azure. Cela marche ailleurs, mais les aspérités sont moins nombreuses sur Azure.
- **Les démos, les ateliers, et les outils internes** où la rapidité F5-to-running compte plus que la flexibilité de déploiement.

Ce n'est *pas* le bon choix quand :

- **La solution est un seul service.** La valeur d'Aspire vient de l'orchestration de plusieurs services. Pour une API seule, le AppHost est du surcoût sans bénéfice.
- **L'équipe a un pipeline de déploiement mature.** S'il y a déjà un setup Kubernetes + Helm + GitOps qui marche, introduire Aspire comme couche d'authoring peut créer de la friction plutôt que la réduire.
- **Des services non-.NET font partie de la topologie.** Aspire peut référencer des containers ou des exécutables de n'importe quel langage, mais sa meilleure intégration est avec les projets .NET. Un système polyglotte avec de gros services Python, Go ou Node.js s'intègre mieux dans un workflow compose-first ou Kubernetes-first.
- **La cible n'est ni Azure ni Kubernetes.** Aspire peut générer des fichiers compose, mais ses chemins de déploiement les plus forts sont ACA et K8s. Pour des VMs nues, [IIS](/fr/posts/hosting-iis/), ou des hôtes Docker classiques, le bénéfice est moindre.

## Wrap-up

.NET Aspire remplace le pattern du "chaque équipe réinvente le même ciment" par une couche d'orchestration C#-first et typée qui décrit toute la topologie dans le projet AppHost, standardise l'observabilité et la résilience via ServiceDefaults, fournit un dashboard local gratuitement, et génère des manifests de déploiement pour Azure Container Apps, Kubernetes ou Docker Compose. Tu peux démarrer une nouvelle application distribuée .NET avec deux projets supplémentaires et une poignée de lignes de code, obtenir les traces et les métriques sur le dashboard sans rien câbler, et déployer sur ACA avec `azd up` en quelques minutes. Tu peux aussi reconnaître quand une solution existante ne bénéficierait pas de la migration et rester sur les outils d'hébergement et de déploiement déjà en place.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Héberger ASP.NET Core avec Docker : un guide pragmatique](/fr/posts/hosting-docker/)
- [Héberger ASP.NET Core sur Kubernetes : l'essentiel pour les devs .NET](/fr/posts/hosting-kubernetes/)
- [Héberger ASP.NET Core sur Azure Container Apps](/fr/posts/hosting-azure-container-apps/)
- [Docker pour le déploiement .NET : Dockerfile et Compose en pratique](/fr/posts/deployment-docker-dockerfile-compose/)
- [Kubernetes : l'essentiel pour les devs .NET, de kubectl à Helm](/fr/posts/deployment-kubernetes-primer/)

## Références

- [Documentation .NET Aspire, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/aspire/)
- [Vue d'ensemble .NET Aspire](https://learn.microsoft.com/fr-fr/dotnet/aspire/get-started/aspire-overview)
- [Documentation Azure Developer CLI](https://learn.microsoft.com/fr-fr/azure/developer/azure-developer-cli/)
- [Dashboard Aspire standalone](https://learn.microsoft.com/fr-fr/dotnet/aspire/fundamentals/dashboard/standalone)
- [.NET Aspire sur GitHub](https://github.com/dotnet/aspire)
