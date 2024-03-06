---
title: Repository Overview
menu:
  docs_{{ .version }}:
    identifier: repository-overview
    name: Repository
    parent: crds
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# Repository

## What is Repository

A `Repository` is a Kubernetes `CustomResourceDefinition`(CRD) which represents the backup information for a specific application. 

The KubeStash operator manages `Repository` objects, creating them based on information from a `BackupConfiguration`. When a `BackupStorage` is created with existing backup data, KubeStash automatically synchronizes the Repositories and Snapshots linked to it from the backend.

## Repository CRD Specification
Like any official Kubernetes resource, a `Repository` object has `TypeMeta`, `ObjectMeta`, `Spec` and `Status` sections.

A sample `Repository` object that uses Google Cloud Storage(GCS) bucket as storage is shown below:
```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Repository
metadata:
  name: gcs-demo-repo
  namespace: demo
spec:
  appRef:
    apiGroup: apps
    kind: StatefulSet
    name: sample-sts
    namespace: demo
  deletionPolicy: Delete
  encryptionSecret:
    name: encrypt-secret
    namespace: demo
  path: /demo/data
  storageRef:
    name: gcs-storage
    namespace: demo
status:
  componentPaths:
  - repository/v1/demo-session/pod-0
  - repository/v1/demo-session/pod-1
  - repository/v1/demo-session/pod-2
  conditions:
  - lastTransitionTime: "2023-12-07T06:37:40Z"
    message: Successfully initialized repository
    reason: RepositoryInitializationSucceeded
    status: "True"
    type: RepositoryInitialized
  integrity: true
  lastBackupTime: "2023-12-07T06:38:00Z"
  phase: Ready
  recentSnapshots:
  - name: gcs-demo-repo-sample-backup-sts-demo-session-1701940920
    phase: Succeeded
    session: demo-session
    size: 12.222 MiB
    snapshotTime: "2023-12-07T06:38:10Z"
  size: 12.222 MiB
  snapshotCount: 1
```
Here, we are going to describe the various sections of the `Repository` crd.

## Repository `Spec`
`Repository` CRD has the following fields in the `.spec` section:
- **spec.appRef :** `spec.AppRef` refers to the application that is being backed up in this `Repository`.
- **spec.storageRef :** `spec.storageRef` refers to the `BackupStorage` CR which contain the storage information where the backed up data will be stored. The `BackupStorage` could be in a different namespace. However, the `Repository` namespace must be allowed to use the `BackupStorage`.
- **spec.path :** `spec.path` represents the directory inside the `BackupStorage` where this Repository is storing its data. This path is relative to the path of `BackupStorage`. This path must be **unique** for each `Repository` referring same `BackupStorage`.
- **spec.deletionPolicy :** `spec.deletionPolicy` specifies what to do when a `Repository` CR is deleted. The valid values for this field are:
  - **Delete :** This will delete the respective `Repository` CR and associated `Snapshot` CRs from the cluster but keep the backed up data in the storage. This is the default behavior.
  - **WipeOut :** This will delete the respective `Repository` CR, associated `Snapshot` CRs and the backed up data in `spec.path` from the storage.
- **spec.encryptionSecret :** refers to the `Secret` containing the encryption key which will be used to encode/decode the backed up data. You can refer to a `Secret` of a different namespace.
- **spec.paused :** `spec.paused` specifies whether the Repository is paused or not. If the Repository is paused, KubeStash will not process any further event for the Repository.

## Repository `Status`
`Repository` crd shows the following statistics in `.status` section:
- **status.phase :** `status.phase` represents the current state of the `Repository`.
- **status.lastBackupTime :** `status.lastBackupTime` specifies the timestamp when the last successful backup has been taken.
- **status.integrity :** `status.integrity` specifies whether the backed up data of this `Repository` has been corrupted or not.
- **status.snapshotCount :** `status.snapshotCount` specifies the number of current `Snapshots` stored in this `Repository`.
- **status.size :** `status.size` specifies the size of the backed up data stored in the `Repository`.
- **status.recentSnapshots :** `status.recentSnapshots` holds a list of recent `Snapshot` (maximum 5) information that has been taken in this `Repository`.
- **status.conditions :** `status.conditions` represents list of conditions regarding this `Repository`. The following condition is set by the KubeStash operator on a `Repository`.

| Condition Type          | Usage                                      |
|-------------------------|--------------------------------------------|
| `RepositoryInitialized` | indicates the `Repository` was initialized |

- **status.componentPaths :** `status.componentPaths` represents list of component paths in this `Repository`.

## Next Steps
- Learn how to create `BackupStorage` for different storages from [here](/docs/guides/backends/overview/index.md).
- Learn how KubeStash backup workloads data from [here](/docs/guides/workloads/overview/index.md).