---
title: "EF Core : Optimisation des Lectures"
date: 2026-04-09
draft: false
tags: ["database", "efcore", "performance", "dotnet"]
categories: ["Database"]
series: ["Database"]
description: "AsNoTracking, projections, split queries, compiled queries, et les pièges classiques N+1 et explosion cartésienne. Tour pratique pour rendre les lectures EF Core rapides sans quitter LINQ."
---

Hello tous le monde, aujourd'hui on va découvrir **comment rendre les lectures EF Core vraiment rapides**, sans sortir de LINQ.

La plupart des problèmes de performance EF Core sont des problèmes de lecture, et ils se ressemblent tous : une page qui chargeait en 40 millisecondes avec 20 lignes en dev prend 3 secondes en staging avec 20 000 lignes. Le développeur sort Dapper, migre cinq requêtes, et le reste du code reste sur LINQ-to-EF. La réponse honnête, c'est qu'EF Core est assez rapide pour 95% des lectures depuis la version 6, à condition de savoir quels leviers actionner. Cet article parle de ces leviers : change tracking, projections, split queries, compiled queries, et les deux pièges (N+1 et explosion cartésienne) qui causent la majorité des tickets "requête lente".

## Le contexte : pourquoi les lectures EF Core peuvent être lentes par défaut

Le mode par défaut d'EF Core est pensé pour l'écriture : chaque entité chargée entre dans le change tracker, chaque navigation peut être lazy-loadée, chaque requête hydrate des entités complètes. Ce défaut est correct pour un command handler qui charge un agrégat, le modifie et sauvegarde. Il est dispendieux pour les requêtes qui ne font que lire. Le change tracker prend de la mémoire et du CPU pour snapshotter chaque entité chargée. L'hydratation construit un graphe d'objets complet alors qu'on n'avait besoin que de quatre colonnes. Les navigations incluses joignent des tables dont on n'a pas besoin pour cette page précise. Et quand un seul `.Include()` traverse deux navigations de type collection, le result set explose en produit cartésien lent à transférer et lent à matérialiser.

Corriger une lecture lente, c'est décider, par requête, lesquels de ces défauts on désactive. La forme du LINQ reste la même ; on ajoute un ou deux appels de méthode.

## Vue d'ensemble : la boîte à outils de l'optimisation de lecture

{{< mermaid >}}
graph TD
    A[Requête lente] --> B{Diagnostic}
    B --> C[Pattern N+1]
    B --> D[Explosion cartésienne]
    B --> E[Coût du tracking]
    B --> F[Sur-lecture de colonnes]
    B --> G[Coût de compilation de la requête]
    C --> H[.Include ou projection]
    D --> I[AsSplitQuery]
    E --> J[AsNoTracking]
    F --> K[Select projection vers DTO]
    G --> L[EF.CompileAsyncQuery]
{{< /mermaid >}}

Cinq leviers, cinq problèmes distincts. On ne les applique pas tous à la fois : on diagnostique d'abord, puis on prend celui qui correspond.

## Zoom : AsNoTracking, le gain le moins cher

Toute requête qui ne modifie pas le résultat devrait sauter le change tracker :

```csharp
var customers = await _db.Customers
    .AsNoTracking()
    .Where(c => c.LoyaltyTier == "gold")
    .ToListAsync(ct);
```

`AsNoTracking()` dit à EF Core de ne pas snapshotter les entités chargées, ce qui économise de la mémoire et du CPU proportionnels à la taille du résultat. Pour un endpoint de liste en lecture seule, c'est en général un gain de 20 à 40% côté .NET sans changer le SQL. Si on a beaucoup de requêtes de ce genre, on le passe par défaut au niveau du contexte :

```csharp
services.AddDbContext<ShopDbContext>(options =>
{
    options.UseNpgsql(conn)
           .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
});
```

Les commandes qui doivent modifier s'y opposent alors explicitement avec `.AsTracking()`. Ça inverse le défaut en faveur des lectures, ce qui est le bon pari pour la plupart des applications web où les lectures dépassent les écritures dans un ratio de 10 pour 1.

> 💡 **Info** — `AsNoTrackingWithIdentityResolution()` est l'intermédiaire. Il saute le change tracking mais déduplique quand même les références à l'intérieur du résultat, ce qui compte quand on charge un graphe avec des parents partagés. À utiliser quand un simple `AsNoTracking()` rend des lignes enfants dupliquées.

## Zoom : les projections, le plus gros gain

La requête la plus rapide est celle qui ne demande que ce que l'endpoint retourne vraiment. Si la réponse API a 5 champs, on lit 5 colonnes, pas les 30 colonnes de l'entité.

```csharp
public sealed record CustomerListItem(Guid Id, string Email, string LoyaltyTier);

var customers = await _db.Customers
    .Where(c => c.LoyaltyTier == "gold")
    .Select(c => new CustomerListItem(c.Id, c.Email, c.LoyaltyTier))
    .ToListAsync(ct);
```

Trois choses se passent automatiquement quand on projette :

1. EF Core génère un `SELECT` avec uniquement les colonnes projetées. Pas de jointures inutiles, pas de colonnes `BLOB` trimballées sur le réseau.
2. Le type du résultat n'est pas une entité, donc il n'y a rien à tracker. `AsNoTracking()` devient redondant sur une requête projetée.
3. EF Core sait traduire une projection à travers une navigation sans avoir besoin d'un `.Include()`.

Ce troisième point, c'est celui que la plupart des développeurs ratent. Pas besoin de `.Include(c => c.Orders)` si on veut juste la date du dernier order :

```csharp
var customers = await _db.Customers
    .Select(c => new
    {
        c.Id,
        c.Email,
        LastOrderDate = c.Orders.Max(o => (DateTime?)o.CreatedAt)
    })
    .ToListAsync(ct);
```

EF Core traduit ça en `LEFT JOIN LATERAL` (ou une sous-requête équivalente selon le provider) et renvoie une ligne par client. Pas de N+1, pas d'aller-retour supplémentaire.

> ✅ **Bonne pratique** — Pour toute requête qui alimente une vue liste, on projette vers un DTO. `.Include()` est fait pour les commandes qui chargent un agrégat, pas pour les lectures qui alimentent une UI.

## Zoom : l'explosion cartésienne

```csharp
var orders = await _db.Orders
    .Include(o => o.Lines)
    .Include(o => o.Payments)
    .Where(o => o.CreatedAt > since)
    .ToListAsync(ct);
```

Ça a l'air innocent et c'est un piège classique. EF Core génère un seul SQL avec deux `LEFT JOIN`. Un order avec 5 lignes et 2 paiements produit 10 lignes dans le result set, une par combinaison. Passe à 1 000 orders et on transfère des dizaines de milliers de lignes sur le réseau pour hydrater ce que le domaine voit comme 1 000 objets. EF Core prévient au démarrage dans les logs quand il détecte ça, et la correction tient en un appel :

```csharp
var orders = await _db.Orders
    .AsSplitQuery()
    .Include(o => o.Lines)
    .Include(o => o.Payments)
    .Where(o => o.CreatedAt > since)
    .ToListAsync(ct);
```

`AsSplitQuery()` fait émettre à EF Core trois `SELECT` distincts (un pour orders, un pour lines, un pour payments) et les recolle en mémoire. On paie trois aller-retours au lieu d'un, mais on transfère `N + L + P` lignes au lieu de `N * L * P`. Pour toute collection qui a plus que quelques enfants, les split queries gagnent.

On peut aussi régler le défaut au niveau du contexte :

```csharp
options.UseNpgsql(conn, npgsql => npgsql.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
```

> ⚠️ **Ça marche, mais...** — Le mode single-query convient aux navigations 1:1 et aux petites collections 1:N. Dès qu'on chaîne deux `.Include()` sur des collections, on est en territoire d'explosion cartésienne. En cas de doute, on bascule en split query.

## Zoom : le piège du N+1

Le N+1 classique vient de l'itération sur des entités chargées avec accès à une navigation sur chacune :

```csharp
var orders = await _db.Orders.Where(o => o.CreatedAt > since).ToListAsync(ct);
foreach (var order in orders)
{
    var customer = order.Customer; // lazy load, un SQL par order
    Console.WriteLine($"{order.Id} - {customer.Email}");
}
```

Le lazy loading tire une requête par itération. La correction, c'est soit un `.Include(o => o.Customer)` explicite, soit, mieux, une projection qui ramène directement l'email du client :

```csharp
var orderSummaries = await _db.Orders
    .Where(o => o.CreatedAt > since)
    .Select(o => new { o.Id, CustomerEmail = o.Customer.Email })
    .ToListAsync(ct);
```

> ❌ **Ne jamais faire** — Ne pas activer le lazy loading global sur une API CRUD. Ça transforme chaque `.Include()` oublié en N+1 silencieux qui ne sort qu'en production. EF Core livre le lazy loading désactivé par défaut précisément pour ça. Si on en a besoin, on l'active par contexte, et on audite chaque requête qui touche une navigation.

## Zoom : compiled queries pour les hot paths

EF Core parse et traduit l'expression tree LINQ à chaque appel. Pour la plupart des requêtes, le coût est négligeable face à l'aller-retour base, mais pour un hot path appelé des milliers de fois par seconde, la traduction commence à apparaître dans le profiler. Les compiled queries mettent la traduction en cache :

```csharp
private static readonly Func<ShopDbContext, Guid, CancellationToken, Task<Customer?>> GetCustomerById =
    EF.CompileAsyncQuery((ShopDbContext db, Guid id, CancellationToken ct) =>
        db.Customers.AsNoTracking().FirstOrDefault(c => c.Id == id));

public Task<Customer?> FindAsync(Guid id, CancellationToken ct) => GetCustomerById(_db, id, ct);
```

`EF.CompileAsyncQuery` prend l'expression LINQ, la traduit une fois, et renvoie un delegate qui réutilise le plan compilé. Sur une requête appelée 10 000 fois, ça économise typiquement 20 à 30% de latence côté .NET. À utiliser chirurgicalement : les 5 requêtes en tête des télémétries, pas chaque méthode de repository.

> 💡 **Info** — Depuis EF Core 6, les résultats de compilation de requête sont aussi mis en cache en interne par forme distincte d'expression LINQ. `EF.CompileAsyncQuery` gagne encore sur le débit brut, mais l'écart est plus étroit qu'avant.

## Zoom : lire le SQL généré

Toute décision d'optimisation dépend de la lecture du SQL réel. En dev, on active le logger et on logge le SQL au niveau debug :

```csharp
options.UseNpgsql(conn)
       .LogTo(Console.WriteLine, LogLevel.Information)
       .EnableSensitiveDataLogging();
```

En production, on garde la catégorie EF Core en `Warning` pour attraper les avertissements d'explosion cartésienne et de multi-collection, mais on ne logge pas chaque statement. À la place, on s'appuie sur l'instrumentation OpenTelemetry, qui émet chaque requête base comme un span avec le SQL attaché.

```csharp
services.AddOpenTelemetry()
        .WithTracing(t => t.AddEntityFrameworkCoreInstrumentation(o => o.SetDbStatementForText = true));
```

Ça donne une timeline de chaque requête qu'une request exécute, et c'est en général assez pour repérer celle qui prend 400 ms quand les 19 autres sont à 2 ms.

## Wrap-up

Tu sais maintenant manier les cinq leviers qui couvrent la majorité des problèmes de performance en lecture. `AsNoTracking` pour chaque requête en lecture seule. Projection vers un DTO pour tout endpoint de liste. `AsSplitQuery` dès qu'on chaîne des `.Include()` sur des collections. Chasser les N+1 de lazy loading en lisant le SQL généré, pas en inspectant le C#. Et compiler les trois ou quatre requêtes qui sortent vraiment dans le hot path. Une fois tout ça en place, les 5% de requêtes encore lentes sont celles qui ont vraiment besoin de SQL brut, sujet du prochain article.

Prêt à repasser sur le repository le plus chaud de ton projet et à appliquer ces leviers, ou à partager cette grille d'analyse avec ton équipe ?

À la prochaine, a++ 👋

## Pour aller plus loin

- [EF Core : Configuration et Seed](/fr/posts/database-efcore-config-seed/)
- [EF Core : les Migrations en pratique](/fr/posts/database-efcore-migrations/)

## Références

- [EF Core : Performance](https://learn.microsoft.com/fr-fr/ef/core/performance/)
- [EF Core : Efficient Querying](https://learn.microsoft.com/fr-fr/ef/core/performance/efficient-querying)
- [EF Core : Split Queries](https://learn.microsoft.com/fr-fr/ef/core/querying/single-split-queries)
- [EF Core : Compiled Queries](https://learn.microsoft.com/fr-fr/ef/core/performance/advanced-performance-topics#compiled-queries)
- [EF Core : Tracking vs No-Tracking](https://learn.microsoft.com/fr-fr/ef/core/querying/tracking)
