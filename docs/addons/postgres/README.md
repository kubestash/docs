---
title: PostgreSQL Addon Overview | KubeStash
description: PostgreSQL Addon Overview | KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-postgres-readme
    name: Readme
    parent: kubestash-postgresql
    weight: -1
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
url: /docs/{{ .version }}/addons/postgres/
aliases:
  - /docs/{{ .version }}/addons/postgres/README/
---

# KubeStash PostgreSQL Addon

KubeStash `v2024.9.30+` supports extending its functionality through addons. KubeStash `PostgreSQL` addon enables KubeStash to backup and restore `PostgreSQL` databases.

This guide will give you an overview of which `PostgreSQL` versions are supported and how the docs are organized.

## Supported PostgreSQL Versions

**Backup Versions**

To find the supported MySQL versions for backup operations in your Kubernetes cluster, you can inspect the `.spec.availableVersions` field of the `mysql-backup` function.

Let's check the supported versions for backup:

```bash
$ kubectl get functions.addons.kubestash.com postgres-backup  -o yaml
apiVersion: addons.kubestash.com/v1alpha1
kind: Function
metadata:
  annotations:
    meta.helm.sh/release-name: kubedb-kubestash-catalog
    meta.helm.sh/release-namespace: stash
  creationTimestamp: "2025-01-31T11:18:00Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: kubedb-kubestash-catalog
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubedb-kubestash-catalog
    app.kubernetes.io/version: v2025.1.9
    helm.sh/chart: kubedb-kubestash-catalog-v2025.1.9
  name: postgres-backup
  resourceVersion: "47922"
  uid: b54e09ea-1099-4a3b-a8ee-94921294b0f5
spec:
  args:
  - backup
  - --namespace=${namespace:=default}
  - --backupsession=${backupSession:=}
  - --enable-cache=${enableCache:=}
  - --scratch-dir=${scratchDir:=}
  - --wait-timeout=${waitTimeout:=300}
  - --pg-args=${args:=}
  - --backup-cmd=${backupCmd:=}
  - --user=${user:=}
  availableVersions:
  - "12.17"
  - "14.10"
  - "16.1"
  - "17.2"
  image: ghcr.io/kubedb/postgres-restic-plugin:v0.14.0_${DB_VERSION}
```

Here,
- `spec.availableVersions` specifies the version list which are supported for backup. 

**Restore Versions**

To find the supported `PostgreSQL` versions for restore operations in your Kubernetes cluster, you can inspect the `.spec.availableVersions` field of the `postgres-restore` function.

Let's check the supported versions for restore:

```bash
$ kubectl get functions.addons.kubestash.com postgres-restore  -o yaml
apiVersion: addons.kubestash.com/v1alpha1
kind: Function
metadata:
  annotations:
    meta.helm.sh/release-name: kubedb-kubestash-catalog
    meta.helm.sh/release-namespace: stash
  creationTimestamp: "2025-01-31T11:18:00Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: kubedb-kubestash-catalog
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubedb-kubestash-catalog
    app.kubernetes.io/version: v2025.1.9
    helm.sh/chart: kubedb-kubestash-catalog-v2025.1.9
  name: postgres-restore
  resourceVersion: "47896"
  uid: cac78178-c1c2-4761-a848-69cb4c0bb9d6
spec:
  args:
  - restore
  - --namespace=${namespace:=default}
  - --restoresession=${restoreSession:=}
  - --snapshot=${snapshot:=}
  - --enable-cache=${enableCache:=}
  - --scratch-dir=${scratchDir:=}
  - --wait-timeout=${waitTimeout:=300}
  - --pg-args=${args:=}
  - --user=${user:=}
  availableVersions:
  - "12.17"
  - "14.10"
  - "16.1"
  - "17.2"
  image: ghcr.io/kubedb/postgres-restic-plugin:v0.14.0_${DB_VERSION}
```

Here,
- `spec.availableVersions` specifies the version list which are supported for restore.

## Addon Version Compatibility

Any database version with a major version matching the supported `PostgreSQL` versions should generally be able to back up the database. For instance, if the function supports the `8.x.x` version, it should work with any `PostgreSQL` database within the `8.x.x` series. However, there may be exceptions where certain versions are incompatible. In such cases, use the specific version name that is explicitly supported.

## Documentation Overview

KubeStash `PostgreSQL` documentations are organized as below:

- [How does it work?](/docs/addons/postgres/overview/index.md) gives an overview of how backup and restore process for PostgreSQL database works in KubeStash.
- [Standalone PostgreSQL Database](/docs/addons/postgres/logical/index.md) shows how to backup and restore an externally managed PostgreSQL database.