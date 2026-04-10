---
title: "Gestion Globale des Erreurs en ASP.NET Core"
date: 2026-04-10
draft: false
tags: ["error-handling", "dotnet", "middleware"]
categories: ["Error Handling"]
series: ["Error Handling"]
description: "Chaque exception qui échappe à un controller ou handler doit atterrir quelque part. ASP.NET Core fournit IExceptionHandler, ProblemDetails et le middleware de status code pages. Voici comment les câbler en un seul pipeline d'erreur prévisible."
---

Hello tous le monde, aujourd'hui on va comprendre **la gestion globale des erreurs en ASP.NET Core**, le filet de sécurité de toute API.

Les exceptions non gérées dans une application ASP.NET Core ne disparaissent pas. Si rien ne les catche, le framework retourne une réponse 500 vide sans body, ce qui est correct du point de vue sécurité et inutile du point de vue débogage. Les clients n'ont pas de code d'erreur à signaler, les logs ont une stack trace sans contexte structuré, et l'équipe ops ne peut pas dire si le 500 était un timeout transitoire de base ou un bug de logique permanent. La gestion globale des erreurs est la couche qui transforme chaque exception non gérée en une réponse structurée pour le client et une entrée de log structurée pour l'équipe, au même endroit, sans disperser des blocs try-catch dans chaque controller.

Cet article couvre les trois mécanismes qu'ASP.NET Core fournit pour ça : l'interface `IExceptionHandler` (le chemin moderne depuis .NET 8), le service `ProblemDetails`, et le middleware de status code pages. On n'a pas besoin des trois à la fois, mais on doit comprendre comment ils se composent.

## Le contexte : pourquoi un seul handler global

Avant ASP.NET Core 8, l'approche standard était un middleware de gestion d'exceptions : soit le `UseExceptionHandler()` intégré avec une lambda, soit un middleware personnalisé qui enveloppait `await _next(context)` dans un try-catch. Les deux marchaient, mais les deux imposaient d'écrire tout le pipeline soi-même : vérifier le type d'exception, choisir le code de statut, sérialiser le body de la réponse, logger l'erreur, et gérer les cas limites comme "la réponse a déjà commencé à être écrite".

`IExceptionHandler`, introduit dans .NET 8, fournit un hook structuré dans le gestionnaire d'exceptions intégré. On implémente l'interface, on l'enregistre en DI, et ASP.NET Core l'appelle quand une exception non gérée atteint le pipeline. Plusieurs handlers peuvent être enregistrés et sont appelés dans l'ordre jusqu'à ce qu'un marque l'exception comme gérée. Ça remplace l'approche par lambda par des classes testables et composables.

L'objectif est toujours le même : un seul endroit dans le code décide quoi retourner quand ça tourne mal.

## Vue d'ensemble : le pipeline d'erreur

{{< mermaid >}}
flowchart LR
    A[Requête] --> B[Pipeline de middleware]
    B --> C[Endpoint / Controller]
    C -->|Exception lancée| D[UseExceptionHandler]
    D --> E[IExceptionHandler 1 :<br/>DomainExceptionHandler]
    E -->|Géré| F[Réponse ProblemDetails 4xx]
    E -->|Non géré| G[IExceptionHandler 2 :<br/>FallbackExceptionHandler]
    G --> H[Réponse ProblemDetails 500]
{{< /mermaid >}}

L'exception remonte depuis l'endpoint, atteint `UseExceptionHandler()`, qui invoque chaque `IExceptionHandler` enregistré dans l'ordre. Le premier qui retourne `true` coupe la chaîne. Si aucun ne gère, le handler intégré retourne la réponse par défaut.

## Zoom : IExceptionHandler pour les exceptions domaine

```csharp
public sealed class DomainExceptionHandler : IExceptionHandler
{
    private readonly ILogger<DomainExceptionHandler> _logger;

    public DomainExceptionHandler(ILogger<DomainExceptionHandler> logger) => _logger = logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is not DomainException domainException)
            return false;

        _logger.LogWarning(exception, "Domain error: {Message}", domainException.Message);

        var statusCode = domainException switch
        {
            OrderNotFoundException => StatusCodes.Status404NotFound,
            OrderAlreadyShippedException => StatusCodes.Status409Conflict,
            _ => StatusCodes.Status422UnprocessableEntity
        };

        httpContext.Response.StatusCode = statusCode;
        await httpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = statusCode,
            Title = domainException.GetType().Name,
            Detail = domainException.Message,
            Instance = httpContext.Request.Path
        }, cancellationToken);

        return true;
    }
}
```

Ce handler ne catche que les [exceptions personnalisées](/fr/posts/error-handling-custom-exceptions/) qui dérivent de `DomainException`. Il mappe chaque type vers un code de statut HTTP, écrit une réponse `ProblemDetails` (RFC 9457), et retourne `true` pour signaler "géré". Pour tout autre type d'exception, il retourne `false`, et le handler suivant dans la chaîne prend le relais.

> 💡 **Info** — `ProblemDetails` est le standard RFC 9457 pour les réponses d'erreur lisibles par les machines dans les API HTTP. ASP.NET Core a un support intégré via `Microsoft.AspNetCore.Http.ProblemDetails` et `Results.Problem()`.

> ✅ **Bonne pratique** — Logger les exceptions domaine en `Warning`, pas en `Error`. Ce sont des conditions attendues (not found, conflit, échec de validation), pas des crashes. On réserve `Error` aux exceptions qui indiquent un vrai bug ou une défaillance d'infrastructure.

## Zoom : un handler de fallback pour tout le reste

```csharp
public sealed class FallbackExceptionHandler : IExceptionHandler
{
    private readonly ILogger<FallbackExceptionHandler> _logger;
    private readonly IHostEnvironment _env;

    public FallbackExceptionHandler(ILogger<FallbackExceptionHandler> logger, IHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "Unhandled exception on {Method} {Path}",
            httpContext.Request.Method, httpContext.Request.Path);

        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;

        var problem = new ProblemDetails
        {
            Status = 500,
            Title = "Internal Server Error",
            Instance = httpContext.Request.Path
        };

        if (_env.IsDevelopment())
            problem.Detail = exception.ToString();

        await httpContext.Response.WriteAsJsonAsync(problem, cancellationToken);
        return true;
    }
}
```

Ce handler catche tout ce que le handler domaine n'a pas pris. Il logge en `Error`, retourne un 500 générique, et n'inclut le détail de l'exception qu'en développement. En production, le body de la réponse n'expose jamais la stack trace.

> ❌ **Ne jamais faire** — Ne pas inclure de stack traces, de chaînes de connexion ou de noms de types internes dans les réponses d'erreur en production. Un body `ProblemDetails` avec `Detail = exception.ToString()`, c'est une fuite de sécurité. Les attaquants utilisent les stack traces pour cartographier l'architecture interne.

## Zoom : câbler le tout

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
builder.Services.AddExceptionHandler<FallbackExceptionHandler>();

var app = builder.Build();

app.UseExceptionHandler();
app.UseStatusCodePages();

app.MapControllers();
app.Run();
```

L'ordre d'enregistrement compte : `DomainExceptionHandler` est appelé en premier. S'il retourne `false`, `FallbackExceptionHandler` tourne. `AddProblemDetails()` enregistre le `IProblemDetailsService` que d'autres parties du framework utilisent pour générer des réponses `ProblemDetails` cohérentes (y compris les erreurs de validation de modèle des controllers).

`UseStatusCodePages()` gère le cas où un middleware pose un code 4xx/5xx sans écrire de body. Sans lui, un `return NotFound()` depuis un controller retourne `404` sans body JSON. Avec lui, le middleware génère un body `ProblemDetails` pour le code de statut.

> 💡 **Info** — `AddProblemDetails()` est disponible depuis .NET 7. Il câble le `IProblemDetailsService` pour que les erreurs générées par le framework (validation de modèle, échecs d'autorisation, routes introuvables) retournent aussi des réponses RFC 9457 au lieu de texte brut.

## Zoom : et les exception filters ?

Les exception filters MVC (`IExceptionFilter`, `IAsyncExceptionFilter`) précèdent l'approche middleware et fonctionnent toujours dans ASP.NET Core. Ils tournent à l'intérieur du pipeline MVC, après le model binding et l'exécution de l'action. Ils ne couvrent pas les exceptions lancées dans le middleware, dans les handlers Minimal API, ou dans le pipeline de requête avant que MVC ne démarre.

Pour un nouveau projet, `IExceptionHandler` est le bon choix. Il se place dans le pipeline middleware, couvre chaque exception quelle que soit sa source, et se teste via la DI. Les exception filters restent utiles si on a un code MVC existant avec une gestion d'erreurs par filtres et aucune raison de migrer.

> ⚠️ **Ça marche, mais...** — Mixer exception filters et `IExceptionHandler` crée deux chemins de gestion d'erreur que les développeurs doivent retenir. Si on adopte `IExceptionHandler`, on retire les exception filters pour éviter la confusion sur lequel tourne quand.

## Zoom : logging structuré dans le handler d'erreur

Le handler d'erreur est le seul endroit par lequel chaque exception non gérée passe. Ça en fait le bon endroit pour attacher des propriétés de log structurées :

```csharp
using (LogContext.PushProperty("TraceId", httpContext.TraceIdentifier))
using (LogContext.PushProperty("UserId", httpContext.User.FindFirst("sub")?.Value))
{
    _logger.LogError(exception, "Unhandled exception on {Method} {Path}",
        httpContext.Request.Method, httpContext.Request.Path);
}
```

`TraceId` et `UserId` atterrissent dans l'entrée de log structurée, ce qui veut dire que l'équipe ops peut chercher "toutes les erreurs pour l'utilisateur X dans la dernière heure" ou "toutes les erreurs avec ce trace ID". C'est le contexte minimum pour une investigation d'incident utile.

## Wrap-up

Tu sais maintenant que la gestion globale des erreurs en ASP.NET Core repose sur trois pièces : des implémentations `IExceptionHandler` enregistrées dans l'ordre de spécificité, `AddProblemDetails()` pour des réponses RFC 9457 cohérentes, et `UseStatusCodePages()` pour les codes de statut sans body. Les exceptions domaine se mappent en 4xx avec un handler typé ; tout le reste se mappe en 500 avec un handler de fallback. La réponse ne fuite jamais les détails internes en production, et le log structuré porte assez de contexte pour diagnostiquer sans reproduire.

Prêt à remplacer le try-catch éparpillé par un pipeline propre, ou à partager ce câblage avec ton équipe ?

À la prochaine, a++ 👋

## Pour aller plus loin

- [Exceptions Personnalisées en .NET](/fr/posts/error-handling-custom-exceptions/)
- [Couche Application : CQS et CQRS](/fr/posts/layer-focused-cqs-cqrs/)

## Références

- [Microsoft : Gérer les erreurs dans ASP.NET Core](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/error-handling)
- [Microsoft : IExceptionHandler](https://learn.microsoft.com/fr-fr/dotnet/api/microsoft.aspnetcore.diagnostics.iexceptionhandler)
- [Microsoft : Utiliser ProblemDetails](https://learn.microsoft.com/fr-fr/aspnet/core/fundamentals/error-handling#problem-details)
- [RFC 9457 : Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
