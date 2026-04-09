---
title: "Accès aux données en .NET : Repository Pattern, ou pas ?"
date: 2026-04-11
draft: false
tags: ["data-access", "repository", "efcore", "dotnet"]
categories: ["Architecture"]
series: ["Layer Focused"]
description: "Le Repository pattern était la bonne réponse pour l'accès aux données en 2004. EF Core a changé la forme du problème. Voilà quand un repository mérite encore sa place, quand il duplique juste le DbContext, et comment trancher sur ton prochain projet."
---

Hello tous le monde, aujourd'hui on va explorer la question qui divise encore la communauté .NET : **faut-il emballer `DbContext` dans un Repository** ?

Demande à dix développeurs .NET, tu auras dix réponses, toutes confiantes et la moitié contradictoires. "Toujours", pour la testabilité. "Jamais", parce que `DbSet<T>` est déjà un repository. "Ça dépend", parce que tout dépend. La réponse honnête ressemble plus à la troisième, mais "ça dépend" n'est utile que si on sait *de quoi* ça dépend.

Cet article pose le problème original que le Repository pattern résolvait, ce qui a changé avec l'arrivée d'EF Core, et le cadre de décision concret pour les applications .NET d'aujourd'hui.

## Le contexte : pourquoi le Repository pattern existe

Le Repository pattern a été formalisé par Martin Fowler dans *Patterns of Enterprise Application Architecture* en 2002, et renforcé par Eric Evans dans *Domain-Driven Design* en 2003. Dans leur monde, "accès aux données" voulait dire ADO.NET brut : `SqlConnection`, `SqlCommand`, `DataReader`, mapping manuel des colonnes vers les propriétés, gestion manuelle des transactions, et des chaînes SQL éparpillées dans la base de code. Le Repository pattern donnait un endroit où mettre tout ça : une classe par agrégat, une interface sur laquelle le domaine pouvait s'appuyer, et une implémentation qui connaissait la base.

Il résolvait quatre vrais problèmes en même temps :

1. **Ignorance de la persistance.** Le modèle de domaine ne savait pas ce qu'était un `SqlCommand`.
2. **SQL centralisé.** La logique des requêtes vivait au même endroit, au lieu d'être saupoudrée dans chaque service.
3. **Testabilité unitaire.** On pouvait mocker `IOrderRepository` dans un test au lieu de monter une base.
4. **Une frontière de traduction plausible.** Les noms de colonnes, de tables et les JOIN restaient dans le repository, et le domaine restait propre.

En 2004, emballer ADO.NET dans des repositories, c'était le bon choix. Le pattern est devenu parole d'évangile, et pendant une décennie on l'a copié-collé dans chaque template .NET d'entreprise.

Puis EF Core est arrivé. `DbContext` est, par conception, un **Unit of Work**. `DbSet<T>` est, par conception, un **Repository**. `SaveChangesAsync()` est, par conception, le commit de ce Unit of Work. Quand on écrit `_db.Orders.Where(...).ToListAsync()`, on utilise déjà une abstraction repository, on ne l'a juste pas nommée comme ça. L'emballer dans un deuxième repository pour dire "on utilise le Repository pattern" ajoute de la cérémonie sans ajouter ce que le pattern original cherchait à ajouter.

## Vue d'ensemble : les trois positions possibles

Trois positions raisonnables existent sur le spectre, et chacune a des contextes où elle gagne.

{{< mermaid >}}
graph TD
    A[Besoin d'accès aux données] --> B{Position}
    B -->|1| C[Utiliser DbContext directement]
    B -->|2| D[Repository fin par agrégat]
    B -->|3| E[Repository abstrait complet<br/>avec interface générique]
    C --> F[Handlers appellent _db.Orders]
    D --> G[IOrderRepository<br/>IOrderReader]
    E --> H[IRepository&lt;T&gt; avec Add, Update, List, Find]
{{< /mermaid >}}

La position 1 fait confiance à EF Core comme couche d'accès aux données. La position 2 garde un repository fin, scopé à un agrégat, là où ça aide vraiment. La position 3, c'est le pattern "generic repository" que les tutoriels des années 2010 ont rendu célèbre et que la plupart des ingénieurs seniors regrettent aujourd'hui.

Le reste de l'article parcourt chaque position, montre le code, et explique où la ligne se situe.

## Zoom : position 1, utiliser DbContext directement

Dans une couche application teintée CQRS (voir [Couche Application : CQS et CQRS](/fr/posts/layer-focused-cqs-cqrs/)), le command handler tient un `DbContext` et écrit les queries directement :

```csharp
public sealed class SubmitOrderHandler : IRequestHandler<SubmitOrderCommand, SubmitOrderResult>
{
    private readonly ShopDbContext _db;
    private readonly IPaymentGateway _payments;

    public SubmitOrderHandler(ShopDbContext db, IPaymentGateway payments)
    {
        _db = db;
        _payments = payments;
    }

    public async Task<SubmitOrderResult> Handle(SubmitOrderCommand cmd, CancellationToken ct)
    {
        var order = await _db.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == cmd.OrderId, ct)
            ?? throw new NotFoundException($"Commande {cmd.OrderId} introuvable.");

        order.Submit();

        var charge = await _payments.ChargeAsync(order.CustomerId, order.Total, ct);
        if (!charge.Success)
            throw new PaymentFailedException(charge.Error);

        await _db.SaveChangesAsync(ct);
        return new SubmitOrderResult(order.Id, order.Status.ToString());
    }
}
```

Pas d'interface repository. Pas de mock d'`IOrderRepository`. Juste le handler, le `DbContext`, et des méthodes de domaine sur l'entité.

**Ce qu'on gagne :**

- Moins de code par handler. Pas d'interface, pas de classe d'implémentation, pas de double de test à câbler.
- L'accès complet aux fonctionnalités d'EF Core : `Include`, `AsNoTracking`, compiled queries, `ExecuteUpdate`, interceptors. Aucun n'a besoin d'être re-exposé à travers une interface repository.
- Les formes de requête peuvent être optimisées par handler. Le `SubmitOrderHandler` inclut les lignes ; un autre handler peut projeter dans un DTO sans passer par le domaine.

**Ce qu'on perd :**

- Une interface `IOrderRepository` mockable. Les handlers sont testés contre une vraie base (avec `Testcontainers` ou SQLite in-memory), pas contre un mock écrit à la main.
- Un foyer central pour la logique de requête réutilisable. Quand deux handlers ont besoin de "commandes passées dans les 30 derniers jours", soit on duplique la clause `Where`, soit on extrait une méthode d'extension.

C'est la position dans laquelle la plupart des bases de code .NET modernes finissent, surtout combinée avec le Vertical Slicing (voir [Vertical Slicing en .NET](/fr/posts/code-structure-vertical-slicing/)) ou une Clean Architecture où la couche application dépend d'un package d'abstraction EF Core.

> 💡 **Info** : `DbContext` implémente le pattern Unit of Work, et `DbSet<T>` implémente le Repository pattern. C'est documenté explicitement par l'équipe EF Core. Si on utilise déjà `DbContext`, on a déjà les deux patterns ; les emballer dans des équivalents écrits à la main est un choix stylistique, pas une évolution architecturale.

> ✅ **Bonne pratique** : Pour les tests d'intégration, utilise une vraie base dans un conteneur (PostgreSQL ou SQL Server) via Testcontainers. Le test tourne contre le même provider qu'en production, et les particularités de traduction LINQ et les soucis de migration remontent dans la CI, pas à 2 h du matin en incident.

## Zoom : position 2, un repository fin par agrégat

Parfois on veut vraiment un endroit nommé pour poser la logique de requête. La position 2 garde le `DbContext` à l'intérieur d'un petit repository scopé à un agrégat, avec des méthodes qui répondent à des questions de domaine, pas à des stubs CRUD :

```csharp
public interface IOrderRepository
{
    Task<Order?> FindForSubmissionAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<Order>> FindStaleAsync(TimeSpan olderThan, CancellationToken ct);
    void Add(Order order);
}

public sealed class OrderRepository : IOrderRepository
{
    private readonly ShopDbContext _db;

    public OrderRepository(ShopDbContext db) => _db = db;

    public Task<Order?> FindForSubmissionAsync(Guid id, CancellationToken ct) =>
        _db.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<Order>> FindStaleAsync(TimeSpan olderThan, CancellationToken ct)
    {
        var cutoff = DateTime.UtcNow - olderThan;
        return await _db.Orders
            .Where(o => o.Status == OrderStatus.Pending && o.PlacedAt < cutoff)
            .ToListAsync(ct);
    }

    public void Add(Order order) => _db.Orders.Add(order);
}
```

Remarque la forme de l'interface. Les méthodes sont **nommées selon les concepts de domaine** (`FindForSubmission`, `FindStale`), pas selon des opérations CRUD (`GetById`, `GetAll`, `Update`). Chaque méthode encode une façon précise dont le domaine veut charger ou sauvegarder un agrégat, avec les bons `Include` et les bons prédicats.

`SaveChangesAsync` n'est volontairement *pas* sur le repository. Ça transformerait chaque repository en son propre Unit of Work, et deux repositories dans le même handler essaieraient chacun de commiter. Le handler (ou un pipeline behavior) possède `SaveChangesAsync`.

**Quand ça paye :**

- L'application a une couche de domaine riche où charger un agrégat nécessite des chaînes de `Include` non triviales, de l'`AsSplitQuery`, ou un ordonnancement soigneux. Cacher ça au même endroit garde les handlers propres.
- Plusieurs handlers chargent le même agrégat de la même façon. Extraire vers une méthode de repository supprime la duplication sans imposer une forme abstraite.
- L'équipe utilise des **tests d'architecture** (voir [Tests d'Architecture en .NET](/fr/posts/testing-architecture-testing/)) pour s'assurer que la couche application ne dépend que d'interfaces, pas directement de `DbContext`.

**Quand ça ne paye pas :**

- L'application est une app CRUD avec 90 % de queries triviales. Chaque handler reçoit une méthode de repository sur mesure qui est utilisée exactement une fois. Le repository devient une couche de pass-through sans valeur.
- L'équipe fait du Vertical Slicing et veut que la forme de la query vive à côté du handler. Un repository éloigne la query de là où elle compte.

> ✅ **Bonne pratique** : Si on écrit un repository, les noms de méthodes décrivent l'**intention**, pas le SQL. `FindForSubmission` vaut mieux que `GetByIdWithLines`. Le premier nom survit à un refactor qui change les tables jointes ; le second non.

## Zoom : position 3, le generic repository, et pourquoi l'éviter

Le pattern "generic repository" est devenu célèbre vers 2010 à travers les tutoriels MVC et les scaffolds originaux unit-of-work plus generic-repository. Ça ressemble à ça :

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(object id, CancellationToken ct);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct);
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct);
    Task AddAsync(T entity, CancellationToken ct);
    void Update(T entity);
    void Delete(T entity);
}
```

Ça a l'air élégant. Une interface, une implémentation, toutes les entités en héritent gratuitement. Le problème, c'est que ça réinvente `DbSet<T>` avec moins de fonctionnalités et une moins bonne API.

- `GetAllAsync` invite à charger une table entière en mémoire.
- `FindAsync(Expression<Func<T, bool>>)` est une abstraction qui fuit : le prédicat doit être traduisible par EF Core, donc l'appelant doit savoir qu'il travaille avec EF Core de toute façon.
- `Update(T)` est ambigu : c'est `Attach + State = Modified`, ou `Update` (qui marque chaque propriété comme modifiée) ?
- Il n'y a pas de façon propre d'exprimer `Include`, `AsNoTracking` ou les projections, donc on finit avec `GetByIdWithLinesAsync`, `GetByIdNoTrackingAsync`, et une douzaine de variantes.
- Il faut quand même un Unit of Work quelque part pour appeler `SaveChanges`, donc l'abstraction ne cache pas complètement le cycle de vie du `DbContext`.

Au bout d'un an, chaque méthode de chaque entité a été ajoutée, l'interface fait deux cents lignes, et l'équipe debugge "pourquoi `Update` perd parfois des champs". Le bon geste, en général, c'est de supprimer le generic repository et de revenir à la position 1 ou 2.

> ❌ **Ne jamais faire** : Un `IRepository<T>` générique avec des prédicats `Expression<Func<T, bool>>` n'est pas une abstraction sur l'accès aux données. C'est `DbSet<T>` avec des trous. Si le but est l'ignorance de la persistance, ça ne le livre pas : l'appelant doit quand même écrire des expressions que seul EF Core peut traduire. Si le but est la testabilité, la position 1 avec Testcontainers donne une meilleure confiance pour un coût de code moindre.

## Zoom : la question de la testabilité

L'argument original pour "toujours repository" était "pour pouvoir le mocker dans les tests". Cet argument était fort quand les tests contre une vraie base voulaient dire installer SQL Server et écrire un script de reset à la main. Il est plus faible aujourd'hui.

- Les **tests unitaires** qui mockent un repository pour vérifier la logique du handler ont toujours de la valeur, et ils sont bon marché à écrire. Ils prouvent que *pour un ensemble d'objets de domaine, le handler produit le bon résultat*.
- Les **tests d'intégration** avec `WebApplicationFactory` et Testcontainers prouvent que *pour du vrai SQL, une vraie traduction LINQ et de vraies migrations, l'endpoint produit la bonne réponse HTTP*. Ils attrapent une classe de bugs que les mocks ne voient jamais : les N+1, les chaînes de `Include` cassées, les échecs de traduction, les dérives de migration.

Voir [Tests d'Intégration avec TestContainers](/fr/posts/testing-integration-testing-testcontainers/) et [Tests API avec WebApplicationFactory](/fr/posts/testing-webapplicationfactory/) pour la mise en place concrète.

En pratique, les équipes qui sautent les repositories au profit d'un usage direct de `DbContext` sont aussi les équipes qui investissent dans les tests d'intégration. Les deux décisions voyagent ensemble : on ne "perd pas la testabilité", on la *déplace* vers une couche qui attrape plus de vrais bugs.

> ⚠️ **Ça marche, mais...** : Mocker `DbContext` directement (avec `Mock<DbSet<T>>` ou équivalent) est possible et populaire dans les tutoriels, mais chaque base de code réelle qui l'a essayé a fini par tomber sur une expression LINQ que le mock ne pouvait pas traduire. Si on veut des handlers mockables, utilise la position 2 (repositories fins par agrégat) et mocke l'interface, pas le DbContext.

## Le cadre de décision

Voilà comment on tranche en pratique, sur un projet .NET neuf ou existant :

1. **Commence en position 1** (`DbContext` direct). C'est le minimum de code et le maximum de fonctionnalités. Ça colle naturellement au Vertical Slicing et au soft CQRS.
2. **Passe en position 2** (repository fin par agrégat) quand on remarque :
   - La même chaîne `Include` est dupliquée sur trois handlers ou plus.
   - La couche application est une frontière imposée (Clean Architecture, tests d'architecture) qui ne doit pas référencer EF Core directement.
   - On a besoin d'un foyer nommé pour des stratégies de chargement complexes, surtout autour de la cohérence des agrégats.
3. **Ne passe jamais en position 3** (`IRepository<T>` générique). Si on hérite d'une base de code qui l'utilise, traite-la comme de la dette technique et factorise-la progressivement vers la position 1 ou 2 à mesure qu'on touche chaque module.

Le choix n'est pas idéologique. Il porte sur l'endroit où la complexité vit réellement dans l'application.

## Wrap-up

Tu sais maintenant pourquoi le Repository pattern existait (ADO.NET brut, 2002, ignorance de la persistance), ce qu'EF Core a changé (`DbContext` *est* un Unit of Work et `DbSet<T>` *est* un repository), et comment choisir entre les trois positions raisonnables sur un projet .NET moderne. Tu peux livrer des handlers qui utilisent `DbContext` directement et les tester quand même avec Testcontainers. Tu peux emballer un agrégat dans un repository fin quand l'intention nommée aide vraiment. Tu peux refuser le générique `IRepository<T>` sans perdre le sommeil. Adapte l'abstraction à la forme du problème, pas aux habitudes de scaffolding d'il y a dix ans.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Endpoints en .NET : Controllers vs Minimal API, la comparaison honnête](/fr/posts/layer-focused-controllers-vs-minimal-api/)
- [Couche Application en .NET : CQS et CQRS sans le hype](/fr/posts/layer-focused-cqs-cqrs/)
- [Vertical Slicing en .NET : organise par fonctionnalité, pas par couche](/fr/posts/code-structure-vertical-slicing/)
- [Clean Architecture en .NET : des dépendances qui pointent dans le bon sens](/fr/posts/code-structure-clean-architecture/)

## Références

- [Repository pattern, Martin Fowler](https://martinfowler.com/eaaCatalog/repository.html)
- [Unit of Work pattern, Martin Fowler](https://martinfowler.com/eaaCatalog/unitOfWork.html)
- [DbContext et Unit of Work, Microsoft Learn](https://learn.microsoft.com/fr-fr/ef/core/miscellaneous/testing/)
- [Tester du code qui utilise EF Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/ef/core/testing/)
- [Conception de la couche de persistance, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)
