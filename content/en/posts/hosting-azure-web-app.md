---
title: "Hosting ASP.NET Core on Azure Web App"
date: 2026-04-08
draft: false
tags: ["hosting", "azure", "dotnet"]
categories: ["Hosting"]
series: ["Hosting"]
description: "Azure App Service Web App is the simplest production host for an ASP.NET Core application on Azure. Deployment slots, built-in scaling, managed certificates, and when to pick it over containers."
---

Not every .NET application needs containers, Kubernetes, or a service mesh. A surprising number of production workloads are best served by the simplest Azure hosting option available: a Web App on Azure App Service. It has been the default for Microsoft-first .NET teams since 2012, it ships with built-in HTTPS and deployment slots and autoscaling, and for a wide class of applications it is the right answer precisely *because* it is simpler than the container-first alternatives covered earlier in this series.

This article closes the [Hosting series](/posts/hosting-iis/) with Azure Web App: what it is, how to deploy an ASP.NET Core application to it, what the built-in features actually do, and when it wins against [containers on ACA](/posts/hosting-azure-container-apps/) or [Kubernetes](/posts/hosting-kubernetes/).

## Why Azure Web App

Azure App Service launched in 2012 and has been evolving continuously since. Web App is its HTTP workload variant, running ASP.NET, ASP.NET Core, Node.js, Python, Java, and PHP applications on Microsoft-managed infrastructure. For .NET specifically, it has native support: no Dockerfile to write, no image to build, no container registry to manage. The application is published directly, the platform runs it, and all the common concerns (TLS, scaling, monitoring, authentication) are available as flipped switches.

The concrete advantages that still matter in 2026:

1. **Simplicity.** A `dotnet publish` output is all it takes to deploy. No container image, no orchestration, no YAML, no Dockerfile. For a team whose primary skill is writing .NET code, that matches the skill set without imposing a new one.
2. **Built-in deployment slots.** Every Standard tier or higher Web App comes with staging slots. Deploy to a slot, validate, then swap slots atomically. The swap is instant and reversible, which makes blue/green deployments a native feature rather than something you have to orchestrate.
3. **Managed TLS certificates.** App Service Managed Certificates are free, auto-renewing, and wired into the custom domain with one click. No cert-manager, no Let's Encrypt cron job, no expiration alerts.
4. **Autoscaling and Always On.** Scale out rules based on CPU, memory, or custom metrics. The Always On setting prevents the worker from going idle during quiet periods, which eliminates the cold start that plagues serverless alternatives for user-facing workloads.
5. **Integration with the Azure ecosystem.** Managed identity, Key Vault references, Application Insights, App Configuration, private endpoints, VNet integration. All of them are configuration settings, not packages to install.

None of this is unique to Web App. It is all available elsewhere. The value is that it is *all in one place* and accessible without extra tooling.

## Overview: the App Service model

{{< mermaid >}}
graph TD
    A[App Service Plan<br/>compute + pricing tier] --> B[Web App<br/>shop-api]
    A --> C[Web App<br/>shop-admin]
    B --> B1[Production slot]
    B --> B2[Staging slot]
    B2 --> B2D[Deployment]
    B1 --> B1T[100% traffic]
{{< /mermaid >}}

The hierarchy is straightforward and has been stable for a decade.

**App Service Plan** is the underlying compute resource: CPU cores, memory, pricing tier (Basic, Standard, Premium v3, Isolated). Multiple Web Apps can share a single plan, which is the standard way to host related applications on the same compute without needing separate billing lines.

**Web App** is the application. It has a name (used in the default URL `<name>.azurewebsites.net`), a runtime stack (`.NET 10 (LTS)`), a deployment source, configuration settings, and optional features (custom domains, identity, scaling rules).

**Deployment slot** is a separate clone of the Web App with its own URL, its own configuration, and its own deployed code. Non-production slots share the App Service Plan's compute but run independently. The value is the ability to swap slot contents atomically: deploy to staging, warm it up with a few requests, run smoke tests, and swap it into production in seconds.

## Zoom: deploying an ASP.NET Core application

The three most common deployment paths, in order of maturity:

**1. Publish profile from Visual Studio or CLI.** Simplest for a single developer. `dotnet publish` produces the output, `az webapp deploy` (or the Visual Studio publish wizard) pushes it. Good for prototypes, not for teams.

**2. GitHub Actions or Azure DevOps pipeline with `azure/webapps-deploy`.** The standard CI path. The pipeline builds, tests, publishes, and deploys, with a single YAML workflow.

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

Four steps: publish the output, deploy to the staging slot, verify health on the staging slot, swap slots. The production traffic flips to the new version at the moment of the swap, without cold start, because Azure warms up the staging slot before the swap completes.

**3. Container deployment.** If the team already builds Docker images for other hosts, Web App can run a custom container from any registry. Configure the Web App to point at the image, and it becomes a managed container host. This loses the "no Dockerfile to write" benefit but keeps the slot and scaling features.

> ✅ **Good practice** : Always deploy to a staging slot first and swap. Direct deployment to production is a habit from the 2000s. With slots, you pay essentially nothing extra for a pre-production validation step and the ability to instantly roll back.

## Zoom: configuration without rebuilding

Web App exposes its configuration through three layers, in order of precedence:

**Slot-specific application settings.** Environment variables defined on the Web App itself, which become `IConfiguration` entries in ASP.NET Core. The double underscore convention maps to nested keys: `ConnectionStrings__Default` becomes `ConnectionStrings:Default`.

**Key Vault references.** An app setting can contain a reference to a secret in Azure Key Vault, and App Service resolves it at startup using the Web App's managed identity. The actual secret never appears in any configuration file or deployment artifact.

```
ConnectionStrings__Default = @Microsoft.KeyVault(SecretUri=https://shop-kv.vault.azure.net/secrets/db-connection/)
```

**App Configuration integration** via the `Microsoft.Extensions.Configuration.AzureAppConfiguration` package, for values that should reload without restart (feature flags, rate limits, toggles). This pairs especially well with Key Vault for the sensitive values and App Configuration for the dynamic ones.

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

> 💡 **Info** : A "slot-specific" app setting stays with the slot during a swap, while a regular setting swaps with the code. This distinction lets you keep `ASPNETCORE_ENVIRONMENT=Staging` on the staging slot permanently, so the same deployment can be tested in staging mode and flipped to production mode by simply swapping.

## Zoom: scaling

Web App offers two scaling dimensions:

**Scale up** changes the size of the App Service Plan (more CPU, more memory). It is an operation that affects all Web Apps on the plan and takes a minute or two. Used when the current tier is too small for the peak load.

**Scale out** adds more instances of the plan, running copies of the same Web Apps in parallel. Azure load-balances traffic across the instances automatically. Scale out rules can be configured based on CPU, memory, queue length, or custom metrics, with cooldown windows to avoid thrashing.

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
    reserved: true  // required for Linux
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

A plan with 2 instances minimum, 10 maximum, adding one instance whenever the average CPU over 5 minutes exceeds 70%, with a 5-minute cooldown. This is the standard shape for autoscaling a steady-traffic Web App.

> ⚠️ **It works, but...** : Autoscale rules on Web App are reactive, not predictive. A burst that exceeds capacity in 30 seconds (see the [spike testing article](/posts/load-testing-spike/)) is faster than the autoscaler's reaction window. For spike-heavy workloads, either run a higher minimum instance count, or accept the queued latency at the start of each spike.

## Zoom: the Always On setting

Web App puts an idle worker to sleep after 20 minutes of inactivity, exactly like [IIS](/posts/hosting-iis/). This is fine for hobby sites and for dev environments, but for user-facing production workloads it introduces a cold start on every first request after idle, which breaks p99 latency targets.

The fix is a single toggle:

```
General settings → Always On → On
```

This keeps the worker warm at all times. It is available on Basic tier and above (not on Free or Shared). For production traffic, it should always be enabled.

Paired with Always On, the `/health/live` endpoint described in the [Docker hosting article](/posts/hosting-docker/) lets you configure the App Service health check ping to periodically hit the endpoint, ensuring the application stays responsive.

```bicep
siteConfig: {
  alwaysOn: true
  healthCheckPath: '/health/live'
  // ...
}
```

## Zoom: when Web App is the right choice

Web App is the right host for:

- **A single .NET web application with steady traffic.** The simplicity pays off: one resource, one deployment path, one set of settings.
- **Teams whose expertise is .NET, not containers or Kubernetes.** No Dockerfile, no `kubectl`, no orchestration knowledge needed.
- **Applications that benefit from deployment slots**: blue/green without extra tooling, A/B testing, gradual rollout with traffic routing percentages.
- **Microsoft-integrated workloads**: Entra ID authentication, managed identity, Key Vault, Application Insights. All of them plug in as configuration options.
- **Workloads that need the Azure hybrid features**: VNet integration, private endpoints, Hybrid Connections for on-prem integration.

It is *not* the right choice when:

- **Container-native deployment is the requirement.** If the application already ships as a Docker image and the team is on the container-first path, [Azure Container Apps](/posts/hosting-azure-container-apps/) or [Kubernetes](/posts/hosting-kubernetes/) fits better.
- **Multi-cloud or on-prem portability matters.** Web App is an Azure-only offering. Porting it elsewhere means rewriting the hosting layer.
- **The workload is event-driven with low baseline traffic.** Scale-to-zero is not a Web App feature (beyond the free tier). Azure Functions or Azure Container Apps with `minReplicas: 0` serves that pattern better.
- **Workload isolation across many small services is required.** Running one Web App per microservice quickly becomes expensive and operationally heavy compared to sharing a cluster or a Container Apps Environment.

## Wrap-up

Azure Web App is the simplest way to run an ASP.NET Core application in Azure in 2026, and for a meaningful share of workloads it is also the best. You can deploy with a GitHub Actions pipeline in half an hour, use deployment slots for zero-downtime swaps, wire Key Vault references directly into configuration, turn on Always On for predictable latency, and configure CPU-based autoscaling with a Bicep template. You can recognize when the workload would benefit more from [Kubernetes](/posts/hosting-kubernetes/) or [Azure Container Apps](/posts/hosting-azure-container-apps/) and choose the right tool for the shape of the problem rather than applying the same hosting pattern to everything.

Ready to level up your next project or share it with your team? See you in the next one, a++ 👋

## Related articles

- [Hosting ASP.NET Core on IIS: The Classic, Demystified](/posts/hosting-iis/)
- [Hosting ASP.NET Core with Docker: A Pragmatic Guide](/posts/hosting-docker/)
- [Hosting ASP.NET Core on Kubernetes: The Essentials for .NET Developers](/posts/hosting-kubernetes/)
- [Hosting ASP.NET Core on Azure Container Apps](/posts/hosting-azure-container-apps/)

## References

- [App Service overview, Microsoft Learn](https://learn.microsoft.com/en-us/azure/app-service/overview)
- [Deploy to App Service, Microsoft Learn](https://learn.microsoft.com/en-us/azure/app-service/deploy-github-actions)
- [Deployment slots, Microsoft Learn](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots)
- [App Service Managed Certificates, Microsoft Learn](https://learn.microsoft.com/en-us/azure/app-service/configure-ssl-certificate)
- [Autoscale in Azure Monitor, Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-overview)
- [Key Vault references for App Service, Microsoft Learn](https://learn.microsoft.com/en-us/azure/app-service/app-service-key-vault-references)
