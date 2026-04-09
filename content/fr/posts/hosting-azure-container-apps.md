---
title: "Héberger ASP.NET Core sur Azure Container Apps"
date: 2026-04-08
draft: false
tags: ["hosting", "azure", "containers", "dotnet"]
categories: ["Hosting"]
series: ["Hosting"]
description: "Azure Container Apps apporte de l'hébergement container de classe Kubernetes sans faire tourner un cluster. Scale à zéro, autoscaling KEDA, Dapr, ingress, et la place qu'il occupe dans le paysage d'hébergement .NET."
---

Hello tous le monde, aujourd'hui on va découvrir **Azure Container Apps**, la plateforme d'hébergement container managée qui comble l'écart entre Docker nu et un vrai cluster Kubernetes.

Entre héberger un seul container sur une VM et faire tourner un vrai [cluster Kubernetes](/fr/posts/hosting-kubernetes/), il y a un écart dans lequel les équipes tombaient régulièrement. Elles voulaient les garanties de Kubernetes (rolling updates, autoscaling, config déclarative, isolation des charges) sans le poids opérationnel (upgrades de cluster, maintenance d'un Ingress Controller, debug de plugins CNI, rotation de certificats). Les plateformes container serverless ont été la réponse, et la version d'Azure, **Azure Container Apps** (ACA), a atteint la disponibilité générale en mai 2022. C'est aujourd'hui une cible de première classe pour les charges ASP.NET Core qui vivent dans l'écosystème Azure.

Cet article couvre ce qu'est réellement ACA sous le capot, comment déployer une image ASP.NET Core dessus, et dans quels cas c'est le bon choix comparé à [Docker nu](/fr/posts/hosting-docker/), à [Kubernetes](/fr/posts/hosting-kubernetes/), ou au prochain article de la série, [Azure Web App](/fr/posts/hosting-azure-web-app/).

## Le contexte : pourquoi Azure Container Apps

Azure Container Apps est une plateforme d'hébergement container managée construite au-dessus de composants open source déjà connus : **Kubernetes** pour l'orchestration, **KEDA** pour l'autoscaling, **Envoy** pour l'ingress, **Dapr** pour la communication service-à-service. Microsoft opère la couche Kubernetes à la place de l'utilisateur, expose une surface d'API simplifiée, et facture à la seconde d'usage. Le résultat est une plateforme qui apporte 80% des capacités de Kubernetes pour environ 20% du coût opérationnel.

Les avantages concrets qui comptent pour une équipe .NET :

1. **Scale à zéro.** Une application inactive ne consomme aucune ressource et ne coûte rien. Quand la première requête arrive, ACA réveille une nouvelle instance en quelques secondes. Couplé à [Native AOT](/fr/posts/performance-aot-compilation/), le démarrage à froid devient réellement rapide.
2. **Autoscaling événementiel via KEDA.** Scaler par nombre de requêtes HTTP, profondeur de file sur Azure Service Bus ou Storage Queues, lag Kafka, métriques Prometheus custom, n'importe lequel des 60+ scalers KEDA. Pas juste le CPU.
3. **Aucun cluster à gérer.** Pas de `kubectl`, pas de pools de nœuds, pas d'upgrades de version, pas d'Ingress Controller à maintenir. Azure s'occupe de tout.
4. **Revisions et partage de trafic.** Chaque déploiement crée une nouvelle revision. Le trafic peut être réparti entre revisions (80/20, canary, blue/green) par un seul appel d'API, et le rollback se fait en réaffectant le trafic à la revision précédente. Aucune orchestration de rolling update à écrire.
5. **Intégration Dapr, optionnelle.** Si on veut abstraire les appels service-à-service, la gestion d'état, le pub/sub, ou les stores de secrets de leur provider sous-jacent, Dapr est disponible via un flag dans la définition du container app. Pas obligatoire, mais présent si la forme correspond.

## Vue d'ensemble : la hiérarchie ACA

{{< mermaid >}}
graph TD
    A[Souscription Azure] --> B[Container Apps Environment<br/>réseau partagé, Log Analytics]
    B --> C[Container App<br/>shop-api]
    B --> D[Container App<br/>shop-worker]
    B --> E[Container App<br/>shop-web]
    C --> C1[Revision v1.4.6<br/>0% trafic]
    C --> C2[Revision v1.4.7<br/>100% trafic]
    C2 --> C2P[Replica 1]
    C2 --> C2Q[Replica 2]
{{< /mermaid >}}

La hiérarchie a trois niveaux qu'il faut comprendre avant de toucher au YAML ou aux commandes `az`.

**Container Apps Environment** est la frontière d'isolation. Correspond grossièrement à un namespace Kubernetes avec son propre réseau virtuel, son propre workspace Log Analytics, et son propre domaine d'ingress. Les applications dans le même environnement peuvent se parler via le réseau interne ; celles dans des environnements différents ne le peuvent pas. Un setup typique a un environnement par stage (dev, staging, prod) ou un par domaine métier.

**Container App** est l'application elle-même. Elle a un nom, une référence d'image, des variables d'environnement, des références de secrets, une configuration d'ingress, et des règles de scaling. On peut la voir comme l'équivalent d'un Deployment Kubernetes plus Service plus Ingress combinés en une seule ressource.

**Revision** est un snapshot immuable de la configuration du Container App. Chaque changement d'image ou de configuration marquée comme "revision-scoped" crée une nouvelle revision. Le trafic entre revisions peut être réparti explicitement, et c'est le mécanisme pour les déploiements canary et blue/green.

**Replica** est un container qui tourne. ACA décide combien de réplicas chaque revision active a besoin selon les règles de scaling et la charge courante.

## Zoom : déployer une image ASP.NET Core

Le chemin le plus simple vers un déploiement qui marche utilise Azure Bicep ou l'Azure CLI. Voici un template Bicep minimal :

```bicep
param location string = resourceGroup().location
param envName string = 'shop-env'
param appName string = 'shop-api'
param imageName string = 'myregistry.azurecr.io/shop-api:1.4.7'

resource logs 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: '${envName}-logs'
  location: location
  properties: {
    sku: { name: 'PerGB2018' }
    retentionInDays: 30
  }
}

resource env 'Microsoft.App/managedEnvironments@2025-01-01' = {
  name: envName
  location: location
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logs.properties.customerId
        sharedKey: logs.listKeys().primarySharedKey
      }
    }
  }
}

resource app 'Microsoft.App/containerApps@2025-01-01' = {
  name: appName
  location: location
  properties: {
    managedEnvironmentId: env.id
    configuration: {
      ingress: {
        external: true
        targetPort: 8080
        transport: 'http'
        allowInsecure: false
      }
      secrets: [
        {
          name: 'db-connection'
          value: 'Host=...'
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'api'
          image: imageName
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
          env: [
            { name: 'ASPNETCORE_ENVIRONMENT', value: 'Production' }
            { name: 'ConnectionStrings__Default', secretRef: 'db-connection' }
          ]
          probes: [
            {
              type: 'Liveness'
              httpGet: { path: '/health/live', port: 8080 }
              periodSeconds: 10
              failureThreshold: 3
            }
            {
              type: 'Readiness'
              httpGet: { path: '/health/ready', port: 8080 }
              periodSeconds: 5
              failureThreshold: 3
            }
          ]
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 10
        rules: [
          {
            name: 'http-scale'
            http: {
              metadata: {
                concurrentRequests: '50'
              }
            }
          }
        ]
      }
    }
  }
}
```

Six détails qui comptent pour un déploiement de prod.

**`ingress.external: true`** expose l'application à internet en HTTPS avec un certificat géré par Azure sur un sous-domaine `*.azurecontainerapps.io`. Pour un domaine custom, le lier séparément et configurer un enregistrement CNAME.

**`targetPort: 8080`** correspond au port que l'application ASP.NET Core écoute à l'intérieur du container. Le port HTTP Kestrel par défaut pour `mcr.microsoft.com/dotnet/aspnet` est 8080 depuis .NET 8, ce que l'[article Docker](/fr/posts/hosting-docker/) recommande.

**Les références `secrets`** gardent les chaînes de connexion hors du template. La `value` peut venir d'un paramètre, de Key Vault via un `keyVaultUrl`, ou d'une autre source. Ne jamais inliner des secrets de prod dans un fichier Bicep commité.

**`probes`** reflètent les probes liveness et readiness de Kubernetes, avec la même sémantique : la liveness redémarre le replica, la readiness le retire temporairement de l'ingress.

**Les règles `scale`** définissent l'autoscaling. Ici, l'application scale sur les requêtes HTTP concurrentes par replica : si chaque replica détient plus de 50 requêtes concurrentes, ACA en ajoute un. On peut combiner plusieurs règles (concurrence HTTP + profondeur de file + CPU) et ACA prend le max.

**`minReplicas: 1`** signifie qu'au moins un replica tourne toujours, ce qui évite le démarrage à froid. Le passer à `0` pour économiser sur les charges à faible trafic (scale à zéro), en acceptant un démarrage à froid de 2 à 5 secondes sur la première requête après inactivité.

> 💡 **Info** : `minReplicas: 0` est la fonctionnalité qui différencie vraiment ACA de [Kubernetes](/fr/posts/hosting-kubernetes/). Scaler à zéro signifie qu'un environnement de dev inactif coûte quelques centimes par jour. Les charges de prod avec un trafic stable gardent généralement `minReplicas: 1` ou plus pour éviter toute latence de démarrage à froid.

## Zoom : revisions et partage de trafic

Chaque fois que le tag d'image ou la configuration revision-scoped change, ACA crée une nouvelle revision. Par défaut, la nouvelle revision reçoit 100% du trafic et la précédente est désactivée. Pour des déploiements canary ou blue/green, le partage explicite de trafic est un seul appel CLI :

```bash
# Déploie une nouvelle image. Crée la revision shop-api--v147.
az containerapp update \
  --name shop-api \
  --resource-group shop-rg \
  --image myregistry.azurecr.io/shop-api:1.4.7 \
  --revision-suffix v147

# Met 10% du trafic sur la nouvelle revision, 90% sur l'ancienne.
az containerapp ingress traffic set \
  --name shop-api \
  --resource-group shop-rg \
  --revision-weight shop-api--v146=90 shop-api--v147=10

# Surveille les métriques pendant 15 minutes. Si tout va bien, bascule à 100%.
az containerapp ingress traffic set \
  --name shop-api \
  --resource-group shop-rg \
  --revision-weight shop-api--v147=100
```

Le rollback est l'inverse : basculer le trafic vers la revision précédente en une commande. Pas de terminaison de pods, pas de rolling update à attendre, pas de scripts.

> ✅ **Bonne pratique** : Automatiser les bascules de trafic dans le pipeline de déploiement avec un gate d'observabilité : répartir 10% vers la nouvelle revision, attendre 10 minutes, comparer le taux d'erreur et la latence au baseline (couvert dans l'[article sur le baseline](/fr/posts/load-testing-baseline/)), et ne passer à 100% que si les métriques tiennent. Revenir automatiquement en arrière sinon.

## Zoom : règles de scaling KEDA

Le scaler de concurrence HTTP vu plus haut est le plus simple. Pour les charges pilotées par des files, des topics Kafka, ou des métriques custom, ACA expose la bibliothèque complète des scalers KEDA.

```bicep
scale: {
  minReplicas: 0
  maxReplicas: 30
  rules: [
    {
      name: 'queue-scale'
      custom: {
        type: 'azure-servicebus'
        metadata: {
          queueName: 'orders-inbound'
          messageCount: '5'
        }
        auth: [
          {
            secretRef: 'servicebus-connection'
            triggerParameter: 'connection'
          }
        ]
      }
    }
  ]
}
```

Cela scale l'application selon la profondeur d'une file Azure Service Bus : s'il y a plus de 5 messages par replica, ACA ajoute un replica, jusqu'à 30 au total. Quand la file se vide, ACA redescend à zéro, et l'application cesse de consommer du compute jusqu'au prochain message. Pour des charges événementielles, c'est une amélioration de coût spectaculaire par rapport à un hébergement always-on.

> ⚠️ **Ça marche, mais...** : Scale-à-zéro plus charges HTTP produit des démarrages à froid de 2 à 5 secondes pour la première requête après inactivité. Pour des API exposées à l'utilisateur, c'est généralement inacceptable, et `minReplicas` doit rester à 1 ou plus. Pour des workers de fond déclenchés par une file, c'est acceptable : la file absorbe la latence, et l'économie de coût est réelle.

## Zoom : configuration et secrets

ACA expose deux endroits pour la configuration. Les variables d'environnement classiques pour les valeurs non sensibles, et une section secrets séparée pour tout ce qui est sensible. Les secrets sont référencés par nom depuis la liste des variables d'environnement :

```bicep
secrets: [
  { name: 'db-connection', keyVaultUrl: 'https://shop-kv.vault.azure.net/secrets/db-connection', identity: 'system' }
  { name: 'jwt-key',        keyVaultUrl: 'https://shop-kv.vault.azure.net/secrets/jwt-key',        identity: 'system' }
]
```

Utiliser `keyVaultUrl` avec une identité managée assignée par le système est le pattern canonique : les secrets vivent dans Azure Key Vault, ACA les tire au déploiement via son identité, et aucune valeur en clair ne touche jamais le template Bicep. Si le secret dans Key Vault tourne, ACA a besoin d'une nouvelle revision pour prendre en compte le changement.

Pour des valeurs qui changent sans un déploiement (feature flags, limites de débit), appairer ACA avec Azure App Configuration et le package `Microsoft.Extensions.Configuration.AzureAppConfiguration`. L'application recharge les valeurs sans redémarrage.

## Quand ACA est le bon choix

Azure Container Apps est le bon hôte pour :

- **Les charges container-natives dans Azure** qui iraient autrement sur Kubernetes mais n'ont pas besoin du contrôle ou de la complexité complète.
- **Les services événementiels** (consommateurs de files, workers de fond, processeurs Kafka) qui bénéficient du scale à zéro.
- **Les microservices où on veut que les appels service-à-service, le pub/sub ou la gestion d'état** soient abstraits via Dapr.
- **Les équipes qui ont de l'expertise container mais pas de budget d'opérations Kubernetes.**
- **Les stratégies de release lourdes en partage de trafic** : canary, blue/green, A/B, où le système natif de revisions supprime le besoin d'outillage de rollout custom.

Ce n'est *pas* le bon choix quand :

- **Il faut un contrôle complet sur Kubernetes** : CRDs custom, operators, NetworkPolicies, personnalisation cluster-wide. Passer à AKS (l'[article Kubernetes](/fr/posts/hosting-kubernetes/)).
- **On gère une seule petite application web avec un trafic stable sans containers** : [Azure Web App](/fr/posts/hosting-azure-web-app/) est encore plus simple et souvent moins cher.
- **L'équipe n'est pas sur Azure** : porter le modèle d'ACA vers AWS ou GCP n'est pas trivial. Si le multi-cloud est une exigence, Kubernetes est une meilleure couche de portabilité.
- **Le démarrage à froid compte et on ne peut pas se permettre `minReplicas: 1`** : le démarrage à froid d'ACA est de 2 à 5 secondes, excellent pour un worker de file et trop lent pour une API exposée à l'utilisateur sans réplicas always-on.

## Wrap-up

Azure Container Apps apporte les bénéfices de l'hébergement container de classe Kubernetes sans le poids opérationnel : revisions, partage de trafic, autoscaling KEDA, ingress avec certificats managés, secrets basés sur Key Vault, et scale-à-zéro pour les charges qui tolèrent le démarrage à froid. Tu peux déployer une image ASP.NET Core avec un template Bicep en un après-midi, la combiner avec du scaling basé sur file pour des workers événementiels, répartir le trafic entre revisions pour des déploiements canary, et reconnaître quand la charge serait mieux servie par [Kubernetes](/fr/posts/hosting-kubernetes/) ou [Azure Web App](/fr/posts/hosting-azure-web-app/).

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Héberger ASP.NET Core avec Docker : un guide pragmatique](/fr/posts/hosting-docker/)
- [Héberger ASP.NET Core sur Kubernetes : l'essentiel pour les devs .NET](/fr/posts/hosting-kubernetes/)
- [La Compilation AOT en .NET : démarrage, taille, et compromis](/fr/posts/performance-aot-compilation/)
- [Le Spike Testing en .NET : survivre au burst soudain](/fr/posts/load-testing-spike/)

## Références

- [Documentation Azure Container Apps, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/container-apps/)
- [Scaler une application dans Azure Container Apps, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/container-apps/scale-app)
- [Gérer les revisions, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/container-apps/revisions)
- [Documentation des scalers KEDA](https://keda.sh/docs/latest/scalers/)
- [Dapr sur Azure Container Apps, Microsoft Learn](https://learn.microsoft.com/fr-fr/azure/container-apps/dapr-overview)
