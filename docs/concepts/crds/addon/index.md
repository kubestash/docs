---
title: Addon Overview
menu:
  docs_{{ .version }}:
    identifier: addon-overview
    name: Addon
    parent: crds
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# Addon

## What is Addon
An `Addon` is a Kubernetes `CustomResourceDefinition`(CRD) which specifies the backup and restore capabilities for a particular resource. For example, MySQL addon specifies the backup and restore capabilities of MySQL database where Postgres addon specifies backup and restore capabilities for PostgreSQL database. An Addon CR defines the backup and restore tasks that can be performed by this addon.

When you install KubeStash, some `Addon`s will be pre-installed for supported targets like workloads, etc. However, you can create your own `Addon` to customize or extend the backup/restore process.

## Addon CRD Specification
Like any official Kubernetes resource, an `Addon` has `TypeMeta`, `ObjectMeta` and `Spec` sections. However, unlike other Kubernetes resources, it does not have a `Status` section.

A sample `Addon` object for a PostgreSQL database is shown below:
```yaml
apiVersion: addons.kubestash.com/v1alpha1
kind: Addon
metadata:
  labels:
    app.kubernetes.io/instance: kubedb
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubedb-kubestash-catalog
    app.kubernetes.io/version: v2024.2.14
    helm.sh/chart: kubedb-kubestash-catalog-v2024.2.14
  name: postgres-addon
spec:
  backupTasks:
    - driver: Restic
      executor: Job
      function: postgres-backup
      name: logical-backup
      parameters:
        - name: args
          required: false
          usage: Arguments to be passed to the dump command.
        - default: pg_dumpall
          name: backupCmd
          required: false
          usage: Backup command to take a database dump (can only be pg_dumpall or pg_dump)
        - default: postgres
          name: user
          required: false
          usage: Specifies database user (not applicable for basic authentication)
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
        - mountPath: /kubestash-tmp
          name: kubestash-tmp-volume
      volumeTemplate:
        - name: kubestash-tmp-volume
          source:
            emptyDir: {}
          usage: Holds temporary files and restic cache.
    - driver: VolumeSnapshotter
      executor: Job
      function: postgres-csi-snapshotter
      name: volume-snapshot
      parameters:
        - name: volumeSnapshotClassName
          required: false
          usage: The VolumeSnapshotClassName to be used by volumeSnapshot
      singleton: true
    - driver: Restic
      executor: Job
      function: kubedbmanifest-backup
      name: manifest-backup
      parameters:
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
        - mountPath: /kubestash-tmp
          name: kubestash-tmp-volume
      volumeTemplate:
        - name: kubestash-tmp-volume
          source:
            emptyDir: {}
          usage: Holds temporary files and restic cache.
  restoreTasks:
    - driver: Restic
      executor: Job
      function: postgres-restore
      name: logical-backup-restore
      parameters:
        - name: args
          required: false
          usage: Arguments to be passed to the dump command.
        - default: postgres
          name: user
          required: false
          usage: Specifies database user (not applicable for basic authentication)
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
        - mountPath: /kubestash-tmp
          name: kubestash-tmp-volume
      volumeTemplate:
        - name: kubestash-tmp-volume
          source:
            emptyDir: {}
          usage: Holds temporary files and restic cache.
    - driver: Restic
      executor: Job
      function: kubedbmanifest-restore
      name: manifest-restore
      parameters:
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
        - mountPath: /kubestash-tmp
          name: kubestash-tmp-volume
      volumeTemplate:
        - name: kubestash-tmp-volume
          source:
            emptyDir: {}
          usage: Holds temporary files and restic cache.
```
This `Addon` has three backup tasks and two restore tasks for a PostgreSQL database. Each task refers to a `Function`.

Here, we are going to describe the various sections of a `Addon` crd.

## Addon `Spec`
A `Addon` object has the following fields in the `spec` section:
- **backupTasks** specifies a list of backup tasks that can be performed by the addon. See [task specification](#task-specification) to learn more about backup tasks.
- **restoreTasks** specifies a list of restore tasks that can be performed by the addon. See [task specification](#task-specification) to learn more about restore tasks.

### Task Specification
Each task contains the following fields:
- **name** specifies the name of the task. The name of a task should indicate what this task does. For example, a name `logical-backup` indicate that this task performs a logical backup of a database.
- **function** specifies the name of a `Function` CR that defines a container definition which will execute the backup/restore logic for a particular application.
- **driver** specifies the underlying tool that will be used to upload the data to the backend storage. Valid values are:
  - **Restic** The underlying tool is [restic](https://restic.net/).
  - **WalG** The underlying tool is [wal-g](https://github.com/wal-g/wal-g).
  - **VolumeSnapshotter** The underlying method is creating Kubernetes [VolumeSnapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) crd and leverages CSI driver to snapshot the PVCs.
- **executor** specifies the type of entity that will execute the task. For example, it can be Job(s), a sidecar container, an ephemeral container, or a Job that creates additional Jobs/Pods for executing the backup/restore logic. Valid values are:
  - **Job** KubeStash will create Job(s) to execute the backup/restore task.
  - **Sidecar** KubeStash will inject a sidecar container into the application to execute the backup/restore task.
  - **EphemeralContainer** KubeStash will attach an ephemeral container to the respective Pods to execute the backup/restore task.
  - **MultiLevelJob** KubeStash will create a Job that will create additional Jobs/Pods to execute the backup/restore task.
- **singleton** specifies whether this task will be executed on a single job or across multiple jobs.
- **parameters** defines a list of parameters that is used by the task to execute its logic. Parameter definition defines the parameter names, their usage, their requirements etc.
  - **name** specifies the name of the parameter.
  - **usage** specifies the usage of this parameter.
  - **required** specify whether this parameter is required or not.
  - **default** specifies a default value for the parameter
- **volumeTemplate** specifies a list of volume templates that is used by the respective backup/restore Job to execute its logic. You can overwrite these volume templates using `addonVolumes` field of `BackupConfiguration`. Each `volumeTemplate` contains:
  - **name** specifies the name of the volume.
  - **usage** specifies the usage of the volume.
  - **source** specifies the source of this volume or a template for volume to use.
- **volumeMounts** specifies the mount path of the volumes specified in the `VolumeTemplate` section. These volumes will be mounted directly on the Job/Container created/injected by KubeStash operator. If the volume type is `VolumeClaimTemplate`, then KubeStash operator is responsible for creating the volume.
- **passThroughMounts** specifies a list of volume mount for the `VolumeTemplates` that should be mounted on second level Jobs/Pods created by the first level executor Job. If the volume needs to be mounted on both first level and second level Jobs/Pods, then specify the mount in both `VolumeMounts` and `PassThroughMounts` section. If the volume type is `VolumeClaimTemplate`, then the first level job is responsible for creating the volume.

## Why Function and Addon?

You might be wondering why we have introduced `Function` and `Addon` crd. We have designed `Function-Addon` model for the following reasons:

- **Customizable:** `Function` and `Addon` enables you to customize backup/recovery process. For example, currently we use [mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html) in `mysql-backup` task to backup MySQL database. You can build a custom `Function` using Percona's [xtrabackup](https://www.percona.com/software/mysql-database/percona-xtrabackup) tool instead of `mysqldump`. Then you can write an `Addon` or add a task in existing MySQL `Addon` with this custom `Function` and use it to backup your target MySQL database.
- **Extensibility:** You can easily backup the databases that are not officially supported by KubeStash. You just need to create `Function`s and an `Addon` for your desired database.
- **Re-usability:** `Function`s are self-sufficient and independent of KubeStash. So, you can reuse them in any application that uses `Function-Addon` model.

## Next Steps

- Learn how KubeStash backup stand-alone PVC using `Function-Addon` model from [here](/docs/guides/volumes/overview/index.md).