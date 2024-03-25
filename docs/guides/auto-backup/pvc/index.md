---
title: Auto Backup PVC | KubeStash
description: An step by step guide on how to configure automatic backup for PVCs.
menu:
  docs_{{ .version }}:
    identifier: auto-backup-pvc
    name: Auto Backup for PVCs
    parent: auto-backup
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Auto Backup for PVC

This tutorial will show you how to configure automatic backup for PersistentVolumeClaims. Here, we are going to backup a PVC provisioned from an NFS server using auto-backup.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).
- Here, we are going to use an NFS server to provision a PVC with `ReadWriteMany` access. If you don't have an NFS server running, deploy one by following the guide [here](https://github.com/appscode/third-party-tools/blob/master/storage/nfs/README.md).
- You should be familiar with the following `KubeStash` concepts:
  - [BackupBlueprint](/docs/concepts/crds/backupblueprint/index.md)
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [Function](/docs/concepts/crds/function/index.md)
  - [Addon](/docs/concepts/crds/addon/index.md)
  - [Snapshot](/docs/concepts/crds/snapshot/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

>**Note:** YAML files used in this tutorial are stored in  [docs/guides/auto-backup/pvc/examples](/docs/guides/auto-backup/pvc/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

**Verify necessary Function and Task:**

KubeStash uses a `Addon-Function` model to automatically backup PVC. When you install KubeStash, it creates the necessary `Addon` and `Function`.

Let's verify that KubeStash has created the necessary `Function` to backup/restore PVC by the following command,

```bash
$ kubectl get addon
NAME             AGE
kubedump-addon   16h
pvc-addon        16h
workload-addon   16h
```

Also, verify that the necessary `Function` has been created,

```bash
$ kubectl get function
NAME                     AGE
kubedump-backup          16h
pvc-backup               16h
pvc-restore              16h
volumesnapshot-backup    16h
volumesnapshot-restore   16h
workload-backup          16h
workload-restore         16h
```

## Prepare Backup Blueprint

We are going to use [GCS Backend](/docs/guides/backends/gcs/index.md) to store the backed up data. You can use any supported backend you prefer. You just have to configure `BackupStorage` and storage secret to match your backend. To learn which backends are supported by KubeStash and how to configure them, please visit [here](/docs/guides/backends/overview/index.md).

**Create Storage Secret:**

At first, let's create a Storage Secret for the GCS backend,

```bash
$ echo -n '<your-project-id>' > GOOGLE_PROJECT_ID
$ mv downloaded-sa-json.key GOOGLE_SERVICE_ACCOUNT_JSON_KEY
$ kubectl create secret generic -n demo gcs-secret \
    --from-file=./GOOGLE_PROJECT_ID \
    --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
secret/gcs-secret created
```

**Create BackupStorage**

Now, let's create a `BackupStorage` to specify the backend information where the backed up data will be stored.

Below is the YAML of the `BackupStorage` object that we are going to create,

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
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true
  deletionPolicy: WipeOut
```

Let’s create the above `BackupStorage`,
```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/pvc/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-storage created
```

**Create RetentionPolicy**

Now, let's create a `RetentionPolicy` to specify how the old Snapshots should be cleaned up.

Below is the YAML of the `RetentionPolicy` object that we are going to create,

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: RetentionPolicy
metadata:
  name: demo-retention
  namespace: demo
spec:
  default: true
  failedSnapshots:
    last: 2
  maxRetentionPeriod: 2mo
  successfulSnapshots:
    last: 5
  usagePolicy:
    allowedNamespaces:
      from: All
```

Let’s create the above `RetentionPolicy`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/pvc/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

**Create Encryption Secret:**

Let's create a secret called `encrypt-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD \
secret "encrypt-secret" created
```

**Create BackupBlueprint:**

Now, we have to create a `BackupBlueprint` CR with a blueprint for `BackupConfiguration` object.

Below is the YAML of the `BackupBlueprint` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupBlueprint
metadata:
  name: pvc-backup-blueprint
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
          name: gcs-storage
          namespace: demo
        retentionPolicy:
          name: demo-retention
          namespace: demo
    sessions:
      - name: frequent-backup
        sessionHistoryLimit: 3
        scheduler:
          schedule: "*/5 * * * *"
          jobTemplate:
            backoffLimit: 1
        repositories:
          - name: ${repoName}
            backend: gcs-backend
            directory: ${namespace}/${targetName}
            encryptionSecret:
              name: encrypt-secret
              namespace: demo
        addon:
          name: pvc-addon
          tasks:
            - name: logical-backup
        retryConfig:
          maxRetry: 2
          delay: 1m
```

Note that we have used some variables (format: `${<variable name>}`) in different fields. KubeStash will substitute these variables with values from the respective target's annotations. You can use arbitrary variables.

Let's create the `BackupBlueprint` that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/pvc/examples/backupblueprint.yaml
backupblueprint.core.kubestash.com/pvc-backup-blueprint created
```

Now, automatic backup is configured for PVC. We just have to add some annotations to the targeted PVC to enable backup.

**Available Auto-Backup Annotations:**

You have to add the auto-backup annotations to the pvc that you want to backup. The following auto-backup annotations are available:

Required Annotations:

- **BackupBlueprint Name:** Specifies the name of the `BackupBlueprint` to be used for this pvc's backup.

```yaml
blueprint.kubestash.com/name: <BackupBlueprint name>
```

- **BackupBlueprint Namespace:** Specifies the namespace where the `BackupBlueprint` resides.

```yaml
blueprint.kubestash.com/namespace: <BackupBlueprint namespace>
```

Optional Annotations:

- **Session Names:** Defines which sessions from the `BackupBlueprint` should be used for the `BackupConfiguration`. If you don’t specify this annotation, all sessions from the `BackupBlueprint` will be used. For multiple sessions, you can provide comma (,) seperated session names.

```yaml
 blueprint.kubestash.com/session-names: <Session names>
```

- **Variables:** Allows you to customize the `BackupConfiguration` by providing values for variables defined within the `BackupBlueprint`. You can define as many annotations as needed for variables.

```yaml
variables.kubestash.com/<variable-name>: <Variable value>
```

Example:

Assuming you are using a variable named `schedule` for a session schedule in the `BackupBlueprint`, you can configure different backup schedules for each target application using annotations like this:
```yaml
variables.kubestash.com/schedule: "*/5 * * * *"
```

## Prepare PVC

At first, let's prepare our desired PVC. Here, we are going to create a [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) (PV) that will use an NFS server as storage. Then, we are going to create a PVC that will bind with the PV. Then, we are going to mount this PVC into a pod. This pod will generate a sample file into the PVC.

**Create PersistentVolume:**

We have deployed an NFS server in `storage` namespace, and it is accessible through a Service named `nfs-service`. Now, we are going to create a PV that uses the NFS server as storage.

Below is the YAML of the PV that we are going to create,

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  csi:
    driver: nfs.csi.k8s.io
    volumeHandle: nfs-server.storage.svc.cluster.local/share##
    volumeAttributes:
      server: nfs-server.storage.svc.cluster.local
      share: /
```

Let's create the PV we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/pvc/examples/nfs_pv.yaml
persistentvolume/nfs-pv created
```

**Create PersistentVolumeClaim:**

Now, create a PVC to bind with the PV we have just created. Below, is the YAML of the PVC that we are going to create,

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  namespace: demo
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  volumeName: nfs-pv
  storageClassName: ""
```

Notice the `spec.volumeName` section. We have specified `nfs-pv` pv name as the volume name.

Let's create the PVC we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/pvc/examples/nfs_pvc.yaml
persistentvolumeclaim/nfs-pvc created
```

Verify that the PVC has bounded with our desired PV,

```bash
$ kubectl get pvc -n demo nfs-pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc   Bound    nfs-pv   1Gi        RWX                           61s
```

Here, we can see that the PVC `nfs-pvc` has been bounded with PV `nfs-pv`.

**Generate Sample Data:**

Now, we are going to deploy two sample pods `demo-pod-1` and `demo-pod-2` that will mount `pod-1/data` and `pod-2/data` [subPath](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath) of the `nfs-pvc` respectively. Each of the pods will generate a sample file named `hello.txt` with some demo data. We are going to backup the entire PVC that contains the sample files using auto-backup.

Below, is the YAML of the first pod that we are going to deploy,

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod-1
  namespace: demo
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["/bin/sh", "-c","echo 'hello from pod 1.' > /sample/data/hello.txt && sleep 3000"]
      volumeMounts:
        - name: my-volume
          mountPath: /sample/data
          subPath: pod-1/data
  volumes:
    - name: my-volume
      persistentVolumeClaim:
        claimName: nfs-pvc
```

Here, we have mounted `pod-1/data` directory of the `nfs-pvc` into `/sample/data` directory of this pod.

Let's deploy the pod we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/pvc/examples/pod-1.yaml
pod/demo-pod-1 created
```

Verify that the sample data has been generated into `/sample/data/` directory,

```bash
$ kubectl exec -n demo demo-pod-1 cat /sample/data/hello.txt
hello from pod 1.
```

Below is the YAML of the second pod that we are going to deploy,

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod-2
  namespace: demo
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["/bin/sh", "-c","echo 'hello from pod 2.' > /sample/data/hello.txt && sleep 3000"]
    volumeMounts:
    - name: my-volume
      mountPath: /sample/data
      subPath: pod-2/data
  volumes:
  - name: my-volume
    persistentVolumeClaim:
      claimName: nfs-pvc
```

Now, we have mounted `pod-2/data` directory of the `nfs-pvc` into `/sample/data` directory of this pod.

Let's create the pod we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/pvc/examples/pod-2.yaml
pod/demo-pod-2 created
```

Verify that the sample data has been generated into `/sample/data/` directory,

```bash
$ kubectl exec -n demo demo-pod-2 cat /sample/data/hello.txt
hello from pod 2.
```

## Backup

Now, we are going to add auto backup specific annotation to the PVC. KubeStash watches for PVC with auto-backup annotations. Once it finds a PVC with auto-backup annotations, it will create a `BackupConfiguration` CR according to respective `BackupBlueprint`. Then, rest of the backup process will proceed as normal backup of a stand-alone PVC as describe [here](/docs/guides/volumes/pvc/index.md).

**Add Annotations:**

Let's add the auto backup specific annotation to the PVC,

```bash
$ kubectl annotate pvc nfs-pvc -n demo --overwrite     \
    blueprint.kubestash.com/name=pvc-backup-blueprint  \
    blueprint.kubestash.com/namespace=demo             \
    variables.kubestash.com/targetName=nfs-pvc         \
    variables.kubestash.com/namespace=demo             \
    variables.kubestash.com/repoName=pvc-repo
```

Verify that the annotations has been added successfully,

```bash
$ kubectl get pvc -n demo nfs-pvc -o yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    blueprint.kubestash.com/name: pvc-backup-blueprint
    blueprint.kubestash.com/namespace: demo
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{},"name":"nfs-pvc","namespace":"demo"},"spec":{"accessModes":["ReadWriteMany"],"resources":{"requests":{"storage":"1Gi"}},"storageClassName":"","volumeName":"nfs-pv"}}
    pv.kubernetes.io/bind-completed: "yes"
    variables.kubestash.com/namespace: demo
    variables.kubestash.com/repoName: pvc-repo
    variables.kubestash.com/targetName: nfs-pvc
  creationTimestamp: "2024-01-09T04:24:36Z"
  finalizers:
    - kubernetes.io/pvc-protection
  name: nfs-pvc
  namespace: demo
  resourceVersion: "1313188"
  uid: 092f3f7d-12f4-498b-9143-2050bcc50f77
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
  volumeMode: Filesystem
  volumeName: nfs-pv
status:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi
  phase: Bound
```

Now, KubeStash will create a `BackupConfiguration` CR according to the blueprint.

**Verify BackupConfiguration:**
If everything goes well, KubeStash should create a `BackupConfiguration` for our PVC and the phase of that `BackupConfiguration` should be `Ready`. Verify the `BackupConfiguration` CR by the following command,

```bash
$ kubectl get backupconfiguration -n demo
NAME                            PHASE   PAUSED   AGE
persistentvolumeclaim-nfs-pvc   Ready            3m31s
```

Now, let's check the YAML of the `BackupConfiguration`.
```bash
$ kubectl get backupconfiguration -n demo persistentvolumeclaim-nfs-pvc -o yaml
```

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  labels:
    app.kubernetes.io/managed-by: kubestash.com
    kubestash.com/invoker-name: pvc-backup-blueprint
    kubestash.com/invoker-namespace: demo
  name: persistentvolumeclaim-nfs-pvc
  namespace: demo
spec:
  ...
    failurePolicy: Fail
    name: frequent-backup
    repositories:
    - backend: gcs-backend
      directory: demo/nfs-pvc
      encryptionSecret:
        name: encrypt-secret
        namespace: demo
      name: pvc-repo
    ...
  target:
    kind: PersistentVolumeClaim
    name: nfs-pvc
    namespace: demo
status:
  ...
```

Notice that the `spec.target` is pointing to the `nfs-pvc` PVC. Also, notice that the `spec.sessions[*].repositories[*].name` and `spec.sessions[*].repositories[*].directory` fields have been populated with the information we had provided as annotation of the PVC.

**Wait for BackupSession:**

Now, wait for the next backup schedule. Run the following command to watch `BackupSession` CR:

```bash
Every 2.0s: kubectl get backupsession -n demo                                   AppsCode-PC-03: Tue Jan  9 12:26:47 2024

NAME                                                       INVOKER-TYPE          INVOKER-NAME                    PHASE       DURATION   AGE
persistentvolumeclaim-nfs-pvc-frequent-backup-1704779101   BackupConfiguration   persistentvolumeclaim-nfs-pvc   Succeeded              41m
```

**Verify Backup:**

When backup session is completed, KubeStash will update the respective `Repository` to reflect the latest state of backed up data.

Run the following command to check if the snapshots are stored in the backend,

```bash
$ kubectl get repository -n demo pvc-repo
NAME       INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
pvc-repo   true        1                2.286 KiB   Ready   45m                      45m
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=pvc-repo
NAME                                                              REPOSITORY   SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       VERIFICATION-STATUS   AGE
pvc-repo-persistentvolumeclaim-n-pvc-frequent-backup-1704779101   pvc-repo     frequent-backup   2024-01-09T05:45:08Z   Delete            Succeeded                         50m
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <target-kind>`
> - `kubestash.com/app-ref-name: <target-name>`
> - `kubestash.com/app-ref-namespace: <target-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
>
> These labels can be used to watch only the `Snapshot`s related to our desired Workload or `Repository`.

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the PVC.

```bash
$ kubectl get snapshots -n demo pvc-repo-persistentvolumeclaim-n-pvc-frequent-backup-1704779101 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: PersistentVolumeClaim
    kubestash.com/app-ref-name: nfs-pvc
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: pvc-repo
  name: pvc-repo-persistentvolumeclaim-n-pvc-frequent-backup-1704779101
  namespace: demo
spec:
  ...
status:
  components:
    dump:
      driver: Restic
      duration: 8.276711625s
      integrity: true
      path: repository/v1/frequent-backup/pvc
      phase: Succeeded
      resticStats:
      - hostPath: /kubestash-data
        id: 960ca348fab17a73b9acf8168a8bb564b1b60bc13bfa163d705f38ee77aefcd2
        size: 36 B
        uploaded: 3.633 KiB
      size: 2.287 KiB
  ...
```
> For PVC, KubeStash takes backup the targeted PVC in `/kubestash-data` file path. For logical backup, KubeStash uses `dump` as the component name for a stand-alone PVC.

Now, if we navigate to the GCS bucket, we will see the backed up data has been stored in the `demo/demo/nfs-pvc/repository/v1/frequent-backup/dump` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/demo/nfs-pvc/snapshots` directory.

> Note: KubeStash stores all dumped data encrypted in the backup directory, meaning it remains unreadable until decrypted.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo backupBlueprint/pvc-backup-blueprint
kubectl delete -n demo repository/persistentvolumeclaim-nfs-pvc
kubectl delete -n demo backupconfiguration/persistentvolumeclaim-nfs-pvc

kubectl delete -n demo pod/demo-pod
kubectl delete -n demo pvc/nfs-pvc
kubectl delete -n demo pv/nfs-pv
```

If you would like to uninstall KubeStash operator, please follow the steps [here](/docs/setup/README.md).
