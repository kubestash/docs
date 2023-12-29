---
title: Populate Volume of a Stand-alone PVC | KubeStash
description: An step by step guide to show how to populate volume of a stand-alone PVC
menu:
  docs_{{ .version }}:
    identifier: volume-populate-pvc
    name: Populate Volume of a Stand-alone PVC
    parent: volume-populator
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Populating a volume of a Standalone PVC

This guide will show you how to use KubeStash to populate the volume of standalone PersistentVolumeClaims (PVCs). Within this guide, we will demonstrate the process of backing up a standalone PVC and subsequently restoring it using Kubernetes native approach, facilitated by KubeStash.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).
- You should be familiar with the following `KubeStash` concepts:
    - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
    - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
    - [BackupSession](/docs/concepts/crds/backupsession/index.md)
    - [SnapShot](/docs/concepts/crds/snapshot/index.md)
    - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
    - [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/volumes/pvc/examples](/docs/guides/volumes/pvc/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Prepare Volume

At first, let's prepare our desired PVC. Here, we are going to create a PVC. Then, we are going to mount this PVC with a Pod. Pod will generate a sample file into the PVC.

**Create PersistentVolumeClaim:**

Below, is the YAML of the PVC that we are going to create,

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: target-pvc
  namespace: demo
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

Notice the `spec.accessModes` section. We are using `ReadWriteMany` access mode so that multiple pods can use this PVC simultaneously. Without this access mode, KubeStash will fail to backup the volume if any other pod mount it during backup.

Let's create the PVC we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/pvc/examples/target-pvc.yaml

persistentvolumeclaim/target-pvc created
```

**Deploy Workload:**

Now, we are going to deploy a sample pod `demo-pod` that will mount `/sample/data` path of the `target-pvc`. This pod will generate a sample file named `hello.txt` with some demo data. We are going to backup the entire PVC using KubeStash that contains the sample files.

Below, is the YAML of the pod that we are going to deploy,

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: demo-pod
  namespace: demo
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["/bin/sh", "-c","echo 'hello from pod.' > /sample/data/hello.txt && sleep 3000"]
    volumeMounts:
    - name: my-volume
      mountPath: /sample/data
  volumes:
  - name: my-volume
    persistentVolumeClaim:
      claimName: target-pvc
```

Let's deploy the pod we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/pvc/examples/demo-pod.yaml
pod/demo-pod created
```

Verify that the sample data has been generated into `/sample/data/` directory,

```bash
$ kubectl exec -n demo demo-pod -- cat /sample/data/hello.txt
hello from pod.
```

## Backup

Now, we are going to backup the PVC `target-pvc` in a GCS bucket using KubeStash. We have to create a Secret and a `BackupStorage` object with access credentials and backend information respectively.

> For GCS backend, if the bucket does not exist, KubeStash needs `Storage Object Admin` role permissions to create the bucket. For more details, please check the following [guide](/docs/guides/backends/gcs/index.md).

**Create Storage Secret:**

Let's create a Secret named `gcs-secret` with access credentials of our desired GCS backend,

```bash
$ echo -n '<your-project-id>' > GOOGLE_PROJECT_ID
$ cat /path/to/downloaded/sa_key_file.json > GOOGLE_SERVICE_ACCOUNT_JSON_KEY
$ kubectl create secret generic -n demo gcs-secret \
    --from-file=./GOOGLE_PROJECT_ID \
    --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
secret/gcs-secret created
```

**Create BackupStorage:**

Now, create a `BackupStorage` custom resource specifying the desired bucket, and directory inside the bucket where the backed up data will be stored.

Below is the YAML of `BackupStorage` object that we are going to create,

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
      bucket: kubestash-qa
      prefix: demo
      secret: gcs-secret
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true 
  deletionPolicy: WipeOut
```

Let's create the `BackupStorage` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/pvc/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-storage created
```

Now, we also have to create another secret with an encryption key `RESTIC_PASSWORD` for Restic. This secret will be used by Restic for both encrypting and decrypting the backup data during backup & restore.

**Create Restic Repository Secret:**

Let's create a Secret named `encry-secret` with an encryption key `RESTIC_PASSWORD` for Restic.

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encry-secret \
    --from-file=./RESTIC_PASSWORD 
secret/encryption-secret created
```

Now, we have to create a `RetentionPolicy` custom resource which specifies how the old `Snapshots` should be cleaned up.

**Create RetentionPolicy:**

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
      from: Same
```

Let's create the `RetentionPolicy` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/pvc/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

We are ready to start taking backup. Now, we have to create a `BackupConfiguration` custom resource targeting `nfs-pvc`.

**Create BackupConfiguration:**

Below is the YAML of the `BackupConfiguration` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: pvc-backup
  namespace: demo
spec:
  target:
    apiGroup:
    kind: PersistentVolumeClaim
    name:  target-pvc
    namespace: demo
  backends:
    - name: gcs-backend
      storageRef:
        namespace: demo
        name: gcs-storage
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
        - name: gcs-repository
          backend: gcs-backend
          directory: /pvc-backup-demo
          encryptionSecret:
            name: encryption-secret
            namespace: demo
          deletionPolicy: WipeOut
      addon:
        name: pvc-addon
        tasks:
          - name: logical-backup
```

Here,
- `spec.target` specifies the backup target. `apiGroup`, `kind`, `namespace` and `name` refers to the `apiGroup`, `kind`, `namespace` and `name` of the targeted workload respectively.
- `spec.sessions[*].scheduler.schedule` specifies a [cron expression](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/#schedule) indicates that `BackupSession` will be created at 5 minute interval.
- `spec.sessions[*].repositories[*].name` specifies the name of the `Repository` object that holds the backend information.
- `spec.sessions[*].repositories[*].backend` specifies the name of the backend that holds the `BackupStorage` information.
- `spec.sessions[*].repositories[*].directory` specifies the path of the `Repository` where the backed up data will be stored.
- `spec.sessions[*].repositories[*].encryptionSecret` specifies the encryption secret for `Restic Repository` which will be used to encrypting the backup data.
- `spec.sessions[*].addon.name` specifies the name of the `Addon` object that specifies addon configuration that will be used to perform backup of a stand-alone PVC.
- `spec.sessions[*].addon.tasks[*].name` specifies the name of the `Task` that holds the `Function` and their order of execution to perform backup of a stand-alone PVC.

Let's create the `BackupConfiguration` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/pvc/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/pvc-backup created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be in `Ready` state. The `Ready` phase indicates that the backup setup is successful. Let's check the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME             PHASE   PAUSED   AGE
pvc-backup       Ready            19s
```

**Verify Repository:**

Verify that KubeStash has created `Repositories` that holds the `BackupStorage` information by the following command,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE   PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository                                       Ready                            28s
```

**Verify CronJob:**

Verify that KubeStash has created a CronJob to trigger a periodic backup of the targeted PVC by the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                     SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-pvc-backup-frequent-backup       */5 * * * *   False     0        5s              40s
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` crd using the following command,

```bash
$ watch -n 1 kubectl get backupsession -n demo -l=kubestash.com/invoker-name=pvc-backup

Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-name=pvc-backup                                                                                           workstation: Wed Jan  3 17:26:00 2024

NAME                                        INVOKER-TYPE          INVOKER-NAME     PHASE       DURATION   AGE
pvc-backup-frequent-backup-1704281100       BackupConfiguration   nfs-pvc-backup   Succeeded              60s

```

**Verify Backup:**

When backup session is created, kubestash operator creates `Snapshot` which represents the state of backup run for each `Repository` which are provided in `BackupConfiguration`.

Run the following command to check `Snapshot` phase,

```bash
$ kubectl get snapshots -n demo
NAME                                                       REPOSITORY       SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       VERIFICATION-STATUS   AGE
gcs-repository-pvc-backup-frequent-backup-1704281100       gcs-repository   frequent-backup   2024-01-03T11:25:13Z   Delete            Succeeded                         2m14s
```

When backup session is completed, KubeStash will update the respective `Repository` to reflect the latest state of backed up data.

Run the following command to check if a backup snapshot has been stored in the backend,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository   true        1                2.262 KiB   Ready   103s                     8m

```

From the output above, we can see that `1` snapshot has been stored in the backend specified by Repository `gcs-repository`.

## Populate Volume

This section will show you through the process of populating a volume by restoring the previously backed-up data using KubeStash.

**Create a Stand-alone PVC:**

Now, we are going to create a stand-alone `PVC` with `spec.dataSourceRef` set and refers our `snapshot` object.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: populate-pv
  namespace: demo
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  dataSourceRef:
    apiGroup: storage.kubestash.com
    kind: Snapshot
    name: gcs-repository-nfs-pvc-backup-frequent-backup-1704281100
```
Here,
  - `spec.dataSourceRef` specifies that which `snapshot` we want to restore into our populate volume.
  - we are referencing a `snapshot` object that we have backed up in the previous section.

Let's create the `PVC` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/pvc/examples/restoresession.yaml
restoresession.core.kubestash.com/populate-pv created
```

**Wait for Populate Volume:**

Now, wait for the volume populate process to complete. You can watch the `PVC` status using the following command,

```bash
$ watch -n 1 kubectl get pvc -n demo populate-pv

Every 1.0s: kubectl get pvc -n demo populate-pv                                workstation: Wed Jan  3 17:30:20 2024

NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
populate-pv         Bound    pvc-8d509c47-c26e-4039-8c03-49b25d126429   2Gi        RWO            standard       5d1
```
From the output of the above command, we can see that `PVC` status is `Bound` that means populate volume has been completed successfully.

**Verify Restored Data :**

We are going to create a new pod with mounting our previously created PVC `populate-pv` to verify whether the volume population with the backed up data has been restored successfully.

Below, the YAML for the Pod we are going to create.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restored-pod
  namespace: demo
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "3600"
    volumeMounts:
    - name: restore-data
      mountPath: /restore/data
  volumes:
  - name: restore-data
    persistentVolumeClaim:
      claimName: populate-pv
      readOnly: false
```
Let's create the Pod we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/pvc/examples/restored-pod.yaml
pod/restored-pod created
```

Now, wait for the Pod to go into the `Running` state.

```bash
$ kubectl get pod -n demo 
NAME           READY   STATUS    RESTARTS   AGE
restored-pod   1/1     Running   0          25s
```

Verify that the backed up data has been restored in `/restore/data` directory for `restored-pod` pod using the following command,

```bash
$ kubectl exec -n demo restored-pod -- cat /restore/data/hello.txt
hello from pod.
```


## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete backupconfiguration -n demo pvc-backup
kubectl delete backupstorage -n demo gcs-storage
kubectl delete retentionPolicy -n demo demo-retention
kubectl delete secret -n demo gcs-secret
kubectl delete secret -n demo encry-secret
kubectl delete pod -n demo demo-pod
kubectl delete -n demo pod restored-pod
kubectl delete pvc -n demo target-pvc
kubectl delete pvc -n demo populate-pv
```
