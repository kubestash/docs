---
title: BackupBlueprint Overview
menu:
  docs_{{ .version }}:
    identifier: backupblueprint-overview
    name: BackupBlueprint
    parent: crds
    weight: 40
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# BackupBlueprint

## What is BackupBlueprint

KubeStash uses 1-1 mapping among `BackupConfiguration` and the target. So, whenever you want to backup a target(workload/PV/PVC/database), you have to create a `BackupConfiguration` object. This could become tiresome when you are trying to backup similar types of target and the `BackupConfiguration` has only slight difference. To mitigate this problem, KubeStash provides a way to specify a blueprint for this object via `BackupBlueprint` crd.

A `BackupBlueprint` is a Kubernetes `CustomResourceDefinition`(CRD) which specifies a blueprint for `BackupConfiguration` in a Kubernetes native way.

You have to create only one  `BackupBlueprint` for all similar types of workloads (i.e. Deployment, DaemonSet, StatefulSet etc.). Then, you just need to add some annotations in the target workload. KubeStash will automatically create respective `BackupConfiguration` object using the blueprint. In KubeStash parlance, we call this process as **auto backup**.

## BackupBlueprint CRD Specification
Like any official Kubernetes resource, a `BackupBlueprint` has `TypeMeta`, `ObjectMeta` and `Spec` sections. However, unlike other Kubernetes resources, it does not have a `Status` section.

A sample `BackupBlueprint` object to auto backup the volumes of workload is shown below,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupBlueprint
metadata:
  name: sample-blueprint
  namespace: demo
spec:
  usagePolicy:
    allowedNamespaces: 
      from: All
  backupConfigurationTemplate:
    deletionPolicy: OnDelete
    backends:
    - name: gcs-backend
      storageRef:
        namespace: demo
        name: gcs-storage
      retentionPolicy:
        name: demo-retention
        namespace: demo
    sessions:
    - name: backup-every-five-minutes
      sessionHistoryLimit: 3
      scheduler: # CronJob specification
        schedule: "*/5 * * * *"
        jobTemplate:
          backoffLimit: 1
      repositories:
      - name: ${repoName}
        backend: gcs-backend
        directory: ${namespace}/${targetName}
        encryptionSecret:
          name: encry-secret
          namespace: demo
      addon:
        name: workload-addon
        tasks:
        - name: LogicalBackup
          params:
            paths: ${paths}
      retryConfig:
        maxRetry: 2
        delay: 1m
```

The sample `BackupBlueprint` that has been shown above can be used to backup Deployments, DaemonSets, StatefulSets. You only have to add some annotations to these workloads. For more details on how auto backup works in KubeStash, please visit [here](/docs/guides/auto-backup/overview/index.md).

Here, we are going to describe the various sections of `BackupBlueprint` crd.

### BackupBlueprint `Spec`

A `BackupBlueprint` object has the following fields in the `spec` section.
- **usagePolicy** specifies a policy of how this `BackupBlueprint` will be used. For example, you can use `allowedNamespaces` policy to restrict the usage of this `BackupBlueprint` to particular namespaces. This field is optional. If you don't provide the usagePolicy, then it can be used only from the current namespace.

Here is an example of `spec.usagePolicy` that limits referencing the `BackupBlueprint` only from the same namespace,
```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      from: Same
```
Here is an example of `spec.usagePolicy` that allows referencing the `BackupBlueprint` from only `prod` and `staging` namespaces,
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
Here is an example of `spec.usagePolicy` that allows referencing the `BackupBlueprint` from all namespaces,
```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      from: All
```

- **backupConfigurationTemplate** specifies the `BackupConfiguration` that will be created by `BackupBlueprint`. It consists of the following fields:
  - **namespace** specifies the namespace of the `BackupConfiguration`. The field is optional. If you don't provide the namespace, then `BackupConfiguration` will be created in the `BackupBlueprint` namespace.
  - **backends** specifies a list of storage references where the backed up data will be stored. The respective `BackupStorage`s can be in a different namespace than the `BackupConfiguration`. However, it must be allowed by the `usagePolicy` of the `BackupStorage` to refer from this namespace. This field is optional, if you don't provide any backend here, KubeStash will use the default `BackupStorage` for the namespace. If a default `BackupStorage` does not exist in the same namespace, then KubeStash will look for a default `BackupStorage` in other namespaces that allows using it from the `BackupConfiguration` namespace. Each backend has the following fields:

| Field             | Usage                                                                                                                                                                                                                                                                                                                                                                                              |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`            | specifies an identifier for this storage. This name must be **unique**.                                                                                                                                                                                                                                                                                                                            |
| `storageRef`      | refers to the CR that holds the information of a storage. You can refer to the `BackupStorage` CR of a different namespace as long as it is allowed by the `usagePolicy` of the `BackupStorage`.                                                                                                                                                                                                   |
| `retentionPolicy` | refers to a `RetentionPolicy` CRs which defines how to cleanup the old `Snapshots`. This field is optional, if you don't provide this field, KubeStash will use the default `RetentionPolicy` for the namespace. If there is no default `RetentionPolicy` for the namespace, then KubeStash will find a `RetentionPolicy` from other namespaces that is allowed to use from the current namespace. |

  - **sessions** specifies a list of session template for backup. You can use custom variables in your template then provide the variable value through annotations. Each session has the following fields:
    - **name** specifies an identifier for this session. This name must be **unique**.
    - **scheduler** specifies the configuration for backup triggering CronJob. To learn about the fields under `scheduler`, see [Scheduler Spec](/docs/concepts/crds/backupconfiguration/index.md#scheduler-spec).
    - **hooks** specifies the backup hooks that should be executed before and/or after the backup. Hooks has two fields:
      - **preBackup** specifies a list of hooks that will be executed before backup. To learn about the fields under `preBackup`, see [HookInfo](/docs/concepts/crds/backupconfiguration/index.md#hookinfo).
      - **postBackup** specifies a list of hooks that will be executed after backup. To learn about the fields under `postBackup`, see [HookInfo](/docs/concepts/crds/backupconfiguration/index.md#hookinfo).
    - **failurePolicy** specifies what to do if the backup fail. Valid values are:
      - **Fail** KubeStash should mark the backup as failed if any component fail to complete its backup. This is the default behavior.
      - **Retry** KubeStash will retry to backup the failed component according to the `retryConfig`.
    - **retryConfig** specifies the behavior of retry in case of a backup failure. RetryConfig has the following fields:
      - **maxRetry** specifies the maximum number of times Stash should retry the backup/restore process. By default, KubeStash will retry only 1 time.
      - **delay** The amount of time to wait before next retry. If you don't specify this field, KubeStash will retry immediately. Format: 30s, 2m, 1h etc.
    - **timeout** specifies the maximum duration of backup. BackupSession will be considered Failed if backup does not complete within this time limit. By default, KubeStash don't set any timeout for backup.
    - **sessionHistoryLimit** specifies how many backup Jobs and associate resources KubeStash should keep for debugging purpose. The default value is 1.
    - **addon** specifies addon configuration that will be used to backup the target. Addon has the following fields:
      - **name** specifies the name of the addon that will be used for the backup purpose.
      - **tasks** specifies a list of backup tasks and their configuration parameters. To learn about the fields under `task`, see [Task Reference](/docs/concepts/crds/backupconfiguration/index.md#task-reference).
      - **containerRuntimeSettings** specifies runtime settings for the backup executor container. More information can be found [here](/docs/concepts/crds/backupconfiguration/index.md#container-level-runtime-settings).
      - **jobTemplate** specifies runtime configurations for the backup Job. More information can be found [here](/docs/concepts/crds/backupconfiguration/index.md#podtemplate-spec).
    - **repositories** specifies a list of repository information where the backed up data will be stored. KubeStash will create the respective `Repository` CRs using this information. Each repository consists of the following fields:
      - **name** specifies the name of the `Repository`.
      - **backend** specifies the name of the backend where this repository will be initialized. This should point to a backend name specified in `.spec.backupConfigurationTemplate.backends` section. For using a default backend, keep this field empty.
      - **directory** specifies the path inside the backend where the backed up data will be stored.
      - **encryptionSecret** refers to the Secret containing the encryption key which will be used to encode/decode the backed up data. You can refer to a Secret of a different namespace by providing `name` and `namespace` fields.
      - **deletionPolicy** specifies what to do when you delete a `Repository` CR. The valid values for this field are:
        - **Delete** This will delete just the `Repository` CR from the cluster but keep the backed up data in the remote backend. This is the default behavior.
        - **WipeOut** This will delete the `Repository` CR as well as the backed up data from the backend.
  - **deletionPolicy** specifies whether the `BackupConfiguration` will be deleted on `BackupBlueprint` deletion. This field is optional, if you don't provide deletionPolicy, then `BackupConfiguration` will not be deleted on `BackupBlueprint` deletion. The only valid value for this field is `OnDelete` which specifies the `BackupConfiguration` will be deleted on `BackupBlueprint` deletion.

## Next Steps

- Learn how to use `BackupBlueprint` for auto backup of workloads data from [here](/docs/guides/auto-backup/workload/index.md).
- Learn how to use `BackupBlueprint` for auto backup of database from [here](/docs/guides/auto-backup/database/index.md).
- Learn how to use `BackupBlueprint` for auto backup of stand-alone PVC from [here](/docs/guides/auto-backup/pvc/index.md).
