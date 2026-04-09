---
title: "Kubernetes Primer for .NET Developers: From kubectl to Helm"
date: 2026-04-09
draft: false
tags: ["deployment", "kubernetes", "dotnet"]
categories: ["Deployment"]
series: ["Deployment"]
description: "A getting-started guide for deploying .NET applications on Kubernetes: kubectl basics, manifest layout, Kustomize for environments, Helm for packaging, and the workflow a .NET team actually needs."
---

The [Hosting series Kubernetes article](/posts/hosting-kubernetes/) covered the primitives you need to run an ASP.NET Core application on Kubernetes: Deployments, Services, Ingress, probes, resource limits. This article covers the other half: **how to actually work with Kubernetes day to day as a .NET developer**. What `kubectl` commands matter, how to organize manifests so that dev, staging, and production do not drift, how Kustomize and Helm fit together, and the concrete workflow a team uses to go from "I wrote a change" to "it is deployed" without hand-crafting YAML for every environment.

The assumption here is that you have read the Hosting article, you understand what a Deployment and a Service are, and you now need to ship the thing. The goal of this primer is to give you the minimum viable tooling to do that, and the judgment to know when to reach for more.

## Why Kubernetes deployment workflow matters

Kubernetes itself is declarative, which is wonderful in theory and unforgiving in practice. A single manifest file with hardcoded values works great for one environment and drifts within a week across three. The gap between "I have a Deployment manifest" and "my team ships reliably to dev, staging, and prod with the same pipeline" is bigger than most introductory tutorials admit.

The workflow this article covers answers four concrete questions:

1. **How do I actually talk to the cluster?** `kubectl` has hundreds of subcommands; maybe fifteen matter for daily work.
2. **How do I keep manifests DRY across environments?** A production deployment needs different resource limits, different replica counts, different secrets, and different ingress hostnames than a dev deployment. Copy-pasting YAML across three folders is the problem that Kustomize and Helm solve.
3. **How do I package and version a release?** A release is not just an image tag. It is a Deployment, a Service, an Ingress, a ConfigMap, a Secret, a HorizontalPodAutoscaler, and whatever else the application needs. All of that should move together.
4. **How do I deploy without touching `kubectl` in production?** Pipelines, GitOps, and reviewable changes are the alternative to "someone typed a command on their laptop".

## Overview: the deployment workflow

{{< mermaid >}}
graph LR
    A[Source code<br/>+ manifests] --> B[CI build]
    B --> C[Image pushed<br/>to registry]
    B --> D[Manifests rendered<br/>Kustomize or Helm]
    D --> E[kubectl apply<br/>or GitOps sync]
    E --> F[Cluster reconciles<br/>desired state]
    F --> G[Running pods]
{{< /mermaid >}}

Every Kubernetes deployment follows the same basic shape. CI builds the image, pushes it to a registry, renders the manifests for the target environment, and applies them to the cluster. The cluster reconciles its running state with the declared state and reports back. The variations between teams are mostly in *how* manifests are rendered and *how* they are applied.

## Zoom: the kubectl commands that matter

Out of everything `kubectl` can do, twelve commands cover 95% of day-to-day work. Learn these first.

```bash
# Where am I?
kubectl config current-context              # which cluster
kubectl config use-context prod             # switch cluster
kubectl get nodes                           # nodes and their status

# What is running?
kubectl get pods -n shop                    # pods in a namespace
kubectl get deployments,svc,ingress -n shop # all common resources at once
kubectl describe pod shop-api-abc123 -n shop  # detailed state of a pod

# Logs and debugging
kubectl logs shop-api-abc123 -n shop --tail=100 --follow
kubectl logs -l app=shop-api -n shop --tail=100  # all pods matching a label
kubectl exec -it shop-api-abc123 -n shop -- /bin/sh  # shell into a pod

# Apply and rollback
kubectl apply -f deployment.yaml             # create or update
kubectl rollout status deployment/shop-api -n shop  # wait for rollout to complete
kubectl rollout undo deployment/shop-api -n shop    # rollback to previous revision

# Port-forward for local debugging
kubectl port-forward svc/shop-api 8080:80 -n shop
```

Two habits save real time. First, set a default namespace so you do not type `-n shop` on every command: `kubectl config set-context --current --namespace=shop`. Second, use `kubectl get` with label selectors (`-l app=shop-api`) to operate on groups of resources, not individual ones.

> 💡 **Info** : `kubectl logs -l app=shop-api --follow` is the command to remember for production log tailing. It aggregates logs from every matching pod in real time, which is what you want when debugging why a specific endpoint is slow across replicas.

## Zoom: manifest layout with Kustomize

A naive approach puts all the manifests in one folder and edits them by hand for each environment. It works for a week and collapses after that. Kustomize solves it with a **base + overlays** pattern that is native to `kubectl` since 1.14.

```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        ├── patch-replicas.yaml
        └── patch-resources.yaml
```

The **base** contains the manifests as they would look in a "default" environment: one replica, minimal resources, no environment-specific values. The **overlays** contain only the *differences* from the base.

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
  - configmap.yaml

commonLabels:
  app: shop-api

images:
  - name: shop-api
    newName: myregistry.azurecr.io/shop-api
    newTag: "1.4.7"
```

```yaml
# k8s/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: shop-prod
resources:
  - ../../base

patches:
  - path: patch-replicas.yaml
  - path: patch-resources.yaml

configMapGenerator:
  - name: shop-api-config
    behavior: merge
    literals:
      - Logging__LogLevel__Default=Warning
      - ASPNETCORE_ENVIRONMENT=Production

images:
  - name: shop-api
    newTag: "1.4.7"
```

```yaml
# k8s/overlays/prod/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-api
spec:
  replicas: 5
```

Three things Kustomize gives you for free. **Namespace substitution**: the overlay declares `namespace: shop-prod` and every resource in the overlay gets deployed there, without editing the base. **Patch-based overrides**: the replica count and resource limits live in small patch files that only describe the delta from the base. **ConfigMap generation with merge semantics**: environment-specific values are layered on top of base values without duplicating the full ConfigMap.

Rendering and applying is a single command:

```bash
# Preview what will be applied
kubectl kustomize k8s/overlays/prod

# Actually apply it
kubectl apply -k k8s/overlays/prod
```

> ✅ **Good practice** : Always `kubectl kustomize` before `kubectl apply` in a new environment. Diffing the rendered output against the current cluster state with `kubectl diff -k ...` shows exactly what will change, which is the closest thing to a dry run Kubernetes offers.

## Zoom: Helm for packaging and reuse

Kustomize is excellent for a team's own manifests. **Helm** solves a different problem: packaging manifests as a reusable artifact that can be versioned, shared, and deployed with parameters. If Kustomize is "my team's manifests, per environment", Helm is "a packaged unit I can install, upgrade, and uninstall like a library".

The practical use cases where Helm wins:

1. **Installing third-party components.** NGINX Ingress Controller, cert-manager, Prometheus, Grafana, external-secrets: all of them ship as Helm charts and installing them is a one-line command.
2. **Packaging your own application for multiple consumers.** A .NET service that multiple teams deploy (a shared auth service, a shared observability agent) is easier as a chart with parameters than as a set of manifests each team has to copy.
3. **Upgrades and rollbacks as first-class operations.** `helm upgrade` and `helm rollback` track the release history in the cluster itself, which is cleaner than manually tracking Git commits.

A minimal chart for the .NET API:

```
chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── _helpers.tpl
```

```yaml
# Chart.yaml
apiVersion: v2
name: shop-api
version: 1.4.7
appVersion: "1.4.7"
description: Shop API service
```

```yaml
# values.yaml
image:
  repository: myregistry.azurecr.io/shop-api
  tag: "1.4.7"
  pullPolicy: IfNotPresent

replicaCount: 3

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: true
  className: nginx
  host: api.shop.example.com
  tls:
    enabled: true
    secretName: shop-api-tls
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "shop-api.fullname" . }}
  labels:
    {{- include "shop-api.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "shop-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "shop-api.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

Installation:

```bash
helm install shop-api ./chart --namespace shop-prod --create-namespace \
  --set image.tag=1.4.7 \
  --set replicaCount=5
```

Upgrade:

```bash
helm upgrade shop-api ./chart --namespace shop-prod \
  --set image.tag=1.4.8
```

Rollback:

```bash
helm rollback shop-api --namespace shop-prod  # back to previous revision
helm rollback shop-api 3 --namespace shop-prod  # back to revision 3 specifically
```

> ⚠️ **It works, but...** : Helm templates are Go text/template over YAML, which is a combination that does not always degrade gracefully. A misplaced indent in a template can produce valid-looking but semantically wrong YAML. `helm template ./chart -f values-prod.yaml | kubectl apply --dry-run=server -f -` is the standard way to catch these before they reach the cluster.

## Zoom: Kustomize or Helm, which one

The choice is not either/or. Most mature Kubernetes setups use both.

**Use Helm for** third-party components, shared services, and anything you publish to a chart repository. Its strength is packaging and upgrade semantics.

**Use Kustomize for** your own team's services, where you control both the base manifests and the overlays. Its strength is simplicity: no templating language, no helpers, just YAML patches.

**Combine them** by using Kustomize to post-process Helm output. Helm renders a chart with base values, Kustomize applies team-specific overrides on top. This is the pattern most production clusters land on after a year or two of experimentation.

```yaml
# kustomization.yaml
helmCharts:
  - name: ingress-nginx
    repo: https://kubernetes.github.io/ingress-nginx
    version: 4.10.0
    releaseName: ingress
    namespace: ingress-nginx
    valuesFile: values-ingress.yaml

patches:
  - path: patch-ingress-resources.yaml
```

> 💡 **Info** : `kubectl` ships with Kustomize built in, but not Helm. Installing Helm is a one-line script (`curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`), and most Kubernetes environments already have it.

## Zoom: GitOps with Flux or ArgoCD

Once the manifests are organized and the rendering works, the next step is removing humans from the deployment path entirely. **GitOps** is the pattern where the cluster continuously reconciles itself with a Git repository: the repository is the source of truth, and the cluster polls it for changes and applies them automatically.

The two widely-used tools are **Flux** and **ArgoCD**. Both work the same way: you install a controller in the cluster, point it at a Git repository, and every change merged to the main branch of that repository is applied to the cluster within seconds. Rollback is a `git revert`.

Minimal ArgoCD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shop-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/shop-manifests.git
    path: overlays/prod
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: shop-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

After this is applied once, ArgoCD watches the `overlays/prod` folder of the manifests repository. Any merge to `main` triggers an automatic sync. The `selfHeal: true` option means the cluster auto-corrects drift: if someone manually edits a resource with `kubectl`, ArgoCD reverts it to match Git.

The benefits are concrete: every deployment is a pull request with reviewers, every rollback is a Git revert, and every environment is auditable by looking at Git history.

> ✅ **Good practice** : Keep application source code and manifests in separate repositories. `shop-api` has the C# code; `shop-manifests` has the YAML. This separation lets the CI pipeline push manifest updates (new image tag) without polluting the code repository history, and it gives the operations team a clear boundary.

## When this is overkill

Everything in this article assumes the team actually runs on Kubernetes and intends to keep doing so. If the current setup is a single container on [Azure Web App](/posts/hosting-azure-web-app/), jumping to Kustomize + Helm + GitOps is overkill. Start with the hosting option that fits the size of the team and the workload, and adopt this toolchain when the scale justifies it.

Rough thresholds:

- **One or two services, one team**: `kubectl apply -f` with plain manifests is fine.
- **A handful of services, one environment other than dev**: add Kustomize.
- **Many services, multiple environments, multiple teams**: add Helm (for shared components) and GitOps (for the deployment pipeline).

## Wrap-up

Deploying .NET applications on Kubernetes as a day-to-day workflow comes down to a small set of tools and habits: fifteen `kubectl` commands for everything operational, Kustomize for base-plus-overlays manifest management, Helm for packaging and third-party charts, and GitOps with Flux or ArgoCD when the scale justifies removing humans from the deployment path. You can adopt these incrementally, start with just Kustomize, add Helm when it pays off, and reach GitOps when "who deployed what when" becomes a real question. You can avoid the common failure mode of copy-pasted YAML across environments, and you can give your team a deployment workflow that is reviewable, auditable, and reversible.

Ready to level up your next project or share it with your team? See you in the next one, .NET Aspire is where we go next.

## Related articles

- [Hosting ASP.NET Core on Kubernetes: The Essentials for .NET Developers](/posts/hosting-kubernetes/)
- [Docker for .NET Deployment: Dockerfile and Compose in Practice](/posts/deployment-docker-dockerfile-compose/)
- [Docker Security for .NET: Hardening, Scanning, and Supply Chain](/posts/deployment-docker-security/)

## References

- [kubectl cheat sheet, Kubernetes docs](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kustomize documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [Helm documentation](https://helm.sh/docs/)
- [ArgoCD documentation](https://argo-cd.readthedocs.io/)
- [Flux documentation](https://fluxcd.io/docs/)
- [Deploy a .NET application to Kubernetes, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/multi-container-microservice-net-applications/)
