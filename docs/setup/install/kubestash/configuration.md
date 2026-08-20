---
title: KubeStash Common Configuration
description: Verify the KubeStash installation
menu:
  docs_{{ .version }}:
    identifier: install-kubestash-config
    name: Common Configuration
    parent: install-kubestash-enterprise
    weight: 60
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Common Configuration

The steps below apply regardless of which [installation method](/docs/setup/install/kubestash/) you used.

## Network Policy

KubeStash can optionally generate NetworkPolicies that restrict traffic to and from the KubeStash operator and webhook server pods so only the required communication is allowed. This is disabled by default. Enable it through `global.networkPolicy`:

```yaml
global:
  # Controls the network policy creation
  networkPolicy:
    enabled: false
    # flavor selects which network policy API is used.
    # Accepted values: "kubernetes" (default) or "cilium".
    flavor: kubernetes
```

Set `enabled: true` to create the policies. The `flavor` field selects which API the generated policies target: `kubernetes` (the built-in `networking.k8s.io` `NetworkPolicy`, the default) or `cilium` (Cilium's `CiliumNetworkPolicy`, for clusters running the Cilium CNI).

Enable it inline with `--set`:

```bash
$ helm upgrade -i kubestash oci://ghcr.io/appscode-charts/kubestash \
        --version {{< param "info.version" >}} \
        --namespace stash --create-namespace \
        --set-file global.license=/path/to/the/license.txt \
        --set global.networkPolicy.enabled=true \
        --set global.networkPolicy.flavor=kubernetes \
        --wait --burst-limit=10000 --debug
```

### Required network communication

KubeStash can run fully disconnected from the internet, as long as every required image is cached in a registry the cluster can reach (see the [offline installation guide](/docs/setup/install/kubestash/helm.md)).

Within the cluster, the following paths must stay open. When `global.networkPolicy.enabled` is `true`, the generated policies allow exactly these; if you maintain your own policies, make sure to permit them yourself:

1. KubeStash operator to the kube-apiserver.
2. KubeStash operator to DNS.
3. kube-apiserver to the webhook server, for the mutating, validating, and conversion webhook endpoints.

In addition, backup and restore jobs need to reach the workloads they protect and the object storage backend (S3, MinIO, and similar).

## Verify installation

To check if KubeStash operator and webhook pods have started, run the following command:

```bash
$ kubectl get pods -n stash -l app.kubernetes.io/instance=kubestash --watch
NAME                                                           READY   STATUS    RESTARTS   AGE
kubestash-kubestash-operator-fcd8bf7c6-psjs6                   2/2     Running   0          5m49s
kubestash-kubestash-operator-webhook-server-6fb8f5cfb9-scrx8   1/1     Running   0          5m49s
```

Once the operator and webhook pods are running, you can cancel the above command by typing `Ctrl+C`.

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

## Verify Catalogs

KubeStash automatically installs the necessary Addon catalogs for workload, PVC and Kubernetes manifest backups. Verify that the Addon catalogs have been installed using the following command.

```bash
$ kubectl get addons.addons.kubestash.com
NAME             AGE
kubedump-addon   7m1s
pvc-addon        7m1s
vault-addon      7m1s
workload-addon   7m1s
```

As you can see from the above output that KubeStash has created `Addon` objects.

Now, you are ready to [take your first backup](/docs/guides/README.md) using KubeStash.
