---
title: Upgrade | KubeStash
description: KubeStash Upgrade
menu:
  docs_{{ .version }}:
    identifier: upgrade-kubestash
    name: Upgrade
    parent: setup
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Upgrading KubeStash

This guide will show you how to upgrade various KubeStash components. Here, we are going to show how to upgrade from an old KubeStash version to the new version, and how to update the license, etc.

## Upgrading KubeStash

In order to upgrade KubeStash from `v2023.xx.xx` to `{{< param "info.version" >}}`, please follow the following steps.

#### 1. Update KubeStash CRDs

Helm [does not upgrade the CRDs](https://github.com/helm/helm/issues/6581) bundled in a Helm chart if the CRDs already exist. So, to upgrade the KubeStash catalog CRDs, please run the following commands below:

```bash
# Update catalog CRDs
$ kubectl apply -f https://github.com/kubestash/installer/raw/{{< param "info.version" >}}/crds/kubestash-crds.yaml
```

#### 2. Upgrade Kubestash Operator

Now, upgrade the KubeStash helm chart using the following command. You can find the latest installation guide [here]().

```bash
helm upgrade kubestash oci://ghcr.io/appscode-charts/kubestash \
    --version {{< param "info.version" >}} \
    --namespace stash \
    --set-file global.license=/path/to/the/license.txt \
    --wait --burst-limit=10000 --debug
```

## Updating License

KubeStash support updating license without requiring any re-installation. KubeStash creates a Secret named `<helm release name>-license` with the license file. You just need to update the Secret. The changes will propagate automatically to the operator and it will use the updated license going forward.

Follow the below instructions to update the license:

- Get a new license and save it into a file.
- Then, run the following upgrade command based on your installation.

<ul class="nav nav-tabs" id="luTabs" role="tablist">
  <li class="nav-item">
    <a class="nav-link active" id="lu-helm3-tab" data-toggle="tab" href="#lu-helm3" role="tab" aria-controls="lu-helm3" aria-selected="true">Helm 3</a>
  </li>
  <li class="nav-item">
    <a class="nav-link" id="lu-yaml-tab" data-toggle="tab" href="#lu-yaml" role="tab" aria-controls="lu-yaml" aria-selected="false">YAML</a>
  </li>
</ul>
<div class="tab-content" id="luTabContent">
  <div class="tab-pane fade show active" id="lu-helm3" role="tabpanel" aria-labelledby="lu-helm3">

#### Using Helm 3

```bash
# detect current version
$ helm ls -A | grep kubestash

# update license key keeping the current version
$ helm upgrade kubestash oci://ghcr.io/appscode-charts/kubestash \
    --version=<cur_version> \
    --namespace stash --create-namespace \
    --reuse-values \
    --set-file global.license=/path/to/new/license.txt
```

</div>
<div class="tab-pane fade" id="lu-yaml" role="tabpanel" aria-labelledby="lu-yaml">

#### Using YAML (with helm 3)

```bash
# detect current version
$ helm ls -A | grep kubestash

# update license key keeping the current version
$ helm template kubestash oci://ghcr.io/appscode-charts/kubestash -n kubestash \
    --version=<cur_version> \
    --namespace stash --create-namespace \
    --set global.skipCleaner=true \
    --show-only appscode/kubestash-operator/templates/license.yaml \
    --set-file global.license=/path/to/new/license.txt | kubectl apply -f -
```

</div>
</div>
