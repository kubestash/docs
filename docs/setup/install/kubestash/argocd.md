---
title: Install KubeStash using ArgoCD
description: Install KubeStash using ArgoCD
menu:
  docs_{{ .version }}:
    identifier: install-kubestash-argocd
    name: ArgoCD
    parent: install-kubestash-enterprise
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Install using ArgoCD

You can deploy KubeStash declaratively using [ArgoCD](https://argo-cd.readthedocs.io/). The setup is composed of three `Application` manifests that should be applied in the following order:

1. `ace-user-roles` — installs the ClusterRoles used by KubeStash (and other AppsCode products) so that user impersonation works correctly.
2. `license-proxyserver` — installs the AppsCode License Proxyserver, which obtains and rotates the KubeStash license from the AppsCode platform automatically. You no longer need to download a license file when this is used.
3. `kubestash` — installs the KubeStash operator itself.

### Prerequisites

- An ArgoCD installation reachable at the `argocd` namespace.
- An AppsCode platform token. You can obtain one from the [AppsCode platform](https://appscode.com/). Replace the placeholder value in the `license-proxyserver` manifest with your token.

### 1. Install ace-user-roles

Save the following manifest as `ace-user-roles.yaml` and apply it with `kubectl apply -f ace-user-roles.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ace-user-roles
  namespace: argocd
spec:
  destination:
    namespace: kubeops
    server: https://kubernetes.default.svc
  project: default
  source:
    chart: ace-user-roles
    helm:
      values: |
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
    repoURL: ghcr.io/appscode-charts
    targetRevision: v2026.2.16
  syncPolicy:
    automated: {}
    syncOptions:
    - CreateNamespace=true
```

### 2. Install license-proxyserver

Save the following manifest as `license-proxyserver.yaml` and replace the `token` value with your AppsCode platform token, then apply it with `kubectl apply -f license-proxyserver.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: license-proxyserver
  namespace: argocd
spec:
  project: default
  source:
    chart: license-proxyserver
    repoURL: ghcr.io/appscode-charts
    targetRevision: v2026.2.16
    helm:
      values: |
        platform:
          baseURL: https://appscode.com
          token: '****************************************'
  destination:
    server: "https://kubernetes.default.svc"
    namespace: kubeops
  syncPolicy:
    automated: {}
    syncOptions:
    - CreateNamespace=true

  ignoreDifferences:
  - jsonPointers:
    - /data
    kind: Secret
    name: license-proxyserver-apiserver-cert
    namespace: kubeops
  - group: apiregistration.k8s.io
    kind: APIService
    name: v1alpha1.proxyserver.licenses.appscode.com
    jsonPointers:
    - /spec/caBundle
  - group: apiextensions.k8s.io
    kind: CustomResourceDefinition
    name: servicemonitors.monitoring.coreos.com
    jsonPointers:
    - /metadata/annotations
    - /spec
```

The `ignoreDifferences` block prevents ArgoCD from continuously fighting the controllers that rotate the apiserver certificate, CA bundle, and ServiceMonitor CRD annotations.

### 3. Install KubeStash

Save the following manifest as `kubestash.yaml` and apply it with `kubectl apply -f kubestash.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kubestash
  namespace: argocd
spec:
  project: default
  source:
    chart: kubestash
    repoURL: ghcr.io/appscode-charts
    targetRevision: {{< param "info.version" >}}
    helm:
      values: |
        ace-user-roles:
          enabled: false
  destination:
    server: "https://kubernetes.default.svc"
    namespace: kubestash
  syncPolicy:
    automated: {}
    syncOptions:
    - CreateNamespace=true

  ignoreDifferences:
  - jsonPointers:
    - /data
    kind: Secret
    name: kubestash-kubestash-operator-cert
    namespace: kubestash
  - group: apps
    kind: Deployment
    name: kubestash-kubestash-operator-operator
    namespace: kubestash
    jsonPointers:
    - /spec/template/metadata/annotations/reload
  - group: apps
    kind: Deployment
    name: kubestash-kubestash-operator-webhook-server
    namespace: kubestash
    jsonPointers:
    - /spec/template/metadata/annotations/reload

  - group: admissionregistration.k8s.io
    kind: MutatingWebhookConfiguration
    name: kubestash-kubestash-operator
    jqPathExpressions:
    - .webhooks[].clientConfig.caBundle
  - group: admissionregistration.k8s.io
    kind: ValidatingWebhookConfiguration
    name: kubestash-kubestash-operator
    jqPathExpressions:
    - .webhooks[].clientConfig.caBundle
```

Notes:

- `ace-user-roles.enabled: false` is set on the KubeStash Application because the `ace-user-roles` chart is already managed by its own Application (step 1).
- The `ignoreDifferences` entries cover the webhook CA bundles, operator TLS secret and the reload annotation that KubeStash flips when its configuration changes. Without them ArgoCD would report the Application as out-of-sync after every reconciliation.

Once all three Applications are healthy, follow the [Common Configuration](/docs/setup/install/kubestash/configuration.md) steps to verify the installation.
