---
title: "Hosting ASP.NET Core on Kubernetes: The Essentials for .NET Developers"
date: 2026-04-08
draft: false
tags: ["hosting", "kubernetes", "dotnet"]
categories: ["Hosting"]
series: ["Hosting"]
description: "Deployment, Service, Ingress, probes, resource limits, graceful shutdown. The Kubernetes primitives a .NET developer actually needs to host an ASP.NET Core application in production."
---

Kubernetes is the default orchestrator for container workloads in 2026, and every serious .NET shop eventually hosts at least one application on it. The learning curve has a reputation for being steep, and it is, but the subset that a .NET developer actually needs to understand to ship an ASP.NET Core application is much smaller than the full platform surface. This article covers exactly that subset: the handful of primitives (Deployment, Service, Ingress, probes, resource limits) that turn a [Docker image](/posts/hosting-docker/) into a production-ready workload on Kubernetes.

## Why Kubernetes

Kubernetes gives you five things that are hard to build on plain Docker:

1. **Declarative desired state.** You describe what should run, and the control plane keeps reality in sync with the description. No scripts, no manual recovery. If a pod dies, a new one is started automatically.
2. **Horizontal scaling.** You specify a replica count or an autoscaler rule, and the cluster maintains the right number of instances, distributing them across nodes.
3. **Rolling updates and rollbacks.** Deploying a new version replaces pods one by one without downtime, and rolling back is a single command.
4. **Service discovery and load balancing.** Pods do not need to know about each other's IPs. They talk to named services, and the cluster routes traffic across the healthy instances.
5. **Resource isolation.** Each pod gets CPU and memory limits, enforced by the kernel, so a misbehaving instance cannot starve its neighbors.

These are the same guarantees that justify moving off a single Docker host to an orchestrator in the first place. The cost is a new vocabulary and a new operational model, which is what this article tries to make concrete.

## Overview: the minimum primitives

{{< mermaid >}}
graph TD
    A[Deployment] --> B[ReplicaSet<br/>manages N pods]
    B --> C[Pod 1<br/>your container]
    B --> D[Pod 2]
    B --> E[Pod 3]
    F[Service] --> C
    F --> D
    F --> E
    G[Ingress] --> F
    H[Internet] --> G
{{< /mermaid >}}

For a typical ASP.NET Core web API, the minimum set of Kubernetes resources is:

**Deployment**: declares what the application is (container image, environment variables, probes, resource limits) and how many replicas should run. The Deployment owns a ReplicaSet, which owns the actual pods.

**Service**: gives the pods a stable virtual IP and DNS name inside the cluster, load-balancing traffic across the healthy replicas. Other services talk to your application via the Service, not to individual pods.

**Ingress**: routes external HTTP traffic from outside the cluster to the Service. Handles TLS termination, host-based routing, and path-based routing via an Ingress Controller (NGINX, Traefik, Azure Application Gateway, etc.).

**ConfigMap and Secret**: externalize configuration and secrets from the image. ConfigMaps for non-sensitive values (log level, feature flags), Secrets for anything sensitive (connection strings, API keys).

These four resources cover 80% of what a .NET application on Kubernetes needs. The rest (HorizontalPodAutoscaler, NetworkPolicy, ServiceAccount, ResourceQuota) is built on top of them.

## Zoom: the Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-api
  labels:
    app: shop-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shop-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: shop-api
    spec:
      terminationGracePeriodSeconds: 45
      containers:
        - name: api
          image: myregistry.azurecr.io/shop-api:1.4.7
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
            - name: ConnectionStrings__Default
              valueFrom:
                secretKeyRef:
                  name: shop-api-secrets
                  key: db-connection
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
```

Six details make this a production-ready Deployment instead of a toy one.

**`strategy.rollingUpdate` with `maxUnavailable: 0`** guarantees that at no point during a deployment does the cluster have fewer than the target replica count available. A new pod is created first (`maxSurge: 1`), it passes its readiness probe, then an old pod is terminated. True zero-downtime rollout.

**`resources.requests` and `resources.limits`** are both declared. Requests tell the scheduler how much room to find on a node. Limits are the hard ceiling enforced by the kernel. A pod without resource limits can eat all CPU on its node, starve other pods, and produce cascading failures. A pod without resource requests gets scheduled anywhere and ends up competing for resources unpredictably.

**`livenessProbe` and `readinessProbe`** pair cleanly with the health check endpoints from the [Docker article](/posts/hosting-docker/). Liveness restarts the pod on failure; readiness removes it from the Service endpoints until it recovers. Never merge them into a single probe, because the consequences of failure are different.

**`terminationGracePeriodSeconds: 45`** extends the default 30-second window to give in-flight requests more time to complete. Must match the `HostOptions.ShutdownTimeout` configured in the application.

**`securityContext`** runs the container as non-root with a read-only root filesystem and no privilege escalation. The chiseled .NET images already run as non-root by default, but declaring it at the pod level is a defense-in-depth measure that also works with full images.

**`env` pulls secrets from a Kubernetes Secret** instead of hardcoding connection strings. The Secret is defined separately and injected at runtime, so the Deployment YAML can be committed to source control without leaking credentials.

> 💡 **Info** : Kubernetes resource requests for CPU are in "millicores" (`m`). `100m` means 0.1 of a CPU core. `500m` means half a core. A typical ASP.NET Core API needs 50-200m at idle and 300-500m under load, but only a load test (covered in the [load testing series](/posts/load-testing-overview/)) tells you the real numbers for your application.

## Zoom: the Service and Ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shop-api
spec:
  type: ClusterIP
  selector:
    app: shop-api
  ports:
    - name: http
      port: 80
      targetPort: http
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-api
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.shop.example.com]
      secretName: shop-api-tls
  rules:
    - host: api.shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-api
                port:
                  name: http
```

The Service type is `ClusterIP`, which means it is only reachable from inside the cluster. External traffic goes through the Ingress, which handles TLS termination with a certificate stored in the `shop-api-tls` Secret (typically managed by cert-manager with Let's Encrypt).

The Ingress annotation `proxy-body-size` is NGINX-specific and increases the maximum upload size from the default 1 MB to 10 MB. Annotations like this are the main way to configure ingress controller behavior; each controller has its own set.

> ✅ **Good practice** : Use a single Ingress resource per domain and a single Service per Deployment. Do not try to be clever with shared services or complex routing rules early. Start simple, and only add complexity when a concrete requirement demands it.

## Zoom: rolling updates and pod lifecycle

When a new version of the application ships, a typical rolling update looks like this:

1. The image tag in the Deployment is updated (via `kubectl set image`, a Helm upgrade, an ArgoCD sync, or similar).
2. Kubernetes creates a new ReplicaSet for the new version.
3. One new pod is created and starts up. The container runs. The readiness probe begins polling. Once it returns 200, the pod is added to the Service endpoints and starts receiving traffic.
4. One old pod is marked for termination. It receives `SIGTERM`. ASP.NET Core stops accepting new connections, drains in-flight requests (up to the grace period), flushes logs, and exits cleanly. Kubernetes removes it from the Service endpoints immediately and waits for the process to exit.
5. Steps 3 and 4 repeat until all old pods are replaced.

Three things can go wrong, and they all look similar from the outside but have different causes.

**The new pod never passes readiness.** The old pods stay in place, the rollout stalls. Usually means the application cannot start: bad configuration, a missing secret, a database migration that failed. `kubectl describe pod` and `kubectl logs` are the first places to look.

**The new pod passes readiness, then crashes under traffic.** Liveness probes start failing, the pod restarts, and the `CrashLoopBackOff` state kicks in. Usually means the application depends on something it did not need during readiness (for example, a downstream API that is only called under real traffic).

**In-flight requests fail during rollout.** Usually means the grace period is too short, or the application does not handle `SIGTERM` correctly (see the [Docker article](/posts/hosting-docker/) on signal handling). Requests get dropped when the old pod exits before they finish.

> ⚠️ **It works, but...** : The default ASP.NET Core behavior is to stop accepting connections on `SIGTERM` and finish pending requests. This works in most cases, but if your application holds long-running operations (large uploads, long-polling, WebSockets), tune `terminationGracePeriodSeconds` upward and configure Kestrel's `KeepAliveTimeout` accordingly.

## Zoom: ConfigMap and Secret

Production configuration should live outside the container image. Kubernetes provides two primitives for this.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shop-api-config
data:
  Logging__LogLevel__Default: Information
  FeatureFlags__NewCheckout: "true"
  AllowedHosts: "api.shop.example.com"
---
apiVersion: v1
kind: Secret
metadata:
  name: shop-api-secrets
type: Opaque
stringData:
  db-connection: "Host=postgres;Database=shop;Username=shop;Password=secret"
  jwt-signing-key: "..."
```

ConfigMap for non-sensitive values, Secret for sensitive ones. The double underscore (`__`) convention in key names maps to nested configuration in ASP.NET Core: `Logging__LogLevel__Default` becomes `Logging:LogLevel:Default` in `IConfiguration`.

Secrets in plain YAML are only base64-encoded, not encrypted. For real security, use one of:

- **Sealed Secrets** (Bitnami) for committing encrypted secrets to Git.
- **External Secrets Operator** to pull secrets from Azure Key Vault, AWS Secrets Manager, HashiCorp Vault at runtime.
- **Kubernetes Secrets with encryption at rest** enabled on the cluster (most managed offerings do this by default).

> ❌ **Never do this** : Do not commit plain Secret YAML to Git, even in a private repository. Treat it the way you would treat a password file. Use one of the external secret management patterns instead.

## Zoom: horizontal autoscaling

Once the application is running with manual replica counts, adding a HorizontalPodAutoscaler lets Kubernetes adjust the count automatically based on CPU or custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shop-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shop-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

The HPA scales the Deployment between 3 and 20 pods, aiming to keep average CPU utilization around 70%. The `stabilizationWindowSeconds: 300` on scale-down prevents thrashing: the HPA waits 5 minutes of low CPU before removing a replica, which avoids flapping when load is spiky.

The [spike testing article](/posts/load-testing-spike/) covers the failure mode where HPA reaction is too slow for sudden bursts. If spikes are a real concern for your workload, either run a higher `minReplicas` count, or move to predictive autoscaling via tools like KEDA.

> 💡 **Info** : KEDA (Kubernetes Event-Driven Autoscaling) is the community-standard way to scale Kubernetes workloads based on external signals: queue depth (RabbitMQ, Azure Service Bus, Kafka), Prometheus metrics, HTTP request rate, and many others. For workloads whose load does not correlate with CPU, KEDA is usually the right answer.

## When Kubernetes is the wrong tool

Kubernetes is powerful, but it is also operationally heavy. Running a production-grade cluster means patching nodes, managing an Ingress Controller, maintaining observability, handling certificate rotation, and debugging issues that do not exist on simpler platforms. For small applications (one service, low traffic, one or two developers), this overhead is disproportionate.

If the workload fits one of these shapes, a lighter alternative is often better:

- **Single small service**: [Azure Web App](/posts/hosting-azure-web-app/) or a plain Docker host.
- **Container-native but low operational tolerance**: [Azure Container Apps](/posts/hosting-azure-container-apps/), which gives you most of Kubernetes's benefits without managing the cluster.
- **Serverless / event-driven**: Azure Functions or AWS Lambda, especially when paired with Native AOT from the [performance series](/posts/performance-aot-compilation/) for fast cold start.

Kubernetes pays off when you have multiple services, multiple teams, variable load that benefits from autoscaling, and enough operational capacity to run the cluster. For a single small API with steady traffic, it is overkill.

## Wrap-up

Hosting ASP.NET Core on Kubernetes comes down to a small set of primitives: a Deployment with probes, resource limits, and a security context; a Service for stable internal routing; an Ingress for external traffic with TLS; ConfigMaps and Secrets for externalized configuration; and optionally a HorizontalPodAutoscaler when load varies. You can turn a Docker image into a production-ready Kubernetes workload by combining those, you can achieve true zero-downtime rolling updates with the right probe and grace period configuration, and you can recognize when the operational cost of Kubernetes is not paying off and a simpler platform would serve better.

Ready to level up your next project or share it with your team? See you in the next one, Azure Container Apps is where we go next.

## Related articles

- [Hosting ASP.NET Core on IIS: The Classic, Demystified](/posts/hosting-iis/)
- [Hosting ASP.NET Core with Docker: A Pragmatic Guide](/posts/hosting-docker/)
- [Load Testing for .NET: An Overview of the Four Types That Matter](/posts/load-testing-overview/)
- [Spike Testing in .NET: Surviving the Sudden Burst](/posts/load-testing-spike/)

## References

- [Kubernetes concepts, official docs](https://kubernetes.io/docs/concepts/)
- [Configure Liveness, Readiness and Startup Probes, Kubernetes docs](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Horizontal Pod Autoscaler, Kubernetes docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Deploy ASP.NET Core apps to Kubernetes, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/multi-container-microservice-net-applications/)
- [KEDA documentation](https://keda.sh/)
