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

## Verify installation

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

## Verify Catalogs

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
