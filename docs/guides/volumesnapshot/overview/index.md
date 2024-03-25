---
title: VolumeSnapshot Overview | KubeStash
description: An overview of how VolumeSnapshot works in KubeStash
menu:
  docs_{{ .version }}:
    identifier: volume-snapshot-overview
    name: How VolumeSnapshot works?
    parent: volume-snapshot
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# VolumeSnapshot Using KubeStash

This guide will give you an overview of how VolumeSnapshot process works in KubeStash.

## How Backup Process Works?

The following diagram shows how KubeStash creates VolumeSnapshot via Kubernetes native API. Open the image in a new tab to see the enlarged version.

<figure align="center">
  <img alt="KubeStash Backup Flow" src="/docs/guides/volumesnapshot/overview/images/volumesnapshot-overview.svg">
<figcaption align="center">Fig: Volume Snapshotting Process in KubeStash</figcaption>
</figure>

The `VolumeSnapshot` process consists of the following steps:

1. At first, a user creates a `Secret` with access credentials of the backend where the backed-up data will be stored. 

2. Then, she creates a `BackupStorage` custom resource that specifies the backend information, along with the `Secret` containing the credentials needed to access the backend. 

3. KubeStash operator watches for `BackupStorage` custom resources. When it finds a `BackupStorage` object, it initializes the `BackupStorage` by uploading the `metadata.yaml` file to the target storage. 

4. Then, she creates a `BackupConfiguration` custom resource that specifies the targeted workload/PVC, the Addon info with a specified task, etc. It also provides information about one or more repositories, each indicating a path and a `BackupStorage` for storing the metadata.

5. KubeStash operator watches for `BackupConfiguration` custom resources. 

6. Once the KubeStash operator finds a `BackupConfiguration` object, it creates `Repository` with the information specified in the BackupConfiguration. 

7. KubeStash operator watches for `Repository` custom resources. When it finds the `Repository` object, it Initializes `Repository` by uploading `repository.yaml` file to the `spec.sessions[*].repositories[*].directory` path specified in the `BackupConfiguration`. 

8. Then, it creates a `CronJob` with the schedule specified in `BackupConfiguration` to trigger backup periodically. 

9. On the next scheduled slot, the `CronJob` triggers a backup by creating a `BackupSession` custom resource. 

10. KubeStash operator watches for `BackupSession` custom resources. 

11. When it finds a `BackupSession` object, it creates a `Snapshot` custom resource for each `Repository` specified in the respective session of the `BackupConfiguration`. 

12. Then, it resolves the respective `Addon` and `Function` and prepares a volume snapshotter `Job` definition. 

13. Then, It creates a volume snapshotter `Job` to capture snapshots of the targeted volumes. 

14. The volume snapshotter `Job` creates a `VolumeSnapshot`(s) object for each Persistent Volume Claims (PVC) associated the target and waits for the Container Storage Interface (CSI) driver to complete snapshotting. The VolumeSnapshot(s) are created with the following naming format:
    ```bash
      <PVC name>-<BackupSession creation timestamp in Unix epoch seconds>
    ```

15. Container Storage Interface (CSI) `external-snapshotter` controller watches for `VolumeSnapshot` resources.

16. When it finds a `VolumeSnapshot` object, it backups `VolumeSnapshot` to the respective cloud storage. 

17. After the snapshotting process is completed, the volume snapshotter `Job` updates the `status.components[*]` field of the `Snapshot` resources with`VolumeSnapshot` information.


## How Restore Process Works?

The following diagram shows how KubeStash restores PersistentVolumeClaims from snapshot using Kubernetes VolumeSnapshot API. Open the image in a new tab to see the enlarged version.

<figure align="center">
  <img alt="KubeStash Restore Flow" src="/docs/guides/volumesnapshot/overview/images/restore-overview.svg">
<figcaption align="center">Fig: Restore process from snapshot in KubeStash</figcaption>
</figure>

The restore process consists of the following steps:

1. At first, a user creates a `RestoreSession` custom resource that specifies the `volumeClaimTemplates`, Addon info with a specified task, and a `DataSource` that determines the `Snapshot` from which the data will be restored.

2. KubeStash operator watches for `RestoreSession` custom resources.

3. Once it finds a `RestoreSession` object, it resolves the respective `Addon` and `Function` and prepares a restore Job definition.

4. Then, it creates a Restore `Job` to restore `PVC` from the `volumeSnapshot`.

5. The restore job retrieves `VolumeSnapshot` information from the Snapshot. It starts by creating PVCs based on `volumeClaimTemplates` and assigns the corresponding `VolumeSnapshot` name to the `spec.dataSourceRef` of each PVC.

6. Container Storage Interface (CSI) `external-snapshotter` controller watches for `PVCs`.

7. When it finds a new `PVC` with `spec.dataSource` field set, it reads the information about the `VolumeSnapshot`.

8. The controller downloads the respective data from the cloud and populates the `PVC`.

9. Finally, when the restore process is completed, the restore `Job` updates the `status.components[*]` field of the `RestoreSession` with restore information of the target application components.


## Next Steps
1. See a step by step guide to snapshot a stand-alone PVC [here](/docs/guides/volumesnapshot/pvc/index.md).
2. See a step by step guide to snapshot the volumes of a Deployment [here](/docs/guides/volumesnapshot/deployment/index.md).
3. See a step by step guide to snapshot the volumes of a StatefulSet [here](/docs/guides/volumesnapshot/statefulset/index.md).
