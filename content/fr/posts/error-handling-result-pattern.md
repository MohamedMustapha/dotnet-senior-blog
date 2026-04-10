---
title: "Le Result Pattern"
date: 2026-04-10
draft: false
tags: ["error-handling", "dotnet", "result-pattern"]
categories: ["Error Handling"]
series: ["Error Handling"]
description: "Les exceptions pour les échecs attendus, c'est du contrôle de flux déguisé. Le Result pattern rend le succès et l'échec des valeurs de retour explicites que le compilateur peut vérifier, sans le coût de performance et le déroulage de pile du throw/catch."
---

Hello tous le monde, aujourd'hui on va explorer **le Result pattern**, l'alternative au throw/catch pour les erreurs métier attendues.

Les exceptions sont le bon outil pour les pannes inattendues : une connexion base qui tombe, une condition out-of-memory, une référence null. Elles sont le mauvais outil pour les échecs attendus : une erreur de validation, une règle métier violée, une ressource qui n'existe pas. La distinction compte parce que les exceptions déroulent la pile, allouent une stack trace, et cassent le flux de contrôle normal. Quand on lance un `OrderNotFoundException` et qu'on le catche trois frames plus haut pour retourner un 404, on utilise le mécanisme de contrôle de flux le plus coûteux de .NET pour faire quelque chose qu'une valeur de retour pourrait faire gratuitement.

Le Result pattern rend le résultat d'une opération explicite : la méthode retourne soit une valeur de succès, soit une valeur d'échec, et l'appelant doit gérer les deux. Ce n'est pas une idée neuve : Rust a `Result<T, E>`, Haskell a `Either`, Go retourne `(value, error)`. En .NET, le pattern prend la forme d'un struct générique qui porte soit le payload de succès, soit une erreur typée, et l'appelant utilise le pattern matching pour brancher sur le résultat.

Cet article montre pourquoi le Result pattern existe dans l'écosystème .NET, comment l'implémenter sans bibliothèque, et les compromis entre results et exceptions.

## Le contexte : pourquoi le Result pattern existe

La communauté .NET débat des exceptions-comme-contrôle-de-flux depuis plus d'une décennie. Les Framework Design Guidelines disent "ne pas utiliser les exceptions pour le flux de contrôle normal", et ensuite chaque tutoriel ASP.NET lance `NotFoundException` dans un handler et la catche dans un middleware pour retourner 404. La contradiction vient du fait que .NET n'avait pas de type result standard, donc les exceptions étaient le seul moyen de communiquer les échecs entre les frontières de méthodes sans paramètres `out` ou tuples.

Le Result pattern comble ce vide. Il est apparu dans l'écosystème .NET vers 2017-2018 via des bibliothèques comme `FluentResults` et `ErrorOr`, et a gagné en traction dans les architectures CQRS où les handlers de commandes et requêtes devaient retourner "ça a marché, voici la donnée" ou "ça a échoué, voici pourquoi" sans lancer d'exception. Les pipelines MediatR, en particulier, ont popularisé le pattern parce que les behaviors (étapes du pipeline) pouvaient inspecter et court-circuiter les résultats sans try-catch.

L'argument performance est réel mais secondaire. L'argument principal, c'est la clarté : quand une méthode retourne `Result<Order>`, l'appelant sait à la compilation que l'opération peut échouer, et le compilateur le force à gérer le chemin d'échec. Quand une méthode retourne `Order` et pourrait lancer, l'appelant doit lire la documentation (si elle existe) pour savoir quelles exceptions catcher.

## Vue d'ensemble : la forme d'un Result

{{< mermaid >}}
flowchart TD
    A[Le handler retourne Result&lt;T&gt;] --> B{Succès ?}
    B -->|Oui| C[Value : T]
    B -->|Non| D[Error : objet erreur typé]
    C --> E[Le controller retourne 200 + body]
    D --> F{Type d'erreur}
    F -->|NotFound| G[Le controller retourne 404]
    F -->|Validation| H[Le controller retourne 422]
    F -->|Conflict| I[Le controller retourne 409]
{{< /mermaid >}}

Le handler retourne un résultat. Le controller fait un pattern match dessus. Pas d'exceptions, pas de middleware, pas de handler global nécessaire pour les échecs attendus.

## Zoom : un type Result minimal

Pas besoin de bibliothèque. Un type Result qui couvre 90% des cas tient en moins de 50 lignes :

```csharp
public readonly struct Result<TValue>
{
    private readonly TValue? _value;
    private readonly Error? _error;

    private Result(TValue value) { _value = value; IsSuccess = true; }
    private Result(Error error) { _error = error; IsSuccess = false; }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;

    public TValue Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException("Cannot access Value on a failed result.");

    public Error Error => IsFailure
        ? _error!
        : throw new InvalidOperationException("Cannot access Error on a successful result.");

    public static implicit operator Result<TValue>(TValue value) => new(value);
    public static implicit operator Result<TValue>(Error error) => new(error);
}

public sealed record Error(string Code, string Message)
{
    public static readonly Error None = new(string.Empty, string.Empty);
    public static Error NotFound(string message) => new("NotFound", message);
    public static Error Validation(string message) => new("Validation", message);
    public static Error Conflict(string message) => new("Conflict", message);
}
```

Les conversions implicites sont ce qui rend l'ergonomie. Un handler peut faire `return order;` pour le succès ou `return Error.NotFound("Order not found");` pour l'échec, sans construire le `Result` explicitement.

> 💡 **Info** — Utiliser un `struct` plutôt qu'une `class` évite une allocation sur le heap par résultat. Sur un hot path qui retourne des milliers de résultats par seconde, ça compte. Sur une requête web typique qui retourne un seul résultat, non, mais il n'y a aucune raison de payer une allocation heap quand le struct fonctionne.

## Zoom : utiliser le Result dans un handler

```csharp
public sealed class GetOrderHandler : IRequestHandler<GetOrderQuery, Result<OrderDto>>
{
    private readonly ShopDbContext _db;

    public GetOrderHandler(ShopDbContext db) => _db = db;

    public async Task<Result<OrderDto>> Handle(GetOrderQuery query, CancellationToken ct)
    {
        var order = await _db.Orders
            .AsNoTracking()
            .Where(o => o.Id == query.OrderId)
            .Select(o => new OrderDto(o.Id, o.Reference, o.Status.ToString(), o.TotalAmount))
            .FirstOrDefaultAsync(ct);

        if (order is null)
            return Error.NotFound($"Order {query.OrderId} was not found.");

        return order;
    }
}
```

Le handler ne lance jamais pour un order manquant. Il retourne un résultat d'échec. Le code appelant, que ce soit un controller, un endpoint Minimal API, ou un autre handler, décide quoi en faire.

## Zoom : mapper le Result vers HTTP dans un controller

```csharp
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly ISender _sender;

    public OrdersController(ISender sender) => _sender = sender;

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> Get(Guid id, CancellationToken ct)
    {
        var result = await _sender.Send(new GetOrderQuery(id), ct);

        if (result.IsFailure)
        {
            return result.Error.Code switch
            {
                "NotFound" => NotFound(ToProblem(result.Error, 404)),
                "Validation" => UnprocessableEntity(ToProblem(result.Error, 422)),
                "Conflict" => Conflict(ToProblem(result.Error, 409)),
                _ => StatusCode(500, ToProblem(result.Error, 500))
            };
        }

        return Ok(result.Value);
    }

    private static ProblemDetails ToProblem(Error error, int status) => new()
    {
        Status = status,
        Title = error.Code,
        Detail = error.Message
    };
}
```

Le mapping du code d'erreur vers le statut HTTP est explicite et local à la couche API. Le domaine ne sait pas ce qu'est un 404. Le controller ne sait pas quelle est la règle métier. Chaque couche fait son travail.

> ✅ **Bonne pratique** — Si le mapping `Error.Code` vers statut HTTP se répète dans beaucoup de controllers, on l'extrait dans une méthode d'extension ou une méthode de base controller. Mais on ne le met pas dans le handler, parce que le handler peut être appelé depuis un job d'arrière-plan où les codes de statut HTTP n'ont aucun sens.

## Zoom : mapper le Result en Minimal API

```csharp
app.MapGet("api/orders/{id:guid}", async (Guid id, ISender sender, CancellationToken ct) =>
{
    var result = await sender.Send(new GetOrderQuery(id), ct);

    return result.IsSuccess
        ? Results.Ok(result.Value)
        : result.Error.Code switch
        {
            "NotFound" => Results.NotFound(result.Error),
            "Validation" => Results.UnprocessableEntity(result.Error),
            "Conflict" => Results.Conflict(result.Error),
            _ => Results.StatusCode(500)
        };
});
```

Même pattern, même clarté. L'endpoint Minimal API est un adaptateur fin entre HTTP et la couche application.

## Zoom : erreurs de validation sous forme de liste

Un seul `Error` fonctionne pour "order introuvable". Les échecs de validation sont typiquement une liste d'erreurs par champ. On étend le record `Error` pour les porter :

```csharp
public sealed record ValidationError : Error
{
    public ValidationError(IReadOnlyList<FieldError> fields)
        : base("Validation", "One or more validation errors occurred.")
    {
        Fields = fields;
    }

    public IReadOnlyList<FieldError> Fields { get; }
}

public sealed record FieldError(string Field, string Message);
```

Le handler retourne un `ValidationError` avec la liste de champs, et le controller le mappe en 422 avec un body qui liste chaque champ :

```csharp
public async Task<Result<OrderDto>> Handle(CreateOrderCommand command, CancellationToken ct)
{
    var errors = new List<FieldError>();

    if (string.IsNullOrWhiteSpace(command.Reference))
        errors.Add(new FieldError("Reference", "La référence est requise."));

    if (command.Lines.Count == 0)
        errors.Add(new FieldError("Lines", "Au moins une ligne est requise."));

    if (errors.Count > 0)
        return new ValidationError(errors);

    // ... créer la commande
}
```

> ⚠️ **Ça marche, mais...** — Pour la validation CRUD simple, FluentValidation avec un behavior de pipeline MediatR attrape la validation avant que le handler ne tourne, ce qui garde le handler propre. La validation par Result montrée ci-dessus convient mieux quand la validation dépend de données que le handler charge depuis la base.

## Zoom : quand continuer à lancer des exceptions

Le Result pattern ne remplace pas les exceptions partout. Il les remplace pour les échecs attendus, au niveau du domaine. Trois cas où les exceptions restent le bon outil :

1. **Pannes d'infrastructure.** Une base morte, un réseau cassé, un disque plein. Ce sont des échecs inattendus et irrécupérables par l'appelant immédiat. Lancer leur permet de se propager jusqu'au [gestionnaire global d'erreurs](/fr/posts/error-handling-global-error-handling/).
2. **Erreurs de programmation.** `ArgumentNullException`, `InvalidOperationException` pour des préconditions violées. Elles signalent un bug, pas une condition runtime.
3. **Frontières de bibliothèques tierces.** Si la bibliothèque lance, catcher et envelopper dans un Result à la frontière de l'adaptateur est correct. Essayer d'empêcher la bibliothèque de lancer ne l'est pas.

La règle simple : si l'appelant peut raisonnablement récupérer, retourner un Result. Si l'appelant ne peut pas, lancer une exception.

> ❌ **Ne jamais faire** — Ne pas envelopper chaque exception dans un Result à chaque couche. `catch (Exception ex) => Result.Failure(ex.Message)` cache la stack trace, avale le type d'exception, et rend le débogage plus difficile que le problème qu'on essayait de résoudre.

## Wrap-up

Tu sais maintenant que le Result pattern rend le succès et l'échec des valeurs de retour explicites. Le handler retourne `Result<T>`, l'appelant fait un pattern match, et le compilateur impose que le chemin d'échec soit géré. Ça ne remplace pas les exceptions pour les crashes d'infrastructure ou les erreurs de programmation, mais ça élimine le surcoût du throw/catch et l'ambiguïté pour chaque échec domaine attendu. Combiné avec le [gestionnaire global d'erreurs](/fr/posts/error-handling-global-error-handling/) pour les échecs inattendus, ça donne une stratégie d'erreur complète avec une séparation propre.

Prêt à introduire le Result pattern dans ta couche application, ou à montrer ce compromis à ton équipe ?

À la prochaine, a++ 👋

## Pour aller plus loin

- [Exceptions Personnalisées en .NET](/fr/posts/error-handling-custom-exceptions/)
- [Gestion Globale des Erreurs en ASP.NET Core](/fr/posts/error-handling-global-error-handling/)
- [Couche Application : CQS et CQRS](/fr/posts/layer-focused-cqs-cqrs/)

## Références

- [Microsoft : Guidelines de conception des exceptions](https://learn.microsoft.com/fr-fr/dotnet/standard/design-guidelines/exceptions)
- [FluentResults sur GitHub](https://github.com/altmann/FluentResults)
- [ErrorOr sur GitHub](https://github.com/amantinband/error-or)
- [RFC 9457 : Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
