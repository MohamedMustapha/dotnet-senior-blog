---
title: "Le Baseline Testing en .NET : savoir à quoi ressemble la normale"
date: 2026-04-08
draft: false
tags: ["load-testing", "baseline", "performance", "dotnet"]
categories: ["Testing"]
series: ["Tests de Charge"]
description: "Un test baseline établit les métriques de référence auxquelles chaque autre test de charge sera comparé. Voilà comment en monter un proprement en .NET avec k6 et NBomber, et quoi faire des chiffres."
---

Hello tous le monde, aujourd'hui on va explorer le **baseline testing**, le premier type de test de charge à mettre en place dans un projet .NET.

Le premier test de charge qu'une équipe devrait lancer n'est presque jamais le plus impressionnant. C'est le plus ennuyeux : le système sous le trafic qu'il est censé gérer au quotidien, assez longtemps pour produire des chiffres stables, et rien de plus. C'est ça, le **baseline**. Sans lui, tous les autres tests de charge n'ont aucun sens, parce qu'il n'y a aucune référence à laquelle comparer. Un p95 à 300 ms ne veut rien dire tant qu'on ne sait pas s'il est meilleur ou pire que la semaine dernière.

L'[article d'ensemble](/fr/posts/load-testing-overview/) de cette série a couvert pourquoi les tests de charge existent et quelles métriques comptent. Cet article zoome sur le premier des quatre types et explique comment réellement monter, lancer et exploiter un baseline dans un projet .NET.

## Le contexte : pourquoi le baseline existe

Une équipe livre une fonctionnalité. La fonctionnalité implique une requête EF Core qui a l'air innocente. Le déploiement suivant passe en prod. Deux semaines plus tard, un client signale que le dashboard est lent. L'équipe regarde Grafana, voit que la latence est effectivement plus haute que d'habitude, et pose la seule question qui compte : "plus haute que *quoi*, exactement ?". Sans baseline, la réponse est "plus haute que mon souvenir de la sensation du mois dernier", ce qui n'est pas une réponse.

Le baseline répond à quatre problèmes concrets :

1. **Il établit une référence.** Un chiffre stable enregistré sous un profil de trafic connu, sauvegardé avec le hash du commit et la date de déploiement. Chaque run suivant peut être comparé à lui.
2. **Il attrape les régressions avant que la prod ne les voie.** Une pull request qui double les allers-retours en base pour le tunnel de checkout échouera à la comparaison baseline en CI, pas à 2 heures du matin un lundi.
3. **Il valide les hypothèses de sizing.** Si le p95 baseline est déjà proche du SLO au trafic attendu, la prod n'a plus de marge, et l'équipe le sait avant l'incident.
4. **Il ancre tous les autres tests de charge.** Les tests soak, stress et spike sont toujours relatifs au baseline. Sans lui, "le système s'est dégradé sous stress" est une phrase sans dénominateur.

## Vue d'ensemble : la forme d'un run baseline

{{< mermaid >}}
graph LR
    A[Pré-prod<br/>fraîche] --> B[Warmup<br/>1-2 min]
    B --> C[Régime stable<br/>5-10 min au<br/>RPS attendu]
    C --> D[Capture des métriques]
    D --> E[Stockage référence<br/>avec hash commit]
    E --> F[Comparaison avec<br/>baseline précédent]
{{< /mermaid >}}

Un run baseline a quatre phases. Le **warmup** existe parce que la compilation JIT, le remplissage du cache, et l'amorçage du pool de connexions faussent tous la première minute de n'importe quel test .NET. Le **régime stable** est la phase où les chiffres sont réellement capturés, assez longtemps pour lisser le bruit du GC et des tâches de fond. La **capture** produit un artefact structuré, pas juste un dump de log. Le **stockage et la comparaison** est la partie que la plupart des équipes sautent et regrettent.

Le profil de trafic pendant le régime stable doit refléter la prod le plus fidèlement possible. Si la prod fait 70% de lectures, 20% d'écritures et 10% de recherches, le baseline fait la même chose. Un baseline qui ne tape que sur `GET /orders` n'est pas un baseline, c'est un microbenchmark avec des prétentions.

## Zoom : un baseline réaliste avec k6

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Trend, Counter } from 'k6/metrics';

const checkoutLatency = new Trend('checkout_flow_duration');
const ordersCreated = new Counter('orders_created');

export const options = {
  stages: [
    { duration: '1m', target: 50 },    // rampe de warmup
    { duration: '10m', target: 50 },   // régime stable
    { duration: '30s', target: 0 },    // cooldown
  ],
  thresholds: {
    'http_req_duration{group:::catalog}': ['p(95)<200'],
    'http_req_duration{group:::checkout}': ['p(95)<500'],
    'http_req_failed': ['rate<0.005'],  // <0,5% d'erreurs
    'checkout_flow_duration': ['p(95)<1200'],
  },
};

const BASE = __ENV.BASE_URL || 'https://shop.preprod.internal';

export default function () {
  // 70% chemin de lecture
  group('catalog', () => {
    const r = http.get(`${BASE}/api/products?page=1&size=20`);
    check(r, { 'catalog ok': (res) => res.status === 200 });
  });

  // 20% chemin d'écriture : tunnel de checkout complet
  if (Math.random() < 0.2) {
    group('checkout', () => {
      const start = Date.now();

      const cart = http.post(`${BASE}/api/cart`, JSON.stringify({
        productId: 'SKU-42',
        quantity: 1,
      }), { headers: { 'Content-Type': 'application/json' } });

      const submit = http.post(`${BASE}/api/orders/${cart.json('id')}/submit`);

      checkoutLatency.add(Date.now() - start);
      if (submit.status === 204) ordersCreated.add(1);
    });
  }

  // 10% chemin de recherche
  if (Math.random() < 0.1) {
    group('search', () => {
      http.get(`${BASE}/api/search?q=jean`);
    });
  }

  sleep(1);
}
```

Trois chemins de trafic, pondérés pour correspondre à la prod. Une rampe de warmup, un régime stable de 10 minutes, un cooldown. Des seuils qui font échouer le run en CI si l'un d'eux casse. Des métriques custom qui suivent la transaction métier (le tunnel de checkout complet), pas seulement les endpoints individuels. Voilà à quoi ressemble un baseline sérieux.

> ✅ **Bonne pratique** : Tagger les requêtes avec `group()` pour que k6 remonte les métriques par chemin. Un p95 global qui mélange lectures et écritures n'a presque jamais d'utilité. Un p95 par groupe dit où la latence vit réellement.

## Zoom : le même baseline avec NBomber

Pour les équipes qui préfèrent garder tout en C# :

```csharp
using NBomber.CSharp;
using NBomber.Http;
using NBomber.Http.CSharp;

using var httpClient = new HttpClient { BaseAddress = new Uri("https://shop.preprod.internal") };

var catalogScenario = Scenario.Create("catalog", async context =>
{
    var request = Http.CreateRequest("GET", "/api/products?page=1&size=20")
        .WithHeader("Accept", "application/json");
    return await Http.Send(httpClient, request);
})
.WithWeight(70)
.WithLoadSimulations(
    Simulation.RampingConstant(copies: 50, during: TimeSpan.FromMinutes(1)),
    Simulation.KeepConstant(copies: 50, during: TimeSpan.FromMinutes(10)));

var checkoutScenario = Scenario.Create("checkout", async context =>
{
    var addToCart = Http.CreateRequest("POST", "/api/cart")
        .WithJsonBody(new { productId = "SKU-42", quantity = 1 });
    var cartResponse = await Http.Send(httpClient, addToCart);
    if (!cartResponse.IsError)
    {
        var cartId = cartResponse.Payload.Value.RootElement.GetProperty("id").GetString();
        var submit = Http.CreateRequest("POST", $"/api/orders/{cartId}/submit");
        return await Http.Send(httpClient, submit);
    }
    return cartResponse;
})
.WithWeight(20)
.WithLoadSimulations(
    Simulation.KeepConstant(copies: 50, during: TimeSpan.FromMinutes(10)));

NBomberRunner
    .RegisterScenarios(catalogScenario, checkoutScenario)
    .WithReportFormats(ReportFormat.Html, ReportFormat.Csv, ReportFormat.Md)
    .WithReportFolder("./reports/baseline")
    .Run();
```

Même idée, exprimée en C#. L'option `WithWeight` permet à NBomber de répartir les utilisateurs virtuels entre les scénarios dans le ratio attendu. Les rapports atterrissent dans `./reports/baseline/` et peuvent être commités, archivés ou poussés vers un bucket de stockage pour la comparaison historique.

## Zoom : quoi capturer, et où le stocker

Un baseline n'est pas utile comme un tas de fichiers CSV. Il est utile comme un enregistrement structuré qu'on peut differ. Au minimum, chaque run baseline devrait stocker :

- **Le hash du commit et la branche** de l'application sous test
- **Le timestamp de déploiement** et l'identifiant de l'environnement
- **La version de k6 ou NBomber** et le hash du fichier source du scénario
- **Les métriques par groupe** : latence p50, p95, p99, p99.9 ; RPS ; taux d'erreur par code de statut
- **Les signaux runtime** : CPU, mémoire, temps de pause GC, longueur de la file du thread pool, usage du pool de base
- **Le statut pass / fail** par rapport aux seuils configurés

Une convention simple qui marche bien : écrire un résumé JSON vers un bucket S3 / blob après chaque run, indexé par `<env>/<yyyy-mm-dd>/<hash-commit>.json`. Un job ultérieur diffe le run le plus récent avec le précédent et poste le delta en commentaire de la pull request. Cela transforme le baseline en un signal vivant de régression, au lieu d'un exercice ponctuel.

> 💡 **Info** : k6 supporte la remontée directe des résultats vers Prometheus (`k6 run --out experimental-prometheus-rw`) et vers Grafana Cloud. NBomber écrit nativement des rapports HTML, CSV et Markdown, et peut aussi alimenter InfluxDB. L'un ou l'autre chemin suffit pour construire la comparaison historique.

## Zoom : baseline par rapport à quoi, exactement

Une question à poser explicitement : quel niveau de trafic "baseline" veut-il dire pour le système ? Trois définitions courantes, chacune valable dans son contexte :

- **Le pic quotidien moyen.** L'heure la plus chargée d'une journée de semaine typique. C'est le point de départ le plus sûr pour la plupart des équipes, parce qu'il correspond à ce que le système gère réellement sur une journée normale.
- **Le pic hebdomadaire.** Le trafic à l'heure la plus chargée du jour le plus chargé de la semaine. Utile pour les systèmes avec des patterns hebdomadaires prévisibles (dashboards du lundi matin, e-commerce du vendredi soir).
- **La charge SLO cible.** Le niveau de trafic que le système est contractuellement censé soutenir, que la prod actuelle l'atteigne ou non. Utilisé quand le SLO est au-dessus du trafic réel et que l'équipe doit prouver que la marge existe.

En choisir un, l'écrire, et s'y tenir. Déplacer silencieusement la cible du baseline entre les runs est la meilleure façon de livrer "des améliorations" qui n'ont l'air d'être des améliorations que parce que la comparaison a bougé sous les pieds.

> ❌ **Ne jamais faire** : Ne pas enregistrer le baseline depuis un système froid un dimanche matin calme et le comparer à un test tournant sur un système chaud sous charge normale. Les deux ne sont pas comparables. Le warmup compte, le régime stable compte, la cohérence de l'environnement de référence compte. Un baseline qui bouge à chaque run n'est pas un baseline.

## Zoom : quand le lancer

Trois cadences couvrent la plupart des équipes :

**Toutes les nuits, en CI.** Un job planifié lance le baseline contre la pré-prod chaque nuit, stocke le résultat, et notifie sur régression. C'est l'automatisation à plus forte valeur que la plupart des équipes peuvent ajouter.

**Avant chaque release importante.** Même avec les runs nocturnes, un run dédié avant release attrape les problèmes qui n'apparaissent que sur le chemin de code spécifique de la version qui sort.

**À la demande, avant de merger une PR sensible à la performance.** Les équipes qui pratiquent ça ont une cible `dotnet run --project LoadTests.Baseline` ou un `k6 run baseline.js` qu'un dev peut déclencher localement contre une pré-prod partagée, avant de demander la review.

> ✅ **Bonne pratique** : Stocker l'artefact de référence du baseline à côté des notes de release. Quand un client signale "c'était plus rapide avant", l'équipe peut sortir le baseline de la dernière version connue comme bonne et prouver, ou réfuter, la réclamation avec des données.

## Wrap-up

Le baseline est le test de charge le moins cher et celui qui rentabilise le plus vite. Le lancer donne à l'équipe un point de référence auquel chaque changement, déploiement et [test soak / stress / spike](/fr/posts/load-testing-overview/) suivant peut être comparé. On peut le monter en k6 ou NBomber en un après-midi, tagger le trafic par chemin métier pour que les métriques par groupe reflètent de vrais flux utilisateurs, stocker des artefacts structurés avec les hashes de commit, et planifier des runs nocturnes contre la pré-prod pour attraper la régression avant la prod.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Les Tests de Charge en .NET : vue d'ensemble des quatre types qui comptent](/fr/posts/load-testing-overview/)
- [Tests d'Intégration avec TestContainers pour .NET](/fr/posts/testing-integration-testing-testcontainers/)
- [Tests API avec WebApplicationFactory en ASP.NET Core](/fr/posts/testing-webapplicationfactory/)

## Références

- [Documentation k6](https://grafana.com/docs/k6/latest/)
- [Seuils et métriques k6](https://grafana.com/docs/k6/latest/using-k6/thresholds/)
- [Documentation NBomber](https://nbomber.com/)
- [Métriques ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/log-mon/metrics/metrics)
