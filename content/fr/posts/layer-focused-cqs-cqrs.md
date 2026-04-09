---
title: "Couche Application en .NET : CQS et CQRS sans le hype"
date: 2026-04-10
draft: false
tags: ["application-layer", "cqrs", "cqs", "dotnet"]
categories: ["Architecture"]
series: ["Layer Focused"]
description: "CQS est une règle de design au niveau méthode. CQRS est un pattern architectural construit par-dessus. Les confondre, c'est comme ça qu'une équipe se retrouve avec deux bases de données pour un CRUD à trois écrans. Voilà le partage honnête, avec du code .NET qui tourne."
---

Hello tous le monde, aujourd'hui on va démystifier **CQS et CQRS**, deux des acronymes les plus mal cités de la sphère .NET.

On entend "on fait du CQRS" pour dire "on a un pipeline MediatR", ou "CQRS c'est overkill pour nous" pour dire en fait "on n'a pas besoin d'event sourcing". Les deux idées sont liées, mais elles vivent à des niveaux différents : CQS est une règle sur les méthodes, CQRS est une règle sur des chemins entiers de lecture et d'écriture. Savoir laquelle on est en train d'appliquer, c'est la différence entre une couche application propre et deux ans de regret.

Cet article déplie les deux, avec du code ASP.NET Core réaliste, et montre où se situe la ligne en pratique.

## Le contexte : pourquoi ces idées existent

**CQS (Command Query Separation)** a été formulé par Bertrand Meyer dans les années 1980 dans *Object-Oriented Software Construction*. Sa règle est simple : une méthode doit soit exécuter une action (une commande, qui change l'état et ne retourne rien), soit répondre à une question (une query, qui retourne des données sans modifier d'état), mais pas les deux. Le but n'est pas une pureté religieuse, c'est de rendre le raisonnement sur le code plus facile. Si on sait qu'une méthode est une query, on peut l'appeler cent fois dans un debugger sans se soucier des effets de bord.

**CQRS (Command Query Responsibility Segregation)** a été introduit par Greg Young vers 2010, en s'appuyant sur les idées d'Udi Dahan et sur sa propre expérience avec l'event sourcing. Young a pris CQS et l'a poussé d'un cran plus haut : au lieu de séparer les commandes et les queries au niveau de la méthode, on les sépare au niveau *architectural*. Les écritures et les lectures deviennent deux chemins différents à travers l'application, potentiellement avec deux modèles différents, deux stores différents, et deux équipes différentes. CQRS est une réponse à une douleur précise : essayer de servir un modèle d'écriture riche et une dizaine de vues de lecture différentes à partir du même ensemble d'entités transforme les classes de domaine en sac de `IsXxx`, `HasYyy`, et "à inclure seulement sur l'écran de résumé".

Les deux idées ont la même lignée, mais elles résolvent des problèmes à des échelles très différentes. CQS, c'est un truc qu'on devrait probablement faire dans chaque classe qu'on écrit. CQRS, c'est un truc qu'on sort quand la complexité des lectures diverge vraiment de la complexité des écritures.

## Vue d'ensemble : où chacun vit

{{< mermaid >}}
graph TD
    subgraph CQS["CQS : règle au niveau méthode"]
        A[OrderService] --> B["PlaceOrder cmd : void"]
        A --> C["GetTotal query : decimal"]
    end
    subgraph CQRS["CQRS : séparation architecturale"]
        D[API] --> E[Côté commande]
        D --> F[Côté query]
        E --> G[Modèle de domaine]
        G --> H[(Store d'écriture)]
        F --> I[(Store de lecture<br/>ou projections)]
    end
{{< /mermaid >}}

CQS vit dans une classe. CQRS vit à travers le code. On peut faire CQS sans CQRS. On peut aussi faire CQRS sans event sourcing, sans deux bases de données, et sans bus de messages. Perdre ces distinctions, c'est comme ça que "ajoutons CQRS" devient un projet de six mois.

## Zoom : CQS dans un service, de bout en bout

Partons d'un service banal qui viole CQS, puis corrigeons-le. Voilà le genre de méthode que tu as presque certainement écrit ou relu :

```csharp
public sealed class OrderService
{
    private readonly ShopDbContext _db;

    public OrderService(ShopDbContext db) => _db = db;

    // Viole CQS : modifie l'état ET retourne des données.
    public async Task<Order> PlaceOrderAsync(PlaceOrderInput input, CancellationToken ct)
    {
        var order = new Order(input.CustomerId, input.Lines);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);
        return order;
    }
}
```

La méthode enregistre une commande *et* retourne l'entité complète. L'appelant a maintenant un `Order` tracké avec des propriétés de navigation chargeables en lazy, qui peuvent ou non être utilisables en sécurité en dehors de cette transaction. Deux préoccupations indépendantes voyagent sur la même valeur de retour. Les tests qui veulent affirmer "la commande a été enregistrée" finissent par aussi affirmer "l'entité retournée a la forme attendue".

La version CQS le sépare :

```csharp
public sealed class OrderService
{
    private readonly ShopDbContext _db;

    public OrderService(ShopDbContext db) => _db = db;

    // Commande : modifie l'état, retourne uniquement ce dont l'appelant a strictement besoin (un id).
    public async Task<Guid> PlaceOrderAsync(PlaceOrderInput input, CancellationToken ct)
    {
        var order = new Order(input.CustomerId, input.Lines);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);
        return order.Id;
    }

    // Query : retourne des données, ne mute jamais.
    public async Task<OrderSummary> GetOrderSummaryAsync(Guid id, CancellationToken ct)
    {
        return await _db.Orders
            .AsNoTracking()
            .Where(o => o.Id == id)
            .Select(o => new OrderSummary(o.Id, o.Status.ToString(), o.Total))
            .FirstAsync(ct);
    }
}
```

Deux méthodes, deux intentions. La commande retourne un id (un puriste retournerait `void`, mais un id est l'information minimale dont l'appelant a besoin pour router vers l'écran suivant, et ça reste trivialement "pas de l'état du domaine"). La query est en `AsNoTracking`, projette dans un DTO, et n'a aucune logique métier.

> 💡 **Info** : Retourner un id depuis une commande est le compromis pragmatique que toute base de code .NET sérieuse finit par faire. Le CQS strict façon Meyer retournerait `void` et obligerait l'appelant à enchaîner avec une query. En pratique, ça coûte un aller-retour pour rien. Retourner `Guid` (ou `Result<Guid>`), c'est très bien.

> ✅ **Bonne pratique** : Écris les queries avec `AsNoTracking()` et des projections vers des DTOs par défaut. Le change tracker est une fonctionnalité qu'on paye à chaque chargement, et les queries n'en ont presque jamais besoin.

## Zoom : CQRS avec MediatR, la version honnête

CQRS au niveau architectural, ça veut dire : le chemin des commandes et le chemin des queries sont deux formes différentes, même s'ils partagent la même base de données. En .NET, la façon la plus courante d'exprimer ça, c'est avec MediatR (ou n'importe quel mediator in-process), en envoyant des objets `ICommand<T>` et `IQuery<T>` depuis la couche API vers des handlers.

Voilà un handler de commande, le chemin d'écriture, qui passe par le modèle de domaine :

```csharp
// Application/Orders/Commands/SubmitOrderCommand.cs
public sealed record SubmitOrderCommand(Guid OrderId) : IRequest<SubmitOrderResult>;
public sealed record SubmitOrderResult(Guid OrderId, string Status);

// Application/Orders/Commands/SubmitOrderHandler.cs
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

        order.Submit(); // fait respecter les invariants dans l'agrégat

        var charge = await _payments.ChargeAsync(order.CustomerId, order.Total, ct);
        if (!charge.Success)
            throw new PaymentFailedException(charge.Error);

        await _db.SaveChangesAsync(ct);
        return new SubmitOrderResult(order.Id, order.Status.ToString());
    }
}
```

Et voilà un handler de query, le chemin de lecture, qui court-circuite complètement le domaine et projette directement depuis le DbContext vers la forme que l'UI veut réellement :

```csharp
// Application/Orders/Queries/GetOrderListQuery.cs
public sealed record GetOrderListQuery(int Page, int PageSize, string? Status)
    : IRequest<PagedResult<OrderListItem>>;

public sealed record OrderListItem(
    Guid Id, string CustomerName, decimal Total, string Status, DateTime PlacedAt);

// Application/Orders/Queries/GetOrderListHandler.cs
public sealed class GetOrderListHandler
    : IRequestHandler<GetOrderListQuery, PagedResult<OrderListItem>>
{
    private readonly ShopDbContext _db;

    public GetOrderListHandler(ShopDbContext db) => _db = db;

    public async Task<PagedResult<OrderListItem>> Handle(
        GetOrderListQuery q, CancellationToken ct)
    {
        var query = _db.Orders.AsNoTracking();

        if (!string.IsNullOrWhiteSpace(q.Status))
            query = query.Where(o => o.Status.ToString() == q.Status);

        var total = await query.CountAsync(ct);

        var items = await query
            .OrderByDescending(o => o.PlacedAt)
            .Skip((q.Page - 1) * q.PageSize)
            .Take(q.PageSize)
            .Select(o => new OrderListItem(
                o.Id, o.Customer.Name, o.Total, o.Status.ToString(), o.PlacedAt))
            .ToListAsync(ct);

        return new PagedResult<OrderListItem>(items, total, q.Page, q.PageSize);
    }
}
```

Même `DbContext`. Une seule base. Deux formes de code très différentes. La commande passe par les agrégats pour protéger les invariants. La query contourne les agrégats pour livrer des pixels. C'est CQRS à son maximum d'utilité : une séparation architecturale propre *sans* deux bases, *sans* event sourcing, *sans* casse-tête de cohérence éventuelle.

> 💡 **Info** : Une seule base, deux chemins de code. On appelle parfois ça "soft CQRS" ou "CQRS lite", et pour la plupart des applications métier, c'est la version qui mérite sa place. Pour l'histoire complète de l'évolution de CQRS et de quand les variantes plus lourdes ont du sens, voir [CQRS expliqué pour les développeurs .NET](/fr/posts/cqrs-explique/).

## Zoom : où s'insère la couche endpoint

Les commandes et les queries doivent être dispatchées depuis quelque part. Les endpoints (Controllers ou Minimal APIs) deviennent de fins dispatchers : ils bindent la requête, appellent `mediator.Send`, et retournent le résultat.

```csharp
// Endpoint, style Minimal API
orders.MapPost("/{id:guid}/submit", async (
    Guid id, ISender mediator, CancellationToken ct) =>
{
    var result = await mediator.Send(new SubmitOrderCommand(id), ct);
    return TypedResults.Ok(result);
});

orders.MapGet("/", async (
    int page, int pageSize, string? status,
    ISender mediator, CancellationToken ct) =>
{
    var result = await mediator.Send(new GetOrderListQuery(page, pageSize, status), ct);
    return TypedResults.Ok(result);
});
```

Aucune logique métier dans l'endpoint. Binding, dispatch, retour. Pour le compromis sur le style d'endpoint (et pourquoi les deux exemples ci-dessus utilisent Minimal APIs), voir [Endpoints en .NET : Controllers vs Minimal API](/fr/posts/layer-focused-controllers-vs-minimal-api/).

> ✅ **Bonne pratique** : Ajoute un pipeline behavior MediatR pour les préoccupations transverses (validation, logging, transactions). Un seul behavior enregistré dans le conteneur de DI s'applique à chaque commande et chaque query, ce qui évite de parsemer des try/catch ou des `using var tx = ...` dans chaque handler.

```csharp
public sealed class TransactionBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ShopDbContext _db;

    public TransactionBehavior(ShopDbContext db) => _db = db;

    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        // On n'emballe que les commandes dans une transaction, pas les queries.
        if (request is not ICommand) return await next();

        await using var tx = await _db.Database.BeginTransactionAsync(ct);
        var response = await next();
        await tx.CommitAsync(ct);
        return response;
    }
}
```

## Zoom : quand CQRS mérite sa complexité

Un setup soft CQRS (handlers séparés, base partagée) est presque toujours une bonne idée dans une application non triviale. Les variantes plus lourdes sont une autre conversation :

- **Store de lecture séparé** : quand le modèle de lecture est vraiment différent (dénormalisé, indexé full-text, distribué géographiquement), et que le garder en phase avec le store d'écriture vaut le coût opérationnel.
- **Event sourcing** : quand on a besoin de reconstruire l'historique, de supporter des queries temporelles, ou d'auditer chaque changement d'état. C'est un engagement énorme. Ça change la façon dont on conçoit les tests, les migrations et les déploiements.
- **APIs de commande et de query séparées** : quand des équipes différentes possèdent les lectures et les écritures, ou quand il faut scaler les deux indépendamment.

Chacune de ces couches résout un vrai problème, et chacune ajoute un vrai coût. La bonne question n'est pas "doit-on faire CQRS", c'est "quelle couche de CQRS est-ce que le problème réclame vraiment".

> ⚠️ **Ça marche, mais...** : Ajouter une base de lecture séparée et un bus de messages à un système avec 3 agrégats et 12 endpoints est un classique de la sur-ingénierie. Les handlers séparés à l'intérieur d'une seule base donnent l'essentiel du bénéfice (chemins de lecture propres, pas de pollution du domaine, optimisation de queries plus facile) avec quasiment aucun du coût.

> ❌ **Ne jamais faire** : Ne force pas les queries à passer par les agrégats du domaine "par cohérence". Une query qui charge un agrégat `Order`, parcourt ses lignes, et les reformate en DTO en C#, sera plus lente, consommera plus de mémoire, et sera plus dure à maintenir qu'une simple projection `Select`. Les lectures et les écritures ont le droit d'avoir des formes différentes, et c'est voulu.

## Wrap-up

Tu sais maintenant la différence entre CQS et CQRS : CQS est la règle au niveau méthode qui dit qu'une méthode doit soit faire soit demander, jamais les deux ; CQRS est le pattern architectural qui prend cette règle et la remonte à l'échelle des chemins de commande et de query de l'application. Tu peux écrire du CQS propre dans n'importe quelle classe de service dès aujourd'hui. Tu peux adopter un soft CQRS avec MediatR et une seule base sur n'importe quel projet .NET greenfield. Et tu peux sortir les variantes plus lourdes (stores séparés, event sourcing) uniquement quand le problème l'exige, pas parce qu'un article de blog l'a dit.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Endpoints en .NET : Controllers vs Minimal API, la comparaison honnête](/fr/posts/layer-focused-controllers-vs-minimal-api/)
- [CQRS expliqué pour les développeurs .NET](/fr/posts/cqrs-explique/)
- [Vertical Slicing en .NET : organise par fonctionnalité, pas par couche](/fr/posts/code-structure-vertical-slicing/)

## Références

- [Command Query Separation, Bertrand Meyer (référence livre original)](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)
- [CQRS Documents, Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
- [MediatR sur GitHub](https://github.com/jbogard/MediatR)
- [Entity Framework Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/ef/core/)
- [Pattern CQRS, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/architecture/patterns/cqrs)
