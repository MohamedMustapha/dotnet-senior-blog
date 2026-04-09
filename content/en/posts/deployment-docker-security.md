---
title: "Docker Security for .NET: Hardening, Scanning, and Supply Chain"
date: 2026-04-09
draft: false
tags: ["deployment", "docker", "security", "dotnet"]
categories: ["Deployment"]
series: ["Deployment"]
description: "Image scanning with Trivy and Scout, SBOMs, signed images with cosign, build provenance, and the concrete .NET-specific security patterns that reduce real attack surface."
---

A container is not automatically secure because it is a container. The default Docker image for a typical .NET application, built without care, runs as root, ships with a full Linux userland, exposes a large attack surface to anyone who can reach the network interface, and carries known CVEs from the base image's last publish date. This is not a hypothetical problem. It is the baseline every .NET team inherits the day they ship their first container, and hardening it is not optional for anything that touches user data.

This article is the security deep dive for the [Deployment series](/posts/deployment-docker-dockerfile-compose/). It complements the [Hosting Docker article](/posts/hosting-docker/), which covered the runtime patterns, with the security-specific concerns that apply to both build and deployment: image scanning, SBOM generation, image signing, secrets handling, and supply chain attestation. The goal is not to turn you into a security engineer. It is to give a .NET team the handful of practices that eliminate the most common real-world risks with the least friction.

## Why container security is different

Traditional .NET security thinking focuses on the application: OWASP Top 10, authentication, input validation, SQL injection, XSS. All of that is still necessary. But a container adds a second surface: the image itself. Three concrete things can go wrong at the container level even in an application with perfect code:

1. **The base image contains a known vulnerability.** A CVE in `glibc`, `openssl`, `zlib`, or any system library ships with every image built on top of the affected base. If the base image has not been rebuilt recently, the vulnerability travels into production.
2. **The running container has more privileges than it needs.** Running as root, having write access to the root filesystem, mounting the Docker socket, and exposing host capabilities all widen the blast radius of any application-level compromise.
3. **The supply chain itself is compromised.** The image pulled from the registry might not be the image the CI pipeline built, if an attacker has write access to the registry or can intercept the pull. Without signatures and provenance, there is no way to prove the image is authentic.

These three risks have dedicated mitigations. The rest of this article covers each one.

## Overview: the layered defense

{{< mermaid >}}
graph TD
    A[Source code] --> B[SBOM generated<br/>at build]
    B --> C[Image scanned<br/>Trivy or Scout]
    C --> D[Image signed<br/>cosign]
    D --> E[Provenance attestation<br/>SLSA level 3]
    E --> F[Registry]
    F --> G[Runtime verification<br/>signature + policy]
    G --> H[Non-root container<br/>read-only FS<br/>no capabilities]
{{< /mermaid >}}

The pipeline adds one security concern per stage. None of them replace the others, and skipping any one of them leaves a specific class of risk uncovered. The good news is that most of these can be added to an existing build pipeline in a day, not a quarter.

## Zoom: the hardened runtime configuration

Before scanning and signing, the container itself should run with minimum privileges. Four settings do most of the work:

```yaml
# Kubernetes pod securityContext (also works on ACA with minor differences)
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

**`runAsNonRoot: true` and `runAsUser: 64198`**: forces the container to run as a non-root user. The chiseled .NET images already default to UID 64198, but declaring it at the pod level is a defense-in-depth measure that catches the case where someone swaps the image for one that still runs as root.

**`allowPrivilegeEscalation: false`**: prevents the process from gaining more privileges than the parent, even if a setuid binary is present. This stops an entire class of privilege escalation exploits at the kernel level.

**`readOnlyRootFilesystem: true`**: mounts the root filesystem read-only. An attacker who gains code execution cannot write a web shell, modify a binary, or drop a persistent payload. ASP.NET Core does not need to write anywhere except `/tmp`, which is provided as a separate `emptyDir` volume.

**`capabilities: drop: [ALL]` and `seccompProfile: RuntimeDefault`**: removes all Linux capabilities (the fine-grained privileges underlying `root`) and restricts the system calls the container can make via the kernel's seccomp filter. ASP.NET Core needs none of the special capabilities, so dropping them costs nothing and closes a large attack surface.

Together, these four settings turn a container from "has a foothold on the host if compromised" into "very constrained sandbox with no easy escalation path". Most .NET applications work under them without modification.

> ✅ **Good practice** : Put these settings in a shared Helm chart or Kustomize base that every service inherits. Standardizing them at the platform level is the only way to prevent drift across tens of services.

## Zoom: image scanning in CI

Every image pushed to production should be scanned for known CVEs before deployment. The two widely-adopted open source tools are **Trivy** (Aqua Security) and **Grype** (Anchore). Microsoft also provides **Docker Scout**, integrated into Docker Desktop and Docker Hub.

A typical CI step using Trivy:

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

Three decisions to make explicit.

**`severity: HIGH,CRITICAL`**: most .NET images have dozens of LOW and MEDIUM CVEs at any given moment, and failing the build on those produces noise that trains the team to ignore the scanner. Fail on HIGH and CRITICAL only, triage the rest in a tracker.

**`exit-code: 1`**: the scan must actually fail the build, not just log warnings. A scanner that does not block deployment is a compliance theater, not a security control.

**`ignore-unfixed: true`**: some CVEs have no fix available yet. Blocking the pipeline on CVEs you cannot fix punishes your team for something outside their control. Log them, track them, revisit weekly, but do not fail the build.

> 💡 **Info** : The chiseled .NET images from Microsoft are rebuilt on base image updates, which means CVEs in `glibc` or similar libraries are patched faster than in the full Debian-based images. This is a significant advantage for teams that scan aggressively: a chiseled image typically has zero HIGH or CRITICAL CVEs on the day of release, while the full image has a handful.

## Zoom: SBOMs and what they are for

A **Software Bill of Materials** (SBOM) is a machine-readable list of every package and version inside an image. It does not prevent any vulnerability by itself, but it enables three important workflows:

1. **Retroactive CVE response.** When a new CVE is disclosed (log4shell, xz, spring4shell), an SBOM lets the team query "which of our 50 deployed images contains the affected package" in seconds, without re-scanning everything.
2. **Compliance and audit.** Customers, regulators, and SOC 2 auditors increasingly ask for SBOMs as proof of what is actually in a shipped product.
3. **Supply chain verification.** Pairing an SBOM with a signature creates an attestation that can be verified at pull time.

BuildKit generates SBOMs natively:

```bash
docker buildx build \
  --sbom=true \
  --provenance=true \
  --tag myregistry.azurecr.io/shop-api:1.4.7 \
  --push \
  .
```

The `--sbom=true` flag attaches an SBOM to the image manifest in SPDX format. The `--provenance=true` flag attaches a SLSA provenance attestation describing *how* the image was built: the source repo, the commit, the builder version, the build parameters. Both are stored as OCI artifacts alongside the image, and neither changes how the image runs.

## Zoom: signing images with cosign

A signed image proves two things: who built it, and that it has not been modified since. The tool of choice in 2026 is **cosign** from the Sigstore project, which supports both keyless signing (via short-lived OIDC tokens from the CI provider) and traditional keypair signing.

Keyless signing from a GitHub Actions workflow:

```yaml
- name: Sign the image
  env:
    COSIGN_EXPERIMENTAL: "true"
  run: |
    cosign sign --yes \
      myregistry.azurecr.io/shop-api@${{ steps.build.outputs.digest }}
```

The signature is stored in the registry next to the image, referencing it by its content digest (not a mutable tag). At deployment time, a verification step fails the deploy if the signature does not match:

```yaml
- name: Verify the image signature
  run: |
    cosign verify \
      --certificate-identity-regexp '^https://github.com/myorg/shop-api/' \
      --certificate-oidc-issuer https://token.actions.githubusercontent.com \
      myregistry.azurecr.io/shop-api:1.4.7
```

This policy says: "only accept this image if it was signed by a GitHub Actions workflow in my organization's `shop-api` repository". An attacker who pushes a modified image to the registry cannot produce a matching signature without also compromising GitHub's OIDC issuer, which is a much higher bar than compromising the registry alone.

> ⚠️ **It works, but...** : Signing without verification is security theater. The signing step in CI is only half of the value; the verification step at deploy time (in Kubernetes with an admission controller like Kyverno or OPA Gatekeeper, or in ACA with an image validation policy) is what actually enforces the guarantee.

## Zoom: secrets, revisited

The [Hosting Docker article](/posts/hosting-docker/) covered the rule: never bake secrets into the image. That rule has two corollaries that deserve explicit attention in a security context.

**Build-time secrets must be passed via `--secret`, not `ENV` or `ARG`.** If a package fetch during `dotnet restore` needs an authentication token, BuildKit provides a mount-based secret mechanism:

```dockerfile
RUN --mount=type=secret,id=nuget-auth,target=/root/.nuget/NuGet/NuGet.Config \
    dotnet restore
```

```bash
docker buildx build \
  --secret id=nuget-auth,src=./nuget-auth.config \
  ...
```

The secret is mounted into the build container during the `RUN` step and is **not** baked into any layer. After the step, the secret is gone. Using `ENV` or `ARG` for the same thing leaks the value into the image history, where anyone with pull access can recover it.

**Runtime secrets should come from a secret store, not environment variables.** Environment variables are visible in process listings, crash dumps, and any container introspection tool. For anything more sensitive than a feature flag, use Kubernetes Secrets mounted as files, Azure Key Vault references, or a sidecar like `vault-agent` that writes to a `tmpfs`. The application reads from the file at startup and never holds the value in an environment variable.

> ❌ **Never do this** : Do not accept the argument "it is a private registry, so it is fine". Private registries are compromised regularly through credential leaks, misconfigured access policies, or supply chain attacks on the registry itself. Defense-in-depth assumes every layer can be compromised.

## Zoom: base image hygiene

The single most impactful security practice for .NET containers is staying current with base image updates. Microsoft rebuilds the .NET base images on every security update to the underlying OS, and the chiseled variants get patched especially fast because they have fewer packages to worry about.

The practical workflow:

1. **Pin to the minor version** (`10.0-noble-chiseled`), not to a patch version or a digest. This way, rebuilds automatically pick up the latest patched base image without manual tag bumps.
2. **Rebuild the image on a schedule**, not only on code changes. A weekly scheduled CI run rebuilds the image with the same source, pulls whatever base image has been patched in the meantime, and pushes a new tag. Any deployed image is at most one week out of date.
3. **Monitor Microsoft security advisories** for .NET and subscribe to the container image advisories. Microsoft releases security updates every second Tuesday of the month, and the base images are usually updated within 24 hours.

```yaml
# .github/workflows/weekly-rebuild.yml
on:
  schedule:
    - cron: '0 2 * * 1'  # Every Monday at 02:00 UTC
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

> ✅ **Good practice** : Pair the weekly rebuild with a rolling deployment to pre-prod and a canary rollout to prod, gated by the baseline tests covered in the [baseline load testing article](/posts/load-testing-baseline/). This turns base image hygiene from "a chore nobody does" into "the pipeline does it automatically".

## Wrap-up

Docker security for .NET in 2026 is not about perfect, it is about the handful of controls that close the biggest gaps: a hardened runtime `securityContext` with non-root, read-only filesystem, and dropped capabilities; image scanning with Trivy or Scout as a blocking CI step; SBOMs and provenance attestation via BuildKit flags; image signing with cosign and verification at deployment time; BuildKit `--mount=type=secret` for build-time secrets; runtime secrets from a store, never from environment variables; and a weekly rebuild schedule to keep the base image current. You can add all of this to an existing deployment pipeline in a day or two, and the result is a container posture that blocks the real attack classes without turning security into a full-time job.

Ready to level up your next project or share it with your team? See you in the next one, Kubernetes Primer is where we go next.

## Related articles

- [Hosting ASP.NET Core with Docker: A Pragmatic Guide](/posts/hosting-docker/)
- [Hosting ASP.NET Core on Kubernetes: The Essentials for .NET Developers](/posts/hosting-kubernetes/)
- [Docker for .NET Deployment: Dockerfile and Compose in Practice](/posts/deployment-docker-dockerfile-compose/)

## References

- [Trivy documentation](https://trivy.dev/)
- [Docker Scout, Docker docs](https://docs.docker.com/scout/)
- [Sigstore cosign documentation](https://docs.sigstore.dev/cosign/overview/)
- [SLSA framework](https://slsa.dev/)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [.NET container security, Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/docker/container-images#security)
- [BuildKit build secrets](https://docs.docker.com/build/building/secrets/)
