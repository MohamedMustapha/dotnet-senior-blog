---
title: "Diagnostic : comment enquêter et résoudre les problèmes de performance de la base"
date: 2026-04-09
draft: false
tags: ["database", "performance", "dotnet"]
categories: ["Database"]
series: ["Database"]
description: "Une méthode reproductible pour les incidents de performance base de données : du symptôme à la cause racine puis au correctif. Plans d'exécution, wait types, index manquants, et les pièges côté .NET qui se font passer pour des problèmes de base."
---

Hello tous le monde, aujourd'hui on va démystifier **le diagnostic d'un problème de performance base de données**, de A à Z.

Les incidents de performance base de données, c'est la catégorie où des ingénieurs expérimentés continuent de dégainer des hypothèses. "C'est sûrement un index manquant." "Ça doit être un mauvais plan d'exécution." "Le pool de connexions est plein." Parfois ils tombent juste du premier coup ; la plupart du temps ils remplacent une hypothèse par une autre jusqu'à ce que les symptômes bougent. Le problème n'est pas la connaissance, c'est la méthode. Un flow de diagnostic reproductible bat une bonne hypothèse à chaque fois, parce qu'il converge sur la vraie cause en minutes plutôt qu'en heures, et parce qu'il produit des preuves qu'on peut attacher au post-mortem.

Cet article pose cette méthode : comment trier le symptôme, comment confirmer le vrai goulot avant de toucher à quoi que ce soit, comment lire un plan d'exécution sans s'y perdre, et les patterns côté .NET qui ressemblent à des problèmes de base mais ne le sont pas.

## Le contexte : pourquoi le diagnostic base demande une méthode

La base de données est une boîte noire avec une interface publique, et cette interface ment sur la cause de la latence. Un endpoint lent mesuré depuis le navigateur peut être la base, le réseau, la sérialisation, la taille de la réponse JSON, un `foreach` qui await une requête par itération, ou une pause GC sur le serveur applicatif. Une requête qui tourne en 200 ms dans SSMS et en 2 secondes depuis l'appli peut être du parameter sniffing, une option de chaîne de connexion différente, une collation différente, ou l'application qui ouvre une seconde transaction que la requête doit attendre. Chacun de ces cas a un correctif différent, et choisir la mauvaise hypothèse gâche les trente premières minutes de l'incident.

La méthode existe pour que ces trente premières minutes produisent des preuves plutôt que des théories.

## Vue d'ensemble : le flow de diagnostic

{{< mermaid >}}
flowchart TD
    A[Symptôme : endpoint lent ou timeout] --> B[Étape 1 : mesurer où va le temps]
    B --> C{Goulot ?}
    C -->|Côté .NET| D[Profiler l'appli : GC, sérialisation, N+1]
    C -->|Côté base| E[Étape 2 : capturer la requête]
    E --> F[Étape 3 : plan d'exécution]
    F --> G{Problème de plan ?}
    G -->|Index manquant| H[Ajouter l'index]
    G -->|Mauvaise estimation| I[Update statistics / réécrire]
    G -->|Plan OK| J[Étape 4 : wait types]
    J --> K{Il attend quoi ?}
    K -->|Locks| L[Chaîne de blocage]
    K -->|IO| M[Stockage / taille des données]
    K -->|CPU| N[Requête chaude ou régression]
{{< /mermaid >}}

Quatre étapes, dans l'ordre, à chaque fois. Sauter l'étape 1, c'est la raison pour laquelle une équipe passe une heure à tuner une requête qui n'était pas le goulot.

## Étape 1 : mesurer où va le temps

Avant d'ouvrir SSMS, on confirme que l'endpoint lent est vraiment lent à cause de la base. Avec de l'instrumentation OpenTelemetry, la réponse est dans la trace :

```csharp
services.AddOpenTelemetry()
        .WithTracing(t => t
            .AddAspNetCoreInstrumentation()
            .AddEntityFrameworkCoreInstrumentation(o => o.SetDbStatementForText = true)
            .AddNpgsql()
            .AddOtlpExporter());
```

Un span de requête avec ses spans enfants dit immédiatement si les 2 secondes de latence sont 1,9 seconde de base sur une seule requête, 2 secondes de 400 requêtes à 5 ms chacune (N+1), ou 100 ms de base et 1,9 seconde de quelque chose d'autre. Si la somme des spans base représente une petite fraction de la requête, le goulot est côté .NET et aucun tuning de requête n'aidera.

Sans traçage, la mesure minimale viable est un chrono autour de l'appel base et un compteur de requêtes par request :

```csharp
var sw = Stopwatch.StartNew();
var count = 0;
_db.Database.SetCommandInterceptor(new CountingInterceptor(() => count++));

var result = await handler.HandleAsync(request, ct);
_logger.LogInformation("Request {Route} took {Elapsed} ms with {Count} queries",
    HttpContext.Request.Path, sw.ElapsedMilliseconds, count);
```

"400 requêtes dans une seule request", c'est déjà un diagnostic avant d'en avoir lu une seule.

> 💡 **Info** — Depuis EF Core 7, `DbCommandInterceptor` donne un hook propre pour compter les commandes sans toucher au LINQ. On garde l'intercepteur en développement uniquement ; en production, le compteur de spans OpenTelemetry porte la même information.

## Étape 2 : capturer la requête exacte

Une fois qu'on sait que la base est le goulot, on a besoin du SQL exact, avec les vraies valeurs de paramètres, pour le reproduire hors de l'appli. `LogTo` d'EF Core ou le span OpenTelemetry donne ça. On copie la requête à l'identique. On ne la réécrit pas "pour plus de clarté" : les décisions de tuning dépendent du SQL littéral que le driver a envoyé.

```sql
SELECT o.id, o.reference, o.total_amount, c.email
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'completed'
  AND o.created_at > '2026-01-01'
ORDER BY o.created_at DESC
LIMIT 50;
```

On la fait tourner sur une copie de production. Une vraie copie, pas la donnée de seed dev. La majorité des requêtes lentes le sont à cause de la forme des données : la base dev a 500 lignes et un hash join ; la production a 40 millions de lignes et un nested loop join sur un index manquant. On ne diagnostique pas ça sur une table de 500 lignes.

## Étape 3 : lire le plan d'exécution

Toute base relationnelle expose le plan d'exécution. Sur PostgreSQL :

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT ... ;
```

Sur SQL Server :

```sql
SET STATISTICS IO, TIME ON;
-- ou activer "Include Actual Execution Plan" dans SSMS
SELECT ... ;
```

On lit le plan pour trois choses, dans l'ordre :

**1. Où va vraiment le temps ?** Le plan est un arbre d'opérations ; chaque nœud rapporte son propre coût et son propre nombre de lignes. Le nœud coûteux est celui sur lequel on met son attention. Ce n'est souvent pas là où on s'attendait.

**2. L'estimation de lignes est-elle loin de la réalité ?** Un écart de 100x entre estimé et réel signale presque toujours un mauvais choix du planificateur : un nested loop là où un hash join aurait été meilleur, un tri qui a débordé sur disque, un bitmap qui n'a pas aidé. La correction est en général `UPDATE STATISTICS`, ou une réécriture du prédicat d'une forme que le planificateur sait estimer.

**3. Y a-t-il un scan séquentiel sur une grosse table ?** Un scan séquentiel sur 40 millions de lignes, c'est la signature classique de "index manquant". Ce n'est pas toujours faux : si la requête ramène 30% des lignes, le scan séquentiel est correct. Mais si la requête ramène 50 lignes et que le plan en scanne 40 millions, il manque un index.

```sql
-- La correction pour la requête d'exemple, sur PostgreSQL :
CREATE INDEX idx_orders_status_created_at
    ON orders (status, created_at DESC)
    WHERE status = 'completed';
```

Un index partiel sur les lignes qui matchent le prédicat le plus fréquent est plus petit, plus rapide à maintenir, et plus sélectif qu'un index complet. La plupart des dashboards filtrent sur un ou deux statuts ; un index partiel épouse exactement ce pattern d'accès.

> ✅ **Bonne pratique** — Ajouter le plan d'exécution (le "avant" et le "après") à la pull request qui introduit l'index. Le toi du futur, en relisant le commit dans six mois, voudra savoir pourquoi cet index précis existe.

> ⚠️ **Ça marche, mais...** — Ajouter un index à partir du missing-index hint sans lire le plan fonctionne assez souvent pour être tentant. Ça produit aussi des index en doublon, des ralentissements côté écriture et des index qui couvrent des requêtes que plus personne ne lance. On lit le plan ; on ajoute l'index dont on a besoin.

## Étape 4 : wait types

Quand le plan est bon et que la requête est toujours lente, la base attend quelque chose. Chaque SGBD majeur expose les wait types :

```sql
-- PostgreSQL
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE state = 'active';

-- SQL Server
SELECT session_id, wait_type, wait_time, blocking_session_id, last_wait_type
FROM sys.dm_exec_requests
WHERE session_id > 50;
```

Trois catégories d'attente couvrent la quasi-totalité des cas :

- **Lock waits** : la requête est bloquée par une autre transaction. La correction est en amont : transactions plus courtes, niveau d'isolation différent, ou découpage de la requête bloquante en plus petites.
- **IO waits** : la requête attend le stockage. La correction est en général une question de forme de données (lignes plus petites, meilleure compression, archivage du vieux) ou d'infrastructure (disques plus rapides, plus de mémoire pour que la donnée chaude tienne en cache).
- **CPU** : pas d'attente, la requête calcule. La correction est une réécriture ou un index couvrant.

> 💡 **Info** — `pg_stat_activity` et `sys.dm_exec_requests` sont des vues point-in-time. Pour les incidents récurrents, on installe un sampler qui en prend un snapshot toutes les 5 secondes dans un log, pour que le prochain incident ait un historique.

## Zoom : les pièges côté .NET qui ressemblent à un problème de base

La moitié des incidents "la base est lente" vus en 15 ans n'étaient pas des incidents base. Les classiques :

**Épuisement du pool de connexions.** Quand toutes les connexions sont prises, la requête suivante bloque jusqu'à la libération d'une. Ça ressemble à une requête lente dans le log applicatif, mais la base montre la requête comme "4 ms une fois démarrée". La correction, c'est trouver la fuite de connexion (en général un `DbContext` gardé plus longtemps qu'une request, ou un `await using` oublié), pas tuner la requête.

**`Task.Run` autour d'un appel EF Core synchrone.** Lancer une requête sync sur un thread du pool a l'air correct jusqu'à ce que le thread pool sature, moment où chaque requête commence à faire la queue sur le thread pool avant même d'atteindre la base. On convertit en async, toujours.

**Un `foreach` qui await une requête par itération.** La latence totale est `N * query_time`, ce qui ressemble à une requête lente jusqu'à ce qu'on réalise qu'il y en a 400. La correction est le pattern projection/include de [EF Core : Optimisation des Lectures](/fr/posts/database-efcore-read-optimization/).

**La sérialisation qui domine la réponse.** Une réponse JSON de 20 Mo prend une seconde à sérialiser, quel que soit l'état de la base. Le span base dans la trace est à 20 ms ; le temps de réponse est à 1200 ms. La correction, c'est la pagination.

> ❌ **Ne jamais faire** — Ne pas "corriger" un goulot côté .NET en ajoutant un index en base. L'index n'aidera pas et le chemin d'écriture ralentira pour tout le monde.

## Wrap-up

Tu sais maintenant la méthode : quatre étapes, dans l'ordre. Mesurer où va le temps, capturer la requête exacte, lire le plan d'exécution, et si le plan est bon, regarder les wait types. Ajouter un index quand le plan le dit, pas quand le ticket d'incident dit "sûrement un index manquant". Et toujours confirmer que la base est le goulot avant d'y toucher, parce que la moitié des incidents "la base est lente" sont des problèmes côté .NET déguisés. Avec ce flow, les incidents cessent d'être du deviner pour devenir une checklist.

Prêt à sortir cette méthode au prochain ticket de perf, ou à la partager avec ton équipe pour que le prochain incident commence par une mesure plutôt qu'une hypothèse ?

À la prochaine, a++ 👋

## Pour aller plus loin

- [EF Core : Optimisation des Lectures](/fr/posts/database-efcore-read-optimization/)
- [Dapper : quand et comment l'utiliser](/fr/posts/database-dapper/)
- [EF Core : les Migrations en pratique](/fr/posts/database-efcore-migrations/)

## Références

- [PostgreSQL : Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL : pg_stat_activity](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ACTIVITY-VIEW)
- [SQL Server : Plans d'exécution](https://learn.microsoft.com/fr-fr/sql/relational-databases/performance/execution-plans)
- [SQL Server : Wait Statistics](https://learn.microsoft.com/fr-fr/sql/relational-databases/system-dynamic-management-views/sys-dm-os-wait-stats-transact-sql)
- [EF Core : Logging et diagnostics](https://learn.microsoft.com/fr-fr/ef/core/logging-events-diagnostics/)
- [OpenTelemetry .NET : instrumentation EF Core](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/tree/main/src/OpenTelemetry.Instrumentation.EntityFrameworkCore)
