---
title: Install KubeStash
description: Installation guide for KubeStash
menu:
  docs_{{ .version }}:
    identifier: install-kubestash-enterprise
    name: KubeStash
    parent: installation-guide
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Install KubeStash

## Get a Free License

Download a FREE license from [AppsCode License Server](https://appscode.com/issue-license?p=stash).

> KubeStash licensing process has been designed to work with CI/CD workflow. You can automatically obtain a license from your CI/CD pipeline by following the guide from [here](https://github.com/appscode/offline-license-server#offline-license-server).

## Install

KubeStash operator can be installed as a Helm chart or simply as Kubernetes manifests.

<ul class="nav nav-tabs" id="installerTab" role="tablist">
  <li class="nav-item">
    <a class="nav-link active" id="helm3-tab" data-toggle="tab" href="#helm3" role="tab" aria-controls="helm3" aria-selected="true">Helm 3 (Recommended)</a>
  </li>
  <li class="nav-item">
    <a class="nav-link" id="script-tab" data-toggle="tab" href="#script" role="tab" aria-controls="script" aria-selected="false">YAML</a>
  </li>
</ul>
<div class="tab-content" id="installerTabContent">
  <div class="tab-pane fade show active" id="helm3" role="tabpanel" aria-labelledby="helm3-tab">

## Using Helm 3

KubeStash can be installed via [Helm](https://helm.sh/) using the [chart](https://github.com/kubestash/installer/tree/master/charts/kubestash) from [AppsCode Charts Repository](https://github.com/appscode/charts). To install the chart with the release name `kubestash`:

```bash
$ helm install kubestash oci://ghcr.io/appscode-charts/kubestash \
        --version {{< param "info.version" >}} \
        --namespace stash --create-namespace \
        --set-file global.license=/path/to/the/license.txt \
        --wait --burst-limit=10000 --debug
```

To see the detailed configuration options, visit [here](https://github.com/kubestash/installer/tree/master/charts/kubestash).

</div>
<div class="tab-pane fade" id="script" role="tabpanel" aria-labelledby="script-tab">

### Using YAML

If you prefer to not use Helm, you can generate YAMLs from KubeStash chart and deploy using `kubectl`. Here we are going to show the prodecure using Helm 3.

```bash
$ helm template kubestash oci://ghcr.io/appscode-charts/kubestash \
        --version {{< param "info.version" >}} \
        --namespace stash --create-namespace \
        --set-file global.license=/path/to/the/license.txt | kubectl apply -f -
```

To see the detailed configuration options, visit [here](https://github.com/kubestash/installer/tree/master/charts/kubestash).

</div>
</div>

### Verify installation

To check if KubeStash operator pods have started, run the following command:

```bash
$ kubectl get pods --all-namespaces -l app.kubernetes.io/name=kubestash-operator --watch
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
stash         kubestash-kubestash-operator-fcd8bf7c6-psjs6   2/2     Running   0          5m49s
```

Once the operator pod is running, you can cancel the above command by typing `Ctrl+C`.

Now, to confirm CRD groups have been registered by the operator, run the following command:
```bash
$ kubectl get crd -l app.kubernetes.io/name=kubestash
NAME                                      CREATED AT
addons.addons.kubestash.com               2023-12-07T06:27:41Z
backupbatches.core.kubestash.com          2023-12-07T06:27:41Z
backupblueprints.core.kubestash.com       2023-12-07T06:27:41Z
backupconfigurations.core.kubestash.com   2023-12-07T06:40:37Z
backupsessions.core.kubestash.com         2023-12-07T06:40:37Z
backupstorages.storage.kubestash.com      2023-12-07T06:40:37Z
functions.addons.kubestash.com            2023-12-07T06:27:41Z
hooktemplates.core.kubestash.com          2023-12-07T06:27:42Z
repositories.storage.kubestash.com        2023-12-07T06:40:37Z
restoresessions.core.kubestash.com        2023-12-07T06:27:42Z
retentionpolicies.storage.kubestash.com   2023-12-07T06:27:42Z
snapshots.storage.kubestash.com           2023-12-07T06:40:37Z
```

### Verify Catalogs

KubeStash automatically installs the necessary Addon catalogs for workload, PVC and Kubernetes manifest backups. Verify that the Addon catalogs have been installed using the following command.

```bash
$ kubectl get addons.addons.kubestash.com
NAME             AGE
kubedump-addon   7m1s
pvc-addon        7m1s
workload-addon   7m1s
```

As you can see from the above output that KubeStash has created `Addon` objects.

Now, you are ready to [take your first backup](/docs/guides/README.md) using KubeStash.

## Install using ArgoCD

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

Once all three Applications are healthy, follow the [Verify installation](#verify-installation) steps above.

## Install in OpenShift

There are two ways to deploy KubeStash in [OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift). Use Option A for the standard upstream chart, or Option B if you require the Red Hat OpenShift certified chart (for example, to satisfy a Red Hat OpenShift certification requirement).

### Option A: Standard chart with OpenShift values

Use the same `kubestash` chart shown above and enable the OpenShift distribution values. This switches the operator to UBI-based images and applies the SecurityContextConstraints and other OpenShift-specific tweaks:

```bash
$ helm install kubestash oci://ghcr.io/appscode-charts/kubestash \
        --version {{< param "info.version" >}} \
        --namespace stash --create-namespace \
        --set-file global.license=/path/to/the/license.txt \
        --set global.distro.openshift=true \
        --set global.distro.ubi=all \
        --wait --burst-limit=10000 --debug
```

Equivalently, in a `values.yaml`:

```yaml
global:
  distro:
    openshift: true
    ubi: "all"
```

- `global.distro.openshift: true` enables OpenShift-specific resources (SCCs, etc.).
- `global.distro.ubi: "all"` switches every component to UBI-based images. Set it to `operator` to only switch the operator images.

### Option B: Red Hat OpenShift certified chart

The `kubestash-certified` chart is the Red Hat certified distribution of KubeStash. Unlike the standard chart, **it does not include CRDs** — the certification process requires CRDs to ship as a separate chart. You must therefore install the CRDs chart first, and then the operator chart.

#### Step 1: Install the CRDs

```bash
$ helm install kubestash-certified-crds oci://ghcr.io/appscode-charts/kubestash-certified-crds \
        --version {{< param "info.version" >}} \
        --namespace stash --create-namespace \
        --wait --burst-limit=10000 --debug
```

#### Step 2: Install the certified operator chart

```bash
$ helm install kubestash oci://ghcr.io/appscode-charts/kubestash-certified \
        --version {{< param "info.version" >}} \
        --namespace stash \
        --set-file global.license=/path/to/the/license.txt \
        --wait --burst-limit=10000 --debug
```

Once installed, follow the [Verify installation](#verify-installation) steps above to confirm the operator and Addon catalogs are running.

## Purchase KubeStash License

If you are interested in purchasing KubeStash license, please contact us via sales@appscode.com for further discussion. You can also set up a meeting via our [calendly link](https://calendly.com/appscode/intro).

If you are willing to purchase KubeStash license but need more time to test in your dev cluster, feel free to contact sales@appscode.com. We will be happy to extend your trial period.
