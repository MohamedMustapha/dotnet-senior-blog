---
title: "Hosting ASP.NET Core on Azure Container Apps"
date: 2026-04-08
draft: false
tags: ["hosting", "azure", "containers", "dotnet"]
categories: ["Hosting"]
series: ["Hosting"]
description: "Azure Container Apps gives you Kubernetes-class container hosting without running a cluster. Scale to zero, KEDA-powered autoscaling, Dapr, ingress, and the place it hits in the .NET hosting landscape."
---

Between hosting a single container on a VM and running a full [Kubernetes cluster](/posts/hosting-kubernetes/), there is a gap that teams kept falling into. They wanted Kubernetes's guarantees (rolling updates, autoscaling, declarative config, workload isolation) without the operational weight (upgrading the cluster, maintaining an Ingress Controller, debugging CNI plugins, rotating certificates). The serverless container platforms were the answer, and Azure's version, **Azure Container Apps** (ACA), reached general availability in May 2022. It is now a first-class target for ASP.NET Core workloads that live in the Azure ecosystem.

This article covers what ACA actually is under the hood, how to deploy an ASP.NET Core image to it, and when it is the right choice compared to [plain Docker](/posts/hosting-docker/), [Kubernetes](/posts/hosting-kubernetes/), or the next-article-in-the-series [Azure Web App](/posts/hosting-azure-web-app/).

## Why Azure Container Apps

Azure Container Apps is a managed container hosting platform built on top of open-source components you already know: **Kubernetes** for orchestration, **KEDA** for autoscaling, **Envoy** for ingress, **Dapr** for service-to-service communication. Microsoft operates the Kubernetes layer for you, exposes a simplified API surface, and bills per second of usage. The result is a platform that gives you 80% of Kubernetes's capabilities with about 20% of the operational cost.

The specific advantages that matter for a .NET team:

1. **Scale to zero.** An idle application consumes no resources and costs nothing. When the first request arrives, ACA wakes a new instance in a few seconds. Combined with [Native AOT](/posts/performance-aot-compilation/), cold start becomes genuinely fast.
2. **Event-driven autoscaling via KEDA.** Scale by HTTP request count, queue depth on Azure Service Bus or Storage Queues, Kafka lag, custom Prometheus metrics, any of the 60+ KEDA scalers. Not just CPU.
3. **No cluster to manage.** No `kubectl`, no node pools, no version upgrades, no Ingress Controller to maintain. Azure handles all of it.
4. **Revisions and traffic splitting.** Every deployment creates a new revision. You can split traffic between revisions (80/20, canary, blue/green) with a single API call, and roll back by reassigning traffic to the previous revision. No rolling update orchestration to write.
5. **Dapr integration, optional.** If you want service-to-service calls, state management, pub/sub, or secret stores abstracted from their underlying provider, Dapr is available with a flag in the container app definition. You do not have to use it, but it is there if the shape fits.

## Overview: the ACA hierarchy

{{< mermaid >}}
graph TD
    A[Azure subscription] --> B[Container Apps Environment<br/>shared network, Log Analytics]
    B --> C[Container App<br/>shop-api]
    B --> D[Container App<br/>shop-worker]
    B --> E[Container App<br/>shop-web]
    C --> C1[Revision v1.4.6<br/>0% traffic]
    C --> C2[Revision v1.4.7<br/>100% traffic]
    C2 --> C2P[Replica 1]
    C2 --> C2Q[Replica 2]
{{< /mermaid >}}

The hierarchy is three levels deep and worth understanding before touching any YAML or `az` commands.

**Container Apps Environment** is the isolation boundary. It corresponds roughly to a Kubernetes namespace with its own virtual network, its own Log Analytics workspace, and its own ingress domain. Apps inside the same environment can talk to each other over the internal network; apps in different environments cannot. A typical setup has one environment per stage (dev, staging, prod) or one per business domain.

**Container App** is the application itself. It has a name, an image reference, environment variables, secret references, ingress configuration, and scaling rules. Think of it as the equivalent of a Kubernetes Deployment plus Service plus Ingress combined into a single resource.

**Revision** is an immutable snapshot of the Container App's configuration. Every change to the image or any configuration marked as "revision-scoped" creates a new revision. Traffic between revisions can be split explicitly, which is the mechanism for canary and blue/green deployments.

**Replica** is a running container. ACA decides how many replicas each active revision needs based on the scaling rules and the current load.

## Zoom: deploying an ASP.NET Core image

The simplest path to a working deployment uses Azure Bicep or the Azure CLI. Here is a minimal Bicep template:

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

Six details that matter for a production deployment.

**`ingress.external: true`** exposes the app to the internet over HTTPS with an Azure-managed certificate on a `*.azurecontainerapps.io` subdomain. For a custom domain, bind it separately and configure a CNAME record.

**`targetPort: 8080`** matches the port the ASP.NET Core app listens on inside the container. The default Kestrel HTTP port for `mcr.microsoft.com/dotnet/aspnet` is 8080 since .NET 8, which is what the [Docker article](/posts/hosting-docker/) recommends.

**`secrets` references** keep connection strings out of the template. The `value` can come from a parameter, from Key Vault via a `keyVaultUrl`, or from another source. Never inline production secrets into a committed Bicep file.

**`probes`** mirror the Kubernetes liveness and readiness probes, with the same semantics: liveness restarts the replica, readiness removes it from ingress temporarily.

**`scale` rules** define autoscaling. Here, the app scales based on concurrent HTTP requests per replica: if each replica holds more than 50 concurrent requests, ACA adds a new one. You can combine multiple rules (HTTP concurrency + queue depth + CPU) and ACA picks the max.

**`minReplicas: 1`** means at least one replica is always running, avoiding cold start. Set it to `0` for cost savings on low-traffic workloads (scale to zero), accepting a cold start of 2-5 seconds on the first request after idle.

> 💡 **Info** : `minReplicas: 0` is the feature that truly differentiates ACA from [Kubernetes](/posts/hosting-kubernetes/). Scaling to zero means an idle dev environment costs cents per day. Production workloads with steady traffic usually keep `minReplicas: 1` or higher to avoid any cold start latency.

## Zoom: revisions and traffic splitting

Every time the image tag or revision-scoped configuration changes, ACA creates a new revision. By default, the new revision receives 100% of traffic and the previous one is deactivated. For canary or blue/green deployments, explicit traffic splitting is a single CLI call:

```bash
# Deploy a new image. Creates revision shop-api--v147.
az containerapp update \
  --name shop-api \
  --resource-group shop-rg \
  --image myregistry.azurecr.io/shop-api:1.4.7 \
  --revision-suffix v147

# Put 10% of traffic on the new revision, 90% on the old.
az containerapp ingress traffic set \
  --name shop-api \
  --resource-group shop-rg \
  --revision-weight shop-api--v146=90 shop-api--v147=10

# Monitor metrics for 15 minutes. If green, shift to 100%.
az containerapp ingress traffic set \
  --name shop-api \
  --resource-group shop-rg \
  --revision-weight shop-api--v147=100
```

Rollback is the inverse: shift traffic back to the previous revision with one command. No pod termination, no rolling update to wait for, no scripts.

> ✅ **Good practice** : Automate traffic shifts in the deployment pipeline with an observability gate: split 10% to the new revision, wait 10 minutes, check error rate and latency against the baseline (covered in the [baseline load testing article](/posts/load-testing-baseline/)), and only proceed to 100% if the metrics hold. Roll back automatically if they do not.

## Zoom: KEDA-powered scaling rules

The HTTP-concurrency scaler shown above is the simplest one. For workloads driven by queues, Kafka topics, or custom metrics, ACA exposes the full KEDA scaler library.

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

This scales the app based on the depth of an Azure Service Bus queue: if there are more than 5 messages per replica, ACA adds a replica, up to 30 total. When the queue empties, ACA scales back to zero, and the app stops consuming compute until the next message arrives. For event-driven workloads, this is a dramatic cost improvement compared to always-on hosting.

> ⚠️ **It works, but...** : Scale-to-zero plus HTTP workloads produces cold starts of 2-5 seconds for the first request after idle. For user-facing APIs, this is usually unacceptable, and `minReplicas` should stay at 1 or higher. For background workers triggered by queues, it is fine: the queue absorbs the latency, and the cost saving is real.

## Zoom: configuration and secrets

ACA exposes two places for configuration. Regular environment variables for non-sensitive values, and a separate secrets section for anything sensitive. Secrets are referenced by name from the environment variable list:

```bicep
secrets: [
  { name: 'db-connection', keyVaultUrl: 'https://shop-kv.vault.azure.net/secrets/db-connection', identity: 'system' }
  { name: 'jwt-key',        keyVaultUrl: 'https://shop-kv.vault.azure.net/secrets/jwt-key',        identity: 'system' }
]
```

Using `keyVaultUrl` with a system-assigned managed identity is the canonical pattern: secrets live in Azure Key Vault, ACA pulls them at deployment time via its identity, and no plain value ever touches the Bicep template. If the secret in Key Vault rotates, ACA needs a new revision to pick up the change.

For values that change without a deployment (feature flags, rate limits), pair ACA with Azure App Configuration and the `Microsoft.Extensions.Configuration.AzureAppConfiguration` package. The app reloads the values without a restart.

## When ACA is the right choice

Azure Container Apps is the right host for:

- **Container-native workloads inside Azure** that would otherwise go on Kubernetes but do not need the full control or complexity.
- **Event-driven services** (queue consumers, background workers, Kafka processors) that benefit from scale-to-zero.
- **Microservices where you want service-to-service calls, pub/sub, or state management** to be abstracted via Dapr.
- **Teams that have container expertise but no Kubernetes operations budget.**
- **Traffic-splitting-heavy release strategies**: canary, blue/green, A/B, where the built-in revision system removes the need for custom rollout tooling.

It is *not* the right choice when:

- **You need full Kubernetes control**: custom CRDs, operators, NetworkPolicies, cluster-wide customization. Go to AKS (the [Kubernetes article](/posts/hosting-kubernetes/)).
- **You run a single small web app with steady traffic and no containers**: [Azure Web App](/posts/hosting-azure-web-app/) is simpler still and often cheaper.
- **Your team is not on Azure**: porting ACA's model to AWS or GCP is non-trivial. If multi-cloud is required, Kubernetes is a better portability layer.
- **Cold start matters and you cannot afford `minReplicas: 1`**: ACA's cold start is 2-5 seconds, which is great for a queue worker and too slow for a user-facing API without always-on replicas.

## Wrap-up

Azure Container Apps gives you the benefits of Kubernetes-class container hosting without the operational weight: revisions, traffic splitting, KEDA autoscaling, ingress with managed certificates, Key Vault-backed secrets, and scale-to-zero for workloads that tolerate cold start. You can deploy an ASP.NET Core image with a Bicep template in an afternoon, combine it with queue-based autoscaling for event-driven workers, split traffic between revisions for canary deployments, and recognize when the workload would be better served by plain [Kubernetes](/posts/hosting-kubernetes/) or [Azure Web App](/posts/hosting-azure-web-app/).

Ready to level up your next project or share it with your team? See you in the next one, Azure Web App is where we go next.

## Related articles

- [Hosting ASP.NET Core with Docker: A Pragmatic Guide](/posts/hosting-docker/)
- [Hosting ASP.NET Core on Kubernetes: The Essentials for .NET Developers](/posts/hosting-kubernetes/)
- [AOT Compilation in .NET: Startup, Size, and the Trade-offs Nobody Mentions](/posts/performance-aot-compilation/)
- [Spike Testing in .NET: Surviving the Sudden Burst](/posts/load-testing-spike/)

## References

- [Azure Container Apps documentation, Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/)
- [Scale an application in Azure Container Apps, Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/scale-app)
- [Manage revisions, Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/revisions)
- [KEDA scalers documentation](https://keda.sh/docs/latest/scalers/)
- [Dapr on Azure Container Apps, Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview)
