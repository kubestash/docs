---
title: Function Overview
menu:
  docs_{{ .version }}:
    identifier: function-overview
    name: Function
    parent: crds
    weight: 35
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# Function

## What is Function

A complete process of a task such as, backup or restore, is called a `Function` in KubeStash.

A `Function` is a Kubernetes `CustomResourceDefinition`(CRD) which basically specifies a template for a container that performs only a specific action or task. For example, `postgres-backup` function only take backup of PostgreSQL Database.

When you install KubeStash, some `Function`s will be pre-installed for supported targets like workloads, pvc etc. However, you can create your own function to customize or extend the backup/restore process.

## Function CRD Specification

Like any official Kubernetes resource, a `Function` has `TypeMeta`, `ObjectMeta` and `Spec` sections. However, unlike other Kubernetes resources, it does not have a `Status` section.

A sample `Function` object to backup a PostgreSQL is shown below,

```yaml
apiVersion: addons.kubestash.com/v1alpha1
kind: Function
metadata:
  annotations:
    meta.helm.sh/release-name: kubedb
    meta.helm.sh/release-namespace: kubedb
  creationTimestamp: "2023-12-12T12:55:59Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: kubedb
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubedb-kubestash-catalog
    app.kubernetes.io/version: v2023.12.11
    helm.sh/chart: kubedb-kubestash-catalog-v2023.12.11
  name: postgres-backup
  resourceVersion: "277644"
  uid: f64449f8-1111-4a4d-8c6e-96c5b877aef6
spec:
  args:
    - backup
    - --namespace=${namespace:=default}
    - --backupsession=${backupSession:=}
    - --enable-cache=${enableCache:=}
    - --scratch-dir=${scratchDir:=}
    - --wait-timeout=${waitTimeout:=300}
    - --pg-args=${args:=}
    - --backup-cmd=${backupCmd:=}
    - --user=${user:=}
  image: ghcr.io/kubedb/postgres-restic-plugin:v0.5.0
```

Here, we are going to describe the various sections of a `Function` crd.

### Function `Spec`

A `Function` object has the following fields in the `spec` section:

#### spec.image

`spec.image` specifies the docker image to use to create a container using the template specified in this `Function`.

#### spec.command

`spec.command` specifies the commands to be executed by the container. Docker image's `ENTRYPOINT` will be executed if no commands are specified.

#### spec.args

`spec.args` specifies a list of arguments that will be passed to the entrypoint. You can templatize this section using `envsubst` style variables. KubeStash will resolve all the variables before creating the respective container. A variable should follow the following patterns:

- ${variableName:=default-value}
- ${variableName:=}

In the first case, if KubeStash can't resolve the variable, the default value will be used in place of this variable. In the second case, if KubeStash can't resolve the variable, an empty string will be used to replace the variable.

##### KubeStash Provided Variables

KubeStash operator provides the following built-in variables based on `BackupSession`, `RestoreSession`, `Function` etc.

| Environment Variable | Usage                                                |
|----------------------|------------------------------------------------------|
| `namespace`          | Namespace of backup or restore job/workload          |
| `backupSession`      | Name of the respective BackupSession object          |
| `restoreSession`     | Name of the respective RestoreSession object         |
| `enableCache`        | Specify whether to use cache while backup or restore |
| `snapshot`           | Name of the respective Snapshot object               |

#### spec.workingDir

`spec.workingDir` specifies the container's working directory. If this field is not specified, the container's runtime default will be used.

#### spec.ports

`spec.ports` specifies a list of the ports to expose from the respective container that will be created for this function.

#### spec.volumeMounts

`spec.volumeMounts` specifies a list of volume names and their `mountPath` that will be mounted into the container that will be created for this function.

#### spec.volumeDevices

`spec.volumeDevices` specifies a list of the block devices to be used by the container that will be created for this function.

#### spec.runtimeSettings

`spec.runtimeSettings` allows to configure runtime environment of a backup job at container level. You can configure the following container level parameters:

| Field             | Usage                                                                                                                                                                                                                      |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `resources`       | Compute resources required by sidecar container or backup job. To know how to manage resources for containers, please visit [here](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/). |
| `livenessProbe`   | Periodic probe of backup sidecar/job container's liveness. Container will be restarted if the probe fails.                                                                                                                 |
| `readinessProbe`  | Periodic probe of backup sidecar/job container's readiness. Container will be removed from service endpoints if the probe fails.                                                                                           |
| `lifecycle`       | Actions that the management system should take in response to container lifecycle events.                                                                                                                                  |
| `securityContext` | Security options that backup sidecar/job's container should run with. For more details, please visit [here](https://kubernetes.io/docs/concepts/policy/security-context/).                                                 |
| `nice`            | Set CPU scheduling priority for the backup process. For more details about `nice`, please visit [here](https://www.askapache.com/optimize/optimize-nice-ionice/#nice).                                                     |
| `ionice`          | Set I/O scheduling class and priority for the backup process. For more details about `ionice`, please visit [here](https://www.askapache.com/optimize/optimize-nice-ionice/#ionice).                                       |
| `env`             | A list of the environment variables to set in the container that will be created for this function.                                                                                                                        |
| `envFrom`         | This allows to set environment variables to the container that will be created for this function from a Secret or ConfigMap.                                                                                               |


## Next Steps

- Learn how KubeStash backup stand-alone PVC using `Function-Addon` model from [here](/docs/guides/volumes/overview/index.md).