---
title: MySQL Addon Overview | KubeStash
description: MySQL Addon Overview | KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-mysql-readme
    name: Readme
    parent: kubestash-mysql
    weight: -1
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
url: /docs/{{ .version }}/addons/mysql/
aliases:
  - /docs/{{ .version }}/addons/mysql/README/
---

# KubeStash MySQL Addon

KubeStash v2024.9.30+ supports extending its functionality through addons. KubeStash MySQL addon enables KubeStash to backup and restore MySQL databases.

This guide will give you an overview of which MySQL versions are supported and how the docs are organized.

## Supported MySQL Versions

**Backup Versions**

To find the supported MySQL versions for backup operations in your Kubernetes cluster, you can inspect the `.spec.availableVersions` field of the `mysql-backup` function.

Let's check the supported versions for backup:

```bash
$  kubectl get functions.addons.kubestash.com mysql-backup  -o yaml
apiVersion: addons.kubestash.com/v1alpha1
kind: Function
metadata:
  annotations:
    meta.helm.sh/release-name: kubedb
    meta.helm.sh/release-namespace: kubedb
  creationTimestamp: "2024-12-11T06:05:43Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: kubedb
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubedb-kubestash-catalog
    app.kubernetes.io/version: v2024.11.18
    helm.sh/chart: kubedb-kubestash-catalog-v2024.11.18
  name: mysql-backup
  resourceVersion: "86023"
  uid: f3b159ba-2f7d-4a6b-bc14-d7e992685632
spec:
  args:
  - backup
  - --namespace=${namespace:=default}
  - --backupsession=${backupSession:=}
  - --enable-cache=${enableCache:=}
  - --scratch-dir=${scratchDir:=}
  - --wait-timeout=${waitTimeout:=300}
  - --mysql-args=${args:=}
  - --db-version=${dbVersion:=}
  - --databases=${databases:=}
  availableVersions:
  - 5.7.25
  - 8.0.3
  - 8.0.21
  image: ghcr.io/kubedb/mysql-restic-plugin:v0.12.0_${DB_VERSION}
```

Here,
 - `spec.availableVersions` specifies the version list which are supported for backup. 


**Restore Versions**

To find the supported MySQL versions for restore operations in your Kubernetes cluster, you can inspect the `.spec.availableVersions` field of the `mysql-restore` function.

Let's check the supported versions for restore:

```bash
$  kubectl get functions.addons.kubestash.com mysql-restore  -o yaml
apiVersion: addons.kubestash.com/v1alpha1
kind: Function
metadata:
  annotations:
    meta.helm.sh/release-name: kubedb
    meta.helm.sh/release-namespace: kubedb
  creationTimestamp: "2024-12-11T06:05:43Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: kubedb
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubedb-kubestash-catalog
    app.kubernetes.io/version: v2024.11.18
    helm.sh/chart: kubedb-kubestash-catalog-v2024.11.18
  name: mysql-restore
  resourceVersion: "85997"
  uid: 94a1e91f-3f2c-4274-a454-0aefdb351eca
spec:
  args:
  - restore
  - --namespace=${namespace:=default}
  - --restoresession=${restoreSession:=}
  - --snapshot=${snapshot:=}
  - --enable-cache=${enableCache:=}
  - --scratch-dir=${scratchDir:=}
  - --wait-timeout=${waitTimeout:=300}
  - --mysql-args=${args:=}
  - --db-version=${dbVersion:=}
  availableVersions:
  - 5.7.25
  - 8.0.3
  - 8.0.21
  image: ghcr.io/kubedb/mysql-restic-plugin:v0.12.0_${DB_VERSION}
```

Here,
- `spec.availableVersions` specifies the version list which are supported for restore.


## Addon Version Compatibility

Any database version with a major version matching the supported MySQL versions should generally be able to back up the database. For instance, if the function supports the `8.x.x` version, it should work with any MySQL database within the `8.x.x` series. However, there may be exceptions where certain versions are incompatible. In such cases, use the specific version name that is explicitly supported.

## Documentation Overview

KubeStash MySQL documentations are organized as below:

- [How does it work?](/docs/addons/mysql/overview/index.md) gives an overview of how backup and restore process for MySQL database works in KubeStash.
- [Standalone MySQL Database](/docs/addons/mysql/logical/index.md) shows how to backup and restore an externally managed MySQL database.