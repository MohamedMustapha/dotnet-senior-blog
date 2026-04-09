---
title: "Endpoints en .NET : Controllers vs Minimal API, la comparaison honnête"
date: 2026-04-09
draft: false
tags: ["endpoints", "controllers", "minimal-api", "dotnet"]
categories: ["Architecture"]
series: ["Layer Focused"]
description: "Controllers ou Minimal APIs ? Les deux livrent des endpoints HTTP en ASP.NET Core, les deux sont pleinement supportés, et leurs défauts respectifs sont différents. Voilà ce qui compte vraiment au moment de choisir."
---

Hello tous le monde, aujourd'hui on va démystifier le choix entre **Controllers et Minimal APIs** en ASP.NET Core.

Tout projet ASP.NET Core commence par la même décision. Tu scaffoldes un dossier `Controllers/` avec des classes `[ApiController]`, ou tu déclares les endpoints dans `Program.cs` avec `app.MapPost(...)` et une lambda ? Les deux styles font le même boulot, vivent dans le même framework, et sont supportés à long terme. Les différences apparaissent dans la quantité de cérémonie qu'ils imposent, dans la façon dont les filtres et le middleware se composent, et dans la manière dont ça tient la route quand l'équipe grossit et que la base de code mûrit.

Cet article compare les deux styles tête à tête, avec du code réaliste, et explique quand chacun mérite sa place.

## Le contexte : pourquoi ce choix existe

Les controllers sont dans ASP.NET depuis MVC 1 en 2009. Les controllers Web API sont arrivés en 2012, et les deux ont été unifiés sous le modèle unique d'ASP.NET Core en 2016. Pendant une décennie, quand on écrivait un endpoint HTTP en .NET, on écrivait un controller. Le model binding, les filtres, les attributs pour la métadonnée, les conventions : tout est profondément enraciné.

Les Minimal APIs sont arrivées avec .NET 6 en novembre 2021, en réponse à une observation très précise : pour un petit service ou un endpoint simple, la cérémonie d'une classe controller, d'une classe de base, d'attributs de routage et d'une méthode d'action représente beaucoup de frappes pour une ligne de travail utile. L'équipe menée par Damian Edwards et David Fowler est partie de la question "quel est le minimum absolu de code pour mapper une URL à un handler ?" et a construit à partir de là.

Pendant la première année, il manquait des morceaux visibles aux Minimal APIs : pas d'endpoint filters, pas de typed results, OpenAPI faible, pas de raccourci `[FromServices]`. .NET 7 et .NET 8 ont comblé la plupart de ces trous. Avec .NET 9, l'écart est suffisamment fin pour que le choix soit vraiment une décision de style et d'architecture, pas une question de capacités.

## Vue d'ensemble : les deux modèles mentaux

Avant le code, voici comment chaque style pose le même endpoint dans ta tête.

{{< mermaid >}}
graph LR
    subgraph Controllers
        direction TB
        C1[Program.cs<br/>AddControllers] --> C2[OrdersController]
        C2 --> C3[Méthode d'action]
        C3 --> C4[Pipeline de filtres]
        C4 --> C5[Logique du handler]
    end
    subgraph Minimal["Minimal API"]
        direction TB
        M1[Program.cs] --> M2[Lambda MapPost]
        M2 --> M3[Endpoint filters]
        M3 --> M4[Logique du handler]
    end
{{< /mermaid >}}

Même requête, même réponse, même middleware, même injection de dépendances. La différence, c'est où l'endpoint est déclaré et comment la métadonnée lui est attachée. Les controllers s'appuient sur les attributs et les conventions. Les Minimal APIs s'appuient sur le chaînage fluent.

## Zoom : le même endpoint, écrit des deux façons

Implémentons un endpoint classique, `POST /orders`, dans les deux styles, avec validation, autorisation, typed results et métadonnée OpenAPI. C'est ce qu'on livrerait vraiment en production.

### Version Controller

```csharp
// Controllers/OrdersController.cs
[ApiController]
[Route("orders")]
[Authorize]
public sealed class OrdersController : ControllerBase
{
    private readonly ISender _mediator;

    public OrdersController(ISender mediator) => _mediator = mediator;

    [HttpPost]
    [ProducesResponseType(typeof(CreateOrderResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<CreateOrderResponse>> Create(
        [FromBody] CreateOrderCommand command,
        CancellationToken ct)
    {
        var response = await _mediator.Send(command, ct);
        return CreatedAtAction(nameof(GetById), new { id = response.OrderId }, response);
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(OrderDetailsResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDetailsResponse>> GetById(
        Guid id, CancellationToken ct)
    {
        var response = await _mediator.Send(new GetOrderDetailsQuery(id), ct);
        return Ok(response);
    }
}
```

Câblé dans `Program.cs` en deux lignes :

```csharp
builder.Services.AddControllers();
// ...
app.MapControllers();
```

### Version Minimal API

```csharp
// Features/Orders/OrdersEndpoints.cs
public static class OrdersEndpoints
{
    public static IEndpointRouteBuilder MapOrders(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapPost("/", async (
                CreateOrderCommand command,
                ISender mediator,
                CancellationToken ct) =>
            {
                var response = await mediator.Send(command, ct);
                return TypedResults.Created($"/orders/{response.OrderId}", response);
            })
            .WithName("CreateOrder")
            .Produces<CreateOrderResponse>(StatusCodes.Status201Created)
            .ProducesValidationProblem();

        group.MapGet("/{id:guid}", async Task<Results<Ok<OrderDetailsResponse>, NotFound>> (
                Guid id,
                ISender mediator,
                CancellationToken ct) =>
            {
                var response = await mediator.Send(new GetOrderDetailsQuery(id), ct);
                return TypedResults.Ok(response);
            })
            .WithName("GetOrderById");

        return app;
    }
}
```

Câblé dans `Program.cs` en une ligne :

```csharp
app.MapOrders();
```

Deux styles, même comportement, même table de routage, même document OpenAPI. La version controller est plus déclarative (les attributs parlent), la version Minimal est plus explicite (la chaîne te dit exactement ce qui s'applique à quoi).

> 💡 **Info** : `TypedResults` (la variante typée de `Results`) est le type de retour recommandé en Minimal APIs depuis .NET 7. Il permet d'exprimer la réponse dans la signature, ce qui améliore à la fois la testabilité et donne au générateur OpenAPI assez d'information pour décrire chaque branche sans attributs supplémentaires.

## Zoom : les filtres, le moment où les deux styles divergent

Les préoccupations transverses (logging, validation, résolution de tenant, idempotence) sont l'endroit où les deux modèles se sentent le plus différents.

Les controllers utilisent les **action filters**, qui existent depuis MVC 1 et forment un pipeline riche : authorization filters, resource filters, action filters, exception filters, result filters. Ils peuvent court-circuiter le pipeline, modifier les arguments de l'action, emballer le résultat. Ils sont puissants et bien compris.

```csharp
public sealed class IdempotencyFilter : IAsyncActionFilter
{
    private readonly IIdempotencyStore _store;

    public IdempotencyFilter(IIdempotencyStore store) => _store = store;

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var key = context.HttpContext.Request.Headers["Idempotency-Key"].ToString();
        if (string.IsNullOrWhiteSpace(key))
        {
            context.Result = new BadRequestObjectResult("Header Idempotency-Key requis.");
            return;
        }

        if (await _store.HasSeenAsync(key))
        {
            context.Result = new StatusCodeResult(StatusCodes.Status409Conflict);
            return;
        }

        await next();
        await _store.MarkAsync(key);
    }
}

// Utilisation
[ServiceFilter(typeof(IdempotencyFilter))]
[HttpPost]
public async Task<ActionResult<CreateOrderResponse>> Create(...) { }
```

Les Minimal APIs utilisent les **endpoint filters**, introduits en .NET 7. Ils sont plus simples (une interface, un delegate), se composent par chaînage de `.AddEndpointFilter(...)`, et s'exécutent après le model binding mais avant le handler.

```csharp
public sealed class IdempotencyFilter : IEndpointFilter
{
    private readonly IIdempotencyStore _store;

    public IdempotencyFilter(IIdempotencyStore store) => _store = store;

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var key = ctx.HttpContext.Request.Headers["Idempotency-Key"].ToString();
        if (string.IsNullOrWhiteSpace(key))
            return TypedResults.BadRequest("Header Idempotency-Key requis.");

        if (await _store.HasSeenAsync(key))
            return TypedResults.Conflict();

        var result = await next(ctx);
        await _store.MarkAsync(key);
        return result;
    }
}

// Utilisation
group.MapPost("/", Handler)
    .AddEndpointFilter<IdempotencyFilter>();
```

L'endpoint filter est plus petit, mais il fait aussi moins de choses. Il n'y a pas de séparation entre les étapes d'autorisation, de ressource, d'action et d'exception. Pour une grosse application avec des dizaines de règles transverses à des étapes différentes, le pipeline de controllers plus riche peut être un avantage. Pour la plupart des applications, une poignée d'endpoint filters est plus simple à comprendre.

> ✅ **Bonne pratique** : En Minimal APIs, attache les filtres au niveau du **groupe de route**, pas de l'endpoint individuel. Un seul `.AddEndpointFilter<IdempotencyFilter>()` sur le groupe `/orders` l'applique à chaque endpoint de commande, ce qui est à la fois moins bruyant et moins sujet aux oublis que de l'ajouter à chaque `MapPost`.

## Zoom : model binding et validation

Les controllers ont une longue histoire avec le model binding. `[FromBody]`, `[FromQuery]`, `[FromRoute]`, `[FromForm]`, `[FromHeader]`, plus le binding implicite basé sur le type et la source. Combiné avec `[ApiController]`, on a les 400 automatiques sur les modèles invalides et le formatage `ProblemDetails` automatique.

Les Minimal APIs bindent par convention : un paramètre de type complexe est lu depuis le body (sauf s'il est décoré de `[FromServices]`), les primitifs sont lus depuis la route ou la query, `IFormFile` vient du form, les services sont résolus depuis l'injection de dépendances automatiquement. On peut toujours utiliser les attributs `[From...]` quand la convention est ambiguë.

La validation est l'endroit où les deux styles laissent un trou. Aucun n'intègre `FluentValidation` par défaut. La réponse canonique dans les deux mondes aujourd'hui est de pousser la validation dans un **pipeline behavior MediatR**, ou dans un **endpoint filter** pour les Minimal APIs. Comme ça le validator vit avec la commande, et la définition de l'endpoint reste propre.

> ⚠️ **Ça marche, mais...** : Les data annotations (`[Required]`, `[StringLength]`) fonctionnent techniquement en Minimal APIs via un `ValidationFilter` ou un middleware équivalent, mais elles laissent fuir les règles de validation dans les records de requête et ne se composent pas bien avec les règles conditionnelles. Dès qu'on dépasse la validation jouet, utilise FluentValidation dans un filtre.

## Zoom : tests, OpenAPI et AOT

Pour les **tests unitaires**, les controllers ont un léger avantage : une méthode d'action est juste une méthode sur une classe. On instancie le controller, on appelle la méthode, on assert sur l'`ActionResult`. Les handlers de Minimal APIs sont en général des lambdas à l'intérieur de `MapPost`, ce qui les rend plus difficiles à tester directement. La solution, c'est d'extraire le handler dans une méthode statique nommée ou dans un handler MediatR, ce qui est une bonne pratique dans les deux cas.

Pour les **tests d'intégration**, les deux styles fonctionnent de façon identique avec `WebApplicationFactory`. Ce qu'on teste, c'est la surface HTTP, et elle ne se soucie pas de la façon dont les endpoints ont été déclarés.

Pour **OpenAPI**, .NET 9 a introduit le nouveau package `Microsoft.AspNetCore.OpenApi` qui remplace Swashbuckle par défaut. Il fonctionne aussi bien avec les deux modèles. Les Minimal APIs récupèrent gratuitement une métadonnée un peu plus riche quand on utilise `TypedResults`, parce que le type de résultat fait partie de la signature.

Pour la **compilation AOT (Ahead of Time)**, les Minimal APIs sont le chemin recommandé. L'équipe ASP.NET Core a beaucoup investi pour rendre `RequestDelegateFactory` généré par source et trim-safe. Les controllers s'appuient sur plus de réflexion au démarrage et ont plus de mal à devenir AOT-friendly.

> 💡 **Info** : Pour le détail de pourquoi l'AOT compte et ce qu'il apporte en .NET, voir [Compilation AOT en .NET](/fr/posts/performance-aot-compilation/).

## La matrice de décision honnête

Aucun style n'est objectivement meilleur. Voilà comment on choisit en pratique :

**Choisir Minimal APIs quand :**

- Le projet est un petit ou moyen service, un microservice, une charge de type fonction.
- On veut la compilation AOT, des démarrages à froid rapides, une image conteneur légère.
- On utilise un code organisé par fonctionnalité (chaque feature mappe ses propres endpoints) et on veut que la déclaration de l'endpoint vive à côté du handler.
- L'équipe préfère le câblage explicite au câblage par convention.

**Choisir Controllers quand :**

- L'application a des dizaines d'endpoints qui partagent des pipelines de filtres riches et des attributs de métadonnée.
- L'équipe a déjà des conventions MVC profondes (model binding, mapping vers des view-models, ordonnancement complexe des filtres).
- On a besoin de toute la taxonomie des action filters (autorisation, ressource, action, résultat, exception) avec le court-circuitage entre les étapes.
- On dépend d'un outil ou d'une bibliothèque qui attend toujours `ControllerBase` (certains vieux analyzers, les vieux scaffolders, quelques extensions tierces).

**Choisir "les deux dans le même projet" quand :**

- Une zone neuve livre en Minimal APIs, une zone legacy garde ses controllers. Ils cohabitent sans souci. `AddControllers()` et `MapControllers()` n'interfèrent pas avec `MapPost(...)`. La seule règle, c'est d'être cohérent à l'intérieur d'une fonctionnalité, pas à l'échelle de la solution entière.

> ❌ **Ne jamais faire** : Ne copie-colle pas des controllers entiers dans des lambdas Minimal API, ou l'inverse, comme "migration". Un changement de style n'est pas une fonctionnalité, il ajoute du risque sans aucune valeur visible pour l'utilisateur. Migre un module quand tu le touches déjà pour une autre raison, et laisse le reste tranquille. La cohérence à l'intérieur d'un module compte plus que la cohérence sur toute la base de code.

## Wrap-up

Tu sais maintenant comment Controllers et Minimal APIs se comparent vraiment dans l'ASP.NET Core moderne : les controllers offrent un pipeline de filtres riche, des conventions profondes et une décennie d'outillage ; les Minimal APIs offrent moins de cérémonie, un support AOT de premier plan, des typed results, et des endpoint filters plus simples à composer. Tu peux écrire le même endpoint dans les deux styles et livrer la même surface HTTP, et tu peux mixer les deux dans un même projet quand ça a du sens. Choisis celui qui colle à la forme de ta base de code, pas celui qui est le plus récent ou le plus ancien.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Références

- [Vue d'ensemble des Minimal APIs, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/minimal-apis/overview)
- [Choisir entre controllers et Minimal APIs, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/apis)
- [Filtres dans les Minimal APIs, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/minimal-apis/min-api-filters)
- [Gestion des erreurs en Minimal APIs, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/minimal-apis/handle-errors)
- [Support OpenAPI dans ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/openapi/overview)
- [Déploiement Native AOT, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/native-aot)
