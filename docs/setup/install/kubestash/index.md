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

## Purchase KubeStash License

If you are interested in purchasing KubeStash license, please contact us via sales@appscode.com for further discussion. You can also set up a meeting via our [calendly link](https://calendly.com/appscode/intro).

If you are willing to purchase KubeStash license but need more time to test in your dev cluster, feel free to contact sales@appscode.com. We will be happy to extend your trial period.
