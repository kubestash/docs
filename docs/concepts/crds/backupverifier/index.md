---
title: BackupVerifier Overview
menu:
  docs_{{ .version }}:
    identifier: backupverifier-overview
    name: BackupVerifier
    parent: crds
    weight: 60
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# BackupVerifier

## What is BackupVerifier

A `BackupVerifier` is a Kubernetes `CustomResourceDefinition`(CRD) which represents backup verification configurations.

## BackupVerifier CRD Specification

Like any official Kubernetes resource, a `BackupVerifier` has `TypeMeta`, `ObjectMeta` and `Spec` sections. However, unlike other Kubernetes resources, it does not have a `Status` section.

A sample `BackupVerifier` is shown below,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupVerifier
metadata:
  creationTimestamp: "2024-11-01T06:30:14Z"
  generation: 1
  name: mysql-script-verifier
  namespace: demo
  resourceVersion: "28771"
  uid: 052b6035-4e16-4864-a3ab-bc0aa2bf1825
spec:
  function: kubedbverifier
  restoreOption:
    addonInfo:
      name: mysql-addon
      tasks:
        - name: logical-backup-restore
    target:
      apiGroup: kubedb.com
      kind: MySQL
      name: sample-mysql
      namespace: verify
  schedule: '*/5 * * * *'
  script:
    location: /tmp/config/test.sh
  sessionHistoryLimit: 2
  type: Script
  volumeMounts:
    - mountPath: /tmp/config
      name: config-vol
  volumes:
    - configMap:
        name: config
      name: config-vol
```

Here, we are going to describe the various sections of a `BackupVerifier` object.

### BackupVerifier `Spec`

A `BackupVerifier` object has the following fields in the `spec` section:

#### spec.restoreOption
`spec.restoreOption` specifies the restore target and addonInfo for backup verification. It consists of the following fields:
- **target :** indicates the target application where the data will be restored for verification. This field consists of `apiGroup`, `kind`, `name` and `namespace`. The backup target can be in a different namespace than the `BackupConfiguration`.
- **addon :** specifies addon configuration that will be used to backup the target. Addon has the following fields:
  - **name :** specifies the name of the addon that will be used for the backup purpose.
  - **tasks :** specifies a list of backup tasks and their configuration parameters. To learn about the fields under `task`, see [Task Reference](/docs/concepts/crds/backupconfiguration#task-reference).
  - **containerRuntimeSettings :** specifies runtime settings for the backup executor container. More information can be found [here](/docs/concepts/crds/backupconfiguration#container-level-runtime-settings).
  - **jobTemplate :** specifies runtime configurations for the backup Job. More information can be found [here](/docs/concepts/crds/backupconfiguration#podtemplate-spec).
  
#### spec.schedule

`spec.schedule` specifies the schedule of backup verification in Cron format, see https://en.wikipedia.org/wiki/Cron.

#### spec.function

`spec.function` specifies the name of a Function CR that defines a container definition which will execute the verification logic for a particular application.

#### spec.volumes

`spec.volumes` indicates the list of volumes that should be mounted on the verification job.

#### spec.volumeMounts

`spec.volumeMounts` specifies the mount for the volumes specified in `Volumes` section.

#### spec.type

`spec.type` indicate the types of verifier that will verify the backup. Valid types are:
- **RestoreOnly :** KubeStash will create a RestoreSession with the tasks provided in BackupConfiguration's verificationStrategies section.
- **Query :** KubeStash operator will restore data and then create a job to run the queries.
- **Script :** KubeStash operator will restore data and then create a job to run the script.

#### spec.query

`spec.query` specifies the queries to be run to verify backup.

#### spec.script 

`spec.script` specifies the script to be run to verify backup. This has the following fields:
- **location :** specifies the absolute path of the script file's location.
- **args :** specifies the arguments to be provided with the script.

#### spec.retryConfig

`spec.retryConfig` specifies the behavior of the retry mechanism in case of a verification failure. This has the following fields:
- **maxRetry :** specifies the maximum number of times KubeStash should retry the backup/restore process. By default, KubeStash will retry only 1 time.
- **delay :** the amount of time to wait before next retry. If you don't specify this field, KubeStash will retry immediately. Format: 30s, 2m, 1h etc.

#### spec.sessionHistoryLimit

`spec.sessionHistoryLimit` specifies how many BackupVerificationSessions and associate resources KubeStash should keep for debugging purpose. The default value is 1.

#### spec.runtimeSettings

`spec.runtimeSettings` allow to specify Resources, NodeSelector, Affinity, Toleration, ReadinessProbe etc. To know more about the fields in `runtimeSettings`, see [Runtime Settings](/docs/concepts/crds/backupconfiguration#runtime-settings)

## Next Steps

- Learn how backup of workloads data works from [here](/docs/guides/workloads/overview/index.md).
- Learn how backup stand alone PVC works from [here](/docs/guides/volumes/overview/index.md).
