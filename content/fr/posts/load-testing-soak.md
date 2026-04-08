---
title: "Le Soak Testing en .NET : les bugs qui n'apparaissent qu'après des heures"
date: 2026-04-08
draft: false
tags: ["load-testing", "soak", "performance", "dotnet"]
categories: ["Testing"]
series: ["Tests de Charge"]
description: "Fuites mémoire, épuisement de pool, croissance des logs, dérive de cache. Les bugs qu'un baseline de 10 minutes ne verra jamais, et qu'un soak test attrape à tous les coups."
---

Hello tous le monde, aujourd'hui on va comprendre le **soak testing**, le type de test qui révèle les bugs invisibles à court terme.

Un système peut passer chaque test unitaire, chaque [test d'intégration](/fr/posts/testing-integration-testing-testcontainers/), chaque [test d'API](/fr/posts/testing-webapplicationfactory/), chaque [test E2E Playwright](/fr/posts/testing-e2e-playwright/), et le [baseline](/fr/posts/load-testing-baseline/), et quand même tomber à 4 heures du matin le troisième jour après le déploiement. Les bugs qui causent ça partagent une signature commune : ils n'apparaissent qu'après des heures d'exécution soutenue. De la mémoire qui fuit d'un kilo-octet par requête. Un pool de connexions qui grimpe de 40 à 99 au long d'un week-end. Un fichier de log qui atteint le quota de disque au sixième jour. Un cache qui dérive parce qu'un événement d'invalidation est occasionnellement perdu sous charge.

Aucun de ces bugs ne se montre dans un baseline de 10 minutes. Tous se montrent dans un soak test. C'est toute la proposition de valeur du soak : faire tourner le système sous une charge modérée et soutenue assez longtemps pour que les bugs dépendants du temps aient une chance d'apparaître.

## Le contexte : pourquoi le soak existe

L'histoire classique sur les incidents de prod soudains est fausse. La plupart des incidents de prod ne sont pas soudains. Ce sont des défaillances lentes qui paraissent soudaines parce que personne ne surveillait la pente. Une croissance mémoire de 2% par jour est invisible sur un graphe qui couvre une heure, et flagrante sur un graphe qui couvre sept jours. Un job de fond qui fuit un thread par run se porte bien à un run par heure et devient catastrophique à dix runs par minute. Ce sont exactement les modes de défaillance qu'un soak est fait pour attraper.

Concrètement, les soak tests répondent à quatre questions qu'aucun autre type de test ne traite :

1. **Est-ce que la mémoire reste stable sous charge soutenue ?** Une vraie fuite mémoire produit un working set qui monte de façon monotone. Un GC qui suit les allocations produit un motif en dents de scie qui reste borné. La différence n'est visible que dans la durée.
2. **Est-ce que les pools de connexions restent sains ?** Pools base de données, pools de clients HTTP, channels gRPC, connexions au broker de messages, tous ont une taille max. Une fuite occasionnelle d'une connexion par heure est invisible à la dixième minute et fatale à la dix-huitième heure.
3. **Est-ce que l'usage disque reste borné ?** Logs, fichiers temporaires, dead-letter queues, tables de jobs en échec. N'importe lequel peut grandir sans borne si la rotation, le pruning ou le nettoyage est cassé.
4. **Est-ce que les caches, les files et l'état des tâches de fond restent cohérents ?** L'invalidation de cache sous écritures concurrentes, la profondeur de file selon la vitesse du consommateur, les jobs planifiés qui ne nettoient pas derrière eux, tout cela dérive avec le temps et ne se révèle qu'après des heures.

## Vue d'ensemble : la forme d'un run soak

{{< mermaid >}}
graph TD
    A[Charge modérée<br/>50-70% du baseline] --> B[Durée<br/>4 à 24 heures]
    B --> C[Métriques en continu]
    C --> D1[Croissance<br/>working set]
    C --> D2[Tailles heap GC<br/>gen0/1/2]
    C --> D3[Temps d'attente pools<br/>DB, HTTP, threads]
    C --> D4[Usage disque<br/>logs, tmp]
    C --> D5[Latence<br/>dérive dans le temps]
{{< /mermaid >}}

Un soak n'est pas un test de débit de pointe. La charge est délibérément modérée, en général 50 à 70 pour cent de ce que le baseline établit comme normal, pour que le système ait de la marge pour le vrai travail et que le test stresse la durée, pas l'intensité. La durée est la variable : quatre heures pour un premier run, une nuit entière pour une validation pré-release, plusieurs jours pour un changement de plateforme (upgrade EF Core, upgrade runtime, migration d'infrastructure).

La sortie d'un soak n'est pas un seul chiffre. C'est un jeu de graphes time-series qui montrent comment les métriques évoluent pendant le run. Un run qui rapporte "p95 moyen à 120 ms" et rien d'autre est un soak raté, parce que la moyenne ne dit rien sur le fait que la latence est peut-être passée de 90 ms à 160 ms sur la fenêtre, ce qui est la vraie question.

## Zoom : configuration soak avec k6

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 30 },     // warmup
    { duration: '8h', target: 30 },     // soak à 30 VUs (~60% du baseline)
    { duration: '1m', target: 0 },      // cooldown
  ],
  thresholds: {
    // La fenêtre glissante : le soak échoue si *n'importe quelle* heure dégrade.
    'http_req_duration': ['p(95)<400'],
    'http_req_failed': ['rate<0.01'],
  },
  ext: {
    loadimpact: { projectID: 0 },
  },
};

const BASE = __ENV.BASE_URL || 'https://shop.preprod.internal';

export default function () {
  group('catalog', () => {
    http.get(`${BASE}/api/products?page=1&size=20`);
  });

  if (Math.random() < 0.2) {
    group('write', () => {
      http.post(`${BASE}/api/cart`, JSON.stringify({
        productId: `SKU-${Math.floor(Math.random() * 1000)}`,
        quantity: 1,
      }), { headers: { 'Content-Type': 'application/json' } });
    });
  }

  sleep(2);
}
```

Huit heures, trente utilisateurs virtuels, charge modérée. Le `sleep(2)` entre requêtes est délibéré : un soak n'est pas fait pour maximiser le débit, il est fait pour maintenir le système sous une pression continue et réaliste sur une longue période.

> ✅ **Bonne pratique** : Lancer le soak avec les résultats streamés en direct vers Grafana (ou n'importe quel dashboard). Le moment le plus utile d'un soak n'est pas la fin, c'est le moment où on remarque que la pente change. Attendre le rapport final supprime l'intérêt du test.

## Zoom : ce qu'il faut regarder pendant le run

Le générateur de charge capture les métriques au niveau requête. Le vrai signal vit côté application. Pour un système .NET, le dashboard minimum pendant un run soak affiche :

**Le working set et le heap GC dans le temps.** La métrique `process.runtime.dotnet.gc.heap.size`, ventilée par génération, tracée en fonction de l'horloge. Un système sain montre un motif stable ou en dents de scie. Une fuite montre une tendance qui monte et ne redescend jamais, même après une collecte gen2.

**Les métriques du pool de connexions base de données.** Les compteurs `pool_wait_time` et `pool_in_use` de Npgsql, SqlClient, ou le provider utilisé. Un pool qui commence à 10 en usage et grimpe à 90 en six heures a une fuite de connexion quelque part, et le soak est le test qui l'attrape.

**La longueur de file du thread pool.** Les compteurs `System.Runtime` exposent `threadpool-queue-length` et `threadpool-thread-count`. Une file qui grandit sans borne signifie que le travail arrive plus vite que les threads ne peuvent le traiter, en général à cause d'un pattern sync-over-async qui n'est visible que sous charge soutenue.

**La distribution de latence, dans le temps, pas en moyenne.** Un heatmap Grafana de `http_server_request_duration` par endpoint dit si le p95 est stable ou dérive vers le haut. La dérive, si elle existe, est le bug.

**L'usage disque sur l'hôte.** Un simple `df` remonté chaque minute attrape les échecs de rotation de logs, les fuites de fichiers temporaires, et le gonflement des dead-letter queues avant qu'ils ne fassent tomber le process.

```csharp
// Program.cs : exposer les métriques dont un soak a besoin
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics
            .AddMeter("Microsoft.AspNetCore.Hosting")
            .AddMeter("Microsoft.AspNetCore.Http.Connections")
            .AddMeter("System.Net.Http")
            .AddMeter("Microsoft.EntityFrameworkCore")
            .AddRuntimeInstrumentation()   // GC, thread pool, contention
            .AddProcessInstrumentation()   // CPU, mémoire, handles
            .AddPrometheusExporter();
    });
```

> 💡 **Info** : `AddRuntimeInstrumentation` vient du package NuGet `OpenTelemetry.Instrumentation.Runtime` et c'est la ligne unique la plus utile qu'une équipe .NET puisse ajouter à un système testable en soak. Elle expose les tailles de heap GC, la longueur de file du thread pool, et la contention de verrous sans une seule ligne de code custom.

## Zoom : lire les résultats

Un soak produit trois résultats typiques.

**Plat et stable.** Toutes les métriques restent dans leur bande de départ pendant toute la durée. La latence fait des dents de scie, le GC récupère, les pools restent stables, l'usage disque reste plat. Le soak passe, et l'équipe a la preuve que le système peut tourner aussi longtemps que la durée du test.

**Dérive graduelle.** La latence monte lentement, la mémoire tend vers le haut, ou un pool grandit. C'est le cas diagnostique pour lequel le soak existe. L'équipe regarde la pente et se demande : "à ce rythme, quand est-ce qu'on atteint la limite ?". Une fuite linéaire de 50 Mo par heure, sur une machine 16 Go, donne à peu près deux semaines. Une dérive sous-linéaire peut encore être acceptable. Une dérive super-linéaire est une alerte rouge, parce qu'elle ne se contente pas de doubler le temps avant l'échec quand la charge double, elle échoue beaucoup plus vite.

**Falaise.** Tout a l'air normal pendant six heures, puis un pool s'épuise, un disjoncteur se déclenche, ou le process tombe en OOM. Le moment de la falaise est une information utile : il dit où se cache la limite cachée et donne à l'équipe une cible concrète à corriger.

> ⚠️ **Ça marche, mais...** : Un soak qui ne montre aucune dérive sur 8 heures n'est pas une preuve que le système peut tourner 8 jours. La couverture de durée croît de façon non linéaire : les crons hebdomadaires, les batchs mensuels, et les motifs de charge saisonniers ne seront exercés que par des runs plus longs. Le soak est un signal de confiance, pas une garantie.

> ❌ **Ne jamais faire** : Ne pas lancer un soak et ne rapporter que le chiffre final. Un p95 de latence moyenné sur 8 heures cache toute l'histoire. L'histoire est dans le graphe time-series. Si le rapport n'inclut pas de graphe, le rapport est incomplet.

## Zoom : quand lancer un soak

Les soak coûtent cher en temps écoulé, même s'ils coûtent peu en compute. Trois cadences couvrent la plupart des équipes :

**Avant chaque upgrade de plateforme.** Un upgrade du runtime .NET, un changement de version majeure d'EF Core, une migration de cluster Kubernetes, un changement de version du moteur de base : chacun justifie un soak d'une nuit entière avant le rollout en prod. C'est là que se cachent les bugs à plus forte valeur.

**Hebdomadaire, planifié.** Un soak de 8 heures une fois par semaine, qui tourne du samedi soir au dimanche matin, attrape les régressions accumulées pendant la semaine et établit un baseline roulant pour le comportement de longue durée.

**Sur suspicion.** Quand un incident de prod contient "dégradation lente sur plusieurs heures" dans son post-mortem, la suite est presque toujours un soak conçu pour reproduire la dégradation en pré-prod, avec le composant fautif instrumenté plus finement que d'habitude.

## Quand le soak est le mauvais outil

Les soak tests sont la bonne réponse pour les modes de défaillance dépendants du temps. Ils sont la mauvaise réponse pour :

- **Les questions de débit de pointe** : c'est un test de stress.
- **La gestion des bursts** : c'est un test de spike.
- **La correction logique sous concurrence** : c'est un test d'intégration avec des workers parallèles, ou une chasse aux race conditions, pas un soak.
- **Trouver le point de rupture** : les tests de stress le trouvent, les soak ne poussent pas assez fort pour l'atteindre.

Lancer un soak pour répondre à une question de stress, c'est attendre huit heures pour une conclusion qu'un test de stress d'une heure aurait donnée.

## Wrap-up

Un soak révèle les bugs qui vivent dans l'écart entre un run sain de dix minutes et un déploiement de prod de plusieurs jours. On peut en monter un en k6 ou NBomber en un après-midi, garder la charge modérée (50 à 70 pour cent du baseline), le faire tourner pendant quatre à vingt-quatre heures contre un environnement de pré-prod réaliste, et regarder les métriques time-series en direct plutôt que d'attendre le rapport final. On peut attraper des pools de connexions qui fuient, des caches qui dérivent, des fichiers de log qui grossissent, et des fuites mémoire linéaires avant qu'ils ne deviennent un incident de prod, et distinguer la dérive graduelle, la falaise et le comportement plat stable à la forme du graphe.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Les Tests de Charge en .NET : vue d'ensemble des quatre types qui comptent](/fr/posts/load-testing-overview/)
- [Le Baseline Testing en .NET : savoir à quoi ressemble la normale](/fr/posts/load-testing-baseline/)
- [Tests API avec WebApplicationFactory en ASP.NET Core](/fr/posts/testing-webapplicationfactory/)

## Références

- [Stages et scénarios k6](https://grafana.com/docs/k6/latest/using-k6/scenarios/)
- [OpenTelemetry Runtime Instrumentation pour .NET](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/tree/main/src/OpenTelemetry.Instrumentation.Runtime)
- [Métriques ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/log-mon/metrics/metrics)
- [Fondamentaux du garbage collection .NET, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/standard/garbage-collection/fundamentals)
