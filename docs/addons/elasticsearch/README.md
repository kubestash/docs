---
title: Elasticsearch Addon Overview | KubeStash
description: Elasticsearch Addon Overview | KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-elasticsearch-readme
    name: Readme
    parent: kubestash-elasticsearch
    weight: -1
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
url: /docs/{{ .version }}/addons/elasticsearch/
aliases:
  - /docs/{{ .version }}/addons/elasticsearch/README/
---

# KubeStash Elasticsearch Addon

KubeStash v2024.9.30+ supports extending its functionality through addons. KubeStash Elasticsearch addon enables KubeStash to backup and restore Elasticsearch databases.

This guide will give you an overview of which Elasticsearch versions are supported and how the docs are organized.

## Supported Elasticsearch Versions

**Backup Versions**

To find the supported Elasticsearch versions for backup operations in your Kubernetes cluster, you can inspect the `.spec.availableVersions` field of the `Elasticsearch-backup` function.

Let's check the supported versions for backup:

```bash
$  kubectl get functions.addons.kubestash.com elasticsearch-backup -oyaml
apiVersion: addons.kubestash.com/v1alpha1
kind: Function
metadata:
  annotations:
    meta.helm.sh/release-name: kubedb
    meta.helm.sh/release-namespace: kubedb
  creationTimestamp: "2025-01-31T11:20:03Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: kubedb
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubedb-kubestash-catalog
    app.kubernetes.io/version: v2025.1.9
    helm.sh/chart: kubedb-kubestash-catalog-v2025.1.9
  name: elasticsearch-backup
  resourceVersion: "2356"
  uid: dec51618-8c80-4d79-9d63-52ec4308b07e
spec:
  args:
  - backup
  - --namespace=${namespace:=default}
  - --backupsession=${backupSession:=}
  - --enable-cache=${enableCache:=}
  - --scratch-dir=${scratchDir:=}
  - --wait-timeout=${waitTimeout:=300}
  - --es-args=${args:=}
  - --interim-data-dir=${interimDataDir:=}
  image: ghcr.io/kubedb/elasticsearch-restic-plugin:v0.14.0
```

Here,
- `spec.availableVersions` specifies the version list which are supported for backup.


**Restore Versions**

To find the supported Elasticsearch versions for restore operations in your Kubernetes cluster, you can inspect the `.spec.availableVersions` field of the `elasticsearch-restore` function.

Let's check the supported versions for restore:

```bash
$  kubectl get functions.addons.kubestash.com elasticsearch-restore -oyaml
apiVersion: addons.kubestash.com/v1alpha1
kind: Function
metadata:
  annotations:
    meta.helm.sh/release-name: kubedb
    meta.helm.sh/release-namespace: kubedb
  creationTimestamp: "2025-01-31T11:20:03Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: kubedb
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubedb-kubestash-catalog
    app.kubernetes.io/version: v2025.1.9
    helm.sh/chart: kubedb-kubestash-catalog-v2025.1.9
  name: elasticsearch-restore
  resourceVersion: "2360"
  uid: 94be8cdc-8f15-40eb-940e-5d58553fc478
spec:
  args:
  - restore
  - --namespace=${namespace:=default}
  - --restoresession=${restoreSession:=}
  - --snapshot=${snapshot:=}
  - --enable-cache=${enableCache:=}
  - --scratch-dir=${scratchDir:=}
  - --wait-timeout=${waitTimeout:=300}
  - --es-args=${args:=}
  - --interim-data-dir=${interimDataDir:=
  image: ghcr.io/kubedb/elasticsearch-restic-plugin:v0.14.0
```

Here,
- `spec.availableVersions` specifies the version list which are supported for restore.


## Addon Version Compatibility

Any database version with a major version matching the supported Elasticsearch versions should generally be able to back up the database. For instance, if the function supports the `8.x.x` version, it should work with any Elasticsearch database within the `8.x.x` series. However, there may be exceptions where certain versions are incompatible. In such cases, use the specific version name that is explicitly supported.

## Documentation Overview

KubeStash Elasticsearch documentations are organized as below:

- [How does it work?](/docs/addons/elasticsearch/overview/index.md) gives an overview of how backup and restore process for Elasticsearch database works in Stash.
- [Standalone Elasticsearch Database](/docs/addons/elasticsearch/logical/index.md) shows how to backup and restore an externally managed Elasticsearch database.