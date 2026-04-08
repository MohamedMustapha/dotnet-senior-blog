---
title: "Zero Allocation en .NET : quand le GC devient le goulet d'étranglement"
date: 2026-04-08
draft: false
tags: ["performance", "dotnet", "memory"]
categories: ["Performance"]
series: ["Performance"]
description: "Span, Memory, stackalloc, ArrayPool, ValueTask, et les patterns qui permettent au code .NET des chemins chauds de tourner sans produire de déchets. Pas pour toutes les méthodes, mais décisif là où ça compte."
---

Hello tous le monde, aujourd'hui on va explorer le **zero allocation** en .NET, et les techniques qui permettent de sortir du rôle passif de "on laisse le GC faire" quand la performance le demande.

Pour 95% du code .NET, le garbage collector est un aide silencieux auquel personne ne pense. Les allocations ont lieu, la mémoire est récupérée, le programme continue. Pour les 5% restants, les chemins chauds d'un système à fort débit, le garbage collector *est* le goulet d'étranglement, et chaque octet alloué par requête devient une requête par seconde que le système ne pourra pas servir. La différence entre ces deux mondes n'est pas la qualité du code, c'est la fréquence : à 100 000 requêtes par seconde, une seule allocation de 1 Ko par requête devient 100 Mo par seconde de pression sur le heap, et le GC se met à tourner en continu pour suivre.

La programmation zero-allocation est l'ensemble des techniques qui permettent au code sensible à la performance d'éviter de produire des déchets sur les chemins chauds. Ce n'est pas un style à appliquer partout. C'est une boîte à outils à sortir quand un [test de stress](/fr/posts/load-testing-stress/) ou un [soak](/fr/posts/load-testing-soak/) montre que les temps de pause GC, la fréquence des collectes gen0, ou la pression sur le heap plafonnent le débit. Utilisé aux bons endroits, cela peut doubler ou tripler la capacité d'un système sans rien changer d'autre.

## Le contexte : pourquoi le zero allocation compte

Le garbage collector .NET est générationnel. Les objets démarrent en gen0, passent en gen1 s'ils vivent assez longtemps, et atteignent gen2 s'ils survivent à deux collectes. Collecter gen0 est peu coûteux (quelques centaines de microsecondes), gen1 est plus cher, gen2 est celui qui produit des pauses applicatives visibles et peut prendre des dizaines de millisecondes. Un système bien élevé garde la plupart de ses allocations en gen0, où la collecte est presque gratuite.

Le problème est que "presque gratuit" n'est pas gratuit. Chaque collecte gen0 arrête les threads managés (en server GC mode, brièvement), mesure les racines, compacte la jeune génération, et reprend. À 100 000 requêtes par seconde, si chaque requête alloue 2 Ko, gen0 se remplit en millisecondes, et le GC tourne plusieurs fois par seconde. Chaque passe introduit du jitter, des pics de latence, et de la contention avec le vrai travail.

Le code zero-allocation change cette équation. Au lieu d'allouer à chaque opération, il réutilise des buffers, utilise la pile pour les données temporaires, et garde le heap managé au calme. Les objectifs sont concrets :

1. **Une latence en queue stable**, parce que moins de pauses GC signifie moins de pics de latence au p99 et au p99.9.
2. **Un débit plus élevé**, parce que le CPU passe moins de temps à collecter et plus de temps à faire tourner le code applicatif.
3. **Moins de pression mémoire**, parce que le working set reste borné et que le système peut tasser plus d'instances par hôte.
4. **Un comportement prévisible sous charge**, parce que le GC n'est plus l'une des pièces mobiles dont le coût croît avec le trafic.

## Vue d'ensemble : la pyramide des allocations

{{< mermaid >}}
graph TD
    A[Types valeur sur la pile<br/>gratuit] --> B[Span&lt;T&gt; sur stackalloc<br/>gratuit]
    B --> C[ArrayPool&lt;T&gt;<br/>réutilisé, pas alloué]
    C --> D[Objets poolés<br/>ObjectPool&lt;T&gt;]
    D --> E[Heap Gen0<br/>peu cher, mais pas gratuit]
    E --> F[LOH / Gen2<br/>cher, à éviter]
{{< /mermaid >}}

Toutes les allocations ne se valent pas. La pyramide ci-dessus classe les options du moins cher au plus cher. Le principe directeur est simple : sur un chemin chaud, essayer de rester le plus haut possible dans la pyramide. Si la donnée tient sur la pile, la mettre sur la pile. Si elle ne tient pas, la louer à un pool. Si ni l'un ni l'autre ne marche, au moins garder l'allocation en gen0 et hors du Large Object Heap.

Cet article couvre quatre techniques dans cette pyramide, chacune s'appliquant à une situation précise.

## Zoom : `Span<T>` et `stackalloc`

`Span<T>` a été introduit dans .NET Core 2.1 comme l'abstraction canonique au-dessus de la mémoire contiguë. Il peut pointer vers un tableau managé, vers un pointeur natif, vers une portion de `string`, ou vers de la mémoire allouée sur la pile, avec la même API. Combiné à `stackalloc`, il permet des buffers zero-allocation pour les opérations de courte durée.

```csharp
public static bool IsValidIban(ReadOnlySpan<char> iban)
{
    if (iban.Length < 15 || iban.Length > 34) return false;

    // Buffer sur la pile, aucune allocation heap.
    Span<char> rearranged = stackalloc char[iban.Length];
    iban[4..].CopyTo(rearranged);
    iban[..4].CopyTo(rearranged[^4..]);

    // Conversion en chiffres, validation modulo 97.
    Span<byte> digits = stackalloc byte[rearranged.Length * 2];
    int digitCount = 0;
    foreach (char c in rearranged)
    {
        if (char.IsDigit(c))
            digits[digitCount++] = (byte)(c - '0');
        else if (c is >= 'A' and <= 'Z')
        {
            int value = c - 'A' + 10;
            digits[digitCount++] = (byte)(value / 10);
            digits[digitCount++] = (byte)(value % 10);
        }
        else return false;
    }

    int remainder = 0;
    for (int i = 0; i < digitCount; i++)
        remainder = (remainder * 10 + digits[i]) % 97;

    return remainder == 1;
}
```

Cette méthode valide un IBAN sans allouer un seul octet sur le heap. Les buffers `stackalloc` vivent dans la frame de pile courante et sont récupérés automatiquement au retour de la méthode. L'appelant passe un `ReadOnlySpan<char>`, qui peut venir d'un `string`, d'un body de requête parsé, ou d'un autre span, sans coût d'allocation.

> 💡 **Info** : `stackalloc` est sûr à l'intérieur d'une méthode qui ne stocke pas le span résultant dans un champ et ne le retourne pas. Le compilateur le garantit via les règles `ref struct` de `Span<T>`. La taille du buffer sur la pile doit rester sous environ 1 Ko pour éviter le risque de `StackOverflowException`. Pour des buffers plus grands, passer à `ArrayPool<T>`.

> ✅ **Bonne pratique** : Accepter `ReadOnlySpan<char>` ou `ReadOnlySpan<byte>` en paramètre de méthode plutôt que `string` ou `byte[]`. Les appelants peuvent passer des tranches de données existantes sans copie, et la méthode gagne un comportement zero-allocation par défaut.

## Zoom : `ArrayPool<T>` pour les buffers loués

Quand le buffer nécessaire est plus grand que ce qu'une allocation sur la pile devrait gérer (disons, 4 Ko ou plus), `ArrayPool<T>.Shared` fournit un pool managé de tableaux réutilisables. Louer un tableau au pool est bien moins cher que d'en allouer un neuf, et le rendre le rend disponible pour le prochain appelant.

```csharp
public static async Task<int> ReadAllToCountAsync(Stream input, CancellationToken ct)
{
    // Location d'un buffer 16 Ko au pool partagé. Aucune allocation heap pour ce buffer.
    byte[] buffer = ArrayPool<byte>.Shared.Rent(16 * 1024);
    try
    {
        int total = 0;
        int read;
        while ((read = await input.ReadAsync(buffer, ct)) > 0)
            total += read;
        return total;
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

Le `try/finally` n'est pas négociable. Louer sans rendre fuite un buffer du pool, ce qui en réduit silencieusement l'efficacité. Le pattern standard est toujours `rent → try → use → finally → return`.

> ⚠️ **Ça marche, mais...** : Si le buffer peut contenir des données sensibles (tokens, informations personnelles), appeler `ArrayPool<T>.Shared.Return(buffer, clearArray: true)` pour zéroter la mémoire avant qu'elle ne retourne au pool. Sinon la prochaine location voit le contenu précédent. Le coût de nettoyage d'un buffer 16 Ko est négligeable face aux conséquences de sécurité de ne pas le nettoyer.

## Zoom : `ValueTask` pour le cas courant du "déjà terminé"

Chaque méthode `async` qui retourne `Task` alloue au moins un objet `Task`, plus un box de state machine si la méthode yield réellement. Pour les méthodes qui retournent souvent synchroniquement (la valeur en cache, la collection vide, le retour précoce sur une garde), cette allocation est du pur gaspillage.

`ValueTask<T>` a été ajouté dans .NET Core 2.0 précisément pour ce cas. C'est un type valeur qui peut représenter soit un résultat terminé inline (zéro allocation), soit une tâche sous-jacente (allocation normale). Utilisé correctement, il supprime les allocations pour les 80% des appels qui se terminent synchroniquement.

```csharp
public sealed class PriceCache
{
    private readonly IDistributedCache _cache;
    private readonly IPriceRepository _repo;
    private readonly ConcurrentDictionary<string, decimal> _local = new();

    public ValueTask<decimal> GetPriceAsync(string sku, CancellationToken ct)
    {
        // Chemin chaud : déjà en cache local, pas d'async, pas d'allocation.
        if (_local.TryGetValue(sku, out var price))
            return new ValueTask<decimal>(price);

        // Chemin froid : cache plus lent, vrai await, vraie allocation.
        return new ValueTask<decimal>(FetchAsync(sku, ct));
    }

    private async Task<decimal> FetchAsync(string sku, CancellationToken ct)
    {
        var bytes = await _cache.GetAsync(sku, ct);
        if (bytes is not null)
        {
            var cached = BitConverter.ToDecimal(bytes);
            _local[sku] = cached;
            return cached;
        }

        var fresh = await _repo.GetPriceAsync(sku, ct);
        _local[sku] = fresh;
        return fresh;
    }
}
```

Dans un service de pricing typique où le cache local hit 95% du temps, ce pattern supprime 95% des allocations `Task<decimal>`. À 100 000 requêtes par seconde, cela fait 95 000 allocations économisées par seconde, auxquelles s'ajoutent celles du box de state machine qui n'ont plus lieu.

> ❌ **Ne jamais faire** : Ne pas `await` un `ValueTask` deux fois, ne pas le stocker dans un champ, ne pas appeler `.Result` sur un qui n'est pas encore terminé. `ValueTask` est optimisé pour une consommation avec un seul await, et une mauvaise utilisation peut corrompre l'objet sous-jacent ou causer des hangs. Le pattern sûr est `await ValueTaskMethod();` une seule fois, sur le site d'appel.

## Zoom : objets poolés avec `ObjectPool<T>`

Pour les objets plus complexes qu'un buffer (un `StringBuilder`, un état de parser custom, un contexte de requête), `Microsoft.Extensions.ObjectPool` fournit un pool que les applications peuvent utiliser directement. C'est le même mécanisme qu'ASP.NET Core utilise en interne pour la réutilisation de `StringBuilder` dans le pipeline.

```csharp
public sealed class ReportFormatter
{
    private readonly ObjectPool<StringBuilder> _builderPool;

    public ReportFormatter(ObjectPoolProvider provider)
    {
        _builderPool = provider.Create(
            new StringBuilderPooledObjectPolicy { MaximumRetainedCapacity = 16 * 1024 });
    }

    public string Format(Order order)
    {
        var sb = _builderPool.Get();
        try
        {
            sb.Append("Order ").Append(order.Id).Append(": ");
            foreach (var line in order.Lines)
                sb.Append(line.ProductName).Append(' ').Append(line.Quantity).Append(", ");

            return sb.ToString();
        }
        finally
        {
            _builderPool.Return(sb);  // la policy Return nettoie le builder
        }
    }
}
```

Le même pattern `rent → try → finally → return` que `ArrayPool`, avec une policy dédiée qui borne la capacité retenue. Le réglage `MaximumRetainedCapacity` compte : un pool qui garde des `StringBuilder` de taille arbitraire trahit son propre intérêt en retenant indéfiniment la mémoire du pire cas.

> 💡 **Info** : `ObjectPoolProvider` est enregistré par défaut dans ASP.NET Core via `services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>()` (ASP.NET Core le fait automatiquement). Pour une application console ou un worker, il faut l'enregistrer explicitement.

## Zoom : mesurer le gain

Le code zero-allocation ne compte que s'il économise réellement des allocations. La seule façon fiable de le vérifier est BenchmarkDotNet avec l'attribut `[MemoryDiagnoser]`. Il rapporte les allocations par opération, ventilées par génération, à côté du runtime.

```csharp
[MemoryDiagnoser]
public class IbanValidationBench
{
    private static readonly string Iban = "FR7630006000011234567890189";

    [Benchmark(Baseline = true)]
    public bool Naive()
    {
        var sb = new StringBuilder();
        sb.Append(Iban.AsSpan(4));
        sb.Append(Iban.AsSpan(0, 4));
        var rearranged = sb.ToString();
        return Validate(rearranged);
    }

    [Benchmark]
    public bool ZeroAlloc() => IsValidIban(Iban.AsSpan());

    private static bool Validate(string s) { /* ... */ return true; }
}
```

Une sortie BenchmarkDotNet typique pour cette comparaison ressemble à :

```
|     Method |      Mean |   Allocated |
|----------- |----------:|------------:|
|      Naive | 412.7 ns  |       216 B |
|  ZeroAlloc |  89.3 ns  |         0 B |
```

Quatre à cinq fois plus rapide, zéro octet alloué, et la différence est directement attribuable à la pression GC qui n'a plus lieu. Sans le benchmark, l'optimisation est de la spéculation. Avec, l'optimisation est un gain mesuré qui vaut la peine d'être livré.

> ✅ **Bonne pratique** : Faire tourner les benchmarks `[MemoryDiagnoser]` comme partie du repository, commités à côté du code qu'ils mesurent. Quand quelqu'un refactore le chemin chaud six mois plus tard, le benchmark dit immédiatement si les allocations ont ressurgi.

## Zoom : quand le zero allocation est le mauvais objectif

Le code zero-allocation est plus difficile à lire, plus difficile à debugger, et plus facile à rater. L'appliquer à une méthode qui tourne deux fois par minute est un retour sur investissement pur négatif. À sortir quand :

- Un [test de stress](/fr/posts/load-testing-stress/) montre que le temps GC domine le chemin chaud.
- Un [soak](/fr/posts/load-testing-soak/) montre une pression heap qui monte, avec une grande fraction du temps passée dans les collectes.
- Un profil BenchmarkDotNet montre une boucle interne qui alloue à chaque itération sur un chemin appelé des milliers de fois par seconde.
- Les percentiles de latence montrent une longue queue qui s'aligne sur les événements de collecte gen2 dans les logs GC.

À *ne pas* sortir quand :

- La méthode n'est pas sur un chemin chaud. Les endpoints CRUD, les opérations d'admin, et les jobs de fond en ont rarement besoin.
- La lisibilité est le goulet d'étranglement. Du code qu'un ingénieur comprend aujourd'hui vaut souvent plus que du code qui tourne 2% plus vite et que personne ne peut modifier.
- Les allocations sont inévitables par construction (sérialisation JSON, rendu d'une page HTML complète). Optimiser les allocations qui sont réellement optionnelles.

## Wrap-up

Le zero-allocation .NET est un outil de précision, pas un mode de vie. Tu peux sortir `Span<T>` et `stackalloc` sur les buffers de courte durée, `ArrayPool<T>` sur les plus grands, `ValueTask<T>` sur les méthodes async qui se terminent souvent synchroniquement, et `ObjectPool<T>` sur les objets complexes réutilisables. Tu peux mesurer chaque changement avec des benchmarks `[MemoryDiagnoser]` pour que les gains soient réels et ne régressent pas en silence. Tu peux appliquer ces techniques là où un [stress](/fr/posts/load-testing-stress/) ou un [soak](/fr/posts/load-testing-soak/) prouve que le GC est le goulet, et laisser le reste du codebase tranquille.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Les Tests de Charge en .NET : vue d'ensemble des quatre types qui comptent](/fr/posts/load-testing-overview/)
- [Le Stress Testing en .NET : trouver le point de rupture et sa forme](/fr/posts/load-testing-stress/)
- [Le Soak Testing en .NET : les bugs qui n'apparaissent qu'après des heures](/fr/posts/load-testing-soak/)
- [Les Tests Unitaires en .NET : rapides, ciblés, et vraiment utiles](/fr/posts/testing-unit-testing/)

## Références

- [Types Memory et Span, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/standard/memory-and-spans/)
- [ArrayPool&lt;T&gt;, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/api/system.buffers.arraypool-1)
- [Guidance ValueTask&lt;T&gt;, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/api/system.threading.tasks.valuetask-1)
- [Pattern object pool, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/performance/objectpool)
- [Documentation BenchmarkDotNet](https://benchmarkdotnet.org/)
- [Fondamentaux du garbage collection, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/standard/garbage-collection/fundamentals)
