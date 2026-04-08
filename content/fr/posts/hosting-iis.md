---
title: "Héberger ASP.NET Core sur IIS : le classique, démystifié"
date: 2026-04-08
draft: false
tags: ["hosting", "iis", "dotnet"]
categories: ["Hosting"]
series: ["Hosting"]
description: "L'ASP.NET Core Module, in-process vs out-of-process, le vrai rôle d'IIS en 2026, et quand l'hébergement sur IIS reste le bon choix pour une équipe .NET."
---

Hello tous le monde, aujourd'hui on va démystifier l'**hébergement d'ASP.NET Core sur IIS**, le grand classique du monde Windows Server qui est encore très présent en 2026, et qu'il vaut mieux savoir configurer correctement.

Pendant vingt ans, IIS a été la réponse par défaut à "où tourne cette application .NET". System.Web, ASP.NET WebForms, MVC jusqu'à la version 5, WCF, WebAPI 2 : tous étaient fortement couplés au pipeline IIS et à `HttpRuntime`. Quand ASP.NET Core est sorti en 2016, il a été explicitement découplé d'IIS : il tournait sur son propre serveur web cross-platform, Kestrel, et IIS est devenu optionnel. Pourtant, dix ans plus tard, IIS reste la cible de prod pour une part significative des boutiques .NET, en général parce qu'un Windows Server on-prem existant, une flotte d'applications legacy, ou une contrainte de conformité le maintient dans le paysage. Cet article traite de l'hébergement d'ASP.NET Core sur IIS en 2026 : ce qu'IIS fait réellement, ce qu'il ne fait pas, et dans quels cas c'est encore le bon choix.

## Le contexte : pourquoi IIS est encore dans le paysage

L'histoire classique est "IIS est legacy, passez aux containers". Cette histoire est à moitié vraie. IIS n'est clairement pas l'avenir, et les nouveaux projets greenfield y démarrent rarement. Mais trois situations en font le choix pragmatique correct :

1. **Une infrastructure Windows Server existante avec une équipe d'ops qui la maîtrise.** Une boîte qui fait tourner cinquante applications .NET sur IIS, avec du monitoring, des pipelines de déploiement et des runbooks construits autour d'IIS, ne gagne rien à déplacer une seule application sur une stack complètement différente. Le coût d'intégration dépasse le gain marginal de l'hébergement.
2. **Applications legacy mélangées avec des modernes.** Une application ASP.NET Core qui doit cohabiter avec une application ASP.NET WebForms, partager l'authentification, partager les certificats SSL, ou répondre sous le même domaine est bien plus facile à héberger sur le même IIS qu'à répartir sur deux stratégies d'hébergement.
3. **Conformité et contraintes de politique.** Certains environnements exigent des configurations TLS spécifiques, des fonctionnalités HTTP.sys, de l'authentification Windows par Kerberos, ou une intégration à Active Directory qui est significativement plus simple sur IIS que sur Kestrel seul.

Aucune de ces raisons ne rend IIS "bon". Elles rendent IIS *adapté* dans son contexte. Le travail consiste à héberger de l'ASP.NET Core moderne correctement dessus, pas à faire semblant que la contrainte n'existe pas.

## Vue d'ensemble : ce qu'IIS fait réellement

{{< mermaid >}}
graph LR
    A[Requête HTTP] --> B[Driver kernel HTTP.sys]
    B --> C[Process worker IIS<br/>w3wp.exe]
    C --> D[ASP.NET Core Module<br/>aspnetcore v2]
    D --> E[Application Kestrel<br/>dotnet.exe]
    E --> F[Réponse]
    F --> C
    C --> B
    B --> A
{{< /mermaid >}}

La pièce clé à comprendre : IIS n'est plus le serveur web de l'application ASP.NET Core. C'est un **reverse proxy** devant un process Kestrel qui fait tourner l'application. Le composant qui fait ce lien est l'**ASP.NET Core Module** (ANCM), un module IIS natif installé avec le .NET Hosting Bundle. ANCM a deux modes, et le choix entre les deux est la décision d'hébergement la plus importante sur IIS.

**Le mode in-process** (défaut depuis .NET Core 2.2) : ANCM charge le CLR directement à l'intérieur du process worker IIS `w3wp.exe`. Kestrel tourne in-process, et les requêtes atteignent l'application via un canal rapide en mémoire. C'est à peu près 2 à 3 fois plus rapide que le mode out-of-process, et c'est le bon défaut pour la plupart des charges.

**Le mode out-of-process** : ANCM lance un process enfant `dotnet.exe` séparé qui héberge Kestrel sur un port localhost, puis forwarde les requêtes IIS vers lui en HTTP. Plus lent, mais nécessaire quand l'application doit s'isoler du process worker ou utiliser des fonctionnalités qui ne marchent pas in-process (par exemple certaines formes d'interop avec des modules).

> 💡 **Info** : L'ASP.NET Core Module s'appelait à l'origine `aspnetcore` (v1), puis a été réécrit en `aspnetcorev2` dans .NET Core 2.2 pour ajouter l'hébergement in-process. Les deux sont installés par le .NET Hosting Bundle ; l'application choisit celui qui est actif via son `web.config`. Les nouveaux projets doivent toujours démarrer en in-process.

## Zoom : le web.config minimal

ASP.NET Core sur IIS utilise toujours un fichier `web.config`, mais il est généré au publish et son seul rôle est de dire à IIS quel module charger et quel exécutable lancer. Pour la plupart des applications, le fichier n'a pas besoin d'édition manuelle :

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore"
             path="*"
             verb="*"
             modules="AspNetCoreModuleV2"
             resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath=".\Shop.Api.exe"
                  stdoutLogEnabled="false"
                  stdoutLogFile=".\logs\stdout"
                  hostingModel="inprocess" />
    </system.webServer>
  </location>
</configuration>
```

L'attribut `hostingModel="inprocess"` est ce qui active le mode in-process. Le `processPath` pointe vers l'exécutable publié, pas vers `dotnet.exe`, parce qu'un publish self-contained ou framework-dependent produit un petit launcher natif pour Windows.

> ✅ **Bonne pratique** : Ne jamais éditer le `web.config` généré à la main pour ajouter des variables d'environnement ou des flags de démarrage. Utiliser le groupe d'items `<EnvironmentVariables>` dans le csproj au moment du publish, ou les définir sur le pool d'applications IIS via le gestionnaire IIS. Un `web.config` édité à la main sera écrasé au prochain publish.

## Zoom : configurer le pool d'applications

Un pool d'applications IIS est le process worker qui héberge l'application. Pour ASP.NET Core, quelques réglages comptent plus que les défauts.

**Version du CLR .NET** : à mettre sur **Aucun code managé**. C'est contre-intuitif mais correct. Le réglage fait référence à l'*ancien* CLR .NET (System.Web), que ASP.NET Core n'utilise pas. Le laisser sur "v4.0" charge du code runtime legacy dans le process worker sans aucune raison.

**Mode de pipeline managé** : Integrated (le défaut). Le mode Classic est pour les applications legacy qui dépendent du vieux pipeline ISAPI.

**Mode et identité du pipeline** : l'`ApplicationPoolIdentity` par défaut est en général correct. Pour les applications qui ont besoin d'accéder à un partage réseau ou à un SQL Server avec authentification intégrée, passer sur un compte de service de domaine dédié au pool.

**Recyclage** : le défaut recycle le pool toutes les 1740 minutes (29 heures). Pour une application ASP.NET Core longue durée qui détient des caches en mémoire, cela force un démarrage à froid chaque jour, ce qui est perturbant. Soit désactiver le recyclage par temps, soit le planifier pendant une fenêtre de trafic faible. Une application ASP.NET Core moderne n'a pas le comportement runtime legacy qui rendait les recyclages quotidiens nécessaires sur System.Web.

**Idle timeout** : le défaut de 20 minutes arrête le process worker quand aucun trafic n'arrive. C'est bien pour des applications intranet mais cela produira des premières requêtes lentes après chaque période d'inactivité. Pour des applications exposées à internet avec un trafic continu, soit le mettre à 0, soit configurer le module Application Initialization pour garder le process au chaud.

> ⚠️ **Ça marche, mais...** : Les réglages de recyclage et d'idle timeout par défaut ont été conçus pour les charges System.Web des années 2000. Ce sont encore les défauts dans IIS moderne. Une équipe qui publie une application ASP.NET Core sans revoir ces réglages paiera chaque matin la taxe de démarrage à froid décrite dans l'[article sur les spike tests](/fr/posts/load-testing-spike/), et ne comprendra pas pourquoi.

## Zoom : déploiement avec Web Deploy

Le mécanisme de déploiement traditionnel pour IIS est Web Deploy (MSDeploy), qui sait comment arrêter le pool, copier les fichiers, et redémarrer proprement. Une release typique depuis un pipeline CI utilise `dotnet publish` pour produire la sortie, puis `msdeploy.exe` pour la pousser vers le serveur cible :

```powershell
dotnet publish -c Release -r win-x64 --self-contained false -o .\publish

msdeploy.exe `
  -verb:sync `
  -source:contentPath=.\publish `
  -dest:contentPath="Default Web Site/shop-api",computerName=https://iis.internal:8172/msdeploy.axd,userName=deployer,password=...,authType=basic `
  -enableRule:AppOffline
```

La règle `AppOffline` dépose un fichier `App_Offline.htm` dans la racine pendant le déploiement, ce qui fait qu'IIS cesse de forwarder vers l'application et affiche une page de maintenance. Sans elle, les requêtes en vol peuvent échouer bruyamment pendant le remplacement des fichiers.

Pour les équipes qui n'aiment pas MSDeploy, un simple `xcopy` ou `robocopy` vers un dossier partagé suivi de `appcmd recycle apppool /apppool.name:ShopApi` est aussi une stratégie de déploiement parfaitement valable. Moins sophistiquée mais plus scriptable.

> 💡 **Info** : Le mécanisme `App_Offline.htm` est un héritage de System.Web, mais ASP.NET Core le respecte toujours. Déposer un fichier nommé exactement `App_Offline.htm` dans la racine de l'application provoque le déchargement du CLR par le module d'hébergement et sert le contenu HTML comme une 503.

## Zoom : observabilité sur IIS

Les applications ASP.NET Core hébergées sur IIS gardent un accès complet à l'observabilité .NET standard : `ILogger`, OpenTelemetry, Application Insights, et toutes les métriques ou traces que l'application émet. Les seules surfaces spécifiques à IIS à connaître sont :

**Les logs IIS** sous `C:\inetpub\logs\LogFiles\W3SVC*`. Ces fichiers enregistrent chaque requête HTTP au niveau IIS, y compris le status code et le temps de réponse, et sont utiles pour corréler avec les logs applicatifs quand quelque chose tourne mal en amont du code applicatif.

**Observateur d'événements → Journaux Windows → Application** : l'ASP.NET Core Module écrit les erreurs de démarrage et les crashs ici. C'est le premier endroit à vérifier quand l'application ne démarre pas après un déploiement.

**`stdoutLogEnabled`** dans le `web.config` : passer temporairement ce réglage à `true` redirige la sortie console de Kestrel vers un fichier. Ne l'activer que pendant un diagnostic et le désactiver après, parce que le fichier n'est pas tourné et grandit sans borne.

```csharp
// Program.cs : router ILogger vers Event Log pour corréler avec IIS
builder.Logging.AddEventLog(new EventLogSettings
{
    SourceName = "Shop.Api",
    LogName = "Application"
});
```

Cela écrit des entrées de log visibles dans l'Observateur d'événements, qu'une équipe d'ops familière avec IIS peut lire sans avoir besoin d'un outil d'agrégation de logs séparé. Ce n'est pas un remplacement du logging structuré, mais c'est un repli pratique.

## Zoom : quand IIS n'est pas la bonne réponse

IIS n'est pas le bon hôte quand :

- **Le déploiement container-natif est la cible.** Si le pipeline de déploiement livre des images Docker et que l'orchestrateur est Kubernetes, Azure Container Apps ou similaire, passer par IIS ajoute un serveur Windows qui n'a pas sa place dans le chemin.
- **Le cross-platform est une exigence.** Les cibles Linux excluent IIS entièrement.
- **L'autoscaling est nécessaire.** IIS scale en ajoutant des process workers ou des hôtes workers ; il ne scale pas élastiquement par nombre d'instances comme une plateforme container. Pour une charge variable, une plateforme container est mieux adaptée.
- **L'équipe est déjà Linux-first.** Faire tourner un Windows Server pour une seule application .NET dans un environnement par ailleurs Linux est le type de "choix d'hébergement" le plus cher parce que le coût opérationnel est élevé et la base de compétences est différente.

Pour ces cas, les prochains articles de la série couvrent [Docker](/fr/posts/hosting-docker/), [Kubernetes](/fr/posts/hosting-kubernetes/), [Azure Container Apps](/fr/posts/hosting-azure-container-apps/), et [Azure Web App](/fr/posts/hosting-azure-web-app/).

## Wrap-up

IIS en 2026 est un reverse proxy qui place un process Kestrel derrière lui via l'ASP.NET Core Module en mode in-process, et c'est un hôte parfaitement raisonnable pour une application ASP.NET Core dans le bon contexte : infrastructure Windows Server existante, mélange d'applications legacy, ou contraintes de conformité qui rendent un autre chemin plus coûteux. Tu peux mettre le pool d'applications sur "Aucun code managé", désactiver le recyclage quotidien legacy, garder l'idle timeout raisonnable, déployer avec Web Deploy et la règle `App_Offline.htm`, et câbler la sortie event log de l'ASP.NET Core Module dans le workflow d'ops existant.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Références

- [Héberger ASP.NET Core sur Windows avec IIS, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/host-and-deploy/iis/)
- [Référence ASP.NET Core Module (ANCM), Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/host-and-deploy/aspnet-core-module)
- [Documentation Web Deploy](https://learn.microsoft.com/fr-fr/iis/publish/using-web-deploy/introduction-to-web-deploy)
- [Réglages de recyclage des pools d'applications IIS, Microsoft Learn](https://learn.microsoft.com/fr-fr/iis/manage/managing-your-configuration-settings/changing-default-application-pool-recycling-settings)
