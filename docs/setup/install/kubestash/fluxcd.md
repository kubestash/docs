---
title: Install KubeStash using FluxCD
description: Install KubeStash using FluxCD
menu:
  docs_{{ .version }}:
    identifier: install-kubestash-fluxcd
    name: FluxCD
    parent: install-kubestash-enterprise
    weight: 40
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Install using FluxCD

You can also deploy KubeStash declaratively with [FluxCD](https://fluxcd.io/). The setup mirrors the ArgoCD flow: one OCI `HelmRepository` pointing at `ghcr.io/appscode-charts`, followed by three `HelmRelease` resources installed in the same order — `ace-user-roles`, `license-proxyserver`, and `kubestash`.

### Prerequisites

- A cluster with the FluxCD `source-controller` and `helm-controller` installed (e.g. via `flux install` or `flux bootstrap`).
- An AppsCode platform token. Replace the placeholder `token` value in the `license-proxyserver` HelmRelease with your token.

### 1. Register the AppsCode OCI Helm repository

Save the following manifest as `appscode-charts.yaml` and apply it with `kubectl apply -f appscode-charts.yaml`:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: appscode-charts
  namespace: flux-system
spec:
  type: oci
  interval: 5m
  url: oci://ghcr.io/appscode-charts
```

### 2. Install ace-user-roles

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ace-user-roles
  namespace: flux-system
spec:
  interval: 10m
  targetNamespace: kubeops
  install:
    createNamespace: true
  chart:
    spec:
      chart: ace-user-roles
      version: v2026.2.16
      sourceRef:
        kind: HelmRepository
        name: appscode-charts
        namespace: flux-system
  values:
    enableClusterRoles:
      ace: false
      appcatalog: true
      catalog: false
      cert-manager: false
      kubedb: true
      kubedb-ui: false
      kubestash: true # enable if used
      kubevault: true # enable if used
      license-proxyserver: true
      metrics: true
      prometheus: false
      secrets-store: false
      stash: true # enable if used
      virtual-secrets: false
    annotations:
      "helm.sh/hook": null
      "helm.sh/hook-delete-policy": null
```

### 3. Install license-proxyserver

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: license-proxyserver
  namespace: flux-system
spec:
  interval: 10m
  targetNamespace: kubeops
  install:
    createNamespace: true
  chart:
    spec:
      chart: license-proxyserver
      version: v2026.2.16
      sourceRef:
        kind: HelmRepository
        name: appscode-charts
        namespace: flux-system
  values:
    platform:
      baseURL: https://appscode.com
      token: '****************************************'
```

### 4. Install KubeStash

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kubestash
  namespace: flux-system
spec:
  interval: 10m
  targetNamespace: kubestash
  install:
    createNamespace: true
  chart:
    spec:
      chart: kubestash
      version: {{< param "info.version" >}}
      sourceRef:
        kind: HelmRepository
        name: appscode-charts
        namespace: flux-system
  values:
    ace-user-roles:
      enabled: false
```

Notes:

- The `HelmRepository` is created once in `flux-system` and reused by every `HelmRelease` via `sourceRef`. Adjust the `namespace` if you keep Flux sources elsewhere.
- `ace-user-roles.enabled: false` is set on the KubeStash HelmRelease because the `ace-user-roles` chart is already managed by its own HelmRelease (step 2).
- FluxCD's `helm-controller` reconciles by re-running `helm upgrade`, so unlike ArgoCD you do not need to declare `ignoreDifferences` for the operator's rotating TLS certs and webhook CA bundles.

Once all three HelmReleases report `Ready`, follow the [Common Configuration](/docs/setup/install/kubestash/configuration.md) steps to verify the installation.
