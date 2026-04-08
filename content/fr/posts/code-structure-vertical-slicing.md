---
title: "Vertical Slicing en .NET : organise par fonctionnalité, pas par couche"
date: 2026-04-08
draft: false
tags: ["architecture", "dotnet", "vertical-slicing"]
categories: ["Architecture"]
series: ["Structure de Code"]
description: "Arrête de découper ton code en dossiers Controllers, Services et Repositories. Organise par fonctionnalité. Voilà le Vertical Slice Architecture en .NET, avec du code réaliste et les compromis dont personne ne parle."
---

Hello tous le monde, aujourd'hui on va découvrir le **Vertical Slicing**, le pattern qui remet en cause à peu près tout ce que les patterns précédents de cette série t'ont appris.

Tous les patterns en couches qu'on a vus jusqu'ici, le [N-Couches](/fr/posts/code-structure-n-layered/), le [UI / Repositories / Services](/fr/posts/code-structure-ui-repos-services/) et la [Clean Architecture](/fr/posts/code-structure-clean-architecture/), partagent la même hypothèse de base : la bonne façon de découper le code, c'est par *rôle technique*. Les controllers ici, les services là, les repositories au fond, les entités au milieu. C'est tellement ancré dans la communauté .NET que la plupart des devs ne remettent jamais ça en question.

Le Vertical Slicing le remet directement en question. Son affirmation est simple : **les fonctionnalités changent ensemble, donc elles doivent vivre ensemble**. Quand tu livres "soumettre une commande", tu touches un controller, un service, un validator, une requête, un DTO de réponse, et probablement un appel à la base. Dans un découpage horizontal, ces six morceaux vivent dans six dossiers différents. Dans une tranche verticale, ils vivent dans un seul. L'idée a été formalisée par Jimmy Bogard vers 2018, en s'appuyant sur son expérience avec MediatR et CQRS sur de vrais codebases .NET, comme réponse à la friction que les architectures en couches ajoutent au travail quotidien sur les fonctionnalités.

## Le contexte : pourquoi ce pattern existe

Supposons que nous soyons en planning de sprint. L'équipe prend quatre stories : "exporter les factures", "rembourser une commande", "envoyer un email de bienvenue", "mettre un produit en vedette". Dans une architecture horizontale, chaque story traverse cinq dossiers. Deux devs qui bossent sur deux stories finissent par éditer `OrderService.cs` en même temps, et à résoudre des conflits de merge dans un fichier qu'aucun des deux ne possède pleinement. Un troisième dev réutilise une méthode dans `OrderRepository` qui était taillée pour une autre fonctionnalité, et un bug subtil passe en prod. La revue de code prend plus de temps parce que les relecteurs doivent sauter entre sept fichiers pour suivre un seul changement. Rien de tout ça n'est la faute de quiconque : c'est le coût d'organiser le code par rôle technique quand le travail arrive fonctionnalité par fonctionnalité.

L'insight, c'est que les architectures en couches optimisent pour l'*axe de réutilisation*, qui est rarement l'axe de *changement*. Les fonctionnalités changent. Les couches, pas vraiment. Alors pourquoi on s'organise autour des couches ?

Le Vertical Slicing inverse le défaut :

1. **Chaque fonctionnalité possède son propre dossier**, avec tout ce dont elle a besoin : requête, handler, validator, réponse.
2. **La réutilisation inter-fonctionnalités est l'exception**, pas la règle. Un peu de duplication est acceptable quand ça isole les changements.
3. **Les abstractions émergent des patterns observés**, pas d'un jardinage préventif d'interfaces.

## Vue d'ensemble : la forme d'une tranche

Avant de rentrer dans le code, voici comment une tranche verticale s'installe dans un projet .NET :

{{< mermaid >}}
graph TD
    subgraph "Features/Orders/SubmitOrder"
        A[SubmitOrderEndpoint.cs]
        B[SubmitOrderCommand.cs]
        C[SubmitOrderHandler.cs]
        D[SubmitOrderValidator.cs]
        E[SubmitOrderResponse.cs]
    end
    subgraph "Features/Orders/GetOrderDetails"
        F[GetOrderDetailsEndpoint.cs]
        G[GetOrderDetailsQuery.cs]
        H[GetOrderDetailsHandler.cs]
        I[GetOrderDetailsResponse.cs]
    end
    subgraph "Shared"
        J[ShopDbContext]
        K[Entités de domaine]
    end
    C --> J
    H --> J
    C --> K
{{< /mermaid >}}

Deux fonctionnalités, deux dossiers, tout ce qu'il faut pour livrer une feature au même endroit. Le seul code partagé, c'est le `DbContext` et les entités de domaine, et c'est fait exprès.

> 💡 **Info** : Le Vertical Slice Architecture a été popularisé par Jimmy Bogard, le créateur de MediatR et AutoMapper. Ce n'est pas une spécification formelle. C'est un ensemble de principes à appliquer avec du jugement, et la disposition des dossiers n'est que la partie visible.

## Zoom : une vraie tranche, de bout en bout

Écrivons la fonctionnalité `SubmitOrder` comme une tranche autonome avec Minimal APIs, MediatR et FluentValidation. Tout vit dans `Features/Orders/SubmitOrder/`.

```csharp
// Features/Orders/SubmitOrder/SubmitOrderCommand.cs
public sealed record SubmitOrderCommand(Guid OrderId) : IRequest<SubmitOrderResponse>;

// Features/Orders/SubmitOrder/SubmitOrderResponse.cs
public sealed record SubmitOrderResponse(Guid OrderId, string Status, decimal Total);

// Features/Orders/SubmitOrder/SubmitOrderValidator.cs
public sealed class SubmitOrderValidator : AbstractValidator<SubmitOrderCommand>
{
    public SubmitOrderValidator()
    {
        RuleFor(x => x.OrderId).NotEmpty();
    }
}

// Features/Orders/SubmitOrder/SubmitOrderHandler.cs
public sealed class SubmitOrderHandler
    : IRequestHandler<SubmitOrderCommand, SubmitOrderResponse>
{
    private readonly ShopDbContext _db;
    private readonly IPaymentGateway _payments;

    public SubmitOrderHandler(ShopDbContext db, IPaymentGateway payments)
    {
        _db = db;
        _payments = payments;
    }

    public async Task<SubmitOrderResponse> Handle(
        SubmitOrderCommand cmd, CancellationToken ct)
    {
        var order = await _db.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == cmd.OrderId, ct)
            ?? throw new NotFoundException($"Commande {cmd.OrderId} introuvable.");

        order.Submit();

        var charge = await _payments.ChargeAsync(
            order.CustomerId, order.Total, ct);
        if (!charge.Success)
            throw new PaymentFailedException(charge.Error);

        await _db.SaveChangesAsync(ct);

        return new SubmitOrderResponse(
            order.Id, order.Status.ToString(), order.Total.Amount);
    }
}

// Features/Orders/SubmitOrder/SubmitOrderEndpoint.cs
public static class SubmitOrderEndpoint
{
    public static void MapSubmitOrder(this IEndpointRouteBuilder app)
    {
        app.MapPost("/orders/{id:guid}/submit", async (
            Guid id,
            ISender mediator,
            CancellationToken ct) =>
        {
            var response = await mediator.Send(new SubmitOrderCommand(id), ct);
            return Results.Ok(response);
        })
        .WithName("SubmitOrder")
        .WithTags("Orders");
    }
}
```

Cinq fichiers, un dossier, une fonctionnalité. Si tu dois comprendre tout le flux, tu ouvres le dossier et tu lis de haut en bas. Si tu dois changer la façon dont une commande est soumise, toutes les lignes que tu vas toucher sont dans le même répertoire. Pas de chasse au grep.

> ✅ **Bonne pratique** : Garde la requête, le validator, le handler et la réponse en `sealed`, scopés à la fonctionnalité. Ne les expose pas à l'extérieur. Si une autre tranche a besoin du même concept, c'est souvent le signe qu'il ne faut *pas* réutiliser : écris une nouvelle commande avec la forme qui colle au nouveau cas.

## Zoom : le côté lecture est encore plus simple

Les lectures n'ont pas besoin de passer par le modèle de domaine. Une query de tranche verticale peut projeter directement depuis EF Core (ou Dapper) vers la forme exacte de la réponse. Pas de repository, pas de mapper, pas de chaîne de montage de DTOs.

```csharp
// Features/Orders/GetOrderDetails/GetOrderDetailsQuery.cs
public sealed record GetOrderDetailsQuery(Guid OrderId)
    : IRequest<GetOrderDetailsResponse>;

// Features/Orders/GetOrderDetails/GetOrderDetailsResponse.cs
public sealed record GetOrderDetailsResponse(
    Guid Id,
    string CustomerName,
    decimal Total,
    string Status,
    IReadOnlyList<LineDto> Lines);

public sealed record LineDto(string ProductName, int Quantity, decimal Subtotal);

// Features/Orders/GetOrderDetails/GetOrderDetailsHandler.cs
public sealed class GetOrderDetailsHandler
    : IRequestHandler<GetOrderDetailsQuery, GetOrderDetailsResponse>
{
    private readonly ShopDbContext _db;

    public GetOrderDetailsHandler(ShopDbContext db) => _db = db;

    public async Task<GetOrderDetailsResponse> Handle(
        GetOrderDetailsQuery q, CancellationToken ct)
    {
        return await _db.Orders
            .AsNoTracking()
            .Where(o => o.Id == q.OrderId)
            .Select(o => new GetOrderDetailsResponse(
                o.Id,
                o.Customer.Name,
                o.Lines.Sum(l => l.Quantity * l.UnitPrice),
                o.Status.ToString(),
                o.Lines.Select(l => new LineDto(
                    l.Product.Name, l.Quantity, l.Quantity * l.UnitPrice))
                    .ToList()))
            .FirstOrDefaultAsync(ct)
            ?? throw new NotFoundException($"Commande {q.OrderId} introuvable.");
    }
}
```

Une seule requête SQL. Une seule projection. Zéro repository. Le côté lecture ne prétend pas respecter le modèle de domaine, parce qu'il n'en a pas besoin : il n'y a pas d'invariant à faire respecter quand tu affiches juste des données.

> 💡 **Info** : C'est l'idée du **CQRS** appliquée au niveau de la tranche. Les commandes passent par le domaine (pour faire respecter les invariants). Les queries le court-circuitent (pour la vitesse et la simplicité). Pas besoin de bases de données séparées ou d'event sourcing pour en profiter.

> ⚠️ **Ça marche, mais...** : Résiste à la tentation d'introduire une interface `IReadRepository` pour les queries. Ça ajoute une couche d'indirection qu'aucune autre fonctionnalité ne réutilisera jamais. Si tu veux le mocker pour les tests, mocke le `DbContext` ou utilise un provider in-memory.

## Zoom : où vivent encore les abstractions

Vertical Slicing, ce n'est pas "pas de code partagé du tout". Certaines choses sont vraiment partagées et ont leur place en dehors des tranches :

- **Les entités de domaine et les value objects** qui portent des invariants. `Order.Submit()` vit toujours dans le domaine. Les tranches l'appellent.
- **L'infrastructure transverse** : le `DbContext`, le bus de messages, l'expéditeur d'emails, l'interface du prestataire de paiement.
- **Les pipeline behaviors** (logging, validation, transaction) qui s'exécutent autour de chaque handler.

Un layout de projet typique ressemble à ça :

```
src/
  Shop.Api/
    Features/
      Orders/
        SubmitOrder/
        GetOrderDetails/
        CancelOrder/
      Customers/
        Register/
        UpdateProfile/
    Domain/              (entités, value objects)
    Infrastructure/      (DbContext, configs EF, clients externes)
    Common/              (pipeline behaviors, problem details, result types)
    Program.cs
```

Remarque l'absence des dossiers `Controllers/`, `Services/` et `Repositories/`. Ces formes émergent par fonctionnalité, pas imposées au niveau du projet.

> ✅ **Bonne pratique** : Ajoute un pipeline behavior MediatR pour la validation, comme ça chaque handler récupère FluentValidation gratuitement. Un fichier dans `Common/`, toutes les tranches en profitent, aucune tranche n'a à le câbler.

```csharp
// Common/Behaviors/ValidationBehavior.cs
public sealed class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!_validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(_validators
                .Select(v => v.ValidateAsync(context, ct))))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next();
    }
}
```

## La question de la duplication

Le Vertical Slicing va te pousser à écrire du code qui a l'air dupliqué. Deux tranches vont toutes les deux lire une commande par id. Deux handlers vont toutes les deux utiliser `ShopDbContext`. Un dev junior va demander "on extrait pas ça dans un helper ?". La réponse, neuf fois sur dix, c'est **non**.

La duplication est bon marché. Le couplage, lui, coûte cher. Dès que tu extrais une méthode partagée utilisée par deux tranches, les tranches cessent d'être indépendantes : changer l'une sans vérifier l'autre devient impossible. Quelques lignes d'EF Core dupliquées, c'est une feature, pas un bug. Ça laisse la tranche A évoluer sans casser la tranche B.

Extrais uniquement quand :

- Tu retrouves le même pattern dans **trois tranches ou plus**.
- Le pattern est vraiment stable et a un nom clair.
- L'extraction supprime un vrai risque, pas juste des lignes.

> ❌ **Ne jamais faire** : Évite de construire une `BaseHandler<TCommand, TResponse>` avec des helpers protégés partagés entre les fonctionnalités. Ça a l'air propre au jour 1, et au bout de six mois le moindre changement sur la classe de base impacte toutes les tranches d'un coup, ce qui est exactement le couplage que le Vertical Slicing cherche à éviter. Garde chaque tranche indépendamment supprimable.

## Là où ça commence à faire mal

Aucun pattern n'est gratuit. Les modes de défaillance du Vertical Slicing sont différents de ceux des patterns en couches, et il faut les connaître :

- **Pas de frontières de domaine imposées** : rien n'empêche un handler de court-circuiter une méthode de domaine et de muter une entité directement. Il faut de la discipline, ou des tests d'architecture, pour garder les invariants à leur place.
- **Discoverability pour les nouveaux** : un dev habitué à "je dois changer le service des commandes, j'ouvre `OrderService.cs`" doit apprendre un nouveau modèle mental. "Je dois changer la façon dont les commandes sont soumises, j'ouvre `Features/Orders/SubmitOrder/`."
- **Coordination inter-tranches** : quand une règle métier s'étend sur quatre fonctionnalités, tu as quatre endroits à mettre à jour. Un bon nommage et les événements de domaine aident, mais c'est du vrai boulot.
- **Peu d'intérêt sur des très petites applis** : si tu as quinze endpoints et pas de vrai domaine, un découpage en tranches verticales a un côté overkill. UI / Repos / Services reste probablement le bon choix.

Le Vertical Slicing brille sur des applications de taille moyenne à large, avec des équipes actives, où les fonctionnalités changent souvent et où deux devs sur deux fonctionnalités ne doivent pas se marcher dessus.

## Wrap-up

Tu sais maintenant ce qu'est vraiment le Vertical Slicing : organiser ton codebase par fonctionnalité pour que tout ce qu'il faut pour livrer un changement vive au même endroit. Tu peux écrire des tranches autonomes avec commande, handler, validator et endpoint, court-circuiter le domaine sur les chemins de lecture pour des queries plus simples, garder les vraies préoccupations partagées dans un dossier `Common` ou `Infrastructure` fin, et résister à la tentation de dédupliquer trop tôt. Tu peux aussi reconnaître quand ce pattern colle et quand un découpage plus traditionnel reste la bonne réponse.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [L'architecture N-Couches en .NET : les fondations que tu dois maîtriser](/fr/posts/code-structure-n-layered/)
- [UI / Repositories / Services : le découpage .NET pragmatique](/fr/posts/code-structure-ui-repos-services/)
- [Clean Architecture en .NET : des dépendances qui pointent dans le bon sens](/fr/posts/code-structure-clean-architecture/)

## Références

- [Architectures d'applications web courantes, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
- [Minimal APIs dans ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/minimal-apis)
- [MediatR sur GitHub](https://github.com/jbogard/MediatR)
- [Documentation FluentValidation](https://docs.fluentvalidation.net/)
- [Entity Framework Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/ef/core/)
