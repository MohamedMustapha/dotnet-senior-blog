---
title: "Hosting ASP.NET Core on IIS: The Classic, Demystified"
date: 2026-04-08
draft: false
tags: ["hosting", "iis", "dotnet"]
categories: ["Hosting"]
series: ["Hosting"]
description: "The ASP.NET Core Module, in-process vs out-of-process, the real role of IIS in 2026, and when hosting on IIS is still the right call for a .NET team."
---

For twenty years, IIS was the default answer to "where does this .NET application run". System.Web, ASP.NET WebForms, MVC up to version 5, WCF, WebAPI 2: all of them were tightly coupled to the IIS pipeline and the `HttpRuntime`. When ASP.NET Core shipped in 2016, it was explicitly decoupled from IIS: it ran on its own cross-platform web server, Kestrel, and IIS became optional. Yet a decade later, IIS is still the production target for a meaningful share of .NET shops, usually because an existing on-prem Windows Server, a fleet of legacy applications, or a compliance requirement keeps it in the picture. This article is about hosting ASP.NET Core on IIS in 2026: what IIS actually does, what it does not do, and when this is still the right call.

## Why IIS is still in the picture

The traditional story is "IIS is legacy, move to containers". That story is half right. IIS is clearly not the future, and new greenfield projects rarely start there. But three situations make it the right pragmatic choice:

1. **Existing Windows Server infrastructure with an operations team that knows it.** A company running fifty .NET applications on IIS, with monitoring, deployment pipelines, and runbooks built around IIS, does not benefit from moving one of them to a completely different stack. The integration cost outweighs the marginal hosting gain.
2. **Legacy applications mixed with modern ones.** An ASP.NET Core application that needs to sit next to an ASP.NET WebForms application, share authentication, share SSL certificates, or respond under the same domain is much easier to host on the same IIS than to split across two hosting strategies.
3. **Compliance and policy constraints.** Some environments require specific TLS configurations, HTTP.sys features, Windows Authentication via Kerberos, or integration with Active Directory that is significantly easier on IIS than on Kestrel alone.

None of these make IIS "good". They make IIS *appropriate* in context. The job is to host modern ASP.NET Core correctly on it, not to pretend the constraint does not exist.

## Overview: what IIS actually does

{{< mermaid >}}
graph LR
    A[HTTP request] --> B[HTTP.sys kernel driver]
    B --> C[IIS worker process<br/>w3wp.exe]
    C --> D[ASP.NET Core Module<br/>aspnetcore v2]
    D --> E[Your Kestrel app<br/>dotnet.exe]
    E --> F[Response]
    F --> C
    C --> B
    B --> A
{{< /mermaid >}}

The critical piece to understand is that IIS is no longer the web server for your ASP.NET Core application. It is a **reverse proxy** in front of a Kestrel process that runs your application. The component that makes this work is the **ASP.NET Core Module** (ANCM), a native IIS module installed with the .NET Hosting Bundle. ANCM has two modes, and the choice between them is the single most important hosting decision on IIS.

**In-process mode** (the default since .NET Core 2.2): ANCM loads the CLR inside the IIS worker process `w3wp.exe` directly. Kestrel runs in-process, and requests reach the application through a fast in-memory channel. This is roughly 2-3x faster than out-of-process mode and is the right default for most workloads.

**Out-of-process mode**: ANCM launches a separate `dotnet.exe` child process that hosts Kestrel on a localhost port, then forwards IIS requests to it over HTTP. Slower, but necessary when the application needs to isolate itself from the worker process or use features that do not work in-process (for example, certain kinds of module interop).

> 💡 **Info** : The ASP.NET Core Module was originally called `aspnetcore` (v1), rewritten as `aspnetcorev2` in .NET Core 2.2 to add in-process hosting. Both are installed by the .NET Hosting Bundle; your application picks the active one through its `web.config`. New projects should always start with in-process.

## Zoom: the minimal web.config

ASP.NET Core on IIS still uses a `web.config` file, but it is generated at publish time and its only job is to tell IIS which module to load and which executable to launch. For most applications, the file needs no manual editing:

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

The `hostingModel="inprocess"` attribute is what activates in-process mode. The `processPath` points at the published executable, not at `dotnet.exe`, because a self-contained or framework-dependent publish produces a small native launcher for Windows.

> ✅ **Good practice** : Never edit the generated `web.config` by hand to add environment variables or startup flags. Use the csproj `<EnvironmentVariables>` item group at publish time, or set them on the IIS application pool through the IIS Manager. A hand-edited `web.config` will be overwritten on the next publish.

## Zoom: configuring the application pool

An IIS application pool is the worker process that hosts your application. For ASP.NET Core, a few settings matter more than the defaults.

**.NET CLR Version**: set to **No Managed Code**. This is counter-intuitive, but correct. The setting refers to the *legacy* .NET CLR (System.Web), which ASP.NET Core does not use. Leaving it on "v4.0" loads legacy runtime code into the worker process for no reason.

**Managed Pipeline Mode**: Integrated (the default). Classic mode is for legacy applications that depend on the old ISAPI pipeline.

**Pipeline Mode and Identity**: the default `ApplicationPoolIdentity` is usually correct. For applications that need access to a network share or a SQL Server with integrated authentication, switch to a domain service account dedicated to the pool.

**Recycling**: the default recycles the pool every 1740 minutes (29 hours). For long-running ASP.NET Core applications that hold in-memory caches, this forces a cold start every day, which is disruptive. Either disable time-based recycling or schedule it during a low-traffic window. A modern ASP.NET Core application does not have the leaky legacy runtime behavior that made daily recycles necessary on System.Web.

**Idle timeout**: the default 20 minutes shuts the worker process down when no traffic arrives. This is fine for intranet applications but will produce slow first-request responses after every idle period. For internet-facing applications with continuous traffic, either set it to 0 or configure the Application Initialization module to keep the process warm.

> ⚠️ **It works, but...** : The default recycling and idle timeout settings were designed for System.Web workloads from the 2000s. They are still the defaults in modern IIS. A team that publishes an ASP.NET Core application without reviewing these settings will pay the cold-start tax described in the [spike testing article](/posts/load-testing-spike/) every morning, and will not understand why.

## Zoom: deployment with Web Deploy

The traditional deployment mechanism for IIS is Web Deploy (MSDeploy), which understands how to stop the application pool, copy files, and restart cleanly. A typical release from a CI pipeline uses `dotnet publish` to produce the output, then `msdeploy.exe` to push it to the target server:

```powershell
dotnet publish -c Release -r win-x64 --self-contained false -o .\publish

msdeploy.exe `
  -verb:sync `
  -source:contentPath=.\publish `
  -dest:contentPath="Default Web Site/shop-api",computerName=https://iis.internal:8172/msdeploy.axd,userName=deployer,password=...,authType=basic `
  -enableRule:AppOffline
```

The `AppOffline` rule drops a `App_Offline.htm` file into the root during deployment, which causes IIS to stop forwarding to the application and display a maintenance page. Without it, in-flight requests can fail noisily during file replacement.

For teams that dislike MSDeploy, a simpler `xcopy` or `robocopy` to a shared folder followed by `appcmd recycle apppool /apppool.name:ShopApi` is also a perfectly valid deployment strategy. It is less sophisticated but more scriptable.

> 💡 **Info** : The `App_Offline.htm` mechanism is a holdover from System.Web, but ASP.NET Core still respects it. Dropping a file named exactly `App_Offline.htm` into the application root causes the hosting module to unload the CLR and serve the HTML content as a 503.

## Zoom: observability on IIS

ASP.NET Core applications hosted on IIS retain full access to standard .NET observability: `ILogger`, OpenTelemetry, Application Insights, and any metrics or traces the application emits. The only IIS-specific surfaces worth knowing about are:

**IIS log files** under `C:\inetpub\logs\LogFiles\W3SVC*`. These record every HTTP request at the IIS level, including status code and response time, and they are useful for correlating with application logs when something goes wrong upstream of the application code.

**Event Viewer → Windows Logs → Application**: the ASP.NET Core Module writes startup errors and crashes here. The first place to check when the application fails to start after a deployment.

**`stdoutLogEnabled`** in `web.config`: temporarily setting this to `true` redirects the Kestrel console output to a file. Only enable it while diagnosing a problem and disable it afterward, because the file is not rotated and grows without bound.

```csharp
// Program.cs: route ILogger to Event Log for IIS correlation
builder.Logging.AddEventLog(new EventLogSettings
{
    SourceName = "Shop.Api",
    LogName = "Application"
});
```

This writes log entries visible in Event Viewer, which an operations team familiar with IIS can read without needing a separate log aggregation tool. It is not a replacement for structured logging, but it is a convenient fallback.

## Zoom: when IIS is not the right answer

IIS is not the right host when:

- **Container-native deployment is the target.** If the deployment pipeline ships Docker images and the orchestrator is Kubernetes, Azure Container Apps, or similar, moving through IIS adds a Windows server that does not belong in the path.
- **Cross-platform is a requirement.** Linux production targets rule IIS out entirely.
- **Autoscaling is needed.** IIS scales by adding worker processes or worker hosts; it does not elastically scale by count of instances the way a container platform does. For variable load, a container-based platform is a better match.
- **The team is already Linux-first.** Running a Windows Server for one .NET application in an otherwise Linux environment is the most expensive kind of "hosting choice" because the operational cost is high and the skill base is different.

For these cases, the next articles in this series cover [Docker](/posts/hosting-docker/), [Kubernetes](/posts/hosting-kubernetes/), [Azure Container Apps](/posts/hosting-azure-container-apps/), and [Azure Web App](/posts/hosting-azure-web-app/).

## Wrap-up

IIS in 2026 is a reverse proxy that fronts a Kestrel process via the ASP.NET Core Module in in-process mode, and it is a perfectly reasonable host for an ASP.NET Core application in the right context: existing Windows Server infrastructure, a legacy application mix, or compliance constraints that make another path more expensive. You can set the app pool to "No Managed Code", disable legacy daily recycling, keep the idle timeout sensible, deploy with Web Deploy and the `App_Offline.htm` rule, and wire the ASP.NET Core Module's event log output into your existing operations workflow.

Ready to level up your next project or share it with your team? See you in the next one, Hosting with Docker is where we go next.

## References

- [Host ASP.NET Core on Windows with IIS, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/)
- [ASP.NET Core Module (ANCM) reference, Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/aspnet-core-module)
- [Web Deploy documentation](https://learn.microsoft.com/en-us/iis/publish/using-web-deploy/introduction-to-web-deploy)
- [IIS application pool recycling settings, Microsoft Learn](https://learn.microsoft.com/en-us/iis/manage/managing-your-configuration-settings/changing-default-application-pool-recycling-settings)
