---
title: Uninstall kubeStash
description: Uninstallation guide for KubeStash
menu:
  docs_{{ .version }}:
    identifier: uninstall-kubestash-enterprise
    name: KubeStash
    parent: uninstallation-guide
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Uninstall KubeStash

To uninstall KubeStash, run the following command:

<ul class="nav nav-tabs" id="installerTab" role="tablist">
  <li class="nav-item">
    <a class="nav-link active" id="helm3-tab" data-toggle="tab" href="#helm3" role="tab" aria-controls="helm3" aria-selected="true">Helm 3</a>
  </li>
  <li class="nav-item">
    <a class="nav-link" id="script-tab" data-toggle="tab" href="#script" role="tab" aria-controls="script" aria-selected="false">YAML</a>
  </li>
</ul>
<div class="tab-content" id="installerTabContent">
  <div class="tab-pane fade show active" id="helm3" role="tabpanel" aria-labelledby="helm3-tab">

## Using Helm 3

In Helm 3, release names are [scoped to a namespace](https://v3.helm.sh/docs/faq/#release-names-are-now-scoped-to-the-namespace). So, provide the namespace you used to install the operator when installing.

```bash
$ helm uninstall kubestash --namespace kubestash
```

</div>
<div class="tab-pane fade" id="script" role="tabpanel" aria-labelledby="script-tab">

## Using YAML (with helm 3)

If you prefer to not use Helm, you can generate YAMLs from KubeStash chart and uninstall using `kubectl`.

```bash
$ helm template kubestash oci://ghcr.io/appscode-charts/kubestash \
        --version {{< param "info.version" >}} \
        --namespace kubestash --create-namespace \
        --set-file global.license="nothing" | kubectl delete -f -
```

</div>
</div>

## Delete CRDs

The above uninstallation process will uninstall the KubeStash operator. However, it will keep the KubeStash registered CRDs so that you don't lose your KubeStash objects i.e. `BackupConfiguration`, `BackupStorage`, etc. during re-installation. If you want to remove the KubeStash CRDs too, please run the following command.

```bash
kubectl delete crd -l=app.kubernetes.io/name=kubestash
```

If you want to delete the `AppBinding` CRD, run the following command.

```bash
 kubectl delete crd -l=app.kubernetes.io/name=catalog
```
