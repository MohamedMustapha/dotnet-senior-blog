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

## Zoom : le manifest naïf en un seul fichier

Chaque tutoriel "démarrer avec Kubernetes" commence de la même façon : un seul fichier YAML, tout inliné.

```yaml
# k8s/all-in-one.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shop-api
  template:
    metadata:
      labels:
        app: shop-api
    spec:
      containers:
        - name: api
          image: myregistry.azurecr.io/shop-api:latest
          ports:
            - containerPort: 8080
          env:
            - name: ConnectionStrings__Default
              value: "Host=postgres;Database=shop;Username=admin;Password=supersecret"
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
---
apiVersion: v1
kind: Service
metadata:
  name: shop-api
spec:
  selector:
    app: shop-api
  ports:
    - port: 80
      targetPort: 8080
```

Cela fonctionne pour une démo de cinq minutes. Dans un contexte réel, plusieurs problèmes apparaissent en même temps :

- **Tag `latest`.** Impossible de savoir quelle version tourne, impossible de faire un rollback vers un build connu, et le comportement de pull d'image dépend du cache du noeud.
- **Secrets en clair dans le manifest.** La chaîne de connexion est visible par toute personne qui peut lire le fichier, et elle finira dans le contrôle de version.
- **Aucune limite de ressources.** Une fuite mémoire dans le container peut faire tomber le noeud entier.
- **Aucune probe.** Kubernetes n'a aucun moyen de savoir si l'application est en bonne santé, et continue de router du trafic vers un pod cassé.
- **Pas d'ingress.** Le service n'est accessible qu'à l'intérieur du cluster.
- **Tout dans un seul fichier.** Comparer les différences entre environnements est pénible, et modifier une seule valeur implique d'éditer un fichier qui contient aussi des ressources sans rapport.

Dès qu'un environnement de staging apparaît à côté de la production, cette approche s'effondre. Les sections suivantes montrent comment décomposer proprement.

## Zoom : décomposer en manifests production-grade

Un layout production-grade sépare chaque ressource dans son propre fichier :

```
k8s/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── configmap.yaml
├── secret.yaml          # ou mieux : utiliser external-secrets
└── hpa.yaml
```

**deployment.yaml** avec probes, limites de ressources et tag d'image fixé :

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shop-api
  template:
    metadata:
      labels:
        app: shop-api
    spec:
      containers:
        - name: api
          image: myregistry.azurecr.io/shop-api:1.4.7
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: shop-api-config
            - secretRef:
                name: shop-api-secrets
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

**configmap.yaml** pour la configuration non sensible :

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shop-api-config
data:
  ASPNETCORE_ENVIRONMENT: Production
  Logging__LogLevel__Default: Warning
  Logging__LogLevel__Microsoft.AspNetCore: Warning
```

**secret.yaml** pour les valeurs sensibles :

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: shop-api-secrets
type: Opaque
stringData:
  ConnectionStrings__Default: "Host=postgres;Database=shop;Username=admin;Password=supersecret"
```

`stringData` accepte du texte brut et Kubernetes l'encode en base64 au repos. Il s'agit d'encodage, pas de chiffrement : toute personne ayant un accès en lecture au namespace peut le décoder. Pour tout ce qui dépasse un cluster de dev local, l'étape suivante est [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) ou l'[External Secrets Operator](https://external-secrets.io/) pointant vers un vrai coffre-fort.

> ❌ **Ne jamais faire** : Ne jamais commiter un manifest Secret en clair dans git. Utiliser `kubectl create secret` de façon impérative, ou utiliser Sealed Secrets / External Secrets Operator pour garder les secrets complètement hors du contrôle de version.

**hpa.yaml** pour l'autoscaling horizontal :

```yaml
# k8s/hpa.yaml
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
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Chaque fichier a une responsabilité unique, chacun peut être diffé indépendamment, et chacun peut être surchargé par environnement avec des patches Kustomize (couvert dans la section suivante).

## Zoom : de Compose à Kubernetes avec kompose

Les équipes qui viennent de [Docker Compose](/fr/posts/deployment-docker-dockerfile-compose/) peuvent utiliser `kompose` comme chemin de migration. L'outil lit un `compose.yaml` et génère des manifests Kubernetes :

```bash
kompose convert -f compose.yaml -o k8s/
```

Pour un fichier Compose typique avec un service API et une base de données, `kompose` génère un Deployment et un Service pour chaque container, et des PersistentVolumeClaims pour les volumes nommés. La structure de base est correcte et le temps de scaffolding initial est réduit.

Ce que `kompose` ne génère pas, et qu'il faut ajouter manuellement :

- **Probes de liveness et readiness.** Compose a `healthcheck`, mais `kompose` ne le mappe pas toujours correctement vers les probes Kubernetes.
- **Requests et limits de ressources.** Le `deploy.resources` de Compose n'est que partiellement supporté.
- **Ingress.** Compose n'a pas de concept équivalent, il n'y a donc rien à convertir.
- **Secrets.** Les `secrets` et variables d'environnement de Compose sont convertis en ConfigMaps, pas en Secrets Kubernetes.
- **Claims de volumes.** Les PVCs générés utilisent des storage classes et des modes d'accès par défaut qui ne correspondent pas forcément au cluster cible.

La checklist de nettoyage post-kompose :

1. Ajouter des probes de readiness et liveness à chaque Deployment.
2. Fixer les requests et limits de ressources.
3. Remplacer les entrées de ConfigMap contenant des secrets par des ressources Secret (ou external-secrets).
4. Vérifier les tailles de PVC et les storage classes générées.
5. Ajouter une ressource Ingress.
6. Fixer les tags d'image (`kompose` conserve le tag du fichier Compose, qui est souvent `latest`).

> ⚠️ **Ça marche, mais...** : `kompose` est un point de départ, pas une sortie production. Les fichiers générés sont du scaffolding, et chacun d'entre eux nécessite une relecture avant d'être appliqué sur un vrai cluster.

## Zoom : passer les secrets depuis la CI/CD

Les secrets doivent vivre dans le store de secrets de la plateforme CI/CD et être injectés au moment du déploiement, jamais committés dans git. Voici deux pipelines concrets.

**GitHub Actions :**

```yaml
- name: Deploy to AKS
  run: |
    kubectl create secret generic shop-secrets \
      --from-literal=ConnectionStrings__Default="${{ secrets.DB_CONNECTION }}" \
      --from-literal=PaymentGateway__ApiKey="${{ secrets.PAYMENT_KEY }}" \
      --namespace shop-prod \
      --dry-run=client -o yaml | kubectl apply -f -
    
    kubectl set image deployment/shop-api \
      api=myregistry.azurecr.io/shop-api:${{ github.sha }} \
      --namespace shop-prod
```

**Azure DevOps :**

```yaml
- task: Kubernetes@1
  inputs:
    connectionType: 'Azure Resource Manager'
    azureSubscriptionEndpoint: '$(azureSubscription)'
    azureResourceGroup: '$(resourceGroup)'
    kubernetesCluster: '$(aksCluster)'
    command: 'apply'
    arguments: '-k k8s/overlays/prod'
    secretType: 'generic'
    secretArguments: '--from-literal=ConnectionStrings__Default=$(DB_CONNECTION)'
```

Le pattern `--dry-run=client -o yaml | kubectl apply -f -` dans l'exemple GitHub Actions mérite une explication : `kubectl create secret` échoue normalement si le secret existe déjà. En le rendant en YAML avec `--dry-run=client` et en le pipant vers `kubectl apply`, la commande devient idempotente, elle crée le secret au premier lancement et le met à jour aux lancements suivants sans erreur.

Pour la gestion de secrets production-grade à grande échelle, l'[External Secrets Operator](https://external-secrets.io/) pointant vers Azure Key Vault, ou la fédération OIDC de GitHub pour une authentification sans clé vers les fournisseurs cloud, sont les approches recommandées. Elles suppriment entièrement le besoin de credentials à longue durée de vie dans les variables CI/CD.

> ✅ **Bonne pratique** : Les secrets vivent dans les variables CI/CD (GitHub Secrets, Azure DevOps Variable Groups), jamais dans git, jamais dans les fichiers de values Helm. Le pipeline CI/CD est le seul endroit où les valeurs de secrets sont résolues, et elles sont injectées dans le cluster au moment du déploiement.

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
