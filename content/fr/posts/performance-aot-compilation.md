---
title: "La Compilation AOT en .NET : démarrage, taille, et les compromis dont personne ne parle"
date: 2026-04-08
draft: false
tags: ["performance", "aot", "dotnet"]
categories: ["Performance"]
series: ["Performance"]
description: "Native AOT, ReadyToRun, et le cadre de décision pour savoir quand la compilation ahead-of-time vaut ses contraintes. Temps de démarrage, taille binaire, empreinte mémoire, limites de la réflexion."
---

Hello tous le monde, aujourd'hui on va comprendre la **compilation AOT** en .NET, ce qu'elle apporte, ce qu'elle coûte, et dans quels cas elle est le bon outil.

Pendant vingt ans, .NET a tourné via un compilateur JIT. Le code managé était chargé, vérifié, compilé en instructions machine au premier appel, et exécuté. Le JIT produisait d'excellentes performances en régime stable parce qu'il pouvait observer le comportement réel à l'exécution et optimiser en conséquence. Le prix était payé au démarrage : un process froid passait ses premières secondes à compiler les chemins chauds avant d'atteindre sa vitesse de croisière, et le runtime lui-même devait être livré dans chaque déploiement. Pour un serveur web qui tourne longtemps, c'était une taxe négligeable. Pour une fonction serverless invoquée une fois puis détruite, un outil CLI, un container qui scale depuis zéro, ou n'importe quelle charge où le démarrage est le coût dominant, c'était un problème.

Native AOT dans .NET 7 (2022) a résolu ce problème en produisant un binaire natif, entièrement compilé et autonome, au moment du build, sans JIT à l'exécution. Le résultat : une application .NET qui démarre en quelques dizaines de millisecondes, prend moins de mémoire, et se livre sans runtime. Les contraintes, elles, sont significatives, et une équipe qui adopte Native AOT sans les comprendre découvrira à ses dépens que les bibliothèques lourdes en réflexion, la génération de code dynamique, et certaines parties de la BCL ne fonctionnent pas de la même façon.

Cet article couvre les deux côtés : ce que l'AOT apporte, ce qu'il coûte, quand l'utiliser, et quand les [techniques zero-allocation](/fr/posts/performance-zero-allocation/) ou le JIT classique sont le meilleur choix.

## Le contexte : pourquoi l'AOT existe

Le problème que l'AOT résout n'est pas le débit brut. Sur les charges longues, un process JIT chaud est souvent *plus rapide* qu'un équivalent compilé AOT, parce que le JIT a de l'optimisation guidée par profil, de la compilation à plusieurs étages, et de l'inlining de méthodes informé par les vrais patterns d'appel. Ce que l'AOT résout, c'est la forme de la courbe de latence au démarrage du process, et la taille de ce qui est livré dans le container.

Trois problèmes concrets motivent l'adoption de l'AOT :

1. **La latence de démarrage à froid.** Une application web .NET 10 standard met plusieurs centaines de millisecondes à démarrer, même avec ReadyToRun. Une version Native AOT de la même application démarre en 30 à 80 millisecondes. Pour une Lambda, une Azure Function, un pod Kubernetes qui scale depuis zéro, ou un outil CLI qui tourne et se termine, cette différence fait tout.
2. **La taille du binaire.** Une application .NET self-contained est livrée avec un runtime qui pèse 70 à 100 Mo après trimming. Un binaire Native AOT pour la même application peut peser 10 à 20 Mo. Dans un registry qui héberge 500 images, ou un pipeline de CI qui tire des images des centaines de fois par jour, la différence s'accumule vite.
3. **L'empreinte mémoire.** Un process JIT a besoin de mémoire pour le runtime, le compilateur lui-même, et le code compilé. Un process AOT n'en a besoin pour rien de tout ça. Le working set de pointe baisse de 30 à 50% sur des charges typiques.

Il y a aussi une quatrième raison, moins discutée : **la simplicité de déploiement**. Un binaire natif est un seul fichier qui tourne sur l'OS cible sans aucun runtime installé. Plus de "c'est la bonne version de .NET ?", plus de "l'image de base a été patchée ?", plus de "pourquoi ça marche sur ma machine et pas sur le serveur ?". Ça tourne ou ça ne tourne pas, et si ça tourne une fois, ça tourne partout où l'OS et l'architecture sont partagés.

## Vue d'ensemble : le paysage AOT

{{< mermaid >}}
graph TD
    A[Options de compilation .NET] --> B[JIT<br/>défaut]
    A --> C[ReadyToRun<br/>depuis .NET Core 3.0]
    A --> D[Native AOT<br/>depuis .NET 7]
    B --> B1[Meilleure perf en régime stable<br/>Démarrage le plus lent]
    C --> C1[Méthodes pré-JIT<br/>Runtime encore nécessaire<br/>~30% plus rapide au démarrage]
    D --> D1[Pas de runtime<br/>Taille minimale<br/>Démarrage le plus rapide<br/>Limites de réflexion]
{{< /mermaid >}}

Trois modèles de compilation sont disponibles en .NET 10, chacun avec un compromis différent.

**JIT** est le défaut et le bon choix pour la grande majorité des charges : serveurs web qui restent up des heures ou des jours, workers de fond, tout ce qui fait que la performance en régime stable compte plus que le démarrage. Le JIT a l'optimisation guidée par profil (PGO) depuis .NET 6, la compilation à plusieurs étages depuis .NET Core 2.1, et produit du code qui, dans beaucoup de benchmarks, bat l'équivalent AOT après warmup.

**ReadyToRun (R2R)** est le compromis intermédiaire. Il pré-compile les méthodes en code natif au build, puis utilise quand même le JIT à l'exécution pour la ré-optimisation et pour les méthodes qui n'ont pas été pré-compilées. R2R est disponible depuis .NET Core 3.0 et s'active en ajoutant `<PublishReadyToRun>true</PublishReadyToRun>` au csproj. Il réduit la latence de démarrage d'environ 30% avec un minimum d'autres changements. C'est le premier pas à faible risque pour les équipes dont le démarrage est pénible mais qui ne peuvent pas se permettre les contraintes AOT.

**Native AOT** est l'engagement complet. Il produit un seul binaire natif sans runtime, sans JIT, et sans génération de code dynamique. Le coût est un jeu de contraintes (couvertes plus bas) qui exclut des catégories entières de bibliothèques. Le bénéfice est le démarrage le plus rapide, la taille minimale, et l'empreinte mémoire la plus basse que .NET puisse produire.

> 💡 **Info** : Native AOT a été publié dans .NET 7 (2022) comme fonctionnalité supportée pour les minimal APIs ASP.NET Core, les workers, et les applications console. Le support s'est étendu dans .NET 8, 9, et 10 à plus de bibliothèques et de scénarios. Dans .NET 10, la plupart des middlewares courants d'ASP.NET Core et `System.Text.Json` fonctionnent correctement sous AOT avec les source generators.

## Zoom : activer Native AOT

Passer une minimal API en Native AOT, c'est un changement de deux lignes dans le csproj et quelques ajustements de code :

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

Deux choses à noter. `CreateSlimBuilder` remplace `CreateBuilder` et produit un hôte trimmer-friendly sans toute la stack MVC. `JsonSerializerContext` remplace la sérialisation JSON basée sur la réflexion runtime par un source generator au moment de la compilation, ce qui est l'approche standard compatible AOT.

```bash
dotnet publish -c Release -r linux-x64
```

La sortie est un seul binaire dans `bin/Release/net10.0/linux-x64/publish/`, pesant typiquement 12 à 18 Mo, sans runtime nécessaire pour le lancer.

> ✅ **Bonne pratique** : Utiliser `CreateSlimBuilder` dès le jour 1 si l'AOT est dans la roadmap. Cela rend la migration progressive au lieu d'un seul flip douloureux, et pousse l'équipe vers des patterns qui fonctionnent à la fois en JIT et en AOT.

## Zoom : les contraintes

C'est la section qu'on saute souvent dans le marketing AOT. Les compromis sont réels et ils disqualifient l'AOT pour une portion significative des codebases du monde réel.

**Pas de réflexion runtime sur des types arbitraires.** Le linker (IL trimmer) supprime le code dont il ne peut pas prouver statiquement qu'il est utilisé. Toute bibliothèque qui utilise `Type.GetType(string)`, `Activator.CreateInstance(type)`, ou `Assembly.Load` au runtime sur des types que le linker n'a pas vus au build échouera. Cela touche beaucoup de bibliothèques populaires : les anciennes versions d'AutoMapper, certains conteneurs IoC avec découverte de types runtime, certaines bibliothèques de sérialisation. Les versions plus récentes de la plupart ont adopté les source generators, mais la migration n'est pas complète dans tout l'écosystème.

**Pas de génération de code dynamique.** `System.Reflection.Emit`, `Expression.Compile()`, et toute bibliothèque construite par-dessus (proxies dynamiques, certains ORM, certains frameworks de mocking) ne fonctionnent pas sous AOT. Entity Framework Core a le support AOT depuis .NET 8+, avec des limitations sur certaines formes de requêtes. Moq et NSubstitute ne fonctionnent pas, parce qu'ils reposent sur la génération de proxies au runtime.

**Source generators pour JSON et autres.** La sérialisation `System.Text.Json` par réflexion ne fonctionne pas sous AOT. La sérialisation par source generator fonctionne. Pareil pour `Regex` (utiliser le regex généré via l'attribut `[GeneratedRegex]`) et pour le logging (utiliser les méthodes de logger générées).

**Globalisation invariante.** Les binaires Native AOT passent par défaut en mode invariant sauf si les données ICU sont livrées explicitement. Pour beaucoup de charges, c'est acceptable. Pour toute application qui formate des dates, des nombres, ou une devise selon la locale de l'utilisateur, c'est une contrainte à adresser.

**Temps de build plus longs.** Un build Native AOT peut prendre 30 secondes à plusieurs minutes par publish, contre moins de 10 secondes pour un build JIT. Pour la boucle de dev, le JIT reste la cible ; l'AOT est pour le pipeline de release.

> ❌ **Ne jamais faire** : Ne pas activer `PublishAot` sur un codebase mature et lancer le publish comme premier test. Le build échouera de façons difficiles à rattacher à la bibliothèque fautive. À la place, ajouter les warnings AOT au build (`<IsAotCompatible>true</IsAotCompatible>`), les corriger de façon itérative, et n'activer `PublishAot` qu'une fois les warnings nettoyés.

## Zoom : ReadyToRun comme alternative à faible risque

Pour les équipes qui veulent une amélioration de démarrage sans les contraintes AOT, ReadyToRun est souvent la bonne réponse. Il demande une propriété dans le csproj et aucun changement de code :

```xml
<PropertyGroup>
  <PublishReadyToRun>true</PublishReadyToRun>
  <PublishReadyToRunComposite>true</PublishReadyToRunComposite>
</PropertyGroup>
```

L'option composite bundle le framework et l'application dans une seule image R2R, améliorant encore le warmup. Le démarrage s'améliore d'environ 30%, la taille augmente de 10 à 20% (parce que le binaire contient maintenant à la fois l'IL et le code natif), et rien d'autre ne change. Le JIT tourne toujours au runtime pour la recompilation tier-1, ce qui veut dire que la performance en régime stable est préservée.

ReadyToRun est le premier pas sensé pour toute équipe dont le démarrage est un problème remonté mais dont le codebase dépend de la réflexion, des proxies dynamiques, ou de tout ce que l'AOT rejetterait.

> ⚠️ **Ça marche, mais...** : R2R améliore le démarrage à froid sur les méthodes qu'il a pré-compilées, typiquement le framework et les chemins chauds déclarés dans l'application. Il n'aide pas pour les méthodes jamais marquées comme cold-compilable, catégorie qui grandit avec la taille du codebase. Pour des cibles de démarrage extrêmes (sous 100 ms), Native AOT reste la réponse.

## Zoom : quand l'AOT est le bon choix

Native AOT est le bon choix pour un ensemble précis de charges. L'utiliser quand au moins deux des conditions suivantes sont vraies :

- **Le process est invoqué fréquemment et de courte durée.** Lambdas, Azure Functions, APIs serverless, outils CLI, jobs planifiés. Chacun paie la taxe de démarrage à froid à chaque invocation.
- **La taille de l'image container compte.** Systèmes multi-tenants qui font tourner des milliers d'images, pipelines de CI qui tirent des images souvent, déploiements edge avec des contraintes de bande passante.
- **L'empreinte mémoire par process est un driver de coût.** Faire tourner des centaines de petites instances par hôte, où chaque 20 Mo économisés par process vaut l'effort de migration.
- **La cible de déploiement n'a aucun runtime installé.** Containers Linux nus, images distroless minimales, systèmes embarqués.
- **Le codebase est greenfield ou assez petit pour être migré en un sprint.** La migration AOT est beaucoup plus facile pour une minimal API de 50 fichiers que pour un monolithe de 500 fichiers avec vingt ans de patterns basés sur la réflexion.

Ne *pas* utiliser l'AOT quand :

- La charge est un serveur web longue durée qui reste up pendant des jours. Le JIT avec PGO battra l'AOT en régime stable, et le temps de démarrage est amorti sur la durée de vie du process.
- Le codebase dépend fortement de la réflexion, des proxies dynamiques, ou du code généré à l'exécution. Soit migrer ces dépendances d'abord, soit accepter que l'AOT n'est pas le bon outil pour ce codebase.
- L'équipe n'a pas d'appétit pour des changements cassants pendant la migration. L'AOT expose des warnings et des erreurs qu'un build JIT tolère en silence, et chaque warning est du vrai travail.

## Zoom : mesurer le gain

Toute adoption AOT doit s'appuyer sur des mesures avant/après, pas sur l'espoir. Trois chiffres comptent :

**Le temps de démarrage à froid**, mesuré du lancement du process à la première requête réussie. Un simple harness qui spawn le process, poll un endpoint de santé, et enregistre le temps écoulé. Répéter dix fois et rapporter la médiane.

**La mémoire résidente de pointe**, mesurée pendant un run stable à 100 RPS. Utiliser `dotnet-counters` ou `ps -o rss` et capturer le max.

**La taille du binaire**, mesurée sur le répertoire de sortie publié (`du -sh` sous Linux, `Get-ChildItem | Measure-Object -Property Length -Sum` sous Windows).

```bash
# Avant, JIT
$ time curl -s http://localhost:5000/health
real    0m0.480s    # ~480 ms de démarrage à froid
$ du -sh ./publish
82M

# Après, Native AOT
$ time curl -s http://localhost:5000/health
real    0m0.052s    #  ~52 ms de démarrage à froid
$ du -sh ./publish
14M
```

Des chiffres comme ça justifient la migration. Des chiffres qui ne montrent que 10% d'amélioration ne la justifient pas, parce que les contraintes que l'AOT impose ont une longue traîne de coûts en aval. La décision doit être guidée par la donnée.

> ✅ **Bonne pratique** : Mettre le harness de mesure dans le repo comme un script dédié. Quand le runtime livre une nouvelle version, rejouer les mesures. L'AOT s'améliore à chaque release .NET, et un codebase qui n'était "pas prêt pour l'AOT" en .NET 8 peut très bien l'être en .NET 10 ou 11.

## Wrap-up

Native AOT est la bonne réponse pour une classe précise de charges .NET où la latence de démarrage à froid, la taille du binaire, et l'empreinte mémoire dominent, et où le codebase peut accepter les contraintes sur la réflexion et la génération de code dynamique. Tu peux l'activer avec `<PublishAot>true</PublishAot>` et `CreateSlimBuilder`, adopter les source generators pour JSON, regex et logging, corriger les warnings AOT de façon itérative avant de basculer le flag de publish, et mesurer le démarrage à froid, la mémoire de pointe et la taille binaire avant et après. Tu peux te rabattre sur ReadyToRun comme intermédiaire à faible risque, ou rester sur le JIT standard quand le débit en régime stable est la vraie métrique qui compte.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Zero Allocation en .NET : quand le GC devient le goulet d'étranglement](/fr/posts/performance-zero-allocation/)
- [Les Tests de Charge en .NET : vue d'ensemble des quatre types qui comptent](/fr/posts/load-testing-overview/)
- [Le Spike Testing en .NET : survivre au burst soudain](/fr/posts/load-testing-spike/)

## Références

- [Déploiement Native AOT, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/core/deploying/native-aot/)
- [Support de Native AOT dans ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/native-aot)
- [Compilation ReadyToRun, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/core/deploying/ready-to-run)
- [Source generation System.Text.Json, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/standard/serialization/system-text-json/source-generation)
- [Options de trimming, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/core/deploying/trimming/trimming-options)
