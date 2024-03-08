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

3. KubeStash operator watches for `BackupStorage` custom resources. When it finds a `BackupStorage` object, it initializes the `BackupStorage` by uploading the `metadata.yaml` file into the target storage. 

4. Then, she creates a `BackupConfiguration` custom resource, targeting either a workload application or a standalone volume. The `BackupConfiguration` object specifies the `Repository` pointing to a `BackupStorage` that contains backend information, indicating where to upload backup data. It also defines the `Addon` information with a specified task and their configuration parameters to be used for backing up the volumes. 

5. KubeStash operator watches for `BackupConfiguration` custom resources. 

6. Once the KubeStash operator finds a `BackupConfiguration` object, it creates `Repository` with the information specified in the BackupConfiguration. 

7. KubeStash operator watches for `Repository` custom resources. When it finds the `Repository` object, it Initializes `Repository` by uploading `repository.yaml` file into the `spec.sessions[*].repositories[*].directory` path specified in `BackupConfiguration`. 

8. Then, it creates a `CronJob` with the schedule specified in `BackupConfiguration` to trigger backup periodically. 

9. On the next scheduled slot, the `CronJob` triggers a backup by creating a `BackupSession` custom resource. 

10. KubeStash operator watches for `BackupSession` custom resources. 

11. When it finds a `BackupSession` object, it creates a `Snapshot` custom resource for each `Repository` specified in the `BackupConfiguration`. 

12. Then, it resolves the respective `Addon` and `Function` and prepares a volume snapshotter `Job` definition. 

13. Then, It creates a volume snapshotter `Job` to capture snapshots of the targeted volumes. 

14. The volume snapshotter `Job` creates a `VolumeSnapshot` object for each Persistent Volume Claim (PVC) of the target and waits for the Container Storage Interface (CSI) driver to complete snapshotting.

    These `VolumeSnasphot` custom resources names follow the following format:
    ```bash
      <PVC name>-<BackupSession creation timestamp in Unix epoch seconds>
    ```

15. Container Storage Interface (CSI) `external-snapshotter` controller watches for `VolumeSnapshot` resources.

16. When it finds a `VolumeSnapshot` object, it backups `VolumeSnapshot` into the respective cloud storage. 

17. After the snapshotting process is completed, the volume snapshotter `Job` updates the `status.components[*]` field of the `Snapshot` resources with each target `PVC` and the corresponding created `VolumeSnapshot` information. It also updates the `status.phase` field of the `BackupSession` to reflect backup completion.

## How Restore Process Works?

The following diagram shows how KubeStash restores PersistentVolumeClaims from snapshot using Kubernetes VolumeSnapshot API. Open the image in a new tab to see the enlarged version.

<figure align="center">
  <img alt="KubeStash Restore Flow" src="/docs/guides/volumesnapshot/overview/images/restore-overview.svg">
<figcaption align="center">Fig: Restore process from snapshot in KubeStash</figcaption>
</figure>

The restore process consists of the following steps:

1. At first, a user creates a `RestoreSession` custom resource, specifying the `volumeClaimTemplates`, the `Repository` object that points to a `BackupStorage` that holds backend information, and the target `Snapshot`, which holds information about the `VolumeSnapshots` created during backup. It also specifies the `Addon` info with a task to use to restore the volume.

2. KubeStash operator watches for `RestoreSession` custom resources.

3. Once it finds a `RestoreSession` object, it resolves the respective `Addon` and `Function` and prepares a restore Job definition.

4. Then, it creates a Restore `Job` to restore `PVC` from the `volumeSnapshot`.

5. The restore `Job` gets VolumeSnapshot's information from Snapshot `status.phase` and creates `PVCs` with `spec.dataSourceRef` field set to the respective VolumeSnapshot name.

6. Container Storage Interface (CSI) `external-snapshotter` controller watches for `PVCs`.

7. When it finds a new `PVC` with `spec.dataSource` field set, it reads the information about the `VolumeSnapshot`.

8. The controller downloads the respective data from the cloud and populates the `PVC`.

9. Once completed, the job updates the `status.phase` field of the `Restoresession` to reflect restore completion.

## Next Steps
1. See a step by step guide to snapshot a stand-alone PVC [here](/docs/guides/volumesnapshot/pvc/index.md).
2. See a step by step guide to snapshot the volumes of a Deployment [here](/docs/guides/volumesnapshot/deployment/index.md).
3. See a step by step guide to snapshot the volumes of a StatefulSet [here](/docs/guides/volumesnapshot/statefulset/index.md).
