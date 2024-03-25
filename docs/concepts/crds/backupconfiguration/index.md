---
title: BackupConfiguration Overview
menu:
  docs_{{ .version }}:
    identifier: backupconfiguration-overview
    name: BackupConfiguration
    parent: crds
    weight: 15
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# BackupConfiguration

## What is BackupConfiguration
A `BackupConfiguration` is a Kubernetes `CustomResourceDefinition`(CRD) which specifies the backup target, the backends references and the sessions that specifies when and how to take backup in a Kubernetes native way.

You have to create a `BackupConfiguration` object for each backup target. A backup target can be a workload, database or a PV/PVC.

## BackupConfiguration CRD Specification

Like any official Kubernetes resource, a `BackupConfiguration` has `TypeMeta`, `ObjectMeta`, `Spec` and `Status` sections.

A sample `BackupConfiguration` object to backup the volumes of a StatefulSet is shown below:
```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-backup-sts
  namespace: demo
spec:
  backends:
    - name: gcs-backend
      retentionPolicy:
        name: demo-retention
        namespace: demo
      storageRef:
        name: gcs-storage
        namespace: demo
  sessions:
    - addon:
        name: workload-addon
        tasks:
          - name: LogicalBackup
            params:
              exclude: /source/data/lost+found
              paths: /source/data
            targetVolumes:
              volumeMounts:
                - mountPath: /source/data
                  name: source-data
      name: demo-session
      repositories:
        - backend: gcs-backend
          directory: /demo/data
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
          name: gcs-demo-repo
      retryConfig:
        delay: 1m0s
        maxRetry: 2
      scheduler:
        jobTemplate:
          backoffLimit: 1
          template:
            controller: {}
            metadata: {}
            spec:
              resources: {}
        schedule: '*/5 * * * *'
      sessionHistoryLimit: 1
  target:
    apiGroup: apps
    kind: StatefulSet
    name: sample-sts
    namespace: demo
status:
  backends:
    - name: gcs-backend
      ready: true
      retentionPolicy:
        found: true
        ref:
          name: demo-retention
          namespace: demo
      storage:
        phase: Ready
        ref:
          name: gcs-storage
          namespace: demo
  conditions:
    - lastTransitionTime: "2023-12-14T08:38:01Z"
      message: Validation has been passed successfully.
      reason: ResourceValidationPassed
      status: "True"
      type: ValidationPassed
  dependencies:
    - found: true
      kind: Addon
      name: workload-addon
  phase: Ready
  repositories:
    - name: gcs-demo-repo
      phase: Ready
  sessions:
    - conditions:
        - lastTransitionTime: "2023-12-14T08:38:01Z"
          message: Scheduler has been ensured successfully.
          reason: SchedulerEnsured
          status: "True"
          type: SchedulerEnsured
      name: demo-session
  targetFound: true
```
Here, we are going to describe the various sections of `BackupConfiguration` crd.

## BackupConfiguration `Spec`

A `BackupConfiguration` object has the following fields in the `spec` section.

**spec.target**

`spec.target` refers to the target of backup. This field consists of `apiGroup`, `kind`, `name` and `namespace`. The backup target can be in a different namespace than the `BackupConfiguration`.

**spec.backends**

`spec.backends` specifies a list of storage references where the backed up data will be stored. The respective `BackupStorages` can be in a different namespace than the `BackupConfiguration`.
However, it must be allowed by the `usagePolicy` of the `BackupStorage` to refer from this namespace. This field is optional, if you don't provide any backend here, KubeStash will use the default `BackupStorage`
for the namespace. If a default `BackupStorage` does not exist in the same namespace, then KubeStash will look for a default `BackupStorage` in other namespaces that allows using it from the `BackupConfiguration`
namespace. Each backend has the following fields:

| Field             | Usage                                                                                                                                                                                                                                                                                                                                                                                                          |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`            | specifies an identifier for this storage. This name must be **unique** among all the backend names provided in this `BackupConfiguration`.                                                                                                                                                                                                                                                                     |
| `storageRef`      | refers to the `BackupStorage` custom resource that holds the information of a storage.                                                                                                                                                                                                                                                                                                                         |
| `retentionPolicy` | refers to a `RetentionPolicy` custom resource which defines how to cleanup the old `Snapshots`. This field is optional, if you don't provide this field, KubeStash will use the default `RetentionPolicy` for the namespace. If there is no default `RetentionPolicy` for the namespace, then KubeStash will find a `RetentionPolicy` from other namespaces that is allowed to use from the current namespace. |

**spec.sessions**

`spec.sessions` defines a list of session configuration that specifies when and how to take backup. Each session has the following fields:
- **name :** specifies an identifier for this session. This name must be **unique** among all the session names provided in this `BackupConfiguration`.
- **scheduler :** specifies the configuration for backup triggering CronJob. To learn about the fields under `scheduler`, see [Scheduler Spec](#scheduler-spec).
- **hooks :** specifies the backup hooks that should be executed before and/or after the backup. Hooks has two fields:
  - **preBackup :** specifies a list of hooks that will be executed before backup. To learn about the fields under `preBackup`, see [HookInfo](#hookinfo).
  - **postBackup :** specifies a list of hooks that will be executed after backup. To learn about the fields under `postBackup`, see [HookInfo](#hookinfo).
- **retryConfig :** specifies the behavior of retry in case of a backup failure. RetryConfig has the following fields:
  - **maxRetry :** specifies the maximum number of times KubeStash should retry the backup/restore process. By default, KubeStash will retry only 1 time.
  - **delay :** The amount of time to wait before next retry. If you don't specify this field, KubeStash will retry immediately. Format: 30s, 2m, 1h etc.
- **timeout :** specifies the maximum duration of backup. BackupSession will be considered Failed if backup does not complete within this time limit. By default, KubeStash don't set any timeout for backup.
- **sessionHistoryLimit :** specifies how many backup Jobs and associate resources KubeStash should keep for debugging purpose. The default value is 1.
- **addon :** specifies addon configuration that will be used to backup the target. Addon has the following fields:
  - **name :** specifies the name of the addon that will be used for the backup purpose.
  - **tasks :** specifies a list of backup tasks and their configuration parameters. To learn about the fields under `task`, see [Task Reference](#task-reference).
  - **containerRuntimeSettings :** specifies runtime settings for the backup executor container. More information can be found [here](#container-level-runtime-settings).
  - **jobTemplate :** specifies runtime configurations for the backup Job. More information can be found [here](#podtemplate-spec).
- **repositories :** specifies a list of repository information where the backed up data will be stored. KubeStash will create the respective `Repository` CRs using this information. Each repository consists of the following fields:
  - **name :** specifies the name of the `Repository`.
  - **backend :** specifies the name of the backend where this repository will be initialized. This should point to a backend name specified in `.spec.backends` section. For using a default backend, keep this field empty.
  - **directory :** specifies the path inside the backend where the backed up data will be stored.
  - **encryptionSecret :** refers to the Secret containing the encryption key which will be used to encrypt the backed up data. You can refer to a Secret of a different namespace by providing `name` and `namespace` fields. This field is optional. No encryption secret is required for `VolumeSnapshot` backups.
  - **deletionPolicy :** specifies what to do when you delete a `Repository` CR. The valid values for this field are:
    - **Delete :** This will delete just the `Repository` CR from the cluster but keep the backed up data in the remote backend. This is the default behavior.
    - **WipeOut :** This will delete the `Repository` CR as well as the backed up data from the backend.

**spec.paused**

`spec.paused` indicates that the BackupConfiguration has been paused from taking backup. Default value is 'false'. If you set `paused` field to `true`, KubeStash will suspend the respective backup triggering CronJob and skip processing any further events for this `BackupConfiguration`.

#### Scheduler Spec
Scheduler Spec specifies the configuration for the backup triggering CronJob for a session. `scheduler` has the following fields:
- **schedule :** The schedule in Cron format, see [here](https://en.wikipedia.org/wiki/Cron) to learn more.
- **startingDeadlineSeconds :** Optional deadline in seconds for starting the job if it misses scheduled time for any reason.  Missed jobs executions will be counted as failed ones.
- **concurrencyPolicy :** Specifies how to treat concurrent executions of a Job. Valid values are:
  - **Allow :** (default) allows CronJobs to run concurrently.
  - **Forbid :** forbids concurrent runs, skipping next run if previous run hasn't finished yet.
  - **Replace :** cancels currently running job and replaces it with a new one.
- **suspend :** This flag tells the controller to suspend subsequent executions, it does not apply to already started executions. Defaults to false.
- **successfulJobsHistoryLimit :** The number of successful finished jobs to retain. Value must be non-negative integer. Defaults to 3.
- **failedJobsHistoryLimit :** The number of failed finished jobs to retain. Value must be non-negative integer. Defaults to 1.
- **jobTemplate :** Specifies the job that will be created when executing a CronJob. JobTemplate has the following fields:

  | Field                     | Usage                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
    |---------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | `parallelism`             | Specifies the maximum desired number of pods the job should run at any given time. The actual number of pods running in steady state will be less than this number when ((`.spec.completions` - `.status.successful)` < `.spec.parallelism`), i.e. when the work left to do is less than max parallelism. More info can be found [here](https://kubernetes.io/docs/concepts/workloads/controllers/job/#parallel-jobs)                                                                                                                                                                                                                                     |
  | `completions`             | Specifies the desired number of successfully finished pods the job should be run with.  Setting to nil means that the success of any pod signals the success of all pods, and allows parallelism to have any positive value. Setting to 1 means that parallelism is limited to 1 and the success of that pod signals the success of the job. More info [here](https://kubernetes.io/docs/concepts/workloads/controllers/job/#parallel-jobs)                                                                                                                                                                                                               |
  | `activeDeadlineSeconds`   | Specifies the duration in seconds relative to the startTime that the job  may be continuously active before the system tries to terminate it; value must be positive integer. If a Job is suspended (at creation or through an update), this timer will effectively be stopped and reset when the Job is resumed again.                                                                                                                                                                                                                                                                                                                                   |
  | `backoffLimit`            | Specifies the number of retries before marking this job failed. Defaults to 6.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
  | `template`                | Describes the pod that will be created when executing a job. To know more about the fields in `template`, see [PodTemplate Spec]()                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
  | `ttlSecondsAfterFinished` | `ttlSecondsAfterFinished` limits the lifetime of a Job that has finished execution (either Complete or Failed). If this field is set, `ttlSecondsAfterFinished` after the Job finishes, it is eligible to be automatically deleted. When the Job is being deleted, its lifecycle guarantees (e.g. finalizers) will be honored. If this field is unset, the Job won't be automatically deleted. If this field is set to zero, the Job becomes eligible to be deleted immediately after it finishes. This field is alpha-level and is only honored by servers that enable the TTLAfterFinished feature.                                                     |
  | `completionMode`          | CompletionMode specifies how Pod completions are tracked. More Information can be found [here](https://kubernetes.io/docs/concepts/workloads/controllers/job/#completion-mode)                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
  | `suspend`                 | Suspend specifies whether the Job controller should create Pods or not. If a Job is created with suspend set to true, no Pods are created by the Job controller. If a Job is suspended after creation (i.e. the flag goes from false to true), the Job controller will delete all active Pods associated with this Job. Users must design their workload to gracefully handle this. Suspending a Job will reset the StartTime field of the Job, effectively resetting the ActiveDeadlineSeconds timer too. This is an alpha field and requires the SuspendJob feature gate to be enabled; otherwise this field may not be set to true. Defaults to false. |

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

#### HookInfo
HookInfo specifies the information about the backup hooks. It contains the following fields:

| Field             | Usage                                                                                                                                                                                                                                                                                                                                                                                 |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`            | specifies a name for the hook                                                                                                                                                                                                                                                                                                                                                         |
| `hookTemplate`    | points to a HookTemplate CR that will be used to execute the hook. You can refer to a HookTemplate from other namespaces as long as your current namespace is allowed by the `usagePolicy` in the respective HookTemplate.                                                                                                                                                            |
| `params`          | specifies parameters for the hook. These parameters must be defined in the respective HookTemplate.                                                                                                                                                                                                                                                                                   |
| `maxRetry`        | MaxRetry specifies how many times KubeStash should retry the hook execution in case of failure. The default value of this field is 0 which means no retry.                                                                                                                                                                                                                            |
| `timeout`         | Timeout specifies a duration in seconds that KubeStash should wait for the hook execution to be completed. If the hook execution does not finish within this time period, KubeStash will consider this hook execution as failure. Then, it will be re-tried according to MaxRetry policy.                                                                                             |
| `executionPolicy` | ExecutionPolicy specifies when to execute the hook. Valid values are: <ul><li>**Always** KubeStash will execute this hook no matter the backup failed. This is the default execution policy.</li><li>**OnSuccess** KubeStash will execute this hook only if the backup has succeeded.</li><li>**OnFailure** KubeStash will execute this hook only if the backup has failed.</li></ul> |
| `variables`       | specifies a list of variables and their sources that will be used to resolve the HookTemplate.                                                                                                                                                                                                                                                                                        |
| `volumes`         | indicates the list of volumes of targeted application that should be mounted on the hook executor. Use this field only for `Function` type hook executor.                                                                                                                                                                                                                             |
| `volumeMounts`    | specifies the mount for the volumes specified in `Volumes` section. Use this field only for `Function` type hook executor.                                                                                                                                                                                                                                                            |
| `runtimeSettings` | specifies runtime configurations for the hook executor Job. Use this field only for `Function` type hook executor. To know more about the fields in `runtimeSettings`, see [Runtime Settings](#runtime-settings)                                                                                                                                                                      |

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


#### Task Reference
Task Reference specifies a task and its configuration parameters. A `task` contains the following fields:
- **name :** indicates to the name of the task.
- **variables :** specifies a list of variables and their sources that will be used to resolve the task. For more details, please visit [here](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
- **params :** specifies parameters for the task. These parameters must be defined in the respective Addon.
- **targetVolumes :** specifies which volumes from the target should be mounted in the backup job/container. It contains the following fields:
  - **volumes :** indicates the list of volumes of targeted application that should be mounted on the backup job.
  - **volumeMounts :** specifies the mount for the volumes specified in `Volumes` section.
  - **volumeClaimTemplates :** specifies a template for the PersistentVolumeClaims that will be created for each Pod in a StatefulSet.
- **addonVolumes :** lets you overwrite the volume sources used in the `VolumeTemplate` section of Addon. Make sure that name of your volume matches with the name of the volume you want to overwrite. Each `addonVolume` contains the following fields:
  - **name :** specifies the name of the volume.
  - **source :** specifies the source of this volume or a template for volume to use.

## BackupConfiguration `Status`
A `BackupConfiguration` object has the following fields in the `status` section.
- **backends :** specifies whether the backends are ready or not. Each backend consists of the following fields:
  - **name :** indicates the backend name.
  - **ready :** indicates whether the respective backend is ready or not.
  - **storage :** indicates the status of the respective `BackupStorage`. It has the following fields:
    - **ref :** indicates to the `BackupStorage` object.
    - **phase :** indicates the current phase of the respective `BackupStorage` which can be `Ready` or `NotReady`.
    - **reason :** specifies the error messages found while checking the `BackupStorage` phase.
  - **retentionPolicy :** indicates the status of the respective `RetentionPolicy`. It has the following fields:
    - **ref :** indicates the `RetentionPolicy` object reference.
    - **found :** indicates whether the `RetentionPolicy` is Found or not.
    - **reason :** specifies the error messages found while checking the `RetentionPolicy`.
- **repositories :** indicates the status of the respective repositories. It consists of the following fields:
  - **name :** indicate the name of the `Repository`.
  - **phase :** indicates the phase of the respective Repository which can be `Ready` or `NotReady`.
  - **reason :** specifies the error messages found while ensuring the respective `Repository`.
- **dependencies :** specifies whether the objects required by this `BackupConfiguration` exist or not.
- **sessions :** specifies status of the session specific resources. It consists of the following fields:
  - **name :** indicates the name of the session.
  - **nextSchedule :** specifies when the next backup will execute for this session.
  - **conditions :** specifies a list of conditions related to this session. The following condition is set by the KubeStash operator on each `.status.sessions`.
  
| Condition Type     | Usage                                               |
|--------------------|-----------------------------------------------------|
| `SchedulerEnsured` | indicates whether the Scheduler was ensured or not. |

- **phase :** represents the current state of the `BackupConfiguration`.
- **targetFound :** specifies whether the backup target exists or not.
- **conditions :** represents list of conditions regarding this `BackupConfiguration`. The following condition is set by the KubeStash operator on a `BackupConfiguration`.

| Condition Type     | Usage                                                       |
|--------------------|-------------------------------------------------------------|
| `ValidationPassed` | indicates whether the validation checks were passed or not. |

## Next Steps

- Learn how to configure `BackupConfiguration` to backup workloads data from [here](/docs/guides/workloads/overview/index.md).
- Learn how to configure `BackupConfiguration` to backup stand-alone PVC from [here](/docs/guides/volumes/overview/index.md).
