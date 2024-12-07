---
title: RestoreSession Overview
menu:
  docs_{{ .version }}:
    identifier: restoresession-overview
    name: RestoreSession
    parent: crds
    weight: 25
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# RestoreSession

## What is RestoreSession

A `RestoreSession` is a Kubernetes `CustomResourceDefinition`(CRD) which specifies a target to restore and the source of data that will be restored in a Kubernetes native way.

You have to create a `RestoreSession` object whenever you want to restore. 

## RestoreSession CRD Specification

Like any official Kubernetes resource, a `RestoreSession` has `TypeMeta`, `ObjectMeta`, `Spec` and `Status` sections.

A sample `RestoreSession` object to restore backed up data of a StatefulSet is shown below:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: sample-restore
  namespace: demo
spec:
  addon:
    name: workload-addon
    tasks:
    - name: logical-backup-restore
  dataSource:
    components:
    - pod-0
    - pod-2
    encryptionSecret:
      name: encrypt-secret
      namespace: demo
    repository: gcs-demo-repo
    snapshot: latest
  target:
    apiGroup: apps
    kind: StatefulSet
    name: restore-sts
    namespace: demo
status:
  components:
    pod-0:
      duration: 10.089085545s
      phase: Succeeded
    pod-2:
      duration: 6.509171185s
      phase: Succeeded
  conditions:
  - lastTransitionTime: "2023-12-18T09:13:35Z"
    message: Validation has been passed successfully.
    reason: ResourceValidationPassed
    status: "True"
    type: ValidationPassed
  - lastTransitionTime: "2023-12-18T09:13:36Z"
    message: Restore Executor has been ensured successfully.
    reason: SuccessfullyEnsuredRestoreExecutor
    status: "True"
    type: RestoreExecutorEnsured
  - lastTransitionTime: "2023-12-18T09:13:59Z"
    message: Metrics have been pushed successfully.
    reason: SuccessfullyPushedMetrics
    status: "True"
    type: MetricsPushed
  dependencies:
  - found: true
    kind: Snapshot
    name: gcs-demo-repo-sample-backup-sts-demo-session-1702543201
    namespace: demo
  - found: true
    kind: Addon
    name: workload-addon
  duration: 25s
  phase: Succeeded
  targetFound: true
  totalComponents: 2
```

Here, we are going to describe the various sections of a `RestoreSession` object.

### RestoreSession `Spec`

A `RestoreSession` object has the following fields in the `spec` section.

#### spec.target

`spec.target` indicates the target application where the data will be restored. You have to specify `apiGroup`, `kind`, `name` and `namespace` of the target.

#### spec.dataSource

`spec.dataSource` specifies the information about the data that will be restored. It consists of the following fields:
- **namespace:** specifies the namespace of the DataSource (i.e. Repository, Snapshot). If you keep this field empty, then KubeStash will consider the DataSource is in the same namespace of the `RestoreSession`.
- **repository :** points to the `Repository` name from which the data will be restored. The `Repository` must be in the same namespace as the `RestoreSession` CR. This field can be empty if the snapshot name is specified.
- **snapshot :**  specifies the `Snapshot` name that will be restored. If you want to use the latest `Snapshot`, use `latest` value in this field. KubeStash will automatically find the latest `Snapshot`.
- **components :** specifies the list of components that will be restored. If you keep this field empty, then all the components that were backed up in the desired `Snapshot` will be restored.
- **encryptionSecret :** refers to the Secret containing the encryption key which will be used to decrypt the backed up data. You can refer to a Secret of a different namespace. You have to specify `name` and `namespace` of the secret. This field is optional. No encryption secret is required for restoring from `VolumeSnapshot` backups.

#### spec.addon

`spec.addon` specifies addon configuration that will be used to restore the target. Addon has the following fields:
  - **name :** specifies the name of the addon that will be used for the restore purpose.
  - **tasks :** specifies a list of restore tasks and their configuration parameters. To learn about the fields under `task`, see [Task Reference](#task-reference).
  - **containerRuntimeSettings :** specifies runtime settings for the restore executor container. More information can be found [here](#container-level-runtime-settings).
  - **jobTemplate :** specifies runtime configurations for the restore Job. More information can be found [here](#podtemplate-spec).

#### spec.hooks
`spec.hooks` specifies the restore hooks that should be executed before and/or after the restore. Hooks has two fields:
  - **preRestore :** specifies a list of hooks that will be executed before restore. To learn about the fields under `preRestore`, see [HookInfo](#hookinfo).
  - **postRestore :** specifies a list of hooks that will be executed after restore. To learn about the fields under `postRestore`, see [HookInfo](#hookinfo).

#### spec.restoreTimeout 
`spec.restoreTimeout` specifies a duration that KubeStash should wait for the restore to be completed. If the restore tasks do not finish within this time period, KubeStash will consider this restore as a failure.

#### spec.manifestOptions
`spec.manifestOptions` provide options to select particular manifest object to restore. It consists of the following fields:
- **restoreNamespace :** specifies the Namespace where the restored manifest files will be created.
- **mongoDB :** specifies the options for selecting particular `MongoDB` components to restore in manifest restore. To know more about the fields in MongoDB manifest option, see [here](#kubedb-manifestoption).
- **postgres :** specifies the options for selecting particular `Postgres` components to restore in manifest restore. To know more about the fields in Postgres manifest option, see [here](#kubedb-manifestoption).
- **mySQL :** specifies the options for selecting particular `MySQL` components to restore in manifest restore. To know more about the fields in MySQL manifest option, see [here](#kubedb-manifestoption).
- **mariaDB :** specifies the options for selecting particular `MariaDB` components to restore in manifest restore. To know more about the fields in MariaDB manifest option, see [here](#kubedb-manifestoption).

#### Task Reference
Task Reference specifies a task and its configuration parameters. A `task` contains the following fields:
- **name :** indicates to the name of the task.
- **variables :** specifies a list of variables and their sources that will be used to resolve the task. For more details, please visit [here](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
- **params :** specifies parameters for the task. These parameters must be defined in the respective Addon.
- **targetVolumes :** specifies which volumes from the target should be mounted in the restore job/container. It contains the following fields:
  - **volumes :** indicates the list of volumes of targeted application that should be mounted on the restore job.
  - **volumeMounts :** specifies the mount for the volumes specified in `Volumes` section.
  - **volumeClaimTemplates :** specifies a template for the PersistentVolumeClaims that will be created for each Pod in a StatefulSet.
- **addonVolumes :** lets you overwrite the volume sources used in the VolumeTemplate section of Addon. Make sure that name of your volume matches with the name of the volume you want to overwrite. Each `addonVolume` contains the following fields:
  - **name :** specifies the name of the volume.
  - **source :** specifies the source of this volume or a template for volume to use.

#### HookInfo
HookInfo specifies the information about the restore hooks. It consists of the following fields:

| Field             | Usage                                                                                                                                                                                                                                                                                                                                                                                                         |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`            | specifies a name for the hook                                                                                                                                                                                                                                                                                                                                                                                 |
| `hookTemplate`    | points to a HookTemplate CR that will be used to execute the hook. You can refer to a HookTemplate from other namespaces as long as your current namespace is allowed by the `usagePolicy` in the respective HookTemplate.                                                                                                                                                                                    |
| `params`          | specifies parameters for the hook. These parameters must be defined in the respective HookTemplates.                                                                                                                                                                                                                                                                                                          |
| `maxRetry`        | MaxRetry specifies how many times KubeStash should retry the hook execution in case of failure.   The default value of this field is 0 which means no retry.                                                                                                                                                                                                                                                  |
| `timeout`         | Timeout specifies a duration in seconds that KubeStash should wait for the hook execution to be completed. If the hook execution does not finish within this time period, KubeStash will consider this hook execution as failure. Then, it will be re-tried according to MaxRetry policy.                                                                                                                     |
| `executionPolicy` | ExecutionPolicy specifies when to execute the hook. Valid values are: <ul><li>**Always** KubeStash will execute this hook no matter the backup/restore failed. This is the default execution policy.</li><li>**OnSuccess** KubeStash will execute this hook only if the backup/restore has succeeded.</li><li>**OnFailure** KubeStash will execute this hook only if the backup/restore has failed.</li></ul> |
| `variables`       | specifies a list of variables and their sources that will be used to resolve the HookTemplate.                                                                                                                                                                                                                                                                                                                |
| `volumes`         | indicates the list of volumes of targeted application that should be mounted on the hook executor. Use this field only for `Function` type hook executor.                                                                                                                                                                                                                                                     |
| `volumeMounts`    | specifies the mount for the volumes specified in `Volumes` section. Use this field only for `Function` type hook executor.                                                                                                                                                                                                                                                                                    |
| `runtimeSettings` | specifies runtime configurations for the hook executor Job. Use this field only for `Function` type hook executor. To know more about the fields in `runtimeSettings`, see [Runtime Settings](#runtime-settings)                                                                                                                                                                                              |

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

#### PodTemplate Spec
PodTemplate Spec describes the data a pod should have when created from a template. `template` has the following fields:
- **metadata :** specifies standard object's metadata. It contains `labels` and `annotations` fields.
- **controller :** specifies workload controller's metadata. It contains `labels` and `annotations` fields.
- **spec :** specifies the desired behavior of the pod. It contains the following fields:

| Field                           | Usage                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
|---------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `volumes`                       | List of volumes that can be mounted by containers belonging to the pod.  More info can be found [here](https://kubernetes.io/docs/concepts/storage/volumes)                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `initContainers`                | List of initialization containers belonging to the pod. More info can be found [here](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `terminationGracePeriodSeconds` | Optional duration in seconds the pod needs to terminate gracefully. May be decreased in delete request. Value must be non-negative integer. The value zero indicates stop immediately via the kill signal (no opportunity to shut down). If this value is nil, the default grace period will be used instead. The grace period is the duration in seconds after the processes running in the pod are sent a termination signal and the time when the processes are forcibly halted with a kill signal. Set this value longer than the expected cleanup time for your process. Defaults to 30 seconds. |
| `dnsPolicy`                     | Set DNS policy for the pod. Defaults to "ClusterFirst". Valid values are `ClusterFirstWithHostNet`, `ClusterFirst`, `Default` or `None`. DNS parameters given in DNSConfig will be merged with the policy selected with DNSPolicy. To have DNS options set along with hostNetwork, you have to specify DNS policy explicitly to `ClusterFirstWithHostNet`.                                                                                                                                                                                                                                            |
| `nodeSelector`                  | NodeSelector is a selector which must be true for the pod to fit on a node. Selector which must match a node's labels for the pod to be scheduled on that node. More info [here](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)                                                                                                                                                                                                                                                                                                                                                  |
| `serviceAccountName`            | ServiceAccountName is the name of the ServiceAccount to use to run this pod. More info [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `hostNetwork`                   | Host networking requested for this pod. Use the host's network namespace. If this option is set, the ports that will be used must be specified. Default to false.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `hostPID`                       | Use the host's pid namespace. It is optional. Default to false.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `hostIPC`                       | Use the host's ipc namespace. It is optional. Default to false.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `shareProcessNamespace`         | Share a single process namespace between all of the containers in a pod. When this is set containers will be able to view and signal processes from other containers in the same pod, and the first process in each container will not be assigned PID 1. HostPID and ShareProcessNamespace cannot both be set. It is optional. Default to false.                                                                                                                                                                                                                                                     |
| `securityContext`               | SecurityContext holds pod-level security attributes and common container settings. It is optional. Defaults to empty.  See type description for default values of each field.                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `imagePullSecrets`              | ImagePullSecrets is an optional list of references to secrets in the same namespace to use for pulling any of the images used by this PodSpec. If specified, these secrets will be passed to individual puller implementations for them to use. More info [here](https://kubernetes.io/docs/concepts/containers/images#specifying-imagepullsecrets-on-a-pod)                                                                                                                                                                                                                                          |
| `affinity`                      | Affinity is a group of affinity scheduling rules. If specified, the pod's scheduling constraints.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `schedulerName`                 | If specified, the pod will be dispatched by specified scheduler. If not specified, the pod will be dispatched by default scheduler.                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `tolerations`                   | If specified, the pod's tolerations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `priorityClassName`             | If specified, indicates the pod's priority. "system-node-critical" and "system-cluster-critical" are two special keywords which indicate the highest priorities with the former being the highest priority. Any other name must be defined by creating a PriorityClass object with that name. If not specified, the pod priority will be default or zero if there is no default.                                                                                                                                                                                                                      |
| `priority`                      | The priority value. Various system components use this field to find the priority of the pod. When Priority Admission Controller is enabled, it prevents users from setting this field. The admission controller populates this field from PriorityClassName. The higher the value, the higher the priority.                                                                                                                                                                                                                                                                                          |
| `dnsConfig`                     | Specifies the DNS parameters of a pod. Parameters specified here will be merged to the generated DNS configuration based on DNSPolicy.                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `runtimeClassName`              | RuntimeClassName refers to a RuntimeClass object in the node.k8s.io group, which should be used to run this pod.  If no RuntimeClass resource matches the named class, the pod will not be run. If unset or empty, the "legacy" RuntimeClass will be used, which is an implicit class with an empty definition that uses the default runtime handler. More info [here](https://git.k8s.io/enhancements/keps/sig-node/585-runtime-class)                                                                                                                                                               |
| `enableServiceLinks`            | EnableServiceLinks indicates whether information about services should be injected into pod's environment variables, matching the syntax of Docker links. It is optional. Defaults to true.                                                                                                                                                                                                                                                                                                                                                                                                           |
| `topologySpreadConstraints`     | TopologySpreadConstraints describes how a group of pods ought to spread across topology domains. Scheduler will schedule pods in a way which abides by the constraints. All topologySpreadConstraints are ANDed.                                                                                                                                                                                                                                                                                                                                                                                      |
| `args`                          | Arguments to the entrypoint. The docker image's CMD is used if this is not provided. Variable references $(VAR_NAME) are expanded using the container's environment. If a variable cannot be resolved, the reference in the input string will be unchanged. The $(VAR_NAME) syntax can be escaped with a double $$, ie: $$(VAR_NAME). Escaped references will never be expanded, regardless of whether the variable exists or not. Cannot be updated. More info [here](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#running-a-command-in-a-shell)      |
| `env`                           | List of environment variables to set in the container. Cannot be updated.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `resources`                     | Compute Resources required by the sidecar container.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `livenessProbe`                 | Periodic probe of container liveness. Container will be restarted if the probe fails. Controllers may set default LivenessProbe if no liveness probe is provided. To ignore defaulting, set the value to empty LivenessProbe "{}". Cannot be updated. More info [here](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes)                                                                                                                                                                                                                                             |
| `readinessProbe`                | Periodic probe of container service readiness. Container will be removed from service endpoints if the probe fails. Cannot be updated. Controllers may set default ReadinessProbe if no readyness probe is provided. To ignore defaulting, set the value to empty ReadynessProbe "{}". More info [here](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes)                                                                                                                                                                                                            |
| `lifecycle`                     | Actions that the management system should take in response to container lifecycle events. Cannot be updated.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `containerSecurityContext`      | Security options the pod should run with.  More info [here](https://kubernetes.io/docs/concepts/policy/security-context/) and [here](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)                                                                                                                                                                                                                                                                                                                                                                                      |                                                                                                    |
| `volumeMounts`                  | Pod volumes to mount into the container's filesystem. Cannot be updated.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

#### KubeDB ManifestOption

KubeDB ManifestOption consists of the following fields:

| Field              | Usage                                                          |
|--------------------|----------------------------------------------------------------|
| `db`               | specifies whether to restore the DB manifest or not.           |
| `dbName`           | specifies the new name of the DB object after restore.         |
| `authSecret`       | specifies whether to restore the AuthSecret manifest or not.   |
| `authSecretName`   | specifies the new name of the AuthSecret object after restore. |
| `configSecret`     | specifies whether to restore the ConfigSecret manifest or not. |
| `configSecretName` | specifies the new name of the ConfigSecret yaml after restore. |
| `issuerRefName`    | specifies the new name of the TLS IssuerRef after restore.     |

### RestoreSession `Status`

`.status` section of `RestoreSession` shows progress, stats, and overall phase of the restore process. `.status` section consists of the following fields:

#### status.phase

`status.phase` represents the current state of the restore process for this `RestoreSession`. `status.phase` will be `Succeeded` only if the phase of all components are `Succeeded` and the post-restore actions are successful. If any of the components fail to complete restore, `status.phase` will be `Failed`.

#### status.targetFound

`status.targetFound` specifies whether the restore target exists or not.

#### status.duration

`status.duration` specifies the total time taken to complete the restore process.

#### status.restoreDeadline

`status.restoreDeadline` indicates the deadline of the restore. Restore will be considered `Failed` if it does not complete within this deadline.

#### status.totalComponents

`status.totalComponents` represents the number of total components for this `RestoreSession`.

#### status.components

`status.components` represents the individual component restore status. Each component consists of the following fields:
- **phase :** represents the restore phase of the component.
- **duration :** specifies the total time taken to complete the restore process for this component.
- **error :** specifies the reason in case of restore failure for the component.

#### status.hooks

`status.hooks` represents the hook execution status. It consists of the following fields:
- **preHooks :** represents the pre-restore hook execution status.
- **postHooks :** represents the post-restore hook execution status.

Each `preHook` or `postHook` has the following fields:
- **name :** indicates the name of the hook whose status is being shown here.
- **phase :** represents the hook execution phase.

#### status.dependencies
`status.dependencies` specifies whether the objects required by this `RestoreSession` exist or not.

#### status.pausedBackups

`status.pausedBackups` represents the list of backups that have been paused before restore.

#### status.conditions

`status.conditions` shows the conditions of various steps of the restore process. KubeStash sets the following conditions for a `RestoreSession`:

| Condition Type                       | Usage                                                                      |
|--------------------------------------|----------------------------------------------------------------------------|
| `RestoreExecutorEnsured`             | Indicates whether the restore executor was ensured or not.                 |
| `RestoreTargetFound`                 | Indicates whether the restore target was found or not.                     |
| `PreRestoreHooksExecutionSucceeded`  | Indicates whether the preRestore hooks were successfully executed or not.  |
| `PostRestoreHooksExecutionSucceeded` | Indicates whether the postRestore hooks were successfully executed or not. |
| `ValidationPassed`                   | Indicates whether the validation checks were passed or not.                |
| `MetricsPushed`                      | Indicates whether the metrics were pushed or not.                          |
| `RestoreIncomplete`                  | Indicates whether the restore is incomplete or not.                        |

## Next Steps

- Learn how restore of workloads data works from [here](/docs/guides/workloads/overview/index.md).
- Learn how restore stand-alone PVC works from [here](/docs/guides/volumes/overview/index.md).
