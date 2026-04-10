---
title: "Exceptions Personnalisées en .NET"
date: 2026-04-10
draft: false
tags: ["error-handling", "dotnet", "exceptions"]
categories: ["Error Handling"]
series: ["Error Handling"]
description: "Lancer System.Exception avec un message string n'est pas de la gestion d'erreur, c'est du signalement d'erreur. Les exceptions personnalisées donnent aux appelants un type à catcher, un payload à logger, et un contrat sur lequel s'appuyer."
---

Hello tous le monde, aujourd'hui on va explorer **les exceptions personnalisées en .NET** et quand elles méritent vraiment leur place dans le code.

Tout code .NET a une couche de gestion d'erreurs, même si cette couche c'est `throw new Exception("quelque chose a foiré")`. Le problème avec ce défaut, ce n'est pas que ça plante l'application, c'est que chaque appelant doit catcher `Exception`, parser le message string, et deviner ce qui s'est passé. Les exceptions personnalisées remplacent la devinette par le système de types : l'appelant catche `OrderAlreadyShippedException`, pas `Exception`, et le type lui dit exactement ce qui a foiré, quelles données sont disponibles, et ce qu'il peut faire.

Cet article déroule quand une exception personnalisée gagne sa place, comment en écrire une qui porte du contexte utile, et les patterns qui empêchent une hiérarchie d'exceptions grandissante de devenir un fardeau de maintenance.

## Le contexte : pourquoi les exceptions personnalisées existent

Le runtime .NET embarque déjà des centaines de types d'exceptions : `ArgumentNullException`, `InvalidOperationException`, `UnauthorizedAccessException`, `TimeoutException`, et bien d'autres. Elles couvrent les pannes d'infrastructure et les erreurs de programmation. Ce qu'elles ne couvrent pas, c'est le domaine : une commande déjà expédiée, un paiement qui échoue à la validation, un utilisateur qui dépasse son rate limit, un document qui échoue un contrôle de conformité. Ces scénarios sont propres à l'application, et le framework ne peut pas les anticiper.

Avant les exceptions personnalisées, les équipes géraient les erreurs de domaine de trois façons : retourner des valeurs magiques (`-1`, `null`, `false`), lancer `InvalidOperationException` avec un message string, ou retourner un tuple `(bool success, string error)`. Les trois partagent le même défaut : l'appelant ne peut pas distinguer les erreurs par type. Il doit lire un string, vérifier un booléen, ou inspecter une valeur, ce qui veut dire que chaque chemin de gestion d'erreur est un branchement conditionnel sur un signal faiblement typé. Les exceptions personnalisées résolvent ça en faisant de l'erreur un type, et les types sont la seule chose que le compilateur sait vérifier.

La pratique remonte à .NET 1.0 et aux Framework Design Guidelines de Krzysztof Cwalina et Brad Abrams, publiées en 2005, qui ont posé la convention : dériver de `Exception` (pas `ApplicationException`), inclure les trois constructeurs standard, et rendre l'exception sérialisable. L'exigence de sérialisation s'est assouplie depuis que .NET Core a abandonné la sérialisation binaire par défaut, mais la convention des trois constructeurs tient toujours.

## Vue d'ensemble : quand créer une exception personnalisée

{{< mermaid >}}
flowchart TD
    A[Scénario d'erreur] --> B{Une exception framework peut l'exprimer ?}
    B -->|Oui| C[Utiliser ArgumentNullException,<br/>InvalidOperationException, etc.]
    B -->|Non| D{Un appelant doit catcher<br/>ça spécifiquement ?}
    D -->|Non| E[Lancer InvalidOperationException<br/>avec un message clair]
    D -->|Oui| F[Créer une exception personnalisée]
    F --> G[Porter le contexte domaine en propriétés]
{{< /mermaid >}}

La porte de décision, c'est : est-ce qu'un appelant a besoin de catcher cette erreur par type ? Si le seul handler est un middleware global qui logge et retourne 500, l'exception personnalisée ajoute de la cérémonie sans valeur. Si un command handler, une politique de retry, ou un contrôleur API doit réagir différemment à cette défaillance précise, l'exception personnalisée est le moyen de savoir ce qui s'est passé.

## Zoom : l'anatomie d'une exception personnalisée

```csharp
public sealed class OrderAlreadyShippedException : Exception
{
    public OrderAlreadyShippedException(Guid orderId)
        : base($"Order {orderId} has already been shipped and cannot be modified.")
    {
        OrderId = orderId;
    }

    public OrderAlreadyShippedException(Guid orderId, Exception innerException)
        : base($"Order {orderId} has already been shipped and cannot be modified.", innerException)
    {
        OrderId = orderId;
    }

    public Guid OrderId { get; }
}
```

Trois choses à noter :

1. **Sealed.** Sauf si on attend des types dérivés (en général non), sceller la classe empêche une hiérarchie incontrôlée.
2. **Payload domaine.** `OrderId` est une propriété typée, pas enfouie dans le message string. L'appelant peut le logger, le retourner dans une réponse ProblemDetails, ou l'utiliser pour rechercher la commande sans parser du texte.
3. **Pas de constructeur sans paramètre.** Si une `OrderAlreadyShippedException` doit toujours porter un `OrderId`, le compilateur l'impose. Impossible d'en créer une sans spécifier quelle commande.

> 💡 **Info** — La convention des trois constructeurs (`()`, `(string message)`, `(string message, Exception inner)`) vient des Framework Design Guidelines. En pratique, on ne garde que les constructeurs qui ont du sens pour le domaine. Un constructeur sans paramètre sur `OrderAlreadyShippedException` n'a aucun sens, donc on ne l'ajoute pas juste pour suivre la convention.

> ✅ **Bonne pratique** — Le contexte domaine va dans des propriétés typées, pas dans le message string. Le message est pour les humains qui lisent les logs. Les propriétés sont pour le code qui prend des décisions.

## Zoom : où lancer

Les exceptions personnalisées se lancent depuis la couche domaine ou application, là où la règle métier est appliquée :

```csharp
public sealed class ShipOrderHandler : IRequestHandler<ShipOrderCommand>
{
    private readonly ShopDbContext _db;

    public ShipOrderHandler(ShopDbContext db) => _db = db;

    public async Task Handle(ShipOrderCommand command, CancellationToken ct)
    {
        var order = await _db.Orders.FindAsync([command.OrderId], ct)
            ?? throw new OrderNotFoundException(command.OrderId);

        if (order.Status == OrderStatus.Shipped)
            throw new OrderAlreadyShippedException(command.OrderId);

        order.Ship();
        await _db.SaveChangesAsync(ct);
    }
}
```

Le handler ne catche pas ces exceptions. Il les lance. Le catch se fait en amont : dans la couche API pour les réponses HTTP, dans un middleware pour le logging, ou dans une politique de retry pour les erreurs transitoires. Cette séparation garde le domaine propre de préoccupations HTTP et la couche API propre de logique métier.

> ⚠️ **Ça marche, mais...** — Lancer et catcher une exception dans la même méthode est un code smell. Si la méthode connaît la condition d'erreur, elle peut retourner un résultat au lieu de lancer. Les exceptions sont faites pour traverser les frontières de couches, pas pour du contrôle de flux à l'intérieur d'une seule méthode.

## Zoom : une classe de base pour les exceptions domaine

Quand plusieurs exceptions personnalisées partagent la même forme (toutes portent un ID d'entité, toutes se mappent au même status HTTP), une classe de base fine coupe la répétition :

```csharp
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    protected DomainException(string message, Exception innerException) : base(message, innerException) { }
}

public sealed class OrderNotFoundException : DomainException
{
    public OrderNotFoundException(Guid orderId)
        : base($"Order {orderId} was not found.")
    {
        OrderId = orderId;
    }

    public Guid OrderId { get; }
}

public sealed class OrderAlreadyShippedException : DomainException
{
    public OrderAlreadyShippedException(Guid orderId)
        : base($"Order {orderId} has already been shipped and cannot be modified.")
    {
        OrderId = orderId;
    }

    public Guid OrderId { get; }
}
```

Le gestionnaire d'erreurs global peut maintenant catcher `DomainException` comme une catégorie et la mapper vers une réponse 4xx, tandis que les exceptions d'infrastructure (`DbException`, `HttpRequestException`) se mappent vers 5xx. La hiérarchie reste plate : une base abstraite, N feuilles scellées. Deux niveaux, c'est le maximum qui reste gérable.

> ❌ **Ne jamais faire** — Ne pas construire une hiérarchie profonde d'exceptions (`DomainException → EntityException → OrderException → OrderAlreadyShippedException`). Chaque niveau ajoute une clause catch dont quelqu'un pourrait dépendre et que personne ne maintiendra. Plat et scellé, c'est l'objectif.

## Zoom : exceptions pour les frontières d'infrastructure

Les exceptions personnalisées ne sont pas toutes des exceptions domaine. Quand on encapsule un service externe, une exception personnalisée encapsule la défaillance :

```csharp
public sealed class PaymentGatewayException : Exception
{
    public PaymentGatewayException(string gatewayCode, string gatewayMessage, Exception inner)
        : base($"Payment gateway returned error {gatewayCode}: {gatewayMessage}", inner)
    {
        GatewayCode = gatewayCode;
        GatewayMessage = gatewayMessage;
    }

    public string GatewayCode { get; }
    public string GatewayMessage { get; }
}
```

L'exception `inner` est le `HttpRequestException` ou `JsonException` brut de l'appel HTTP. Le wrapper donne à l'appelant un type à catcher et des propriétés à logger, sans fuiter le détail d'implémentation de quel client HTTP ou quel sérialiseur a été utilisé.

> ✅ **Bonne pratique** — Toujours passer l'exception originale en `innerException`. La stack trace de la cause racine, c'est ce dont on a besoin dans l'incident. L'avaler rend le débogage plus difficile pour tout le monde.

## Zoom : ce qui ne mérite pas une exception personnalisée

Chaque erreur ne demande pas un type dédié. Trois cas où une exception framework suffit :

1. **Erreurs de programmation.** `ArgumentNullException`, `ArgumentOutOfRangeException`, `InvalidOperationException` communiquent déjà "l'appelant a fait une erreur". Ajouter `OrderIdCannotBeEmptyException` ne fait que dupliquer ce que `ArgumentException("orderId")` dit déjà.
2. **Erreurs que personne ne catche spécifiquement.** Si le seul handler est le middleware global, `InvalidOperationException("Order has no lines")` suffit. Le middleware logge le message et retourne 500 quel que soit le type.
3. **Erreurs de validation.** Une liste de défaillances de validation par champ n'est pas une exception, c'est un résultat. Lancer `ValidationException` avec une liste d'erreurs et la catcher dans un filtre pour retourner 400 fonctionne, mais mène au contrôle-de-flux-par-exception. Le [Result pattern](/fr/posts/error-handling-result-pattern/) est plus propre.

## Wrap-up

Tu sais maintenant que les exceptions personnalisées sont un outil de communication entre couches. On en crée une quand un appelant doit catcher une défaillance domaine spécifique, porter du contexte typé pour le logging et les réponses, et garder le domaine libre de préoccupations d'infrastructure. On les scelle, on garde la hiérarchie plate, et on attache toujours l'exception interne quand on encapsule une défaillance externe. Pour le reste, les exceptions framework suffisent.

Prêt à revoir les exceptions de ton projet et à remplacer les `throw new Exception("...")` par des types qui parlent, ou à partager ces guidelines avec ton équipe ?

À la prochaine, a++ 👋

## Pour aller plus loin

- [Couche Application : CQS et CQRS](/fr/posts/layer-focused-cqs-cqrs/)

## Références

- [Microsoft : Guidelines de conception des exceptions](https://learn.microsoft.com/fr-fr/dotnet/standard/design-guidelines/exceptions)
- [Microsoft : Créer et lancer des exceptions](https://learn.microsoft.com/fr-fr/dotnet/csharp/fundamentals/exceptions/creating-and-throwing-exceptions)
- [Microsoft : Bonnes pratiques pour les exceptions](https://learn.microsoft.com/fr-fr/dotnet/standard/exceptions/best-practices-for-exceptions)
