---
title: "Les Tests de Charge en .NET : vue d'ensemble des quatre types qui comptent"
date: 2026-04-08
draft: false
tags: ["load-testing", "performance", "dotnet"]
categories: ["Testing"]
series: ["Tests de Charge"]
description: "Baseline, soak, stress, spike : quatre types de tests de charge, quatre questions sur la tenue en production. Un tour pragmatique pour les équipes .NET, avec les outils et les métriques qui comptent vraiment."
---

Hello tous le monde, aujourd'hui on va démystifier les **tests de charge** en .NET, et plus particulièrement les quatre types qui structurent une vraie stratégie de performance.

Les tests unitaires, les tests d'intégration, les tests d'API et les tests end-to-end partagent tous une hypothèse silencieuse : ils tournent un utilisateur à la fois. Cette hypothèse est confortable, productive, et complètement aveugle à la question que la prod finit toujours par poser : "que se passe-t-il quand mille utilisateurs arrivent en même temps ?". Les tests de charge existent pour répondre à cette question avant que la réponse ne soit un appel téléphonique à 3 heures du matin.

La [série Testing](/fr/posts/testing-unit-testing/) a couvert la correction à travers toute la pyramide, des tests unitaires aux [tests d'intégration avec TestContainers](/fr/posts/testing-integration-testing-testcontainers/), aux [tests d'API avec WebApplicationFactory](/fr/posts/testing-webapplicationfactory/), et aux [tests end-to-end avec Playwright](/fr/posts/testing-e2e-playwright/). Les tests de charge sont sur un axe totalement différent. Ils ne demandent pas "est-ce que la logique marche", ils demandent "est-ce que la logique tient sous concurrence, sous trafic soutenu, sous des bursts soudains, et au-delà de la capacité prévue". Quatre questions, quatre types de tests, quatre articles dans cette série. Cet article est la carte.

## Le contexte : pourquoi les tests de charge existent

L'excuse classique pour les sauter a longtemps été "on scalera quand ce sera nécessaire". Ça marche jusqu'au jour où une campagne marketing, un moment viral, ou une intégration avec un partenaire devenu populaire envoient dix fois le trafic habituel en trente secondes. À ce moment-là, l'équipe découvre, d'un coup, que le pool de connexions à la base est plafonné à 100, que le cache ne se reconstruit pas proprement quand tout le monde tape en même temps, qu'un framework de log tient un verrou qui sérialise chaque requête, et que l'autoscaler prend quatre minutes pour réagir à un burst qui en dure deux.

Les tests de charge font remonter tout cela *avant* l'incident. Plus concrètement, ils répondent à quatre questions précises que la prod finira par poser :

1. **À quoi ressemble "la normale" ?** Sans point de référence, il est impossible de détecter qu'un déploiement a dégradé les choses.
2. **Est-ce que le système se dégrade gracieusement sur des heures ou des jours ?** Fuites mémoire, épuisement de pool, bugs de rotation de logs, dérive de cache : tout cela n'apparaît qu'après une exécution prolongée.
3. **Où le système casse-t-il, et comment ?** Comprendre le mode de défaillance est aussi important que de connaître le point de rupture.
4. **Comment le système réagit-il à un burst soudain ?** L'autoscaling, la backpressure, la profondeur de file, et les caches froids se comportent différemment selon qu'on monte progressivement ou qu'on passe de 0 à 100 en quelques secondes.

Chaque question a son type de test dédié. Aucun ne remplace les autres.

## Vue d'ensemble : les quatre types

{{< mermaid >}}
graph TD
    A[Tests de charge] --> B[Baseline<br/>Établir la normale<br/>régime stable]
    A --> C[Soak<br/>Longue durée<br/>charge modérée]
    A --> D[Stress<br/>Au-delà de la capacité<br/>trouver le point de rupture]
    A --> E[Spike<br/>Burst soudain<br/>de faible à très élevé]
    B --> B1[Référence<br/>pour la régression]
    C --> C1[Fuites, pool épuisé,<br/>logs, dérive cache]
    D --> D1[Point de rupture,<br/>capacity planning]
    E --> E1[Réactivité autoscale,<br/>cache froid, backpressure]
{{< /mermaid >}}

**Le baseline** fait tourner le système sous le trafic qu'il est censé gérer au quotidien, assez longtemps pour produire des chiffres stables. La sortie est un jeu de métriques de référence : requêtes par seconde, percentiles de latence, taux d'erreur, CPU, mémoire, usage du pool de base de données. Chaque test de charge ultérieur est comparé à cette référence.

**Le soak** fait tourner la même charge modérée pendant des heures, souvent toute une nuit, parfois plusieurs jours. Son rôle n'est pas de mesurer le débit de pointe, c'est de vérifier que le système ne se dégrade pas avec le temps. Les fuites mémoire, l'épuisement du pool de connexions, la croissance des fichiers de log, la dérive de l'invalidation de cache, et l'accumulation de tâches de fond, tout cela apparaît ici et nulle part ailleurs.

**Le stress** pousse le système au-delà de sa capacité prévue, et continue à pousser jusqu'à ce que quelque chose cède. L'objectif n'est pas de prouver que le système peut supporter une charge infinie, c'est de caractériser le *mode de défaillance* : la latence grandit-elle linéairement, puis explose-t-elle ? Le taux d'erreur monte-t-il avant la latence ? Le système récupère-t-il proprement quand le stress est retiré ?

**Le spike** part d'un état calme et monte à une charge très élevée en quelques secondes. C'est le test qui met en évidence la latence de l'autoscaling, les pénalités de cache froid, la gestion des bursts de connexions, et le coût de chauffe des chemins de code JIT. Un système qui encaisse parfaitement une montée progressive peut quand même s'effondrer sur un spike.

Chacun de ces types a son propre article dans la série. Le reste de cette vue d'ensemble couvre le vocabulaire partagé et les choix d'outillage qui s'appliquent aux quatre.

## Zoom : les métriques qui comptent

Chaque test de charge, quel que soit son type, doit rapporter le même jeu de chiffres. Si l'un d'eux manque, le test est incomplet.

**Le débit** mesuré en requêtes par seconde (RPS). Le compte brut du travail que le système traite par unité de temps. Un RPS élevé n'a de sens que couplé à la métrique suivante.

**Les percentiles de latence** : p50, p95, p99, et p99.9. La latence moyenne n'est presque jamais utile, parce qu'un système où 90% des requêtes prennent 20 ms et 10% prennent 2 secondes a la même moyenne qu'un système où chaque requête prend 220 ms, et les deux ne sont pas équivalents pour un utilisateur. Rapporter des percentiles, toujours.

**Le taux d'erreur**, ventilé par code de statut. Un test qui maintient un p95 sous 100 ms tout en renvoyant silencieusement 4% des requêtes en 500 n'est pas un test qui passe, c'est un test qui induit en erreur.

**Les signaux de saturation** du runtime .NET et de l'infrastructure : CPU, mémoire, temps de pause GC (gen0/1/2), longueur de la file du thread pool, temps d'attente sur le pool de connexions de la base, nombre de connexions HTTP client. Ces signaux disent *pourquoi* la latence a monté, ce qui est la moitié actionnable de l'information.

**La corrélation avec les transactions métier**, pas seulement les endpoints HTTP. Un test qui rapporte "POST /orders p95 à 300 ms" est moins utile qu'un test qui rapporte "le tunnel de checkout (ajout au panier, remise, soumission, confirmation de paiement) p95 à 1,2 s". L'expérience utilisateur est la composition des endpoints individuels, pas un seul d'entre eux.

> 💡 **Info** : Dans .NET moderne (8+), `System.Diagnostics.Metrics` et l'histogramme natif `http.server.request.duration` exposent ces chiffres nativement. Les envoyer vers Prometheus et Grafana se fait en deux lignes de configuration, et c'est la fondation de tout ce qui est décrit dans cette série.

## Zoom : le paysage des outils en 2026

Deux outils compatibles .NET couvrent 90% des cas réels, et le choix entre les deux se joue surtout sur l'endroit où vit le code de test.

**k6** (Grafana Labs) est le standard actuel de l'industrie. Les tests s'écrivent en JavaScript, tournent sur un runner en Go, et passent à l'échelle de centaines de milliers d'utilisateurs virtuels depuis une seule machine. k6 s'intègre proprement avec Grafana pour la visualisation, avec Prometheus comme puits de métriques, et avec la plupart des systèmes de CI. Il est agnostique du langage, ce qui est un atout si l'équipe livre plusieurs stacks backend, et un point neutre si elle ne livre que du .NET.

```javascript
// k6 : un test baseline, 50 utilisateurs virtuels pendant 5 minutes
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 50,
  duration: '5m',
  thresholds: {
    http_req_duration: ['p(95)<300'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('https://shop.test/api/orders');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

**NBomber** est l'option native .NET. Les tests s'écrivent en C# ou F#, vivent dans un projet .NET normal, partagent les types avec l'application sous test, et tournent depuis `dotnet test` ou un host console. L'avantage est que la suite de tests de charge est du code que l'équipe sait déjà lire, relire et refactorer.

```csharp
// NBomber : même baseline, écrit en C#
using NBomber.CSharp;
using NBomber.Http;
using NBomber.Http.CSharp;

var scenario = Scenario.Create("get_orders", async context =>
{
    var response = await Http.CreateRequest("GET", "https://shop.test/api/orders")
        .WithHeader("Accept", "application/json")
        .SendAsync(httpClient, context);

    return response;
})
.WithLoadSimulations(
    Simulation.KeepConstant(copies: 50, during: TimeSpan.FromMinutes(5)));

NBomberRunner.RegisterScenarios(scenario).Run();
```

Les deux sont de qualité production. Pour les équipes .NET qui préfèrent garder tout en C#, NBomber est le choix qui a le moins de friction. Pour les équipes qui veulent la communauté et l'écosystème les plus larges, k6 est le pari le plus sûr. JMeter, Gatling, Artillery et Locust existent et ont des cas d'usage légitimes, mais pour un projet .NET greenfield en 2026, k6 ou NBomber est la recommandation par défaut.

> ✅ **Bonne pratique** : Écrire le code des tests de charge dans le même repository que l'application, à côté des tests d'intégration. Les tests de charge font partie du codebase, ce ne sont pas un dossier à part sur le poste de quelqu'un.

## Zoom : où les tests de charge tournent

Un test de charge contre un poste de développeur est presque toujours sans valeur. Le réseau, la base locale, le CPU partagé avec tous les IDE et navigateurs ouverts, et l'absence d'infrastructure réaliste faussent tous le résultat. Les environnements utiles sont :

- **Un environnement de pré-prod dédié** qui reflète le sizing et la topologie de la prod. C'est la cible par défaut pour les tests baseline, soak et spike.
- **Un clone de la prod**, monté pour une fenêtre de test planifiée. Plus coûteux, plus précis, réservé aux tests de stress et aux exercices de capacity planning.
- **La prod elle-même, avec un sous-ensemble de trafic contrôlé**, pour les équipes avancées qui pratiquent le load testing continu. Cela demande une maturité d'observabilité que la plupart des équipes n'ont pas, et ce n'est pas le point de départ.

Pour la plupart des équipes, la bonne réponse est un environnement de pré-prod provisionné depuis le même Infrastructure-as-Code que la prod, avec la même classe de sizing de base, le même cache, et les mêmes dépendances démarrées via [TestContainers](/fr/posts/testing-integration-testing-testcontainers/) quand un vrai service managé n'est pas disponible.

> ⚠️ **Ça marche, mais...** : Faire tourner des tests de charge contre une base cloud en free tier ou contre un petit container de dev va produire des chiffres qui ont l'air terribles par rapport à la prod, ou pire, des chiffres qui ont l'air excellents et qui sont totalement faux. L'attention doit autant porter sur le sizing de la cible que sur celui du générateur de charge.

## Zoom : ce que les tests de charge n'attrapent pas

Les tests de charge ne remplacent aucune autre couche de la pyramide de tests. Ils n'attrapent pas :

- **Les bugs de logique** : le calculator peut être faux et tenir 10 000 RPS. C'est un problème de [test unitaire](/fr/posts/testing-unit-testing/).
- **Les trous d'autorisation** : un contrôle de rôle cassé est rapide. Rapide et faux est pire que lent et correct. C'est un problème de [test WebApplicationFactory](/fr/posts/testing-webapplicationfactory/).
- **La correction des migrations de données** : un test de charge contre une migration cassée échouera simplement avec des données cassées. Les migrations se testent d'abord contre une vraie base, via les [tests d'intégration avec TestContainers](/fr/posts/testing-integration-testing-testcontainers/).
- **Les race conditions au niveau UI** : ce sont des [tests Playwright E2E](/fr/posts/testing-e2e-playwright/).

Les tests de charge se posent sur un système correct, pas à la place d'un système correct. Les lancer avant que le reste de la pyramide soit au vert est une perte de temps pour le générateur de charge, et une source de fausse confiance.

## Wrap-up

Tu as maintenant la carte des quatre types de tests de charge qui comptent pour un système .NET : le baseline pour établir ce que "la normale" veut dire, le soak pour vérifier que le système tient dans la durée, le stress pour trouver le point de rupture et sa forme, et le spike pour valider l'autoscaling et la gestion des bursts. Tu peux choisir k6 ou NBomber comme runner par défaut, capturer le débit, les percentiles de latence, le taux d'erreur et les signaux de saturation pour chaque test, et les lancer contre un environnement de pré-prod qui reflète réellement la prod.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Les Tests Unitaires en .NET : rapides, ciblés, et vraiment utiles](/fr/posts/testing-unit-testing/)
- [Tests d'Intégration avec TestContainers pour .NET](/fr/posts/testing-integration-testing-testcontainers/)
- [Tests API avec WebApplicationFactory en ASP.NET Core](/fr/posts/testing-webapplicationfactory/)
- [Tests End-to-End avec Playwright pour .NET](/fr/posts/testing-e2e-playwright/)

## Références

- [Documentation k6](https://grafana.com/docs/k6/latest/)
- [Documentation NBomber](https://nbomber.com/)
- [Métriques ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/log-mon/metrics/metrics)
- [Diagnostics et performance .NET, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/core/diagnostics/)
- [Google SRE book : Handling Overload](https://sre.google/sre-book/handling-overload/)
