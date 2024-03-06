---
title: Troubleshooting KubeStash Installation
description: Troubleshooting guide for KubeStash installation
menu:
  docs_{{ .version }}:
    identifier: install-kubestash-troubleshoot
    name: Troubleshooting
    parent: installation-guide
    weight: 40
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Troubleshooting Kubestash

This guide will show you how to troubleshoot some common KubeStash installation issues.

## Installing in GKE Cluster

If you are installing KubeStash on a GKE cluster, you will need cluster admin permissions to install KubeStash operator. Run the following command to grant admin permission to the cluster.

```bash
$ kubectl create clusterrolebinding "cluster-admin-$(whoami)" \
  --clusterrole=cluster-admin                                 \
  --user="$(gcloud config get-value core/account)"
```

In addition, if your GKE cluster is a [private cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters), you will need to either add an additional firewall rule that allows master nodes access port `8443/tcp` on worker nodes, or change the existing rule that allows access to ports `443/tcp` and `10250/tcp` to also allow access to port `8443/tcp`. The procedure to add or modify firewall rules is described in the official GKE documentation for private clusters mentioned before.

## Configuring Network Volume Accessor

For network volumes like NFS, KubeStash requires deploying a helper deployment within the same namespace as the BackupStorage. This deployment mounts the NFS volume to access the necessary resources. We call this helper deployment `network volume accessor`. You can configure its resources, user id, privileged permission etc. To enable the network volume accessor, run the following command:

<ul class="nav nav-tabs" id="installerTab" role="tablist">
  <li class="nav-item">
    <a class="nav-link active" id="new-installer-tab" data-toggle="tab" href="#new-installation-tab" role="tab" aria-controls="new-installation-tab" aria-selected="true">New Installation</a>
  </li>
  <li class="nav-item">
    <a class="nav-link" id="existing-installation" data-toggle="tab" href="#existing-installation-tab" role="tab" aria-controls="existing-installation-tab" aria-selected="false">Existing Installation</a>
  </li>
</ul>
<div class="tab-content" id="installerTabContent">
  <div class="tab-pane fade show active" id="new-installation-tab" role="tabpanel" aria-labelledby="new-installation-tab">

### New Installation

If you haven't installed KubeStash yet, run the following command to configure the network volume accessor during installation

```bash
$ helm install -i kubestash oci://ghcr.io/appscode-charts/kubestash \
--version {{< param "info.version" >}} \
--namespace kubestash --create-namespace \
--set kubestash-operator.netVolAccessor.cpu=200m \
--set kubestash-operator.netVolAccessor.memory=128Mi \
--set kubestash-operator.netVolAccessor.runAsUser=0 \
--set kubestash-operator.netVolAccessor.privileged=true \
--set-file global.license=/path/to/license-file.txt \
--wait --burst-limit=10000 --debug
```

</div>
<div class="tab-pane fade" id="existing-installation-tab" role="tabpanel" aria-labelledby="existing-installation-tab">

### Existing Installation

If you have installed KubeStash already in your cluster but didn't configure the network volume accessor, you can use `helm upgrade` command to configure it in the existing installation.

```bash
$ helm upgrade -i kubestash oci://ghcr.io/appscode-charts/kubestash \
--version {{< param "info.version" >}} \
--namespace kubestash --create-namespace \
--set kubestash-operator.netVolAccessor.cpu=200m \
--set kubestash-operator.netVolAccessor.memory=128Mi \
--set kubestash-operator.netVolAccessor.runAsUser=0 \
--set kubestash-operator.netVolAccessor.privileged=true \
--set-file global.license=/path/to/license-file.txt \
--wait --burst-limit=10000 --debug
```
</div>
</div>



## Detect KubeStash version

To detect KubeStash version, exec into the operator pod and run `kubestash version` command.

```bash
$ POD_NAMESPACE=kubestash
$ POD_NAME=$(kubectl get pods -n $POD_NAMESPACE -l app.kubernetes.io/name=kubestash-operator -o jsonpath={.items[0].metadata.name})
$ kubectl exec $POD_NAME -c operator -n $POD_NAMESPACE -- /kubestash version

Version = {{< param "info.version" >}}
VersionStrategy = tag
Os = alpine
Arch = amd64
CommitHash = 85b0f16ab1b915633e968aac0ee23f877808ef49
GitBranch = release-0.5
GitTag = {{< param "info.version" >}}
CommitTimestamp = 2020-08-10T05:24:23

$ kubectl exec -it $POD_NAME -c operator -n $POD_NAMESPACE restic version
restic 0.9.6
compiled with go1.9 on linux/amd64
```
