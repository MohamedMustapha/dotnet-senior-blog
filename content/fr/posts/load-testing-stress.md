---
title: "Le Stress Testing en .NET : trouver le point de rupture et sa forme"
date: 2026-04-08
draft: false
tags: ["load-testing", "stress", "performance", "dotnet"]
categories: ["Testing"]
series: ["Tests de Charge"]
description: "Le stress ne demande pas 'est-ce que le système tient la charge normale'. Il demande 'quand casse-t-il, comment casse-t-il, et comment récupère-t-il'. Trois questions qui comptent plus qu'un RPS de pointe."
---

Hello tous le monde, aujourd'hui on va découvrir le **stress testing**, le type de test qui pousse le système au-delà de ses limites pour en apprendre quelque chose d'utile.

Un baseline dit à quoi ressemble la normale. Un [soak](/fr/posts/load-testing-soak/) dit si le système tient dans la durée. Aucun des deux ne répond à la question que la prod finit toujours par imposer : **quand est-ce que ça casse, et comment**. C'est le rôle du stress test. Pas de prouver que le système peut gérer une charge arbitraire (aucun système ne le peut), mais de caractériser la forme exacte de sa défaillance pour que l'équipe puisse concevoir autour.

L'[article d'ensemble](/fr/posts/load-testing-overview/) a présenté les quatre types. L'[article baseline](/fr/posts/load-testing-baseline/) a couvert le run de référence. Cet article couvre celui qui casse délibérément le système, qui apprend quelque chose de cette cassure, et qui en sort avec un plan de capacité concret.

## Le contexte : pourquoi le stress existe

Chaque système a un point au-delà duquel plus de trafic dégrade les choses au lieu de les améliorer. Ajouter une requête par seconde de plus commence à empiler du travail plus vite que les workers ne peuvent le traiter. La latence monte, puis grimpe fortement, puis le taux d'erreur commence à croître. Finalement, quelque chose cède : un pool de connexions sature, un thread pool meurt de faim, un disjoncteur s'ouvre, ou le process tombe en OOM et redémarre. L'équipe qui apprend ça en prod paie la leçon par un incident. L'équipe qui l'apprend en stress test paie la même leçon par un tableur.

Les stress tests répondent à quatre questions qu'aucun autre type de test ne traite :

1. **Où est le point de rupture ?** La charge (en RPS ou en utilisateurs concurrents) à laquelle la latence explose, le taux d'erreur grimpe, ou le process lâche. Le chiffre lui-même est utile pour le capacity planning.
2. **Quelle est la forme de la défaillance ?** Dégradation linéaire, exponentielle, effondrement en falaise, et défaillance en cascade demandent tous des réponses différentes. La forme est plus actionnable que le chiffre brut.
3. **Quel composant cède en premier ?** Est-ce le pool de connexions base, le thread pool, la mémoire, l'API en aval, le rate limiter ? Le premier goulet d'étranglement est celui qui vaut la peine d'être corrigé.
4. **Est-ce que le système récupère ?** Une fois le stress retiré, le système revient-il à une latence et un débit sains, ou reste-t-il dégradé et demande un redémarrage ? Le comportement de récupération compte autant que le point de rupture.

Sans stress test, le capacity planning est de la devinette. Avec un, l'équipe a un chiffre, une forme, et un profil de récupération.

## Vue d'ensemble : la forme d'un run stress

{{< mermaid >}}
graph LR
    A[Charge baseline<br/>50 VUs] --> B[Rampe<br/>+50 VUs toutes les 2 min]
    B --> C[Observation<br/>point de rupture]
    C --> D[Maintien au-dessus<br/>du seuil 1-2 min]
    D --> E[Rampe descendante<br/>observation de la récupération]
{{< /mermaid >}}

Un stress test est une rampe contrôlée, pas un burst soudain. Le système démarre à la charge baseline, monte par paliers mesurés, et le test capture le point auquel l'objectif de service pré-défini est franchi. Ce point est le point de rupture. La rampe continue un court moment au-delà pour caractériser le mode de défaillance, puis descend pour observer la récupération.

Trois règles façonnent un run stress utile :

**Monter, pas sauter.** Un burst soudain est un test de spike, qui est une autre question. Un stress veut voir la pente de la dégradation, ce qui demande une montée progressive et mesurée.

**Définir l'échec avant le run.** "Le système est cassé" n'est pas une affirmation objective. Décider à l'avance : par exemple, le point de rupture est atteint quand le p95 dépasse 1 seconde ou que le taux d'erreur dépasse 5%. Sans ça, l'équipe va discuter des résultats après coup.

**Toujours redescendre en rampe.** Observer comment le système récupère (ou pas) représente la moitié de la valeur du test. Un stress qui coupe le trafic au pic et rapporte "on a tenu 5000 RPS" n'a rien appris sur la capacité réelle que la prod pourrait soutenir.

## Zoom : un run stress avec k6

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  stages: [
    { duration: '2m',  target: 50 },    // maintien baseline
    { duration: '2m',  target: 100 },   // +50 VUs
    { duration: '2m',  target: 150 },
    { duration: '2m',  target: 200 },
    { duration: '2m',  target: 300 },
    { duration: '2m',  target: 400 },
    { duration: '2m',  target: 500 },
    { duration: '2m',  target: 500 },   // maintien au pic
    { duration: '3m',  target: 0 },     // descente, observation récupération
  ],
  thresholds: {
    // Ces seuils sont la définition de l'échec.
    // Un seuil franchi fait échouer le run, ce qui est attendu au-delà du point de rupture.
    'http_req_duration{group:::hot}': ['p(95)<1000'],
    'http_req_failed': ['rate<0.05'],
  },
};

const BASE = __ENV.BASE_URL || 'https://shop.preprod.internal';

export default function () {
  group('hot', () => {
    http.get(`${BASE}/api/products/featured`);
  });

  if (Math.random() < 0.3) {
    group('write', () => {
      http.post(`${BASE}/api/cart`, JSON.stringify({
        productId: 'SKU-1',
        quantity: 1,
      }), { headers: { 'Content-Type': 'application/json' } });
    });
  }

  sleep(0.5);
}
```

Un plafond à 500 VUs, atteint en six paliers de +50 à +100 VUs chacun. Chaque palier dure deux minutes, ce qui est assez long pour que le système se stabilise à ce niveau de charge avant le palier suivant. La descente est courte et délibérée : trois minutes du pic à zéro, et c'est là que le comportement de récupération est capturé.

> ✅ **Bonne pratique** : Choisir la taille des paliers pour que la rampe entière prenne 15 à 25 minutes. Les runs plus courts manquent le comportement en régime stable à chaque niveau. Les runs plus longs brûlent du budget et rendent le résultat difficile à interpréter.

## Zoom : le même run avec NBomber

```csharp
using NBomber.CSharp;
using NBomber.Http;
using NBomber.Http.CSharp;

using var httpClient = new HttpClient { BaseAddress = new Uri("https://shop.preprod.internal") };

var scenario = Scenario.Create("hot_path", async context =>
{
    var request = Http.CreateRequest("GET", "/api/products/featured");
    return await Http.Send(httpClient, request);
})
.WithLoadSimulations(
    Simulation.KeepConstant(copies: 50,  during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 100, during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 200, during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 300, during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 400, during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 500, during: TimeSpan.FromMinutes(2)),
    Simulation.KeepConstant(copies: 500, during: TimeSpan.FromMinutes(2)),
    Simulation.RampingConstant(copies: 0,   during: TimeSpan.FromMinutes(3))
);

NBomberRunner.RegisterScenarios(scenario)
    .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
    .WithReportFolder("./reports/stress")
    .Run();
```

Même profil en escalier, exprimé sous forme de liste de stages `LoadSimulation`. Le rapport HTML de NBomber trace la latence et le débit par palier, ce qui est exactement la forme qu'un stress test doit produire.

## Zoom : identifier le point de rupture

Le point de rupture n'est pas toujours évident depuis un seul graphe. C'est l'intersection de trois signaux.

**La courbe p95 de latence.** Tracer le p95 en fonction du nombre de VUs. Dans un système sain, la courbe est quasiment plate, puis commence à monter, puis monte fortement. Le point de rupture est là où la montée devient super-linéaire, en général visible comme un point d'inflexion sur le graphe.

**La courbe du taux d'erreur.** Tracer le taux d'erreur en fonction du nombre de VUs. Dans la plupart des systèmes .NET, le taux d'erreur reste proche de zéro jusqu'au point de rupture, puis monte vite. Si le taux d'erreur commence à monter avant la latence, le goulet est une limite dure (un pool de connexions, un rate limiter, un disjoncteur). Si la latence monte d'abord, le goulet est une limite molle (CPU, mémoire, file d'attente du thread pool).

**La courbe du débit.** Tracer le RPS réussi en fonction du nombre de VUs. Dans un système sain, le débit croît avec les VUs, puis plafonne au maximum du système. Dans un système qui défaille, le débit atteint un pic, puis *baisse* à mesure que le système passe plus de temps à gérer des échecs qu'à traiter du vrai travail. La baisse est le signal le plus actionnable : elle signifie que le système fait pire sous plus de charge, pas juste moins bien.

L'intersection de ces trois courbes donne un chiffre défendable : "le système supporte 320 RPS avant que le p95 ne dépasse 1 seconde et que le taux d'erreur ne dépasse 1%". Ce chiffre est utilisable pour le capacity planning, pour la négociation de contrats, et pour le sizing du déploiement.

## Zoom : la forme de la défaillance

La courbe elle-même compte autant que le chiffre. Quatre formes de défaillance sont courantes.

**Dégradation linéaire.** La latence monte doucement, le taux d'erreur reste proche de zéro, le débit plafonne proprement. La meilleure forme possible, parce qu'elle veut dire que l'équipe peut scaler horizontalement de façon linéaire pour suivre la demande et prédire le comportement au-delà du point de rupture. Indique en général un système CPU-bound avec des pools bien paramétrés.

**Courbe en coude.** La latence est plate, puis se plie brutalement vers le haut à un niveau de charge précis. Indique une limite de ressource dure : un pool de connexions qui atteint son max, une tempête de cache miss, un thread pool qui sature. Le correctif est en général un seul changement de configuration, une fois la ressource identifiée.

**Effondrement en falaise.** La latence est plate, tout a l'air bien, puis le système s'effondre en 30 secondes : les erreurs explosent, le débit tombe à zéro. Indique une défaillance en cascade : un disjoncteur qui s'ouvre et affame un service dépendant, un deadlock qui se propage à travers les requêtes, un OOM qui redémarre le process. Les défaillances en falaise sont les plus dangereuses parce qu'il n'y a aucun avertissement avant l'incident.

**Spirale de la mort.** La latence monte, puis le débit baisse, puis la latence monte encore parce que les retries s'empilent sur un système déjà surchargé. Le système empire à mesure qu'il reçoit du trafic, même si le trafic arrête de croître. Le correctif est en général de la backpressure ou du load shedding, pas plus de capacité.

> 💡 **Info** : Le runtime .NET a un limiteur de concurrence natif (`Microsoft.AspNetCore.RateLimiting`, disponible depuis .NET 7) conçu précisément pour prévenir les spirales de la mort. Ajouter un rate limiter basé sur une file devant les endpoints sensibles transforme une spirale de la mort en rejet contrôlé, qui est beaucoup plus facile à raisonner.

## Zoom : la récupération

Une fois la descente commencée, la question passe de "jusqu'où c'est descendu" à "est-ce que le système revient". Trois issues.

**Récupération propre.** En quelques secondes après la baisse de charge, la latence revient au baseline, le taux d'erreur revient à zéro, le débit suit la demande. C'est le résultat attendu, et il confirme que le système peut se délester de la charge sans effet secondaire.

**Récupération lente.** La latence met plusieurs minutes à revenir au baseline, même après la baisse de charge. Indique en général que quelque chose est encore en train de se drainer : une file qui a accumulé du backlog, un pool de connexions qui libère lentement ses connexions coincées, un cache qui se reconstruit à froid après une tempête d'invalidation. Le temps de récupération est lui-même une métrique, et c'est souvent là que le coût caché de la défaillance vit.

**Pas de récupération.** La latence reste élevée, ou le système continue de renvoyer des erreurs, même à charge zéro. Indique des dommages permanents : un thread fuité qui tient un verrou, une state machine async en deadlock, un disjoncteur bloqué ouvert, un cache qui ne peut pas se réhydrater. Le process a besoin d'un redémarrage pour retrouver la santé, et c'est une information que l'équipe doit avoir avant que la même défaillance ne se produise en prod.

> ⚠️ **Ça marche, mais...** : Un stress qui ne mesure que le RPS de pointe sans mesurer la récupération ne rapporte que la moitié de l'histoire. L'équipe peut atteindre un pic que la prod ne peut pas, si la récupération après ce pic est impossible. Le capacity planning doit tenir compte de la marge nécessaire pour éviter le pic, pas seulement du pic lui-même.

> ❌ **Ne jamais faire** : Ne pas lancer un stress test contre la prod sans un blast radius strict et une condition d'arrêt convenue à l'avance. Les stress tests sont faits pour casser des choses, et la base de prod n'est pas l'endroit pour découvrir ce qui casse.

## Wrap-up

Un stress test est le seul test qui produit un chiffre de capacité que l'équipe peut réellement défendre. On peut en monter un en k6 ou NBomber en un après-midi, utiliser une rampe en escalier de 15 à 25 minutes, définir la condition d'échec avant le run pour éviter de discuter des résultats après coup, capturer les courbes de latence, de taux d'erreur et de débit côte à côte, identifier la forme de la défaillance, et toujours inclure une phase de descente pour mesurer la récupération. On peut sortir d'un stress avec un chiffre défendable pour le capacity planning, un premier goulet d'étranglement nommé à corriger, et de la confiance sur le comportement du système quand le trafic de prod dépasse les limites attendues.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Les Tests de Charge en .NET : vue d'ensemble des quatre types qui comptent](/fr/posts/load-testing-overview/)
- [Le Baseline Testing en .NET : savoir à quoi ressemble la normale](/fr/posts/load-testing-baseline/)
- [Le Soak Testing en .NET : les bugs qui n'apparaissent qu'après des heures](/fr/posts/load-testing-soak/)

## Références

- [Patterns de stress testing k6](https://grafana.com/docs/k6/latest/testing-guides/test-types/stress-testing/)
- [Load simulations NBomber](https://nbomber.com/docs/using-nbomber/load-simulations/)
- [Middleware de rate limiting ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/performance/rate-limit)
- [Google SRE book : Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
