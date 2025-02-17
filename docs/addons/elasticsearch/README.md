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

## Elasticsearch Addon

```bash
$  kubectl get functions.addons.kubestash.com elasticsearch-addon -oyaml
apiVersion: addons.kubestash.com/v1alpha1
kind: Addon
metadata:
  creationTimestamp: "2025-02-17T07:40:02Z"
  generation: 1
  name: elasticsearch-addon
  resourceVersion: "293799"
  uid: a1386f01-5c39-4291-8604-b19ef1775005
spec:
  backupTasks:
  - driver: Restic
    executor: Job
    function: elasticsearch-backup
    name: logical-backup
    parameters:
    - default: --match=^(?![.])(?!apm-agent-configuration)(?!kubedb-system).+
      name: args
      required: false
      usage: Arguments to be passed to the dump command.
    - default: /kubestash-interim/data
      name: interimDataDir
      required: false
      usage: Directory where data will be stored temporarily before uploading to the
        backend.
    - default: "true"
      name: enableCache
      required: false
      usage: Enable or disable caching. Disabling caching may impact backup performance.
    - default: /kubestash-tmp
      name: scratchDir
      required: false
      usage: Directory for holding temporary files and restic cache.
    singleton: true
    volumeMounts:
    - mountPath: /kubestash-interim
      name: kubestash-interim-volume
    - mountPath: /kubestash-tmp
      name: kubestash-tmp-volume
    volumeTemplate:
    - name: kubestash-interim-volume
      source:
        emptyDir: {}
      usage: Holds backed up data temporarily before uploading to the backend.
    - name: kubestash-tmp-volume
      source:
        emptyDir: {}
      usage: Holds temporary files and restic cache.
  - driver: Restic
    executor: Job
    function: elasticsearch-dashboard-backup
    name: dashboard-backup
    parameters:
    - default: /kubestash-interim/data
      name: interimDataDir
      required: false
      usage: Directory where data will be stored temporarily before uploading to the
        backend.
    - default: "true"
      name: enableCache
      required: false
      usage: Enable or disable caching. Disabling caching may impact backup performance.
    - default: /kubestash-tmp
      name: scratchDir
      required: false
      usage: Directory for holding temporary files and restic cache.
    singleton: true
    volumeMounts:
    - mountPath: /kubestash-interim
      name: kubestash-interim-volume
    - mountPath: /kubestash-tmp
      name: kubestash-tmp-volume
    volumeTemplate:
    - name: kubestash-interim-volume
      source:
        emptyDir: {}
      usage: Holds backed up data temporarily before uploading to the backend.
    - name: kubestash-tmp-volume
      source:
        emptyDir: {}
      usage: Holds temporary files and restic cache.
  restoreTasks:
  - driver: Restic
    executor: Job
    function: elasticsearch-restore
    name: logical-backup-restore
    parameters:
    - default: --match=^(?![.])(?!apm-agent-configuration)(?!kubedb-system).+
      name: args
      required: false
      usage: Arguments to be passed to the dump command.
    - default: /kubestash-interim/data
      name: interimDataDir
      required: false
      usage: Specifies the directory where data will be stored temporarily before
        dumping to the database.
    - default: "true"
      name: enableCache
      required: false
      usage: Enable or disable caching. Disabling caching may impact backup performance.
    - default: /kubestash-tmp
      name: scratchDir
      required: false
      usage: Directory for holding temporary files and restic cache.
    singleton: true
    volumeMounts:
    - mountPath: /kubestash-interim
      name: kubestash-interim-volume
    - mountPath: /kubestash-tmp
      name: kubestash-tmp-volume
    volumeTemplate:
    - name: kubestash-interim-volume
      source:
        emptyDir: {}
      usage: Holds backed up data temporarily before dumping to the database.
    - name: kubestash-tmp-volume
      source:
        emptyDir: {}
      usage: Holds temporary files and restic cache.
  - driver: Restic
    executor: Job
    function: elasticsearch-dashboard-restore
    name: dashboard-restore
    parameters:
    - default: /kubestash-interim/data
      name: interimDataDir
      required: false
      usage: Specifies the directory where data will be stored temporarily before
        restoring to the dashboard.
    - default: "true"
      name: enableCache
      required: false
      usage: Enable or disable caching. Disabling caching may impact backup performance.
    - default: /kubestash-tmp
      name: scratchDir
      required: false
      usage: Directory for holding temporary files and restic cache.
    singleton: true
    volumeMounts:
    - mountPath: /kubestash-interim
      name: kubestash-interim-volume
    - mountPath: /kubestash-tmp
      name: kubestash-tmp-volume
    volumeTemplate:
    - name: kubestash-interim-volume
      source:
        emptyDir: {}
      usage: Holds backed up data temporarily before dumping to the database.
    - name: kubestash-tmp-volume
      source:
        emptyDir: {}
      usage: Holds temporary files and restic cache.
```

## Elasticsearch Backup Function


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



# Elasticsearch Restore Function

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

## Documentation Overview

KubeStash Elasticsearch documentations are organized as below:

- [How does it work?](/docs/addons/elasticsearch/overview/index.md) gives an overview of how backup and restore process for Elasticsearch database works in Stash.
- [Standalone Elasticsearch Database](/docs/addons/elasticsearch/logical/index.md) shows how to backup and restore an externally managed Elasticsearch database.