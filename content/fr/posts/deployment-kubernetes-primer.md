---
title: "Kubernetes : l'essentiel pour les devs .NET, de kubectl à Helm"
date: 2026-04-09
draft: false
tags: ["deployment", "kubernetes", "dotnet"]
categories: ["Deployment"]
series: ["Deployment"]
description: "Un guide de démarrage pour déployer des applications .NET sur Kubernetes : bases de kubectl, layout des manifests, Kustomize pour les environnements, Helm pour le packaging, et le workflow dont une équipe .NET a réellement besoin."
---

Hello tous le monde, aujourd'hui on va découvrir **Kubernetes vu sous l'angle déploiement**, et le workflow quotidien qui transforme des manifests en releases fiables.

L'[article Kubernetes de la série Hosting](/fr/posts/hosting-kubernetes/) a couvert les primitives nécessaires pour faire tourner une application ASP.NET Core sur Kubernetes : Deployments, Services, Ingress, probes, limites de ressources. Cet article couvre l'autre moitié : **comment travailler au quotidien avec Kubernetes en tant que dev .NET**. Quelles commandes `kubectl` comptent, comment organiser les manifests pour que dev, staging et prod ne dérivent pas, comment Kustomize et Helm s'articulent, et le workflow concret qu'une équipe utilise pour passer de "j'ai écrit un changement" à "c'est déployé" sans fabriquer du YAML à la main pour chaque environnement.

L'hypothèse ici est que l'article Hosting a été lu, que ce qu'est un Deployment et un Service est compris, et qu'il faut maintenant livrer la chose. Le but de ce primer est de donner le minimum viable d'outillage, et le jugement pour savoir quand en ajouter.

## Le contexte : pourquoi le workflow de déploiement Kubernetes compte

Kubernetes lui-même est déclaratif, ce qui est magnifique en théorie et impitoyable en pratique. Un seul fichier manifest avec des valeurs en dur marche très bien pour un environnement et dérive en une semaine sur trois. L'écart entre "j'ai un manifest Deployment" et "mon équipe livre de façon fiable en dev, staging et prod avec le même pipeline" est plus grand que ce que la plupart des tutoriels d'introduction admettent.

Le workflow couvert ici répond à quatre questions concrètes :

1. **Comment on parle réellement au cluster ?** `kubectl` a des centaines de sous-commandes ; peut-être quinze comptent pour le travail quotidien.
2. **Comment garder les manifests DRY entre les environnements ?** Un déploiement de prod a besoin de limites de ressources différentes, d'un nombre de réplicas différent, de secrets différents, et d'un hostname d'ingress différent d'un déploiement de dev. Copier-coller du YAML entre trois dossiers est exactement le problème que Kustomize et Helm résolvent.
3. **Comment packager et versionner une release ?** Une release n'est pas juste un tag d'image. C'est un Deployment, un Service, un Ingress, un ConfigMap, un Secret, un HorizontalPodAutoscaler, et tout ce dont l'application a besoin. Tout cela doit bouger ensemble.
4. **Comment déployer sans toucher à `kubectl` en prod ?** Les pipelines, le GitOps, et les changements revus sont l'alternative à "quelqu'un a tapé une commande sur son laptop".

## Vue d'ensemble : le workflow de déploiement

{{< mermaid >}}
graph LR
    A[Code source<br/>+ manifests] --> B[Build CI]
    B --> C[Image poussée<br/>au registry]
    B --> D[Manifests rendus<br/>Kustomize ou Helm]
    D --> E[kubectl apply<br/>ou sync GitOps]
    E --> F[Cluster réconcilie<br/>l'état désiré]
    F --> G[Pods qui tournent]
{{< /mermaid >}}

Chaque déploiement Kubernetes suit la même forme de base. La CI build l'image, la pousse au registry, rend les manifests pour l'environnement cible, et les applique au cluster. Le cluster réconcilie son état courant avec l'état déclaré et rapporte le résultat. Les variations entre équipes se jouent surtout dans le *comment* les manifests sont rendus et *comment* ils sont appliqués.

## Zoom : les commandes kubectl qui comptent

Parmi tout ce que `kubectl` peut faire, douze commandes couvrent 95% du travail quotidien. À apprendre en premier.

```bash
# Où suis-je ?
kubectl config current-context              # quel cluster
kubectl config use-context prod             # changer de cluster
kubectl get nodes                           # nœuds et leur statut

# Qu'est-ce qui tourne ?
kubectl get pods -n shop                    # pods dans un namespace
kubectl get deployments,svc,ingress -n shop # toutes les ressources courantes d'un coup
kubectl describe pod shop-api-abc123 -n shop  # état détaillé d'un pod

# Logs et debug
kubectl logs shop-api-abc123 -n shop --tail=100 --follow
kubectl logs -l app=shop-api -n shop --tail=100  # tous les pods qui matchent un label
kubectl exec -it shop-api-abc123 -n shop -- /bin/sh  # shell dans un pod

# Apply et rollback
kubectl apply -f deployment.yaml             # créer ou mettre à jour
kubectl rollout status deployment/shop-api -n shop  # attendre que le rollout termine
kubectl rollout undo deployment/shop-api -n shop    # rollback à la revision précédente

# Port-forward pour le debug local
kubectl port-forward svc/shop-api 8080:80 -n shop
```

Deux habitudes économisent du vrai temps. D'abord, fixer un namespace par défaut pour ne pas taper `-n shop` sur chaque commande : `kubectl config set-context --current --namespace=shop`. Ensuite, utiliser `kubectl get` avec des sélecteurs de labels (`-l app=shop-api`) pour opérer sur des groupes de ressources, pas sur des ressources individuelles.

> 💡 **Info** : `kubectl logs -l app=shop-api --follow` est la commande à retenir pour du log tailing en prod. Elle agrège les logs de chaque pod qui matche en temps réel, ce qui est exactement ce qu'il faut pour debugger pourquoi un endpoint précis est lent sur plusieurs réplicas.

## Zoom : layout des manifests avec Kustomize

Une approche naïve met tous les manifests dans un dossier et les édite à la main pour chaque environnement. Cela marche une semaine et s'effondre après. Kustomize résout le problème avec un pattern **base + overlays** natif à `kubectl` depuis la 1.14.

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

La **base** contient les manifests tels qu'ils seraient dans un environnement "par défaut" : un replica, ressources minimales, aucune valeur spécifique à un environnement. Les **overlays** ne contiennent que les *différences* par rapport à la base.

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

Trois choses que Kustomize donne gratuitement. **Substitution de namespace** : l'overlay déclare `namespace: shop-prod` et toutes les ressources de l'overlay sont déployées là, sans éditer la base. **Overrides par patches** : le nombre de réplicas et les limites de ressources vivent dans de petits fichiers de patch qui ne décrivent que le delta par rapport à la base. **Génération de ConfigMap avec sémantique de merge** : les valeurs spécifiques à un environnement se superposent aux valeurs de base sans dupliquer le ConfigMap complet.

Rendre et appliquer se fait en une seule commande :

```bash
# Prévisualiser ce qui sera appliqué
kubectl kustomize k8s/overlays/prod

# Appliquer pour de vrai
kubectl apply -k k8s/overlays/prod
```

> ✅ **Bonne pratique** : Toujours faire `kubectl kustomize` avant `kubectl apply` sur un nouvel environnement. Differ la sortie rendue contre l'état courant du cluster avec `kubectl diff -k ...` montre exactement ce qui va changer, ce qui est le plus proche d'un dry run que Kubernetes offre.

## Zoom : Helm pour le packaging et la réutilisation

Kustomize est excellent pour les manifests d'une équipe. **Helm** résout un autre problème : packager les manifests en artefact réutilisable qui peut être versionné, partagé, et déployé avec des paramètres. Si Kustomize c'est "les manifests de mon équipe, par environnement", Helm c'est "une unité packagée que je peux installer, mettre à jour, et désinstaller comme une bibliothèque".

Les cas pratiques où Helm gagne :

1. **Installer des composants tiers.** NGINX Ingress Controller, cert-manager, Prometheus, Grafana, external-secrets : tous sont livrés comme des charts Helm et s'installent en une commande.
2. **Packager sa propre application pour plusieurs consommateurs.** Un service .NET que plusieurs équipes déploient (un service d'auth partagé, un agent d'observabilité partagé) est plus facile en chart avec des paramètres qu'en jeu de manifests que chaque équipe doit copier.
3. **Upgrades et rollbacks comme opérations de première classe.** `helm upgrade` et `helm rollback` suivent l'historique des releases dans le cluster lui-même, ce qui est plus propre que suivre manuellement les commits Git.

Une chart minimale pour l'API .NET :

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

Installation :

```bash
helm install shop-api ./chart --namespace shop-prod --create-namespace \
  --set image.tag=1.4.7 \
  --set replicaCount=5
```

Upgrade :

```bash
helm upgrade shop-api ./chart --namespace shop-prod \
  --set image.tag=1.4.8
```

Rollback :

```bash
helm rollback shop-api --namespace shop-prod  # revient à la révision précédente
helm rollback shop-api 3 --namespace shop-prod  # revient à la révision 3 précisément
```

> ⚠️ **Ça marche, mais...** : Les templates Helm sont du Go text/template par-dessus du YAML, une combinaison qui ne se dégrade pas toujours proprement. Une indentation mal placée dans un template peut produire du YAML qui a l'air valide mais qui est sémantiquement faux. `helm template ./chart -f values-prod.yaml | kubectl apply --dry-run=server -f -` est la façon standard d'attraper ça avant que cela n'atteigne le cluster.

## Zoom : Kustomize ou Helm, lequel

Le choix n'est pas soit l'un soit l'autre. La plupart des setups Kubernetes matures utilisent les deux.

**Utiliser Helm pour** les composants tiers, les services partagés, et tout ce qu'on publie dans un dépôt de charts. Sa force est le packaging et la sémantique d'upgrade.

**Utiliser Kustomize pour** les services de sa propre équipe, où on contrôle à la fois les manifests de base et les overlays. Sa force est la simplicité : pas de langage de template, pas de helpers, juste des patches YAML.

**Combiner les deux** en utilisant Kustomize pour post-traiter la sortie de Helm. Helm rend une chart avec des valeurs de base, Kustomize applique les overrides spécifiques à l'équipe par-dessus. C'est le pattern sur lequel atterrissent la plupart des clusters de prod après un an ou deux d'expérimentation.

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

> 💡 **Info** : `kubectl` livre Kustomize nativement, mais pas Helm. Installer Helm est un script d'une ligne (`curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`), et la plupart des environnements Kubernetes l'ont déjà.

## Zoom : GitOps avec Flux ou ArgoCD

Une fois les manifests organisés et le rendering qui marche, l'étape suivante est de retirer les humains du chemin de déploiement entièrement. **GitOps** est le pattern où le cluster réconcilie en continu avec un repository Git : le repository est la source de vérité, et le cluster le poll pour les changements et les applique automatiquement.

Les deux outils largement utilisés sont **Flux** et **ArgoCD**. Les deux marchent de la même façon : on installe un controller dans le cluster, on le pointe vers un repository Git, et chaque changement mergé dans la branche principale de ce repository est appliqué au cluster en quelques secondes. Le rollback est un `git revert`.

Une Application ArgoCD minimale :

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

Après avoir appliqué cela une fois, ArgoCD surveille le dossier `overlays/prod` du repository de manifests. Tout merge dans `main` déclenche une sync automatique. L'option `selfHeal: true` signifie que le cluster auto-corrige la dérive : si quelqu'un édite manuellement une ressource avec `kubectl`, ArgoCD la ramène à l'état Git.

Les bénéfices sont concrets : chaque déploiement est une pull request avec des relecteurs, chaque rollback est un `git revert`, et chaque environnement est auditable en regardant l'historique Git.

> ✅ **Bonne pratique** : Garder le code source de l'application et les manifests dans des repositories séparés. `shop-api` a le code C# ; `shop-manifests` a les YAML. Cette séparation laisse le pipeline CI pousser des mises à jour de manifests (nouveau tag d'image) sans polluer l'historique du repository de code, et donne à l'équipe d'ops une frontière claire.

## Quand c'est surdimensionné

Tout ce qui est dans cet article suppose que l'équipe tourne réellement sur Kubernetes et a l'intention de continuer. Si le setup courant est un seul container sur [Azure Web App](/fr/posts/hosting-azure-web-app/), passer à Kustomize + Helm + GitOps est surdimensionné. Commencer avec l'option d'hébergement qui correspond à la taille de l'équipe et à la charge, et adopter cette chaîne d'outils quand l'échelle le justifie.

Seuils approximatifs :

- **Un ou deux services, une équipe** : `kubectl apply -f` avec des manifests bruts suffit.
- **Une poignée de services, un environnement autre que dev** : ajouter Kustomize.
- **Beaucoup de services, plusieurs environnements, plusieurs équipes** : ajouter Helm (pour les composants partagés) et GitOps (pour le pipeline de déploiement).

## Wrap-up

Déployer des applications .NET sur Kubernetes au quotidien se résume à un petit jeu d'outils et d'habitudes : quinze commandes `kubectl` pour tout ce qui est opérationnel, Kustomize pour la gestion de manifests en base-plus-overlays, Helm pour le packaging et les charts tierces, et GitOps avec Flux ou ArgoCD quand l'échelle justifie de retirer les humains du chemin de déploiement. Tu peux les adopter de façon incrémentale, commencer avec juste Kustomize, ajouter Helm quand cela rentabilise, et atteindre GitOps quand "qui a déployé quoi quand" devient une vraie question. Tu peux éviter le mode de défaillance classique du YAML copié-collé entre environnements, et tu peux donner à l'équipe un workflow de déploiement relisable, auditable, et réversible.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Héberger ASP.NET Core sur Kubernetes : l'essentiel pour les devs .NET](/fr/posts/hosting-kubernetes/)
- [Docker pour le déploiement .NET : Dockerfile et Compose en pratique](/fr/posts/deployment-docker-dockerfile-compose/)
- [Docker : bonnes pratiques de sécurité pour .NET](/fr/posts/deployment-docker-security/)

## Références

- [Cheat sheet kubectl, docs Kubernetes](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Documentation Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [Documentation Helm](https://helm.sh/docs/)
- [Documentation ArgoCD](https://argo-cd.readthedocs.io/)
- [Documentation Flux](https://fluxcd.io/docs/)
- [Déployer une application .NET sur Kubernetes, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/architecture/microservices/multi-container-microservice-net-applications/)
