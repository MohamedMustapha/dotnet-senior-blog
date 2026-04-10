---
title: "Quand lever une exception, quand logger"
date: 2026-04-10
draft: false
tags: ["error-handling", "dotnet", "logging"]
categories: ["Error Handling"]
series: ["Error Handling"]
description: "Throw là où on détecte. Log là où on gère. Jamais les deux au même endroit. Un cadre de décision pour savoir quand lancer, quand retourner un Result, quand logger, et où chaque responsabilité se situe dans l'architecture."
---

Hello tous le monde, aujourd'hui on va démystifier **la question qui revient dans chaque code review : quand lancer une exception et quand logger**.

Le bug de gestion d'erreurs le plus fréquent dans les codebases .NET, ce n'est pas un catch manquant ou un mauvais code de statut. C'est un try-catch qui logge l'exception et la relance, ce qui produit trois entrées de log identiques pour la même erreur à mesure qu'elle remonte à travers trois couches. Le deuxième bug le plus fréquent, c'est un catch qui avale l'exception, logge un "error occurred" générique, et retourne null, ce qui cache la cause racine et fait croire à l'appelant que l'opération a réussi. Les deux bugs viennent de la même confusion : ne pas savoir quelle couche est responsable de quoi.

Cet article pose le cadre de décision : quelle couche détecte l'erreur, quelle couche la gère, quelle couche la logge, et que fait chaque couche avec. Il s'appuie sur les trois articles précédents de la série, les [exceptions personnalisées](/fr/posts/error-handling-custom-exceptions/), le [gestionnaire global d'erreurs](/fr/posts/error-handling-global-error-handling/), et le [Result pattern](/fr/posts/error-handling-result-pattern/), pour les relier en une stratégie cohérente.

## Le contexte : pourquoi la confusion existe

.NET rend le throw facile, le catch facile, et le log facile. Les trois opérations se ressemblent depuis chaque couche, donc sans convention, chaque développeur ajoute un try-catch là où il ne se sent pas tranquille. Le résultat est un code où les exceptions sont catchées quatre fois, loggées trois fois, et gérées zéro fois, parce que personne n'était sûr de qui devait s'en occuper.

La convention est simple : **throw là où on détecte, handle là où on peut agir, log là où on handle**. Si une couche détecte une erreur et ne peut rien faire d'utile, elle laisse l'exception se propager. Si une couche peut traduire l'erreur en réponse significative (un statut HTTP, un retry, une valeur de fallback), elle catche, gère et logge au même endroit.

## Vue d'ensemble : la carte des responsabilités

{{< mermaid >}}
graph TD
    A[Domaine / Entité] -->|Lance DomainException| B[Couche Application / Handler]
    B -->|Retourne Result ou laisse l'exception se propager| C[Couche API / Controller]
    C -->|Mappe Result vers HTTP ou laisse l'exception se propager| D[Gestionnaire global d'erreurs]
    D -->|Logge et retourne ProblemDetails| E[Client]

    style A fill:#f9f,stroke:#333
    style D fill:#ff9,stroke:#333
{{< /mermaid >}}

Deux couches ont des responsabilités explicites :
- **Domaine** (rose) : détecte les violations de règles métier et lance ou retourne un échec.
- **Gestionnaire global d'erreurs** (jaune) : catche tout ce que personne d'autre n'a géré, logge, et retourne une réponse sûre.

Tout ce qui est entre les deux est un relais : ça traduit l'erreur dans une forme différente (Result vers HTTP, exception domaine vers Result) ou ça laisse passer sans toucher.

## Règle 1 : throw là où on détecte

La méthode qui découvre la violation est la méthode qui lance. Elle ne logge pas. Elle ne catche pas. Elle lance et laisse l'appelant décider.

```csharp
public void Ship()
{
    if (Status == OrderStatus.Shipped)
        throw new OrderAlreadyShippedException(Id);

    if (Lines.Count == 0)
        throw new InvalidOperationException("Cannot ship an order with no lines.");

    Status = OrderStatus.Shipped;
    ShippedAt = DateTime.UtcNow;
}
```

L'entité n'a pas accès à `ILogger`. Elle ne sait pas si l'appelant est un controller API, un job d'arrière-plan, ou un test unitaire. Son travail est de protéger ses propres invariants. Lancer est la façon dont elle fait ça.

> ✅ **Bonne pratique** — Les méthodes du domaine lancent des exceptions domaine pour les violations de règles métier et des exceptions framework pour les erreurs de programmation (`ArgumentNullException`, `InvalidOperationException`). Elles ne loggent jamais.

## Règle 2 : les handlers traduisent, ils ne catch-and-rethrow pas

La couche application (le handler de commande/requête) appelle les méthodes de domaine et l'infrastructure. Si elle utilise le [Result pattern](/fr/posts/error-handling-result-pattern/), elle catche les exceptions domaine à la frontière et les traduit en échecs Result :

```csharp
public async Task<Result<Unit>> Handle(ShipOrderCommand command, CancellationToken ct)
{
    var order = await _db.Orders.FindAsync([command.OrderId], ct);
    if (order is null)
        return Error.NotFound($"Order {command.OrderId} was not found.");

    try
    {
        order.Ship();
    }
    catch (OrderAlreadyShippedException)
    {
        return Error.Conflict($"Order {command.OrderId} has already been shipped.");
    }

    await _db.SaveChangesAsync(ct);
    return Unit.Value;
}
```

C'est le seul catch dans le handler, et il traduit une exception domaine spécifique en Result. Il ne catche pas `Exception`. Il ne logge pas. Les exceptions d'infrastructure (`DbException`, `HttpRequestException`) se propagent jusqu'au gestionnaire global.

Si on n'utilise pas le Result pattern, le handler ne catche rien du tout. L'exception domaine se propage au gestionnaire global, qui la mappe en réponse 4xx.

> ❌ **Ne jamais faire** — Ne pas écrire `catch (Exception ex) { _logger.LogError(ex, "..."); throw; }` dans un handler. Ça produit une entrée de log au niveau du handler et une autre au niveau du gestionnaire global pour la même exception. Le gestionnaire global est le point unique de logging pour les exceptions non gérées.

## Règle 3 : les adaptateurs d'infrastructure enveloppent et relancent

Quand on appelle un service externe, l'adaptateur catche l'exception d'infrastructure et l'enveloppe dans une [exception personnalisée](/fr/posts/error-handling-custom-exceptions/) qui a du sens pour la couche application :

```csharp
public async Task<PaymentResult> ChargeAsync(Guid orderId, decimal amount, CancellationToken ct)
{
    try
    {
        var response = await _httpClient.PostAsJsonAsync("/charges", new { orderId, amount }, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<PaymentResult>(ct)
            ?? throw new PaymentGatewayException("EMPTY_RESPONSE", "Gateway returned empty body", null!);
    }
    catch (HttpRequestException ex)
    {
        throw new PaymentGatewayException("HTTP_ERROR", ex.Message, ex);
    }
    catch (JsonException ex)
    {
        throw new PaymentGatewayException("PARSE_ERROR", "Failed to parse gateway response", ex);
    }
}
```

L'adaptateur ne logge pas. Il enveloppe l'exception d'infrastructure dans un type qui a du sens pour le domaine et passe l'originale en `innerException`. Le gestionnaire global logge toute la chaîne.

> 💡 **Info** — Garder l'exception originale en `innerException` préserve la stack trace complète. Quand le gestionnaire global logge `PaymentGatewayException`, la chaîne `innerException` montre exactement quel appel HTTP a échoué, quelle était la réponse, et où dans le code ça s'est passé.

## Règle 4 : le gestionnaire global est le point unique de logging

Le [gestionnaire global d'erreurs](/fr/posts/error-handling-global-error-handling/) est le seul endroit où chaque exception non gérée est loggée. Il logge une fois, avec tout le contexte, et retourne une réponse structurée :

```csharp
public async ValueTask<bool> TryHandleAsync(HttpContext httpContext, Exception exception, CancellationToken ct)
{
    var level = exception is DomainException ? LogLevel.Warning : LogLevel.Error;

    _logger.Log(level, exception, "Exception on {Method} {Path}",
        httpContext.Request.Method, httpContext.Request.Path);

    // ... écrire la réponse ProblemDetails
    return true;
}
```

Les exceptions domaine loggent en `Warning` parce que ce sont des conditions attendues. Les exceptions d'infrastructure loggent en `Error` parce qu'elles indiquent un vrai problème. Le handler ne relance pas : c'est le bout de la ligne.

## Règle 5 : logger au point de décision, pas au point de détection

Le corollaire de "throw là où on détecte, log là où on gère" est que le logging se fait là où une décision est prise. Trois exemples :

**La politique de retry décide de retenter.** Le middleware de retry logge en `Warning` parce qu'il gère l'erreur en retentant, il ne la propage pas :

```csharp
retryPolicy.Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)),
        onRetry: (ex, delay, attempt, _) =>
            _logger.LogWarning(ex, "Retry {Attempt} after {Delay}s", attempt, delay.TotalSeconds));
```

**Le fallback décide de retourner une valeur par défaut.** Le fallback logge en `Warning` et retourne la valeur par défaut :

```csharp
public async Task<ExchangeRate> GetRateAsync(string currency, CancellationToken ct)
{
    try
    {
        return await _rateService.GetAsync(currency, ct);
    }
    catch (HttpRequestException ex)
    {
        _logger.LogWarning(ex, "Rate service unavailable, using cached rate for {Currency}", currency);
        return _cache.GetLastKnownRate(currency);
    }
}
```

**Le job d'arrière-plan décide de sauter l'élément.** Le job logge en `Warning` et continue avec l'élément suivant :

```csharp
foreach (var order in pendingOrders)
{
    try
    {
        await ProcessOrderAsync(order, ct);
    }
    catch (Exception ex)
    {
        _logger.LogWarning(ex, "Failed to process order {OrderId}, skipping", order.Id);
    }
}
```

Dans les trois cas, le catch-and-log est au point où la décision est prise. Pas avant, pas après.

> ⚠️ **Ça marche, mais...** — Catcher `Exception` dans une boucle de job d'arrière-plan est un choix pragmatique pour garder le job en marche. Mais on logge chaque skip, et on surveille le taux de skip. Si le taux de skip grimpe, le job cache une défaillance systémique derrière des retries individuels.

## La matrice de décision

| Couche | Détecte l'erreur | Lance | Retourne Result | Catche | Logge |
|---|---|---|---|---|---|
| Entité domaine | Oui | Oui | Non | Non | Non |
| Handler application | Parfois | Rarement | Oui (si Result) | Seulement pour traduire | Non |
| Adaptateur infrastructure | Oui | Oui (enveloppé) | Non | Seulement pour envelopper | Non |
| Politique retry/fallback | Non | Non | Non | Oui | Oui (Warning) |
| Controller API | Non | Non | Non | Non (mappe Result) | Non |
| Gestionnaire global | Non | Non | Non | Oui (tout) | Oui (Error ou Warning) |

Le tableau a un seul loggeur par ligne. Aucune ligne n'a à la fois "Lance : Oui" et "Logge : Oui". C'est la convention qui arrête les doublons dans les logs.

## Wrap-up

Tu sais maintenant que toute la stratégie d'erreur se résume à un principe : séparer la détection de la gestion. La couche qui trouve le problème lance ou retourne un échec. La couche qui peut agir sur le problème catche, décide quoi faire, et logge. Le gestionnaire global est le filet de sécurité pour tout ce que personne d'autre n'a géré. Quand chaque couche suit ça, les exceptions se propagent proprement, les entrées de log sont uniques, et le post-mortem a exactement une stack trace par incident, pas trois.

Prêt à auditer les try-catch de ton code et à supprimer ceux qui ne décident de rien, ou à poser cette convention avec ton équipe ?

À la prochaine, a++ 👋

## Pour aller plus loin

- [Exceptions Personnalisées en .NET](/fr/posts/error-handling-custom-exceptions/)
- [Gestion Globale des Erreurs en ASP.NET Core](/fr/posts/error-handling-global-error-handling/)
- [Le Result Pattern](/fr/posts/error-handling-result-pattern/)
- [Couche Application : CQS et CQRS](/fr/posts/layer-focused-cqs-cqrs/)

## Références

- [Microsoft : Bonnes pratiques pour les exceptions](https://learn.microsoft.com/fr-fr/dotnet/standard/exceptions/best-practices-for-exceptions)
- [Microsoft : Logging en .NET](https://learn.microsoft.com/fr-fr/dotnet/core/extensions/logging)
- [Serilog : Logging Structuré](https://serilog.net/)
