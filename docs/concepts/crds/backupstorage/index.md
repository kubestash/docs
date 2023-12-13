---
title: BackupStorage Overview
menu:
  docs_{{ .version }}:
    identifier: backupstorage-overview
    name: BackupStorage
    parent: crds
    weight: 5
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# BackupStorage

## What is BackupStorage

A `BackupStorage` is a Kubernetes `CustomResourceDefinition` (CRD) which specifies the backend information where the backed up data of different applications will be stored.
A `BackupStorage` can be considered as a representation of a bucket in Kubernetes native way. This is a namespaced object. However, the `BackupStorage` can be used from any namespace
as long as it is permitted by the usage policy.

You have to create at least one `BackupStorage` object and refer the `BackupStorage` in the `BackupConfiguration`.

## BackupStorage CRD Specification
Like any official Kubernetes resource, a `BackupStorage` has `TypeMeta`, `ObjectMeta`, `Spec` and `Status` sections.

A sample `BackupStorage` object that uses Google Cloud Storage(GCS) bucket as storage is shown below:
```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: BackupStorage
metadata:
  name: gcs-storage
  namespace: demo
spec:
  storage:
    provider: gcs
    gcs:
      bucket: kubestash-demo
      prefix: demo
      secretName: gcs-secret
  default: true
  deletionPolicy: WipeOut
  runtimeSettings: {}
  usagePolicy:
    allowedNamespaces:
      from: All
status:
  conditions:
  - lastTransitionTime: "2023-12-05T13:14:04Z"
    message: Successfully initialized backend.
    reason: BackendInitializationSucceeded
    status: "True"
    type: BackendInitialized
  - lastTransitionTime: "2023-12-05T13:14:04Z"
    message: Backend secret exists.
    reason: BackendSecretAvailable
    status: "True"
    type: BackendSecretFound
  phase: Ready
  repositories:
  - name: gcs-demo-repo
    namespace: demo
    path: /demo/data
    size: 12.222 MiB
    synced: true
  totalSize: 12.222 MiB
```
Here, we are going to describe the various sections of the `BackupStorage` crd.

## BackupStorage `Spec`
`BackupStorage` CRD has the following fields in the `.spec` section:
- **spec.storage** `spec.storage` specifies the storage information where the backed up data will be stored. To learn how to configure `BackupStorage` crd for various storages, please visit [here](/docs/guides/backends/overview).
- **spec.default** `spec.default` specifies whether to use this `BackupStorage` as default storage for the current namespace as well as the allowed namespaces. One namespace can have **at most one** default BackupStorage configured.
- **spec.deletionPolicy** `spec.deletionPolicy` specifies whether KubeStash operator should delete backed up files from the storage or not when a `BackupStorage` is deleted. The valid values for this field are:
    - **Delete** This will delete the respective `Repository` and `Snapshot` CRs from the cluster but keep the backed up data in the storage. This is the default behavior.
    - **WipeOut** This will delete the respective `Repository` and `Snapshot` CRs as well as the backed up data from the storage.
- **spec.runtimeSettings** `spec.runtimeSettings` allow to specify Resources, NodeSelector, Affinity, Toleration, ReadinessProbe etc. for the storage initializer/cleaner jobs. To learn more visit [here](/docs/concepts/crds/backupconfiguration/index.md#runtime-settings).
- **spec.usagePolicy** `spec.usagePolicy` specifies a policy of how this `BackupStorage` will be used. For example, you can use `allowedNamespaces` policy to restrict the usage of this `BackupStorage` to particular namespaces. This field is optional. If you don't provide the `usagePolicy`, then it can be used only from the current namespace.

Here is an example of `spec.usagePolicy` that limits referencing the `BackupStorage` only from the same namespace,
```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      from: Same
```
Here is an example of `spec.usagePolicy` that allows referencing the `BackupStorage` from only `prod` and `staging` namespaces,
```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      from: Selector
      selector:
        matchExpressions:
          - key: "kubernetes.io/metadata.name"
            operator: In
            values: ["prod","staging"]
```
Here is an example of `spec.usagePolicy` that allows referencing the `BackupStorage` from all namespaces,
```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      from: All
```

## BackupStorage `Status`
KubeStash operator updates `.status` of a BackupStorage. BackupStorage shows the following statistics in status section:
- **status.phase** `status.phase` indicates the overall phase of the `BackupStorage`. Phase will be **Ready** only if the Backend is initialized, Backend secret exists and Repositories are synced.
- **status.totalSize** `status.totalSize` represents the total backed up data size in this storage. This is simply the summation of sizes of all Repositories using this `BackupStorage`.
- **status.repositories** `status.repositories` holds the information of all Repositories using this `BackupStorage`. Each entry has the following fields:
    - **name** `name` of the `Repository`.
    - **namespace** `namespace` of the `Repository`.
    - **path** `path` is relative to the path of `BackupStorage` where the data of this `Repository` has been stored.
    - **size** the amount of data that has been stored in this `Repository`.
    - **synced** indicates whether the repo is synced with the storage state or not.
    - **error** specifies the reason in case of `Repository` sync failure.
- **status.conditions** `status.conditions` represents list of conditions regarding this `BackupStorage`. The following conditions are set by the KubeStash operator on a `BackupStorage`.

| Condition Type       | Usage                                       |
|----------------------|---------------------------------------------|
| `BackendInitialized` | indicates that the backend is initialized.  |
| `BackendSecretFound` | indicates that the backend secret is found. |

## Deleting BackupStorage
KubeStash allows users to delete only `BackupStorage` or `BackupStorage` along with respective backed up data depending on `.spec.deletionPolicy`. Here, we are going to show how to perform these operations.

**Delete only BackupStorage keeping backed up data:**

First check the value in `.spec.deletionPolicy`. If the value of this field is `Delete`, then just run the following command:
```bash
$ kubectl delete backupstorage <backupstorage-name>

# Example
$ kubectl delete backupstorage gcs-storage
backupstorage "gcs-storage" deleted
```

If the value of the `.spec.deletionPolicy` field is `WipeOut`, then run the following commands:
```bash
$ kubectl patch backupstorage <backupstorage-name> --type="merge" --patch='{"spec": {"deletionPolicy": "Delete"}}'

# Example 
$ kubectl patch backupstorage gcs-storage --type="merge" --patch='{"spec": {"deletionPolicy": "Delete"}}'
backupstorage "gcs-storage" patched

# Then delete the BackupStorage
$ kubectl delete backupstorage gcs-storage
backupstorage "gcs-storage" deleted
```
This will delete only `BackupStorage` and its corresponding Repositories. It won’t delete any backed up data from the storage. You can recreate the `BackupStorage` object later to reuse existing data. It will sync the corresponding Repositories.

**Delete BackupStorage along with backed up data:**

First check the value in `.spec.deletionPolicy`. If the value of this field is `WipeOut`, then just run the following command:
```bash
$ kubectl delete backupstorage <backupstorage-name>

# Example
$ kubectl delete backupstorage gcs-storage
backupstorage "gcs-storage" deleted
```

If the value of the `.spec.deletionPolicy` field is `Delete`, then run the following commands:
```bash
$ kubectl patch backupstorage <backupstorage-name> --type="merge" --patch='{"spec": {"deletionPolicy": "WipeOut"}}'

# Example 
$ kubectl patch backupstorage gcs-storage --type="merge" --patch='{"spec": {"deletionPolicy": "WipeOut"}}'
backupstorage "gcs-storage" patched

# Then delete the BackupStorage
$ kubectl delete backupstorage gcs-storage
backupstorage "gcs-storage" deleted
```
This will delete `BackupStorage` and its corresponding Repositories along with the backed up data from the storage. You can browse your storage bucket to verify that the backed up data has been wiped out.

## Next Steps
- Learn how to create `BackupStorage` for different storages from [here](/docs/guides/backends/overview/index.md).
- Learn how KubeStash backup workloads data from [here](/docs/guides/workloads/overview/index.md).
- Learn how KubeStash backup databases from [here](/docs/guides/addons/overview/index.md).
