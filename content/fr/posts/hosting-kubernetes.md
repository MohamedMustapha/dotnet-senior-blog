---
title: "Héberger ASP.NET Core sur Kubernetes : l'essentiel pour les devs .NET"
date: 2026-04-08
draft: false
tags: ["hosting", "kubernetes", "dotnet"]
categories: ["Hosting"]
series: ["Hosting"]
description: "Deployment, Service, Ingress, probes, limites de ressources, arrêt gracieux. Les primitives Kubernetes dont un dev .NET a réellement besoin pour héberger une application ASP.NET Core en prod."
---

Hello tous le monde, aujourd'hui on va comprendre l'**hébergement d'ASP.NET Core sur Kubernetes**, en se concentrant sur le sous-ensemble de la plateforme qui compte vraiment pour un dev .NET.

Kubernetes est l'orchestrateur par défaut des charges container en 2026, et toute boutique .NET sérieuse finit par y héberger au moins une application. La courbe d'apprentissage a la réputation d'être raide, et c'est vrai, mais le sous-ensemble qu'un dev .NET doit réellement comprendre pour livrer une application ASP.NET Core est bien plus petit que la surface complète de la plateforme. Cet article couvre exactement ce sous-ensemble : la poignée de primitives (Deployment, Service, Ingress, probes, limites de ressources) qui transforment une [image Docker](/fr/posts/hosting-docker/) en une charge prête pour la prod sur Kubernetes.

## Le contexte : pourquoi Kubernetes

Kubernetes apporte cinq choses difficiles à construire sur du Docker nu :

1. **Un état désiré déclaratif.** On décrit ce qui doit tourner, et le plan de contrôle garde la réalité en phase avec la description. Pas de scripts, pas de récupération manuelle. Si un pod meurt, un nouveau démarre automatiquement.
2. **Le scaling horizontal.** On spécifie un nombre de réplicas ou une règle d'autoscaler, et le cluster maintient le bon nombre d'instances, en les distribuant sur les nœuds.
3. **Les rolling updates et les rollbacks.** Déployer une nouvelle version remplace les pods un à un sans downtime, et revenir en arrière est une seule commande.
4. **La découverte de services et le load balancing.** Les pods n'ont pas besoin de connaître les IPs des autres. Ils parlent à des services nommés, et le cluster route le trafic vers les instances saines.
5. **L'isolation des ressources.** Chaque pod a des limites CPU et mémoire, imposées par le noyau, pour qu'une instance mal élevée ne puisse pas affamer ses voisins.

Ce sont les garanties qui justifient le passage d'un hôte Docker unique à un orchestrateur. Le coût, c'est un nouveau vocabulaire et un nouveau modèle opérationnel, et c'est ce que cet article essaie de rendre concret.

## Vue d'ensemble : les primitives minimales

{{< mermaid >}}
graph TD
    A[Deployment] --> B[ReplicaSet<br/>gère N pods]
    B --> C[Pod 1<br/>ton container]
    B --> D[Pod 2]
    B --> E[Pod 3]
    F[Service] --> C
    F --> D
    F --> E
    G[Ingress] --> F
    H[Internet] --> G
{{< /mermaid >}}

Pour une API web ASP.NET Core typique, le jeu minimal de ressources Kubernetes est :

**Deployment** : déclare ce qu'est l'application (image container, variables d'environnement, probes, limites de ressources) et combien de réplicas doivent tourner. Le Deployment possède un ReplicaSet, qui possède les pods réels.

**Service** : donne aux pods une IP virtuelle et un nom DNS stables à l'intérieur du cluster, et load balance le trafic entre les réplicas sains. Les autres services parlent à l'application via le Service, pas aux pods individuels.

**Ingress** : route le trafic HTTP externe depuis l'extérieur du cluster vers le Service. Gère la terminaison TLS, le routing par hôte, et le routing par chemin via un Ingress Controller (NGINX, Traefik, Azure Application Gateway, etc.).

**ConfigMap et Secret** : externalisent la configuration et les secrets hors de l'image. ConfigMaps pour les valeurs non sensibles (niveau de log, feature flags), Secrets pour tout ce qui est sensible (chaînes de connexion, clés d'API).

Ces quatre ressources couvrent 80% de ce dont une application .NET sur Kubernetes a besoin. Le reste (HorizontalPodAutoscaler, NetworkPolicy, ServiceAccount, ResourceQuota) est construit par-dessus.

## Zoom : le Deployment

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

Six détails font de ce Deployment un Deployment prêt pour la prod plutôt qu'un Deployment de tutoriel.

**`strategy.rollingUpdate` avec `maxUnavailable: 0`** garantit qu'à aucun moment pendant un déploiement le cluster n'a moins que le nombre cible de réplicas disponibles. Un nouveau pod est créé en premier (`maxSurge: 1`), il passe sa readiness probe, puis un ancien pod est terminé. Vrai rollout zero-downtime.

**`resources.requests` et `resources.limits`** sont tous les deux déclarés. Les requests disent au scheduler combien d'espace trouver sur un nœud. Les limits sont le plafond dur imposé par le noyau. Un pod sans limites de ressources peut manger tout le CPU de son nœud, affamer les autres pods, et produire des défaillances en cascade. Un pod sans requests de ressources est planifié n'importe où et finit par se battre pour les ressources de façon imprévisible.

**`livenessProbe` et `readinessProbe`** s'appairent proprement avec les endpoints de health check vus dans l'[article Docker](/fr/posts/hosting-docker/). La liveness redémarre le pod en cas d'échec ; la readiness le retire des endpoints du Service jusqu'à récupération. Ne jamais fusionner les deux dans une seule probe, parce que les conséquences d'un échec sont différentes.

**`terminationGracePeriodSeconds: 45`** étend la fenêtre par défaut de 30 secondes pour laisser aux requêtes en vol plus de temps pour terminer. Doit correspondre au `HostOptions.ShutdownTimeout` configuré dans l'application.

**`securityContext`** fait tourner le container en non-root avec un système de fichiers racine en lecture seule et sans escalade de privilèges. Les images .NET chiseled tournent déjà en non-root par défaut, mais le déclarer au niveau du pod est une mesure de défense en profondeur qui marche aussi avec les images complètes.

**`env` tire les secrets depuis un Secret Kubernetes** au lieu de coder en dur les chaînes de connexion. Le Secret est défini séparément et injecté au runtime, pour que le YAML du Deployment puisse être commité en gestion de version sans fuiter les credentials.

> 💡 **Info** : Les requests de ressources Kubernetes pour le CPU sont en "millicores" (`m`). `100m` veut dire 0,1 de cœur CPU. `500m` veut dire un demi cœur. Une API ASP.NET Core typique a besoin de 50 à 200m au repos et de 300 à 500m sous charge, mais seul un test de charge (couvert dans la [série load testing](/fr/posts/load-testing-overview/)) donne les vrais chiffres pour l'application.

## Zoom : le Service et l'Ingress

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

Le type de Service est `ClusterIP`, ce qui veut dire qu'il n'est joignable que depuis l'intérieur du cluster. Le trafic externe passe par l'Ingress, qui gère la terminaison TLS avec un certificat stocké dans le Secret `shop-api-tls` (typiquement géré par cert-manager avec Let's Encrypt).

L'annotation d'Ingress `proxy-body-size` est spécifique à NGINX et augmente la taille maximale d'upload du défaut de 1 Mo à 10 Mo. Les annotations comme celle-ci sont la façon principale de configurer le comportement d'un ingress controller ; chaque controller a son propre jeu.

> ✅ **Bonne pratique** : Utiliser une seule ressource Ingress par domaine et un seul Service par Deployment. Ne pas essayer d'être malin avec des services partagés ou des règles de routing complexes dès le départ. Commencer simple, et n'ajouter de la complexité que quand un besoin concret l'exige.

## Zoom : rolling updates et cycle de vie des pods

Quand une nouvelle version de l'application est livrée, un rolling update typique ressemble à ça :

1. Le tag de l'image dans le Deployment est mis à jour (via `kubectl set image`, un upgrade Helm, une synchro ArgoCD, ou similaire).
2. Kubernetes crée un nouveau ReplicaSet pour la nouvelle version.
3. Un nouveau pod est créé et démarre. Le container tourne. La readiness probe commence à poller. Une fois qu'elle renvoie 200, le pod est ajouté aux endpoints du Service et commence à recevoir du trafic.
4. Un ancien pod est marqué pour terminaison. Il reçoit `SIGTERM`. ASP.NET Core cesse d'accepter de nouvelles connexions, draine les requêtes en vol (dans la limite de la période de grâce), flush les logs, et sort proprement. Kubernetes le retire des endpoints du Service immédiatement et attend que le process sorte.
5. Les étapes 3 et 4 se répètent jusqu'à ce que tous les anciens pods soient remplacés.

Trois choses peuvent mal tourner, et de l'extérieur elles se ressemblent mais ont des causes différentes.

**Le nouveau pod ne passe jamais la readiness.** Les anciens pods restent en place, le rollout stagne. En général, cela veut dire que l'application ne démarre pas : mauvaise configuration, secret manquant, migration de base qui a échoué. `kubectl describe pod` et `kubectl logs` sont les premiers endroits à regarder.

**Le nouveau pod passe la readiness, puis crashe sous trafic.** Les liveness probes commencent à échouer, le pod redémarre, et l'état `CrashLoopBackOff` se déclenche. En général, cela veut dire que l'application dépend de quelque chose dont elle n'avait pas besoin pendant la readiness (par exemple, une API en aval qui n'est appelée que sous vrai trafic).

**Les requêtes en vol échouent pendant le rollout.** En général, cela veut dire que la période de grâce est trop courte, ou que l'application ne gère pas `SIGTERM` correctement (voir l'[article Docker](/fr/posts/hosting-docker/) sur la gestion des signaux). Les requêtes sont perdues quand l'ancien pod sort avant qu'elles ne terminent.

> ⚠️ **Ça marche, mais...** : Le comportement par défaut d'ASP.NET Core est de cesser d'accepter les connexions sur `SIGTERM` et de finir les requêtes en attente. Cela marche dans la plupart des cas, mais si l'application gère des opérations longues (gros uploads, long-polling, WebSockets), augmenter `terminationGracePeriodSeconds` et configurer `KeepAliveTimeout` de Kestrel en conséquence.

## Zoom : ConfigMap et Secret

La configuration de prod doit vivre hors de l'image container. Kubernetes fournit deux primitives pour ça.

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

ConfigMap pour les valeurs non sensibles, Secret pour les sensibles. La convention du double underscore (`__`) dans les noms de clés mappe sur la configuration imbriquée d'ASP.NET Core : `Logging__LogLevel__Default` devient `Logging:LogLevel:Default` dans `IConfiguration`.

Les Secrets en YAML clair sont seulement encodés en base64, pas chiffrés. Pour une vraie sécurité, utiliser l'un de :

- **Sealed Secrets** (Bitnami) pour commiter des secrets chiffrés dans Git.
- **External Secrets Operator** pour tirer les secrets depuis Azure Key Vault, AWS Secrets Manager, HashiCorp Vault au runtime.
- **Kubernetes Secrets avec chiffrement au repos** activé sur le cluster (la plupart des offres managées le font par défaut).

> ❌ **Ne jamais faire** : Ne pas commiter de YAML Secret en clair dans Git, même dans un repo privé. Le traiter comme un fichier de mots de passe. Utiliser l'un des patterns externes de gestion de secrets à la place.

## Zoom : autoscaling horizontal

Une fois l'application qui tourne avec des comptes de réplicas manuels, ajouter un HorizontalPodAutoscaler laisse Kubernetes ajuster le compte automatiquement selon le CPU ou des métriques custom.

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

Le HPA scale le Deployment entre 3 et 20 pods, en visant une utilisation CPU moyenne autour de 70%. Le `stabilizationWindowSeconds: 300` sur le scale-down évite le thrashing : le HPA attend 5 minutes de CPU bas avant de retirer un réplica, ce qui évite les oscillations quand la charge est erratique.

L'[article sur les spike tests](/fr/posts/load-testing-spike/) couvre le mode de défaillance où la réaction du HPA est trop lente pour les bursts soudains. Si les spikes sont un vrai sujet pour la charge, soit faire tourner un `minReplicas` plus élevé, soit passer à de l'autoscaling prédictif via des outils comme KEDA.

> 💡 **Info** : KEDA (Kubernetes Event-Driven Autoscaling) est la façon standard communautaire de scaler des charges Kubernetes selon des signaux externes : profondeur de file (RabbitMQ, Azure Service Bus, Kafka), métriques Prometheus, taux de requêtes HTTP, et bien d'autres. Pour les charges dont la charge ne corrèle pas avec le CPU, KEDA est en général la bonne réponse.

## Quand Kubernetes est le mauvais outil

Kubernetes est puissant, mais il est aussi lourd opérationnellement. Faire tourner un cluster de qualité prod signifie patcher les nœuds, gérer un Ingress Controller, maintenir l'observabilité, gérer la rotation des certificats, et debugger des problèmes qui n'existent pas sur des plateformes plus simples. Pour une petite application (un service, peu de trafic, un ou deux devs), ce coût est disproportionné.

Si la charge correspond à l'une de ces formes, une alternative plus légère est souvent meilleure :

- **Un seul petit service** : [Azure Web App](/fr/posts/hosting-azure-web-app/) ou un simple hôte Docker.
- **Container-natif mais faible tolérance opérationnelle** : [Azure Container Apps](/fr/posts/hosting-azure-container-apps/), qui apporte la plupart des bénéfices de Kubernetes sans gérer le cluster.
- **Serverless / événementiel** : Azure Functions ou AWS Lambda, surtout couplé à Native AOT vu dans la [série performance](/fr/posts/performance-aot-compilation/) pour un démarrage à froid rapide.

Kubernetes rentabilise quand on a plusieurs services, plusieurs équipes, une charge variable qui bénéficie de l'autoscaling, et assez de capacité opérationnelle pour faire tourner le cluster. Pour une seule petite API avec un trafic stable, c'est surdimensionné.

## Wrap-up

Héberger ASP.NET Core sur Kubernetes se résume à un petit jeu de primitives : un Deployment avec probes, limites de ressources et security context ; un Service pour un routing interne stable ; un Ingress pour le trafic externe avec TLS ; des ConfigMaps et Secrets pour la configuration externalisée ; et optionnellement un HorizontalPodAutoscaler quand la charge varie. Tu peux transformer une image Docker en une charge Kubernetes prête pour la prod en les combinant, tu peux obtenir de vrais rolling updates zero-downtime avec la bonne config de probes et de période de grâce, et tu peux reconnaître quand le coût opérationnel de Kubernetes ne rentabilise pas et qu'une plateforme plus simple servirait mieux.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Héberger ASP.NET Core sur IIS : le classique, démystifié](/fr/posts/hosting-iis/)
- [Héberger ASP.NET Core avec Docker : un guide pragmatique](/fr/posts/hosting-docker/)
- [Les Tests de Charge en .NET : vue d'ensemble des quatre types qui comptent](/fr/posts/load-testing-overview/)
- [Le Spike Testing en .NET : survivre au burst soudain](/fr/posts/load-testing-spike/)

## Références

- [Concepts Kubernetes, docs officielles](https://kubernetes.io/docs/concepts/)
- [Configurer les probes Liveness, Readiness et Startup, docs Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Horizontal Pod Autoscaler, docs Kubernetes](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Déployer des applications ASP.NET Core sur Kubernetes, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/architecture/microservices/multi-container-microservice-net-applications/)
- [Documentation KEDA](https://keda.sh/)
