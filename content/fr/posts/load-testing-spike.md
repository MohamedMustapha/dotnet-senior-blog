---
title: "Le Spike Testing en .NET : survivre au burst soudain"
date: 2026-04-08
draft: false
tags: ["load-testing", "spike", "performance", "dotnet"]
categories: ["Testing"]
series: ["Tests de Charge"]
description: "Black Friday, contenu viral, cache froid, latence d'autoscale. Le spike test prouve si un système peut survivre à un burst soudain de bas à très haut, pas juste à une montée progressive."
---

Hello tous le monde, aujourd'hui on va démystifier le **spike testing**, le dernier des quatre types de tests de charge, et celui qui révèle comment le système réagit aux moments les plus visibles de sa vie.

Un système peut passer un [baseline](/fr/posts/load-testing-baseline/), tenir un [soak](/fr/posts/load-testing-soak/), récupérer proprement d'un [stress](/fr/posts/load-testing-stress/), et quand même échouer sur son moment public le plus visible : la seconde précise où le trafic passe du calme à l'écrasement, sans préavis. Le Black Friday à minuit, un tweet viral qui pointe vers une landing page, un email marketing envoyé à cinq cent mille boîtes en même temps, une intégration partenaire dont le cron se déclenche à l'heure pile. Ce sont les moments dont l'équipe se souvient, et ce n'est pas à ça que prépare une rampe de stress progressive.

Le spike testing est le dernier des quatre types présentés dans l'[article d'ensemble](/fr/posts/load-testing-overview/). Il répond à une seule question précise : **quand le trafic passe de presque zéro à très haut en moins de dix secondes, est-ce que le système tient, se dégrade gracieusement, ou s'effondre**.

## Le contexte : pourquoi le spike existe

Un stress test avec une rampe douce donne au système toutes les chances de s'adapter : les caches CPU chauffent, le JIT compile les chemins chauds, le pool de connexions base grandit pour suivre la demande, l'autoscaler réagit et provisionne de nouvelles instances. Un spike ne donne rien de tout ça. Ça part du calme, et quinze secondes plus tard c'est submergé. Les systèmes qui meurent sur un spike sont ceux qui avaient besoin de la rampe.

Concrètement, les spikes exposent quatre faiblesses distinctes qu'aucun autre type de test ne stresse aussi fort :

1. **La pénalité de cache froid.** Les caches distribués vont bien jusqu'à ce que tous les nœuds ratent en même temps. La base reçoit alors tout le trafic, amplifié par une tempête de requêtes identiques concurrentes, et s'effondre avant que le cache n'ait le temps de se réhydrater.
2. **La latence de l'autoscaling.** Les horizontal pod autoscalers Kubernetes, Azure Container Apps, AWS ECS, et chaque autre autoscaler ont un temps de réaction. Ce temps est en général de l'ordre de la minute. Un spike qui dure quatre-vingt-dix secondes est fini avant qu'une seule nouvelle instance ne soit prête.
3. **Le coût de démarrage des pools de connexions.** Les drivers de base de données, les clients HTTP, et les connexions aux brokers de messages mettent du temps à s'établir. Une application qui démarre avec un pool de 10 connexions et en a besoin de 200 passera les trente premières secondes du spike en timeout pendant que le pool grossit.
4. **La compilation JIT et le warmup.** .NET JIT les méthodes au premier appel. Les méthodes tier-0 sont re-JIT en tier-1 une fois qu'elles se révèlent chaudes. Un spike touche le système avant que les chemins chauds ne soient compilés en tier-1, ce qui peut doubler la latence des premiers milliers de requêtes.

Aucun de ces problèmes n'est visible dans un test en régime stable. Tous sont visibles dans un spike, et tous sont corrigeables, en général avec des changements de configuration et des stratégies de warmup qui coûtent très peu.

## Vue d'ensemble : la forme d'un run spike

{{< mermaid >}}
graph LR
    A[Idle ou très bas<br/>5-10 VUs] --> B[Saut soudain<br/>10 -> 500 VUs<br/>en moins de 30s]
    B --> C[Maintien au pic<br/>2-5 min]
    C --> D[Redescente<br/>vers idle]
    D --> E[Observation<br/>second spike<br/>si besoin]
{{< /mermaid >}}

Un spike test a quatre phases, chacune avec un rôle précis.

**La phase idle** établit que le système est au calme. Trafic bas ou nul, pendant une ou deux minutes. C'est l'état que le spike va interrompre.

**Le saut** est la caractéristique définissante du test. La montée se fait en secondes, pas en minutes. Un spike est fait pour prendre le système au dépourvu. Si la montée est progressive, le test est un stress, pas un spike.

**Le maintien au pic** garde la charge haute pendant deux à cinq minutes. Assez longtemps pour que l'autoscaler (s'il y en a un) réagisse, que le JIT chauffe, que le cache se réhydrate, et que les pools de connexions grandissent. Cette phase répond à la question "est-ce que le système récupère pendant qu'il est encore sous charge".

**La redescente** revient à l'idle. Optionnellement, un second spike suit une minute plus tard, pour tester si le système est réellement prêt pour le prochain burst ou s'il est encore en train de récupérer du premier.

## Zoom : un spike avec k6

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m',  target: 10 },    // idle
    { duration: '10s', target: 500 },   // le spike : 10 -> 500 en 10s
    { duration: '3m',  target: 500 },   // maintien au pic
    { duration: '10s', target: 10 },    // redescente
    { duration: '30s', target: 10 },    // observation de la récupération
    { duration: '10s', target: 500 },   // second spike (optionnel)
    { duration: '1m',  target: 500 },
    { duration: '10s', target: 0 },
  ],
  thresholds: {
    // Un spike a des seuils volontairement plus lâches : l'objectif est "encore debout", pas "latence baseline".
    'http_req_duration': ['p(95)<2000'],
    'http_req_failed': ['rate<0.10'],
  },
};

const BASE = __ENV.BASE_URL || 'https://shop.preprod.internal';

export default function () {
  http.get(`${BASE}/api/products/featured`);
  sleep(0.1);  // boucle serrée : un spike maximise la pression
}
```

De dix à cinq cents utilisateurs virtuels en dix secondes, maintenu trois minutes, redescendu, maintenu bas, puis re-spiké. Les seuils sont délibérément plus lâches qu'un baseline ou un stress, parce que la question n'est pas "est-ce que la performance est restée au niveau baseline" mais "est-ce que le système est resté disponible pendant le spike et le second spike".

> ✅ **Bonne pratique** : Lancer le spike contre un système qui est au repos depuis au moins dix minutes avant le début du test. Un spike contre un système chaud n'est pas un spike, c'est un stress. La froideur fait tout l'intérêt du test.

## Zoom : le même test avec NBomber

```csharp
using NBomber.CSharp;
using NBomber.Http;
using NBomber.Http.CSharp;

using var httpClient = new HttpClient { BaseAddress = new Uri("https://shop.preprod.internal") };

var scenario = Scenario.Create("spike_hot_path", async context =>
{
    var request = Http.CreateRequest("GET", "/api/products/featured");
    return await Http.Send(httpClient, request);
})
.WithLoadSimulations(
    // Idle
    Simulation.KeepConstant(copies: 10, during: TimeSpan.FromMinutes(1)),
    // Le spike : rampe 10 -> 500 en 10 secondes
    Simulation.RampingConstant(copies: 500, during: TimeSpan.FromSeconds(10)),
    // Maintien
    Simulation.KeepConstant(copies: 500, during: TimeSpan.FromMinutes(3)),
    // Redescente
    Simulation.RampingConstant(copies: 10, during: TimeSpan.FromSeconds(10)),
    // Récupération
    Simulation.KeepConstant(copies: 10, during: TimeSpan.FromSeconds(30)),
    // Second spike
    Simulation.RampingConstant(copies: 500, during: TimeSpan.FromSeconds(10)),
    Simulation.KeepConstant(copies: 500, during: TimeSpan.FromMinutes(1)),
    Simulation.RampingConstant(copies: 0,  during: TimeSpan.FromSeconds(10))
);

NBomberRunner.RegisterScenarios(scenario)
    .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
    .WithReportFolder("./reports/spike")
    .Run();
```

`RampingConstant` avec une durée de 10 secondes de 10 à 500 utilisateurs virtuels est l'équivalent NBomber du stage spike de k6. Tout le reste est une question de séquencement de phases, que NBomber exprime comme une liste ordonnée d'entrées `LoadSimulation`.

## Zoom : ce qu'il faut regarder pendant un spike

Cinq signaux comptent pendant un spike, et tous ont besoin d'une résolution inférieure à la seconde dans le dashboard pour être lisibles.

**Le temps-jusqu'à-la-première-réponse après le début du spike.** Combien de secondes s'écoulent entre le saut de charge et la première `200 OK` servie sous la nouvelle charge. C'est souvent le chiffre le plus utile à lui seul : il capture le warmup JIT, la croissance du pool de connexions, et la réhydratation du cache dans une seule métrique.

**La courbe de croissance du pool de connexions.** Pour Npgsql ou SqlClient, tracer `pool_in_use` dans le temps. Pendant un spike, le pool devrait grandir rapidement pour suivre la demande. S'il plafonne tôt, le pool a atteint son maximum configuré et l'équipe a trouvé le premier goulet d'étranglement.

**La distribution de latence des requêtes base de données.** Pendant un spike avec un cache froid, la base est le premier endroit à encaisser. Tracer le p95 par seconde de la durée des requêtes. Repérer le moment où il atteint son pic, puis revient au baseline. Le delta est le coût du cache froid.

**Les événements de l'autoscaler.** Si le système tourne sur Kubernetes ou un orchestrateur de containers avec de l'autoscaling, logger le nombre de pods dans le temps. Comparer le moment du scale-up au début du spike. L'écart est la latence de l'autoscaling, et c'est presque toujours plus long que ce que les équipes attendent.

**Le taux d'erreur par endpoint.** Pendant un spike, certains endpoints cassent avant les autres. Tracer le taux d'erreur par endpoint pour identifier lequel est tombé en premier. C'est la prochaine cible à corriger.

```csharp
// Program.cs : exposer les métriques minimales nécessaires à un spike
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddMeter("Microsoft.AspNetCore.Hosting")
        .AddMeter("Microsoft.EntityFrameworkCore")  // durée des requêtes
        .AddMeter("Npgsql")                          // pool_in_use
        .AddRuntimeInstrumentation()                 // GC, thread pool
        .AddPrometheusExporter());
```

> 💡 **Info** : La résolution temporelle par défaut de Grafana est de 15 ou 30 secondes, ce qui est trop grossier pour un spike de 90 secondes. Il faut passer l'intervalle de scrape à 1 seconde et le rafraîchissement du dashboard à 1 seconde pendant les spike tests. Sinon le graphe n'affichera que deux points sur tout le spike et rien ne sera diagnosticable.

## Zoom : les quatre échecs courants sur un spike

**Tempête de cache froid.** Chaque requête tape le cache, chaque lookup rate, chaque miss tape la base, et la base voit 500 requêtes identiques concurrentes. Le correctif est la coalescence de requêtes ou un verrou autour de la réhydratation du cache, pour que seul le premier miss déclenche une requête base pendant que les autres attendent.

**Épuisement du pool de connexions.** Le pool Npgsql par défaut est plafonné à 100 connexions. Une instance qui gère 400 requêtes concurrentes pendant un spike en bloquera 300 en attente d'une connexion. Le correctif est soit un pool plus grand (si la base peut l'encaisser) soit un limiteur de concurrence devant l'endpoint (pour délester plutôt qu'empiler).

**Latence de l'autoscaling.** L'autoscaler est configuré pour ajouter des pods quand le CPU dépasse 70%. Le spike envoie le CPU à 100% en 10 secondes, l'autoscaler réagit en 60 secondes, et le premier nouveau pod est prêt 45 secondes plus tard. Les 90 premières secondes du spike tournent avec la moitié de la capacité nécessaire. Le correctif est le pré-chauffage : faire tourner plus de capacité idle, ou utiliser un autoscaling prédictif, ou pré-scaler avant un événement attendu (solde de minuit).

**Coût de warmup JIT.** Les premiers milliers de requêtes après un démarrage à froid sont servis par du code JIT tier-0, plus lent que le tier-1. Dans un spike, ces premiers milliers de requêtes arrivent en quelques secondes, et leur latence est deux à trois fois celle du baseline. Le correctif est la compilation ReadyToRun (R2R), l'AOT, ou un endpoint de warmup que l'orchestrateur appelle avant de déclarer le pod healthy.

> ⚠️ **Ça marche, mais...** : Un spike qui ne déclenche aucune de ces défaillances au premier run est généralement le signe que le système cible n'est pas configuré comme la prod le sera. Vérifier que le cache est réellement vide, que le pool de la base est à son défaut de prod, et que le compte de répliques correspond au minimum de prod. Sinon le test confirme la mauvaise chose.

> ❌ **Ne jamais faire** : Ne pas lancer un spike test juste après un autre test de charge. Le système est chaud, les pools sont pleins, les caches sont peuplés. Un spike contre un système chaud ne dit rien. Attendre dix minutes d'idle, ou redémarrer la cible.

## Zoom : quand lancer un spike

Les spike tests sont moins routiniers que les baselines mais plus ciblés. Trois déclencheurs :

**Avant un événement de trafic attendu.** Un lancement de produit, une campagne marketing, une intégration externe connue qui va être mise en service. Si l'équipe sait qu'un spike arrive en prod, autant le répéter en pré-prod d'abord.

**Après un changement de topologie de déploiement.** De nouvelles règles d'autoscaling, un type d'instance différent, un nouveau backend de cache, une migration de base. Chacun peut changer le comportement en spike sans apparaître dans un baseline ou un stress.

**Quand un incident de prod dit "le trafic a sauté et on est tombés".** La suite est toujours un spike en pré-prod, avec la forme de trafic exacte de l'incident, et la configuration d'infrastructure exacte de l'incident. L'objectif est de reproduire la défaillance, la corriger, et prouver que le correctif marche.

## Wrap-up

Un spike test est le seul test qui mesure comment un système survit à un saut soudain du calme à l'écrasement. On peut en monter un en k6 ou NBomber en un après-midi, partir d'un état idle (pas d'un état chaud), sauter de bas à haut en moins de trente secondes, maintenir au pic quelques minutes, déclencher optionnellement un second spike pour tester la capacité à encaisser tout de suite après, et surveiller le temps-jusqu'à-la-première-réponse, la croissance des pools, la latence d'autoscaling et le coût du cache froid avec un dashboard à résolution inférieure à la seconde. On peut en sortir en sachant lequel des quatre échecs spike courants le système rencontrerait, et planifier les correctifs (coalescence de requêtes, pools plus grands, pré-chauffage, compilation ReadyToRun) avant que la prochaine campagne marketing ne les rende urgents.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Les Tests de Charge en .NET : vue d'ensemble des quatre types qui comptent](/fr/posts/load-testing-overview/)
- [Le Baseline Testing en .NET : savoir à quoi ressemble la normale](/fr/posts/load-testing-baseline/)
- [Le Soak Testing en .NET : les bugs qui n'apparaissent qu'après des heures](/fr/posts/load-testing-soak/)
- [Le Stress Testing en .NET : trouver le point de rupture et sa forme](/fr/posts/load-testing-stress/)

## Références

- [Guide spike testing k6](https://grafana.com/docs/k6/latest/testing-guides/test-types/spike-testing/)
- [Load simulations NBomber](https://nbomber.com/docs/using-nbomber/load-simulations/)
- [Compilation ReadyToRun, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/core/deploying/ready-to-run)
- [Horizontal Pod Autoscaler dans Kubernetes](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Google SRE book : Handling Overload](https://sre.google/sre-book/handling-overload/)
