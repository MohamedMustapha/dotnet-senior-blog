---
title: "Héberger ASP.NET Core sur Azure Web App"
date: 2026-04-08
draft: false
tags: ["hosting", "azure", "dotnet"]
categories: ["Hosting"]
series: ["Hosting"]
description: "Azure App Service Web App est l'hôte de production le plus simple pour une application ASP.NET Core sur Azure. Deployment slots, scaling natif, certificats managés, et quand le choisir plutôt que les containers."
---

Hello tous le monde, aujourd'hui on va démystifier **Azure Web App**, l'option d'hébergement la plus simple pour une application ASP.NET Core sur Azure, et celle qui reste souvent le meilleur choix malgré toutes les alternatives plus modernes.

Toutes les applications .NET n'ont pas besoin de containers, de Kubernetes, ou d'un service mesh. Un nombre surprenant de charges de prod sont mieux servies par l'option d'hébergement Azure la plus simple disponible : une Web App sur Azure App Service. C'est le défaut pour les équipes .NET Microsoft-first depuis 2012, elle livre nativement HTTPS, deployment slots et autoscaling, et pour une large classe d'applications c'est la bonne réponse précisément *parce qu'*elle est plus simple que les alternatives container-first vues plus tôt dans cette série.

Cet article clôt la [série Hosting](/fr/posts/hosting-iis/) avec Azure Web App : ce que c'est, comment y déployer une application ASP.NET Core, ce que font réellement les fonctionnalités natives, et dans quels cas elle gagne face aux [containers sur ACA](/fr/posts/hosting-azure-container-apps/) ou à [Kubernetes](/fr/posts/hosting-kubernetes/).

## Le contexte : pourquoi Azure Web App

Azure App Service a été lancé en 2012 et évolue en continu depuis. Web App est sa variante pour les charges HTTP, qui fait tourner des applications ASP.NET, ASP.NET Core, Node.js, Python, Java et PHP sur une infrastructure gérée par Microsoft. Pour .NET en particulier, le support est natif : pas de Dockerfile à écrire, pas d'image à construire, pas de registry container à gérer. L'application est publiée directement, la plateforme la fait tourner, et toutes les préoccupations habituelles (TLS, scaling, monitoring, authentification) sont disponibles comme des interrupteurs à basculer.

Les avantages concrets qui comptent encore en 2026 :

1. **La simplicité.** Une sortie `dotnet publish` suffit pour déployer. Pas d'image container, pas d'orchestration, pas de YAML, pas de Dockerfile. Pour une équipe dont la compétence principale est d'écrire du .NET, cela correspond au skill set sans en imposer un nouveau.
2. **Les deployment slots natifs.** Chaque Web App au tier Standard ou plus haut livre avec des slots de staging. Déployer sur un slot, valider, puis swap les slots atomiquement. Le swap est instantané et réversible, ce qui fait des déploiements blue/green une fonctionnalité native plutôt que quelque chose à orchestrer.
3. **Certificats TLS managés.** Les App Service Managed Certificates sont gratuits, s'auto-renouvellent, et se branchent sur le domaine custom en un clic. Pas de cert-manager, pas de cron Let's Encrypt, pas d'alertes d'expiration.
4. **Autoscaling et Always On.** Règles de scale-out basées sur le CPU, la mémoire ou des métriques custom. Le réglage Always On empêche le worker de passer en inactif pendant les périodes calmes, ce qui supprime le démarrage à froid qui pénalise les alternatives serverless pour les charges exposées à l'utilisateur.
5. **Intégration avec l'écosystème Azure.** Identité managée, références Key Vault, Application Insights, App Configuration, private endpoints, intégration VNet. Tout cela est disponible sous forme d'options de configuration, pas de packages à installer.

Rien de tout cela n'est unique à Web App. Tout est disponible ailleurs. La valeur, c'est que *tout est au même endroit* et accessible sans outillage supplémentaire.

## Vue d'ensemble : le modèle App Service

{{< mermaid >}}
graph TD
    A[App Service Plan<br/>compute + tier de prix] --> B[Web App<br/>shop-api]
    A --> C[Web App<br/>shop-admin]
    B --> B1[Slot Production]
    B --> B2[Slot Staging]
    B2 --> B2D[Déploiement]
    B1 --> B1T[100% trafic]
{{< /mermaid >}}

La hiérarchie est directe et stable depuis une décennie.

**App Service Plan** est la ressource compute sous-jacente : cœurs CPU, mémoire, tier de prix (Basic, Standard, Premium v3, Isolated). Plusieurs Web Apps peuvent partager un seul plan, ce qui est la façon standard d'héberger des applications liées sur le même compute sans avoir besoin de lignes de facturation séparées.

**Web App** est l'application. Elle a un nom (utilisé dans l'URL par défaut `<nom>.azurewebsites.net`), une stack de runtime (`.NET 10 (LTS)`), une source de déploiement, des paramètres de configuration, et des fonctionnalités optionnelles (domaines custom, identité, règles de scaling).

**Deployment slot** est un clone séparé de la Web App avec sa propre URL, sa propre configuration, et son propre code déployé. Les slots non-production partagent le compute de l'App Service Plan mais tournent indépendamment. La valeur vient de la capacité à swapper le contenu des slots atomiquement : déployer sur staging, le chauffer avec quelques requêtes, lancer des smoke tests, puis le faire basculer en production en quelques secondes.

## Zoom : déployer une application ASP.NET Core

Les trois chemins de déploiement les plus courants, par ordre de maturité :

**1. Publish profile depuis Visual Studio ou la CLI.** Le plus simple pour un seul développeur. `dotnet publish` produit la sortie, `az webapp deploy` (ou l'assistant de publish de Visual Studio) la pousse. Bon pour des prototypes, pas pour des équipes.

**2. Pipeline GitHub Actions ou Azure DevOps avec `azure/webapps-deploy`.** Le chemin CI standard. Le pipeline build, teste, publie, et déploie, avec un seul workflow YAML.

```yaml
# .github/workflows/deploy.yml
name: Deploy Shop API
on:
  push:
    branches: [main]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '10.0.x' }

      - name: Publish
        run: dotnet publish Shop.Api/Shop.Api.csproj -c Release -o ./publish

      - name: Deploy to staging slot
        uses: azure/webapps-deploy@v3
        with:
          app-name: shop-api
          slot-name: staging
          package: ./publish
          publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE_STAGING }}

      - name: Health check staging
        run: |
          curl --fail https://shop-api-staging.azurewebsites.net/health/ready

      - name: Swap slots
        uses: azure/CLI@v2
        with:
          inlineScript: |
            az webapp deployment slot swap \
              --resource-group shop-rg \
              --name shop-api \
              --slot staging \
              --target-slot production
```

Quatre étapes : publier la sortie, déployer sur le slot de staging, vérifier la santé sur le slot de staging, swapper les slots. Le trafic de prod bascule sur la nouvelle version au moment du swap, sans démarrage à froid, parce qu'Azure chauffe le slot de staging avant que le swap ne se termine.

**3. Déploiement container.** Si l'équipe construit déjà des images Docker pour d'autres hôtes, Web App peut faire tourner un container custom depuis n'importe quel registry. Configurer la Web App pour pointer sur l'image, et elle devient un hôte container managé. Cela perd le bénéfice "pas de Dockerfile à écrire" mais garde les fonctionnalités de slots et de scaling.

> ✅ **Bonne pratique** : Toujours déployer sur un slot de staging d'abord et swapper. Le déploiement direct en prod est une habitude des années 2000. Avec les slots, on ne paie essentiellement rien de plus pour une étape de validation pré-prod et la capacité à revenir instantanément en arrière.

## Zoom : configuration sans rebuilder

Web App expose sa configuration par trois couches, par ordre de précédence :

**Paramètres d'application spécifiques au slot.** Des variables d'environnement définies sur la Web App elle-même, qui deviennent des entrées `IConfiguration` dans ASP.NET Core. La convention du double underscore mappe sur les clés imbriquées : `ConnectionStrings__Default` devient `ConnectionStrings:Default`.

**Références Key Vault.** Un paramètre d'application peut contenir une référence à un secret dans Azure Key Vault, et App Service la résout au démarrage via l'identité managée de la Web App. Le vrai secret n'apparaît jamais dans un fichier de configuration ou un artefact de déploiement.

```
ConnectionStrings__Default = @Microsoft.KeyVault(SecretUri=https://shop-kv.vault.azure.net/secrets/db-connection/)
```

**Intégration App Configuration** via le package `Microsoft.Extensions.Configuration.AzureAppConfiguration`, pour les valeurs qui doivent se recharger sans redémarrage (feature flags, limites de débit, toggles). Cela s'appaire particulièrement bien avec Key Vault pour les valeurs sensibles et App Configuration pour les dynamiques.

```csharp
// Program.cs
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(new Uri(builder.Configuration["AppConfig:Endpoint"]!),
                    new DefaultAzureCredential())
        .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential()))
        .Select(KeyFilter.Any, LabelFilter.Null)
        .Select(KeyFilter.Any, builder.Environment.EnvironmentName)
        .ConfigureRefresh(refresh =>
        {
            refresh.Register("Sentinel", refreshAll: true)
                   .SetRefreshInterval(TimeSpan.FromSeconds(30));
        });
});
```

> 💡 **Info** : Un paramètre d'application "spécifique au slot" reste avec le slot pendant un swap, tandis qu'un paramètre classique swap avec le code. Cette distinction permet de garder `ASPNETCORE_ENVIRONMENT=Staging` en permanence sur le slot de staging, pour que le même déploiement puisse être testé en mode staging et basculé en mode production simplement en swappant.

## Zoom : scaling

Web App offre deux dimensions de scaling :

**Scale up** change la taille de l'App Service Plan (plus de CPU, plus de mémoire). C'est une opération qui affecte toutes les Web Apps du plan et prend une à deux minutes. À utiliser quand le tier actuel est trop petit pour la charge de pointe.

**Scale out** ajoute plus d'instances du plan, faisant tourner des copies des mêmes Web Apps en parallèle. Azure load balance le trafic sur les instances automatiquement. Les règles de scale-out peuvent être configurées selon le CPU, la mémoire, la longueur de file, ou des métriques custom, avec des fenêtres de cooldown pour éviter le thrashing.

```bicep
resource plan 'Microsoft.Web/serverfarms@2024-04-01' = {
  name: 'shop-plan'
  location: location
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
    capacity: 2
  }
  kind: 'linux'
  properties: {
    reserved: true  // obligatoire pour Linux
  }
}

resource autoscale 'Microsoft.Insights/autoscalesettings@2022-10-01' = {
  name: 'shop-plan-autoscale'
  location: location
  properties: {
    targetResourceUri: plan.id
    enabled: true
    profiles: [
      {
        name: 'default'
        capacity: { minimum: '2', maximum: '10', default: '2' }
        rules: [
          {
            metricTrigger: {
              metricName: 'CpuPercentage'
              metricResourceUri: plan.id
              timeGrain: 'PT1M'
              statistic: 'Average'
              timeWindow: 'PT5M'
              timeAggregation: 'Average'
              operator: 'GreaterThan'
              threshold: 70
            }
            scaleAction: {
              direction: 'Increase'
              type: 'ChangeCount'
              value: '1'
              cooldown: 'PT5M'
            }
          }
        ]
      }
    ]
  }
}
```

Un plan avec 2 instances minimum, 10 maximum, qui ajoute une instance chaque fois que le CPU moyen sur 5 minutes dépasse 70%, avec un cooldown de 5 minutes. C'est la forme standard pour autoscaler une Web App à trafic stable.

> ⚠️ **Ça marche, mais...** : Les règles d'autoscale sur Web App sont réactives, pas prédictives. Un burst qui dépasse la capacité en 30 secondes (voir l'[article sur les spike tests](/fr/posts/load-testing-spike/)) est plus rapide que la fenêtre de réaction de l'autoscaler. Pour des charges à forte composante spike, soit faire tourner un nombre minimum d'instances plus élevé, soit accepter la latence mise en file d'attente au début de chaque spike.

## Zoom : le réglage Always On

Web App endort un worker inactif après 20 minutes d'inactivité, exactement comme [IIS](/fr/posts/hosting-iis/). C'est acceptable pour des sites hobby et pour des environnements de dev, mais pour des charges de prod exposées à l'utilisateur cela introduit un démarrage à froid sur chaque première requête après inactivité, ce qui casse les cibles de latence p99.

Le correctif est un seul toggle :

```
General settings → Always On → On
```

Cela garde le worker au chaud en permanence. C'est disponible au tier Basic et au-dessus (pas sur Free ou Shared). Pour du trafic de prod, ce réglage doit toujours être activé.

Appairé à Always On, l'endpoint `/health/live` décrit dans l'[article Docker](/fr/posts/hosting-docker/) permet de configurer le ping de santé d'App Service pour taper périodiquement l'endpoint, s'assurant que l'application reste réactive.

```bicep
siteConfig: {
  alwaysOn: true
  healthCheckPath: '/health/live'
  // ...
}
```

## Zoom : quand Web App est le bon choix

Web App est le bon hôte pour :

- **Une seule application web .NET avec un trafic stable.** La simplicité rentabilise : une ressource, un chemin de déploiement, un jeu de réglages.
- **Les équipes dont l'expertise est .NET, pas les containers ni Kubernetes.** Pas de Dockerfile, pas de `kubectl`, pas de connaissance d'orchestration nécessaire.
- **Les applications qui bénéficient des deployment slots** : blue/green sans outillage supplémentaire, tests A/B, rollout progressif avec des pourcentages de routage de trafic.
- **Les charges intégrées à l'écosystème Microsoft** : authentification Entra ID, identité managée, Key Vault, Application Insights. Tout se branche comme options de configuration.
- **Les charges qui ont besoin des fonctionnalités hybrides Azure** : intégration VNet, private endpoints, Hybrid Connections pour l'intégration on-prem.

Ce n'est *pas* le bon choix quand :

- **Le déploiement container-natif est une exigence.** Si l'application est déjà livrée comme image Docker et que l'équipe est sur le chemin container-first, [Azure Container Apps](/fr/posts/hosting-azure-container-apps/) ou [Kubernetes](/fr/posts/hosting-kubernetes/) est mieux adapté.
- **La portabilité multi-cloud ou on-prem compte.** Web App est une offre Azure-only. La porter ailleurs signifie réécrire la couche d'hébergement.
- **La charge est événementielle avec un trafic de base faible.** Le scale à zéro n'est pas une fonctionnalité de Web App (au-delà du tier gratuit). Azure Functions ou Azure Container Apps avec `minReplicas: 0` servent mieux ce pattern.
- **L'isolation de charges sur beaucoup de petits services est nécessaire.** Faire tourner une Web App par microservice devient vite coûteux et opérationnellement lourd comparé à partager un cluster ou un Container Apps Environment.

## Wrap-up

Azure Web App est la façon la plus simple de faire tourner une application ASP.NET Core sur Azure en 2026, et pour une part significative des charges c'est aussi la meilleure. Tu peux déployer avec un pipeline GitHub Actions en une demi-heure, utiliser les deployment slots pour des swaps zero-downtime, câbler des références Key Vault directement dans la configuration, activer Always On pour une latence prévisible, et configurer un autoscaling basé sur le CPU avec un template Bicep. Tu peux reconnaître quand la charge bénéficierait plus de [Kubernetes](/fr/posts/hosting-kubernetes/) ou d'[Azure Container Apps](/fr/posts/hosting-azure-container-apps/) et choisir le bon outil selon la forme du problème plutôt que d'appliquer le même pattern d'hébergement à tout.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Héberger ASP.NET Core sur IIS : le classique, démystifié](/fr/posts/hosting-iis/)
- [Héberger ASP.NET Core avec Docker : un guide pragmatique](/fr/posts/hosting-docker/)
- [Héberger ASP.NET Core sur Kubernetes : l'essentiel pour les devs .NET](/fr/posts/hosting-kubernetes/)
- [Héberger ASP.NET Core sur Azure Container Apps](/fr/posts/hosting-azure-container-apps/)

## Références

- [Vue d'ensemble App Service, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/app-service/overview)
- [Déployer sur App Service, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/app-service/deploy-github-actions)
- [Deployment slots, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/app-service/deploy-staging-slots)
- [App Service Managed Certificates, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/app-service/configure-ssl-certificate)
- [Autoscale dans Azure Monitor, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/azure-monitor/autoscale/autoscale-overview)
- [Références Key Vault pour App Service, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/app-service/app-service-key-vault-references)
