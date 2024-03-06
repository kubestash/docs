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
- **spec.storage :** `spec.storage` specifies the storage information where the backed up data will be stored. To learn how to configure `BackupStorage` crd for various storages, please visit [here](/docs/guides/backends/overview).
- **spec.default :** `spec.default` specifies whether to use this `BackupStorage` as default storage for the current namespace as well as the allowed namespaces. One namespace can have **at most one** default BackupStorage configured.
- **spec.deletionPolicy :** `spec.deletionPolicy` specifies whether KubeStash operator should delete backed up files from the storage or not when a `BackupStorage` is deleted. The valid values for this field are:
    - **Delete :** This will delete the respective `Repository` and `Snapshot` custom resources from the cluster but keep the backed up data in the storage. This is the default behavior.
    - **WipeOut :** This will delete the respective `Repository` and `Snapshot` custom resources as well as the backed up data from the storage.
- **spec.runtimeSettings :** `spec.runtimeSettings` allow to specify Resources, NodeSelector, Affinity, Toleration, ReadinessProbe etc. for the storage initializer/cleaner jobs. To learn more visit [here](#runtime-settings).
- **spec.usagePolicy :** `spec.usagePolicy` lets you control which namespaces are allowed to use the `BackupStorage` and which are not. If you refer to a `BackupStorage` from a restricted namespace, KubeStash will reject creating the respective `BackupConfiguration` from validating webhook. You can use the `usagePolicy` to allow only the same namespace, a subset of namespaces, or all the namespaces to refer to the `BackupStorage`. If you don’t specify any `usagePolicy`, KubeStash will allow referencing the `BackupStorage` only from the namespace where it was created.

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

#### Runtime Settings
Runtime Settings allows to configure runtime environment for the corresponding job. You can specify runtime settings at both pod level and container level.

##### Container Level Runtime Settings
`runtimeSettings.container` is used to configure the corresponding job at container level. You can configure the following container level parameters:

| Field             | Usage                                                                                                                                                                                                                        |
|-------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `resources`       | Compute resources required by the corresponding job containers. To learn how to manage resources for containers, please visit [here](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/). |
| `livenessProbe`   | Periodic probe of corresponding job container's liveness. Container will be restarted if the probe fails.                                                                                                                    |
| `readinessProbe`  | Periodic probe of corresponding job container's readiness. Container will be removed from service endpoints if the probe fails.                                                                                              |
| `lifecycle`       | Actions that the management system should take in response to container lifecycle events.                                                                                                                                    |
| `securityContext` | Security options that corresponding job's container should run with. For more details, please visit [here](https://kubernetes.io/docs/concepts/policy/security-context/).                                                    |
| `nice`            | Set CPU scheduling priority for current process. For more details about `nice`, please visit [here](https://www.askapache.com/optimize/optimize-nice-ionice/#nice).                                                          |
| `ionice`          | Set I/O scheduling class and priority for current process. For more details about `ionice`, please visit [here](https://www.askapache.com/optimize/optimize-nice-ionice/#ionice).                                            |
| `env`             | A list of the environment variables to set in the corresponding job's container.                                                                                                                                             |
| `envFrom`         | This allows to set environment variables to the container that will be created for this function from a Secret or ConfigMap.                                                                                                 |

##### Pod Level Runtime Settings
`runtimeSettings.pod` is used to configure the corresponding job at pod level. You can configure the following pod level parameters:

| Field                          | Usage                                                                                                                                                                                                                                    |
|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `podLabels`                    | The labels that will be attached with the respective Pod                                                                                                                                                                                 |
| `serviceAccountName`           | Name of the `ServiceAccount` to use for the corresponding job.                                                                                                                                                                           |
| `nodeSelector`                 | Selector which must be true for corresponding job pod to fit on a node.                                                                                                                                                                  |
| `automountServiceAccountToken` | Indicates whether a service account token should be automatically mounted into the pod.                                                                                                                                                  |
| `nodeName`                     | `nodeName` is used to request to schedule job's pod onto a specific node.                                                                                                                                                                |
| `securityContext`              | Security options that job's pod should run with. For more details, please visit [here](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/).                                                                      |
| `imagePullSecrets`             | A list of secret names in the same namespace that will be used to pull image from private Docker registry. For more details, please visit [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/). |
| `affinity`                     | Affinity and anti-affinity to schedule job's pod on a desired node. For more details, please visit [here](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity).                                |
| `schedulerName`                | Name of the scheduler that should dispatch the job.                                                                                                                                                                                      |
| `tolerations`                  | Taints and Tolerations to ensure that job's pod is not scheduled in inappropriate nodes. For more details about `toleration`, please visit [here](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/).              |
| `priorityClassName`            | Indicates the job pod's priority class. For more details, please visit [here](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/).                                                                               |
| `priority`                     | Indicates the job pod's priority value.                                                                                                                                                                                                  |
| `readinessGates`               | Specifies additional conditions to be evaluated for Pod readiness. For more details, please visit [here](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-readiness-gate).                                          |
| `runtimeClassName`             | RuntimeClass is used for selecting the container runtime configuration. For more details, please visit [here](https://kubernetes.io/docs/concepts/containers/runtime-class/)                                                             |
| `enableServiceLinks`           | EnableServiceLinks indicates whether information about services should be injected into pod's environment variables.                                                                                                                     |

## BackupStorage `Status`
KubeStash operator updates `.status` of a BackupStorage. BackupStorage shows the following statistics in status section:
- **status.phase :** `status.phase` indicates the overall phase of the `BackupStorage`. Phase will be **Ready** only if the Backend is initialized, Backend secret exists and Repositories are synced.
- **status.totalSize :** `status.totalSize` represents the total backed up data size in this storage. This is simply the summation of sizes of all Repositories using this `BackupStorage`.
- **status.repositories :** `status.repositories` holds the information of all Repositories using this `BackupStorage`. Each entry has the following fields:
    - **name :** `name` of the `Repository`.
    - **namespace :** `namespace` of the `Repository`.
    - **path :** `path` is relative to the path of `BackupStorage` where the data of this `Repository` has been stored.
    - **size :** the amount of data that has been stored in this `Repository`.
    - **synced :** indicates whether the repo is synced with the storage state or not.
    - **error :** specifies the reason in case of `Repository` sync failure.
- **status.conditions :** `status.conditions` represents list of conditions regarding this `BackupStorage`. The following conditions are set by the KubeStash operator on a `BackupStorage`.

| Condition Type       | Usage                                        |
|----------------------|----------------------------------------------|
| `BackendInitialized` | indicates that the backend was initialized.  |
| `BackendSecretFound` | indicates that the backend secret was found. |

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
This will delete only `BackupStorage` CR and its associated Repository and Snapshot CRs. It won’t delete any backed up data from the storage. You can recreate the `BackupStorage` object later to reuse existing data, and it will automatically sync its Repositories and Snapshots.

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
This will delete `BackupStorage` CR, associated Repository and Snapshot CRs along with the backed up data from the storage. To do the cleanup task, KubeStash operator will create a cleanup job in the same namespace of the `BackupStorage` with the name `cleanup-<backupstorage-name>`. You can browse your storage bucket to verify that the backed up data has been wiped out.

> It is not recommended to set `.spec.deletionPolicy` to `WipeOut`. Set this field to `WipeOut` for testing purpose only.

## Next Steps
- Learn how to create `BackupStorage` for different storages from [here](/docs/guides/backends/overview/index.md).
- Learn how KubeStash backup workloads data from [here](/docs/guides/workloads/overview/index.md).
