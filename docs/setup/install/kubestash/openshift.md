---
title: Install KubeStash on OpenShift
description: Install KubeStash on OpenShift
menu:
  docs_{{ .version }}:
    identifier: install-kubestash-openshift
    name: OpenShift
    parent: install-kubestash-enterprise
    weight: 50
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Install in OpenShift

There are two ways to deploy KubeStash in [OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift). Use Option A for the standard upstream chart, or Option B if you require the Red Hat OpenShift certified chart (for example, to satisfy a Red Hat OpenShift certification requirement).

### Option A: Standard chart with OpenShift values

Use the standard [`kubestash` chart](/docs/setup/install/kubestash/helm.md) and enable the OpenShift distribution values. This switches the operator to UBI-based images and applies the SecurityContextConstraints and other OpenShift-specific tweaks:

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

Once installed, follow the [Common Configuration](/docs/setup/install/kubestash/configuration.md) steps to verify the operator and Addon catalogs are running.
