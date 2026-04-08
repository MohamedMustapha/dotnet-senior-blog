---
title: "Clean Architecture en .NET : des dépendances qui pointent dans le bon sens"
date: 2026-04-08
draft: false
tags: ["architecture", "dotnet", "clean-architecture"]
categories: ["Architecture"]
series: ["Structure de Code"]
description: "Clean Architecture, ce n'est pas quatre projets et un diagramme en cercles. C'est une seule règle : les dépendances pointent vers l'intérieur. Voilà comment la mettre en place en .NET, sans dogme."
---

Hello tous le monde, aujourd'hui on va démystifier la **Clean Architecture**, probablement le pattern le plus cité et le plus mal compris dans le monde .NET.

Clean Architecture a un problème de branding. Beaucoup de codebases .NET qui l'affichent dans leur README ressemblent en fait à du N-Couches avec des dossiers renommés, et d'autres accumulent assez d'interfaces, de mappers et de sauts de DTOs pour qu'un simple listing produit devienne un parcours à travers plusieurs fichiers. Ces deux situations s'expliquent facilement : le pattern est souvent introduit sans son cadrage d'origine, et les équipes comblent le vide avec du cérémonial. L'idée d'origine, celle que Robert Martin a formalisée en 2012 (en s'appuyant sur des travaux plus anciens comme l'Hexagonal Architecture d'Alistair Cockburn en 2005 et l'Onion Architecture de Jeffrey Palermo en 2008), est beaucoup plus petite et beaucoup plus utile : **tes règles métier ne doivent dépendre ni de ton framework, ni de la base de données, ni de ta stack HTTP**. Tout le reste, c'est du détail d'implémentation.

Si tu as lu les articles précédents de cette série, tu connais déjà les deux patterns qui sont venus avant : [l'architecture N-Couches](/fr/posts/code-structure-n-layered/) avec ses projets physiquement séparés, et [UI / Repositories / Services](/fr/posts/code-structure-ui-repos-services/) avec son découpage pragmatique à l'intérieur d'un seul projet. Clean Architecture, c'est ce que tu sors du placard quand ces patterns commencent à fuir et que tu as besoin du compilateur pour tenir la frontière entre le domaine et le monde extérieur.

## Le contexte : pourquoi ce pattern existe

Imaginons que nous ayons un codebase .NET mature. L'équipe livre depuis trois ans. Les requêtes EF Core vivent à l'intérieur des classes de service. Les controllers construisent des objets métier à la main depuis le body des requêtes. Un `UserService` référence `HttpContext` pour lire l'utilisateur courant. Une règle métier sur l'éligibilité aux promotions vit moitié dans une procédure stockée, moitié dans un `if` d'un controller, et moitié dans un fichier JavaScript côté front. Quand un nouveau prestataire de paiement arrive, personne ne peut modifier `OrderService` sans casser deux fonctionnalités, parce que la logique métier est emmêlée avec des appels spécifiques à Stripe.

Voilà la douleur que Clean Architecture soigne. Son contrat :

1. **Ton modèle de domaine ne connaît rien** à EF Core, ASP.NET, MediatR ou Stripe.
2. **Ta couche application** orchestre les cas d'usage en ne manipulant que des abstractions.
3. **L'infrastructure se branche depuis l'extérieur** et peut être remplacée sans toucher au domaine.

Le gain n'est pas théorique. C'est la possibilité de monter EF Core de version, de changer de bus de messages, ou de remplacer ton prestataire de paiement sans ouvrir le projet de domaine. C'est aussi la possibilité d'écrire des tests unitaires rapides sur tes règles métier sans démarrer une base de données.

## Vue d'ensemble : les couches et la règle

Avant de rentrer dans le code, voici les grandes briques de la Clean Architecture telles qu'on va les utiliser en .NET :

{{< mermaid >}}
graph TD
    A[Api / Presentation<br/>Controllers, Minimal APIs, SignalR] --> B[Application<br/>Use cases, commandes, queries, ports]
    B --> C[Domain<br/>Entités, value objects, services de domaine, invariants]
    D[Infrastructure<br/>EF Core, clients HTTP, fichiers, bus de messages] --> B
    D --> C
    A --> D
{{< /mermaid >}}

Les flèches, c'est la seule chose qui compte. **Tout pointe vers Domain.** Domain ne dépend de rien. Application ne dépend que de Domain. Infrastructure implémente les interfaces déclarées dans Application (ou dans Domain). Le projet Api fait le câblage au démarrage. Si les flèches sont bonnes, tu as de la Clean Architecture. Sinon, tu as quatre projets qui te coûtent le découpage sans te rendre le bénéfice.

> 💡 **Info** : Le diagramme d'origine montre quatre cercles concentriques (Entities, Use Cases, Interface Adapters, Frameworks). En pratique, la plupart des équipes .NET ramènent ça à quatre csproj : `Domain`, `Application`, `Infrastructure`, `Api`. Ce mapping est suffisant et c'est celui que j'utilise dans cet article.

## Zoom : Domain, le cœur

Le projet Domain contient tes concepts métier et leurs invariants. Il ne référence rien. Ni EF Core, ni MediatR, ni `Microsoft.Extensions.*`. Juste `net10.0` et tes propres types.

```csharp
// Domain/Orders/Order.cs
namespace Shop.Domain.Orders;

public sealed class Order
{
    private readonly List<OrderLine> _lines = new();

    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();
    public Money Total => new(_lines.Sum(l => l.Subtotal.Amount), Currency.Eur);

    private Order(OrderId id, CustomerId customerId)
    {
        Id = id;
        CustomerId = customerId;
        Status = OrderStatus.Draft;
    }

    public static Order Create(CustomerId customerId)
        => new(OrderId.New(), customerId);

    public void AddLine(ProductId productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("On ne peut pas modifier une commande déjà soumise.");
        if (quantity <= 0)
            throw new DomainException("La quantité doit être positive.");

        _lines.Add(new OrderLine(productId, quantity, unitPrice));
    }

    public void Submit()
    {
        if (_lines.Count == 0)
            throw new DomainException("Une commande vide ne peut pas être soumise.");
        Status = OrderStatus.Submitted;
    }
}
```

Remarque ce qui n'est **pas** là : pas d'attribut `[Table]`, pas de `DbContext`, pas de mot-clé `virtual` pour le lazy loading, pas de propriété de navigation qui présuppose EF Core. L'entité fait respecter ses propres invariants. Casser une règle lève une exception de domaine, pas un HTTP 400.

> ✅ **Bonne pratique** : Mets tes constructeurs en privé ou internal et expose des méthodes factory (`Order.Create(...)`). Ça force tous les appelants à passer par tes vérifications d'invariants. Il n'y a aucun moyen d'obtenir un `Order` cassé depuis l'extérieur.

> ❌ **Ne jamais faire** : Ne colle pas d'attributs `[Column]` ou `[Required]` sur tes entités de domaine pour "gagner du temps". Dès que tu le fais, le projet Domain gagne une dépendance dure sur un ORM, et l'invariant "Domain ne référence rien" cesse d'être vrai. L'API fluent d'EF Core dans Infrastructure te donne le même mapping sans faire fuiter le framework jusque dans tes entités.

## Zoom : Application, les cas d'usage

La couche Application décrit ce que le système *fait*, exprimé sous forme de cas d'usage. C'est ici que vivent les commandes et les queries, que les transactions sont coordonnées, et que tu déclares les *ports* (interfaces) que l'Infrastructure viendra brancher. Si le découpage commandes / queries est nouveau pour toi, l'article dédié [CQRS expliqué pour les développeurs .NET](/fr/posts/cqrs-explique/) traite le pattern en profondeur, cette section se contente d'en supposer la forme.

```csharp
// Application/Orders/SubmitOrder/SubmitOrderCommand.cs
public sealed record SubmitOrderCommand(Guid OrderId) : IRequest<Result>;

// Application/Orders/SubmitOrder/SubmitOrderHandler.cs
public sealed class SubmitOrderHandler : IRequestHandler<SubmitOrderCommand, Result>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;
    private readonly IPaymentGateway _payments;

    public SubmitOrderHandler(
        IOrderRepository orders,
        IUnitOfWork uow,
        IPaymentGateway payments)
    {
        _orders = orders;
        _uow = uow;
        _payments = payments;
    }

    public async Task<Result> Handle(SubmitOrderCommand cmd, CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(new OrderId(cmd.OrderId), ct);
        if (order is null)
            return Result.NotFound($"Commande {cmd.OrderId} introuvable.");

        order.Submit();

        var charge = await _payments.ChargeAsync(order.CustomerId, order.Total, ct);
        if (!charge.Success)
            return Result.Failure(charge.Error);

        await _uow.SaveChangesAsync(ct);
        return Result.Success();
    }
}
```

Les interfaces `IOrderRepository`, `IUnitOfWork` et `IPaymentGateway` vivent dans le projet Application, juste à côté du use case qui en a besoin. Elles décrivent *ce dont* l'application a besoin, pas *comment* c'est implémenté.

```csharp
// Application/Abstractions/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}
```

> 💡 **Info** : C'est le **Dependency Inversion Principle** rendu physique. La politique haute (Application) possède l'abstraction. Le détail bas (Infrastructure) l'implémente. La flèche va d'Infrastructure vers Application, et pas l'inverse.

> ⚠️ **Ça marche, mais...** : Tu verras des équipes mettre toutes leurs interfaces dans un projet séparé `Application.Contracts`, "pour la réutilisation". Dans 90% des cas, ce projet n'est importé que par Infrastructure et n'apporte rien. Garde les interfaces à côté de leurs cas d'usage, tant que tu n'as pas un vrai second consommateur.

## Zoom : Infrastructure, les prises

Infrastructure, c'est là où EF Core, les clients HTTP, les écritures fichiers et le bus de messages apparaissent enfin. Elle référence Application (pour implémenter les ports) et Domain (pour mapper les entités). Rien dans Application ou Domain ne référence Infrastructure.

```csharp
// Infrastructure/Persistence/OrderRepository.cs
internal sealed class OrderRepository : IOrderRepository
{
    private readonly ShopDbContext _db;

    public OrderRepository(ShopDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct)
        => _db.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct)
        => await _db.Orders.AddAsync(order, ct);
}
```

La configuration EF Core qui connaît les tables et les colonnes vit aussi ici, pas sur l'entité :

```csharp
// Infrastructure/Persistence/Configurations/OrderConfiguration.cs
internal sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.ToTable("Orders");
        b.HasKey(o => o.Id);
        b.Property(o => o.Id)
            .HasConversion(id => id.Value, value => new OrderId(value));
        b.Property(o => o.Status).HasConversion<string>();
        b.OwnsMany(o => o.Lines, lines =>
        {
            lines.ToTable("OrderLines");
            lines.WithOwner().HasForeignKey("OrderId");
        });
    }
}
```

> ✅ **Bonne pratique** : Marque tes implémentations d'Infrastructure en `internal`. La seule façon pour le monde extérieur d'obtenir un `IOrderRepository`, ça doit être via la DI. Si un controller peut faire `new OrderRepository(...)`, il y a un problème.

## Zoom : Api, la composition root

Le projet Api, c'est là où tout est câblé. Il référence Application et Infrastructure, enregistre les services, et expose les endpoints. Il reste fin : parser, déléguer, mapper le résultat.

```csharp
// Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplication();      // MediatR, validators, pipeline behaviors
builder.Services.AddInfrastructure(     // DbContext, repositories, clients HTTP
    builder.Configuration);

var app = builder.Build();
app.MapOrderEndpoints();
app.Run();

// Api/Endpoints/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/orders");

        g.MapPost("/{id:guid}/submit", async (
            Guid id,
            ISender mediator,
            CancellationToken ct) =>
        {
            var result = await mediator.Send(new SubmitOrderCommand(id), ct);
            return result.IsSuccess
                ? Results.NoContent()
                : result.ToProblemDetails();
        });
    }
}
```

Aucune logique métier ici. L'endpoint parse la route, dispatche la commande, traduit le résultat. Si tu dois changer de transport demain, un service gRPC ou un worker de fond, tu écris une nouvelle composition root et tu réutilises Application et Domain sans y toucher.

> 💡 **Info** : `AddApplication` et `AddInfrastructure` sont des méthodes d'extension qui vivent dans leur projet respectif. Ça garde chaque couche en charge de ses propres enregistrements, et le projet Api n'a pas besoin de savoir ce qu'est un `DbContext`.

## La règle que le compilateur doit faire respecter

L'intérêt de découper en quatre projets, c'est que le compilateur vérifie les flèches pour toi. Ton graphe de csproj doit ressembler à ça :

```
Api           -> Application, Infrastructure
Infrastructure-> Application, Domain
Application   -> Domain
Domain        -> (rien)
```

Si Domain gagne une référence vers quoi que ce soit, tu as un bug dans le pattern. Une bonne pratique, c'est d'ajouter un **test d'architecture** pour que la règle devienne exécutable :

```csharp
[Fact]
public void Domain_ne_doit_dependre_de_rien()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .ShouldNot()
        .HaveDependencyOnAny("Shop.Application", "Shop.Infrastructure", "Shop.Api")
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}
```

> ✅ **Bonne pratique** : Ajoute un test d'architecture par couche. Ça prend cinq minutes avec NetArchTest ou ArchUnitNET et ça attrape le `using Shop.Infrastructure;` accidentel avant qu'il ne devienne, en silence, un point d'appui du code.

## Quand Clean Architecture est le mauvais choix

Clean Architecture, c'est du surcoût. Quatre projets, une chorégraphie d'injection de dépendances, des handlers, des ports, des mappers. Pour une appli CRUD de trente endpoints où les "règles métier" se résument à "enregistre ça et renvoie-le", la structure coûte plus qu'elle ne rapporte. Dans ces cas-là, [UI / Repositories / Services](/fr/posts/code-structure-ui-repos-services/) est honnêtement meilleur et ça te fera livrer plus vite.

Sors la Clean Architecture quand au moins deux de ces conditions sont vraies :

- Tu as de vraies règles métier, pas juste du CRUD. Des invariants, des machines à états, des calculs, de la cohérence entre agrégats.
- L'application doit survivre à plusieurs frameworks, moteurs de stockage ou transports pendant sa vie.
- Plusieurs équipes ou développeurs ont besoin de frontières vraiment respectées.
- Tu vas écrire une quantité non négligeable de tests unitaires sur le domaine et tu ne veux pas embarquer une base de données dedans.

Si aucune de ces conditions n'est remplie, tu paies la taxe sans toucher les bénéfices.

## Wrap-up

Tu sais maintenant ce qu'est vraiment la Clean Architecture : une seule règle sur le sens des dépendances, respectée par les références de projets et parfois par des tests d'architecture. Tu peux monter les quatre projets, mettre tes invariants dans le Domain, garder tes cas d'usage dans Application, brancher Infrastructure depuis l'extérieur, et garder l'Api comme une composition root fine. Tu peux aussi reconnaître quand le codebase n'a pas besoin de ce niveau de structure et choisir une option plus légère.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [L'architecture N-Couches en .NET : les fondations que tu dois maîtriser](/fr/posts/code-structure-n-layered/)
- [UI / Repositories / Services : le découpage .NET pragmatique](/fr/posts/code-structure-ui-repos-services/)

## Références

- [Architectures d'applications web courantes, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
- [Pattern Ports et Adapters, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/architecture/patterns/ports-and-adapters)
- [Injection de dépendances dans ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/dependency-injection)
- [Configuration fluent d'Entity Framework Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/ef/core/modeling/)
- [NetArchTest sur GitHub](https://github.com/BenMorris/NetArchTest)
