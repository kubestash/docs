---
title: Snapshot Overview
menu:
  docs_{{ .version }}:
    identifier: snapshot-overview
    name: Snapshot
    parent: crds
    weight: 55
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---
> New to KubeStash? Please start [here](/docs/concepts/README.md).

# Snapshot

## What is Snapshot

A `Snapshot` is a Kubernetes `CustomResourceDefinition`(CRD) which represents the state of a backup run for one or more components of an application. For every `BackupSession`, KubeStash operator creates `Snapshot` CRs. If a `BackupSession` involves multiple repositories, a `Snapshot` is created for each repository. 

> End users are not meant to create or edit Snapshot CRs.

## Snapshot CRD Specification

Like any official Kubernetes resource, a `Snapshot` has `TypeMeta`, `ObjectMeta`, `Spec` and `Status` sections.

A sample `Snapshot` object is shown below,

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: StatefulSet
    kubestash.com/app-ref-name: sample-sts
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-demo-repo
  name: gcs-demo-repo-sample-backup-sts-demo-session-1702543201
  namespace: demo
spec:
  appRef:
    apiGroup: apps
    kind: StatefulSet
    name: sample-sts
    namespace: demo
  backupSession: sample-backup-sts-demo-session-1702543201
  deletionPolicy: Delete
  repository: gcs-demo-repo
  session: demo-session
  snapshotID: 01HHKQQ59XN4979HMDFNCANA9V
  type: FullBackup
  version: v1
status:
  components:
    dump-pod-0:
      driver: Restic
      duration: 10.595356615s
      integrity: true
      path: repository/v1/demo-session/dump-pod-0
      phase: Succeeded
      resticStats:
        - hostPath: /source/data
          id: cb64ffb35297f15cd93cfedc280b7e10267aaef283626fa564a32f69468a6bc2
          size: 10.240 MiB
          uploaded: 10.242 MiB
      size: 10.242 MiB
    dump-pod-1:
      driver: Restic
      duration: 8.789930136s
      integrity: true
      path: repository/v1/demo-session/dump-pod-1
      phase: Succeeded
      resticStats:
        - hostPath: /source/data
          id: 489d5a7614daa663df0c69a53a36d353936b6c8be61b8472694eeabcedbde16f
          size: 1.978 MiB
          uploaded: 1.979 MiB
      size: 1.979 MiB
    dump-pod-2:
      driver: Restic
      duration: 6.261958475s
      integrity: true
      path: repository/v1/demo-session/dump-pod-2
      phase: Succeeded
      resticStats:
        - hostPath: /source/data
          id: 17a7bc582a1f39fa54beb96f37f6f806f86d39ca575777391fcf69ddfc1bbab7
          size: 13 B
          uploaded: 1.056 KiB
      size: 809 B
  conditions:
    - lastTransitionTime: "2023-12-14T08:40:01Z"
      message: Recent snapshot list updated successfully
      reason: SuccessfullyUpdatedRecentSnapshotList
      status: "True"
      type: RecentSnapshotListUpdated
    - lastTransitionTime: "2023-12-18T08:50:57Z"
      message: Metadata uploaded to backend successfully
      reason: SuccessfullyUploadedSnapshotMetadata
      status: "True"
      type: SnapshotMetadataUploaded
  integrity: true
  phase: Succeeded
  size: 12.222 MiB
  snapshotTime: "2023-12-14T08:40:10Z"
  totalComponents: 3
```

Here, we are going to describe the various sections of a `Snapshot` object.

### Snapshot `Metadata`

**metadata.name**

`metadata.name` specifies the name of the `Snapshot` object. It follows the following pattern, `<Repository name>-<BackupSession name>`.

**metadata.labels**

A `Snapshot` object holds `repository` and target application `kind`, `name` and `namespace` as a label in `metadata.labels` section. This helps a user to query the Snapshots of a particular repository and/or a particular target application.


### Snapshot `Spec`

**spec.snapshotID**

`spec.snapshotID` represents a **Universally Unique Lexicographically Sortable Identifier**(ULID) for the Snapshot. For more details about ULID, please see [here](https://github.com/oklog/ulid)

**spec.type** 

`spec.type` specifies whether this snapshot represents a `FullBackup` or `IncrementalBackup`.

**spec.repository** 

`spec.repository` specifies the name of the `Repository` where this `Snapshot` is being stored.

**spec.session**

`spec.session` specifies the name of the session which is responsible for this `Snapshot`.

**spec.backupSession** 

`spec.backupSession` represents the name of the respective `BackupSession` which is responsible for this `Snapshot`.

**spec.version**

`spec.version` denotes the respective data organization structure inside the `Repository`.

**spec.appRef**

`spec.appRef` specifies the reference of the application that has been backed up in this `Snapshot`.

**spec.deletionPolicy**

`spec.deletionPolicy` specifies what to do when you delete a `Snapshot` CR. The valid values are:
- **Delete** This will delete just the Snapshot CR from the cluster but keep the backed up data in the remote backend. This is the default behavior.
- **WipeOut** This will delete the Snapshot CR as well as the backed up data from the backend.

**spec.paused**

`spec.paused` specifies whether the `Snapshot` is paused or not. If the `Snapshot` is paused, KubeStash will not process any further event for the `Snapshot`.

### Snapshot `Status`

`Snapshot` object has the following fields in `.status` section:

**status.phase**

`status.phase` represents the backup state of this `Snapshot`.

**status.verificationStatus**

`status.verificationStatus` specifies whether this `Snapshot` has been verified or not. The valid values for this field are `Verified`, `NotVerified` and `VerificationFailed`.

**status.verificationSession**

`status.verificationSession` specifies which BackupVerificationSession verified this `Snapshot`.

**status.snapshotTime**

`status.snapshotTime` represents the original creation timestamp for this `Snapshot`.

**status.lastUpdateTime**

`status.lastUpdateTime` specifies the timestamp when this Snapshot was last updated.

**status.size**

`status.size` represents the size of the `Snapshot`.

**status.integrity**

`status.integrity` represents whether the `Snapshot` data has been corrupted or not.

**status.conditions**

`status.conditions` represents list of conditions regarding this `Snapshot`. KubeStash sets the following conditions for a `Snapshot`:

| Field                       | Usage                                                           |
|-----------------------------|-----------------------------------------------------------------|
| `SnapshotMetadataUploaded`  | Indicates whether the metadata of Snapshot was uploaded or not. |
| `RecentSnapshotListUpdated` | Indicates whether the recent Snapshot list was updated or not.  |

**status.totalComponents**

`status.totalComponents` represents the number of total components for this `Snapshot`.

**status.components**

`status.components` represents the backup information of the individual components of this `Snapshot`. Each component consists of the following fields:

- **path :** specifies the path inside the `Repository` where the backed up data for this component has been stored. This path is relative to `Repository` path.
- **phase :** represents the backup phase of the component.
- **size :** represents the size of the restic repository for this component.
- **duration :** specifies the total time taken to complete the backup process for this component.
- **integrity :** represents the result of the restic repository integrity check for this component.
- **error :** specifies the reason in case of backup failure for the component.
- **driver :** specifies the name of the tool that has been used to upload the underlying backed up data.
- **resticStats :** specifies the **Restic** driver specific information. Each resticStat consists of the following fields:
  - **id :** represents the restic snapshot id.
  - **uploaded :** specifies the amount of data that has been uploaded in the restic snapshot.
  - **hostPath :** represents the backup path for which restic snapshot is taken.
  - **size :** represents the restic snapshot size.
- **volumeSnapshotterStats :** specifies the **VolumeSnapshotter** driver specific information. Each volumeSnapshotterStat consists of the following fields:
  - **pvcName :** represents the backup PVC name for which volumeSnapshot was created.
  - **hostPath :** represents the mount path of PVC for which volumeSnapshot was created.
  - **volumeSnapshotName :** represents the name of created volumeSnapshot.
  - **volumeSnapshotTime :** indicates the timestamp at which the volumeSnapshot was created.
- **walSegments :** specifies a list of wall segment for individual component. Each walSegment consists of `start` time and `end` time fields.

## Next Steps

- Learn how to configure `BackupConfiguration` to backup workloads data from [here](/docs/guides/workloads/overview/index.md).
- Learn how to configure `BackupConfiguration` to backup stand-alone PVC from [here](/docs/guides/volumes/overview/index.md).
