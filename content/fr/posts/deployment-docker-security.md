---
title: "Docker : bonnes pratiques de sécurité pour .NET"
date: 2026-04-09
draft: false
tags: ["deployment", "docker", "security", "dotnet"]
categories: ["Deployment"]
series: ["Deployment"]
description: "Scan d'image avec Trivy et Scout, SBOMs, images signées avec cosign, provenance de build, et les patterns de sécurité spécifiques à .NET qui réduisent la vraie surface d'attaque."
---

Hello tous le monde, aujourd'hui on va comprendre **la sécurité Docker pour .NET**, la couche de contrôles qu'on applique aux containers et qui change leur posture réelle en prod.

Un container n'est pas automatiquement sécurisé parce que c'est un container. L'image Docker par défaut d'une application .NET typique, construite sans précaution, tourne en root, embarque tout un userland Linux, expose une large surface d'attaque à quiconque peut joindre l'interface réseau, et porte des CVE connues depuis la dernière date de publication de son image de base. Ce n'est pas un problème hypothétique. C'est le baseline dont chaque équipe .NET hérite le jour où elle livre son premier container, et le durcir n'est pas optionnel pour tout ce qui touche à de la donnée utilisateur.

Cet article est le zoom sécurité de la [série Deployment](/fr/posts/deployment-docker-dockerfile-compose/). Il complète l'[article Hosting Docker](/fr/posts/hosting-docker/), qui couvrait les patterns runtime, avec les préoccupations spécifiquement sécurité qui s'appliquent au build et au déploiement : scan d'image, génération de SBOM, signature d'image, gestion des secrets, et attestation de supply chain. L'objectif n'est pas de transformer un dev .NET en ingénieur sécurité. C'est de donner à une équipe .NET la poignée de pratiques qui éliminent les risques les plus courants avec le moins de friction.

## Le contexte : pourquoi la sécurité container est différente

La pensée sécurité traditionnelle en .NET se concentre sur l'application : OWASP Top 10, authentification, validation des entrées, injection SQL, XSS. Tout cela reste nécessaire. Mais un container ajoute une seconde surface : l'image elle-même. Trois choses concrètes peuvent mal tourner au niveau container même dans une application au code parfait :

1. **L'image de base contient une vulnérabilité connue.** Une CVE dans `glibc`, `openssl`, `zlib`, ou n'importe quelle bibliothèque système est livrée avec chaque image construite par-dessus la base affectée. Si l'image de base n'a pas été reconstruite récemment, la vulnérabilité voyage jusqu'en prod.
2. **Le container qui tourne a plus de privilèges qu'il n'en a besoin.** Tourner en root, avoir un accès en écriture sur le système de fichiers racine, monter le socket Docker, et exposer des capabilities de l'hôte élargissent tous le rayon d'impact de n'importe quelle compromission applicative.
3. **La supply chain elle-même est compromise.** L'image tirée du registry peut ne pas être l'image que le pipeline de CI a construite, si un attaquant a un accès en écriture au registry ou peut intercepter le pull. Sans signatures ni provenance, il n'y a aucun moyen de prouver que l'image est authentique.

Ces trois risques ont des mitigations dédiées. Le reste de l'article les couvre une par une.

## Vue d'ensemble : la défense en couches

{{< mermaid >}}
graph TD
    A[Code source] --> B[SBOM généré<br/>au build]
    B --> C[Image scannée<br/>Trivy ou Scout]
    C --> D[Image signée<br/>cosign]
    D --> E[Attestation de provenance<br/>SLSA niveau 3]
    E --> F[Registry]
    F --> G[Vérification au runtime<br/>signature + policy]
    G --> H[Container non-root<br/>FS en lecture seule<br/>aucune capability]
{{< /mermaid >}}

Le pipeline ajoute une préoccupation sécurité par étape. Aucune ne remplace les autres, et sauter l'une d'elles laisse une classe de risque précise à découvert. La bonne nouvelle, c'est que la plupart peuvent être ajoutées à un pipeline de build existant en une journée, pas en un trimestre.

## Zoom : la configuration runtime durcie

Avant de scanner et signer, le container lui-même doit tourner avec le minimum de privilèges. Quatre réglages font l'essentiel du travail :

```yaml
# securityContext d'un pod Kubernetes (marche aussi sur ACA avec des différences mineures)
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 64198
    fsGroup: 64198
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: api
      image: myregistry.azurecr.io/shop-api:1.4.7
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: [ALL]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

**`runAsNonRoot: true` et `runAsUser: 64198`** : forcent le container à tourner sous un utilisateur non-root. Les images .NET chiseled utilisent déjà UID 64198 par défaut, mais le déclarer au niveau du pod est une mesure de défense en profondeur qui attrape le cas où quelqu'un substitue l'image par une qui tourne encore en root.

**`allowPrivilegeEscalation: false`** : empêche le process de gagner plus de privilèges que le parent, même si un binaire setuid est présent. Cela bloque toute une classe d'exploits d'escalade de privilèges au niveau kernel.

**`readOnlyRootFilesystem: true`** : monte le système de fichiers racine en lecture seule. Un attaquant qui obtient une exécution de code ne peut pas écrire de web shell, modifier un binaire, ou déposer un payload persistant. ASP.NET Core n'a pas besoin d'écrire ailleurs que dans `/tmp`, qui est fourni comme volume `emptyDir` séparé.

**`capabilities: drop: [ALL]` et `seccompProfile: RuntimeDefault`** : suppriment toutes les capabilities Linux (les privilèges fins sous `root`) et restreignent les appels système que le container peut faire via le filtre seccomp du kernel. ASP.NET Core n'a besoin d'aucune des capabilities spéciales, donc les supprimer ne coûte rien et ferme une grande surface d'attaque.

Ensemble, ces quatre réglages transforment un container de "a un pied sur l'hôte si compromis" en "sandbox très contraint sans chemin d'escalade facile". La plupart des applications .NET fonctionnent dessous sans modification.

> ✅ **Bonne pratique** : Mettre ces réglages dans une chart Helm ou une base Kustomize partagée dont chaque service hérite. Les standardiser au niveau plateforme est la seule façon d'éviter la dérive sur des dizaines de services.

## Zoom : scan d'image en CI

Toute image poussée en prod doit être scannée pour les CVE connues avant le déploiement. Les deux outils open source largement adoptés sont **Trivy** (Aqua Security) et **Grype** (Anchore). Microsoft fournit aussi **Docker Scout**, intégré dans Docker Desktop et Docker Hub.

Une étape CI typique avec Trivy :

```yaml
# .github/workflows/deploy.yml
- name: Scan image for CVEs
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myregistry.azurecr.io/shop-api:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: HIGH,CRITICAL
    exit-code: 1
    ignore-unfixed: true

- name: Upload scan results
  uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: trivy-results.sarif
```

Trois décisions à rendre explicites.

**`severity: HIGH,CRITICAL`** : la plupart des images .NET ont des dizaines de CVE LOW et MEDIUM à tout moment, et faire échouer le build sur celles-ci produit du bruit qui entraîne l'équipe à ignorer le scanner. Échouer seulement sur HIGH et CRITICAL, trier le reste dans un tracker.

**`exit-code: 1`** : le scan doit réellement faire échouer le build, pas juste logger des avertissements. Un scanner qui ne bloque pas le déploiement est du théâtre de conformité, pas un contrôle de sécurité.

**`ignore-unfixed: true`** : certaines CVE n'ont pas encore de fix disponible. Bloquer le pipeline sur des CVE qu'on ne peut pas corriger punit l'équipe pour quelque chose hors de son contrôle. Les logger, les tracker, les revoir chaque semaine, mais ne pas faire échouer le build.

> 💡 **Info** : Les images .NET chiseled de Microsoft sont reconstruites à chaque mise à jour d'image de base, ce qui veut dire que les CVE dans `glibc` ou des bibliothèques similaires sont patchées plus vite que dans les images complètes basées sur Debian. C'est un avantage significatif pour les équipes qui scannent agressivement : une image chiseled a typiquement zéro CVE HIGH ou CRITICAL le jour de sa sortie, tandis que l'image complète en a une poignée.

## Zoom : les SBOMs et leur utilité

Un **Software Bill of Materials** (SBOM) est une liste lisible par machine de chaque package et version à l'intérieur d'une image. Il ne prévient aucune vulnérabilité par lui-même, mais il active trois workflows importants :

1. **Réponse rétroactive aux CVE.** Quand une nouvelle CVE est divulguée (log4shell, xz, spring4shell), un SBOM permet à l'équipe de demander "laquelle de nos 50 images déployées contient le package affecté" en quelques secondes, sans tout re-scanner.
2. **Conformité et audit.** Les clients, les régulateurs, et les auditeurs SOC 2 demandent de plus en plus des SBOMs comme preuve de ce qu'il y a réellement dans un produit livré.
3. **Vérification de supply chain.** Appairer un SBOM avec une signature crée une attestation qui peut être vérifiée au moment du pull.

BuildKit génère les SBOMs nativement :

```bash
docker buildx build \
  --sbom=true \
  --provenance=true \
  --tag myregistry.azurecr.io/shop-api:1.4.7 \
  --push \
  .
```

Le flag `--sbom=true` attache un SBOM au manifeste de l'image au format SPDX. Le flag `--provenance=true` attache une attestation de provenance SLSA qui décrit *comment* l'image a été construite : le repo source, le commit, la version du builder, les paramètres de build. Les deux sont stockés comme artefacts OCI à côté de l'image, et aucun ne change la façon dont l'image tourne.

## Zoom : signer les images avec cosign

Une image signée prouve deux choses : qui l'a construite, et qu'elle n'a pas été modifiée depuis. L'outil de choix en 2026 est **cosign** du projet Sigstore, qui supporte à la fois la signature sans clé (via des tokens OIDC de courte durée du provider CI) et la signature traditionnelle par paire de clés.

Signature sans clé depuis un workflow GitHub Actions :

```yaml
- name: Sign the image
  env:
    COSIGN_EXPERIMENTAL: "true"
  run: |
    cosign sign --yes \
      myregistry.azurecr.io/shop-api@${{ steps.build.outputs.digest }}
```

La signature est stockée dans le registry à côté de l'image, en la référençant par son digest de contenu (pas un tag mutable). Au moment du déploiement, une étape de vérification fait échouer le déploiement si la signature ne correspond pas :

```yaml
- name: Verify the image signature
  run: |
    cosign verify \
      --certificate-identity-regexp '^https://github.com/myorg/shop-api/' \
      --certificate-oidc-issuer https://token.actions.githubusercontent.com \
      myregistry.azurecr.io/shop-api:1.4.7
```

Cette policy dit : "n'accepter cette image que si elle a été signée par un workflow GitHub Actions dans le repo `shop-api` de mon organisation". Un attaquant qui pousse une image modifiée dans le registry ne peut pas produire de signature correspondante sans compromettre aussi l'issuer OIDC de GitHub, ce qui est une barre bien plus haute que compromettre le registry seul.

> ⚠️ **Ça marche, mais...** : Signer sans vérifier est du théâtre de sécurité. L'étape de signature en CI n'est que la moitié de la valeur ; l'étape de vérification au déploiement (dans Kubernetes avec un admission controller comme Kyverno ou OPA Gatekeeper, ou dans ACA avec une policy de validation d'image) est ce qui impose réellement la garantie.

## Zoom : les secrets, revus

L'[article Hosting Docker](/fr/posts/hosting-docker/) a couvert la règle : ne jamais cuire de secrets dans l'image. Cette règle a deux corollaires qui méritent une attention explicite dans un contexte sécurité.

**Les secrets au build doivent passer par `--secret`, pas par `ENV` ou `ARG`.** Si un package fetch pendant `dotnet restore` a besoin d'un token d'authentification, BuildKit fournit un mécanisme de secret basé sur le mount :

```dockerfile
RUN --mount=type=secret,id=nuget-auth,target=/root/.nuget/NuGet/NuGet.Config \
    dotnet restore
```

```bash
docker buildx build \
  --secret id=nuget-auth,src=./nuget-auth.config \
  ...
```

Le secret est monté dans le container de build pendant l'étape `RUN` et **n'est pas** cuit dans un layer. Après l'étape, le secret a disparu. Utiliser `ENV` ou `ARG` pour la même chose fait fuiter la valeur dans l'historique de l'image, où quiconque avec un accès pull peut la récupérer.

**Les secrets runtime doivent venir d'un store de secrets, pas de variables d'environnement.** Les variables d'environnement sont visibles dans la liste des process, les dumps de crash, et tout outil d'introspection de container. Pour tout ce qui est plus sensible qu'un feature flag, utiliser des Secrets Kubernetes montés comme fichiers, des références Azure Key Vault, ou un sidecar comme `vault-agent` qui écrit dans un `tmpfs`. L'application lit depuis le fichier au démarrage et ne garde jamais la valeur dans une variable d'environnement.

> ❌ **Ne jamais faire** : Ne pas accepter l'argument "c'est un registry privé, donc c'est bon". Les registries privés sont compromis régulièrement via des fuites de credentials, des policies d'accès mal configurées, ou des attaques de supply chain sur le registry lui-même. La défense en profondeur suppose que chaque couche peut être compromise.

## Zoom : l'hygiène de l'image de base

La pratique de sécurité à l'impact le plus important pour les containers .NET est de rester à jour avec les mises à jour d'image de base. Microsoft reconstruit les images de base .NET à chaque mise à jour de sécurité de l'OS sous-jacent, et les variantes chiseled sont patchées particulièrement vite parce qu'elles ont moins de packages à gérer.

Le workflow concret :

1. **Pinner sur la version mineure** (`10.0-noble-chiseled`), pas sur une version de patch ou un digest. Ainsi, les rebuilds récupèrent automatiquement la dernière image de base patchée sans bump de tag manuel.
2. **Reconstruire l'image sur une planification**, pas uniquement sur les changements de code. Un run CI planifié hebdomadaire reconstruit l'image avec la même source, tire l'image de base qui a été patchée entre-temps, et pousse un nouveau tag. Toute image déployée est à une semaine maximum d'écart.
3. **Surveiller les advisories de sécurité Microsoft** pour .NET et s'abonner aux advisories d'images container. Microsoft publie des mises à jour de sécurité chaque deuxième mardi du mois, et les images de base sont généralement mises à jour dans les 24 heures.

```yaml
# .github/workflows/weekly-rebuild.yml
on:
  schedule:
    - cron: '0 2 * * 1'  # Chaque lundi à 02:00 UTC
  workflow_dispatch:

jobs:
  rebuild:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Rebuild and push
        run: |
          VERSION=$(date +%Y%m%d) docker buildx bake --push
```

> ✅ **Bonne pratique** : Appairer le rebuild hebdomadaire avec un déploiement progressif en pré-prod et un rollout canary en prod, gate par les tests baseline couverts dans l'[article sur le baseline load testing](/fr/posts/load-testing-baseline/). Cela transforme l'hygiène d'image de base de "une corvée que personne ne fait" en "le pipeline le fait automatiquement".

## Wrap-up

La sécurité Docker pour .NET en 2026 n'est pas une question de perfection, c'est la poignée de contrôles qui ferment les plus grosses brèches : un `securityContext` runtime durci avec non-root, système de fichiers en lecture seule, et capabilities droppées ; un scan d'image avec Trivy ou Scout comme étape bloquante en CI ; SBOMs et attestation de provenance via les flags BuildKit ; signature d'image avec cosign et vérification au déploiement ; `--mount=type=secret` BuildKit pour les secrets au build ; secrets runtime depuis un store, jamais depuis des variables d'environnement ; et une planification de rebuild hebdomadaire pour garder l'image de base à jour. Tu peux ajouter tout cela à un pipeline de déploiement existant en un jour ou deux, et le résultat est une posture container qui bloque les vraies classes d'attaque sans transformer la sécurité en job à temps plein.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Héberger ASP.NET Core avec Docker : un guide pragmatique](/fr/posts/hosting-docker/)
- [Héberger ASP.NET Core sur Kubernetes : l'essentiel pour les devs .NET](/fr/posts/hosting-kubernetes/)
- [Docker pour le déploiement .NET : Dockerfile et Compose en pratique](/fr/posts/deployment-docker-dockerfile-compose/)

## Références

- [Documentation Trivy](https://trivy.dev/)
- [Docker Scout, docs Docker](https://docs.docker.com/scout/)
- [Documentation cosign Sigstore](https://docs.sigstore.dev/cosign/overview/)
- [Framework SLSA](https://slsa.dev/)
- [Pod Security Standards Kubernetes](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Sécurité des containers .NET, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/core/docker/container-images#security)
- [Secrets de build BuildKit](https://docs.docker.com/build/building/secrets/)
