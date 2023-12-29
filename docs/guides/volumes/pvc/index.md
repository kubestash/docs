---
title: Backup Stand-alone PVC | KubeStash
description: A step by step guide on how to backup a stand-alone PVC using KubeStash.
menu:
  docs_{{ .version }}:
    identifier: volume-backup-pvc
    name: Backup & Restore a Stand-alone PVC
    parent: volume-backup
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup Stand-alone PVC using KubeStash

This guide will show you how to backup a stand-alone PersistentVolumeClaim (PVC) using KubeStash. Here, we are going to backup a PVC provisioned using an NFS server into a GCS bucket.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).

- You will need to have a PVC with `ReadWriteMany` access mode. Here, we are going to use an NFS server to provision a PVC with `ReadWriteMany` access mode. If you don't have an NFS server running, deploy one by following the guide [here](https://github.com/kubernetes-csi/csi-driver-nfs).

- You should be familiar with the following `KubeStash` concepts:
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/volumes/pvc/examples](/docs/guides/volumes/pvc/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

**Verify necessary Addons and Functions:**

When you install KubeStash, it automatically creates the necessary `Addon` and `Function` to backup a stand-alone volume.

Let's verify that KubeStash has created the necessary `Function` to backup/restore PVC by the following command,

```bash
$ kubectl get functions
NAME                     AGE
pvc-backup               3m25s
pvc-restore              3m25s
volumesnapshot-backup    3m25s
volumesnapshot-restore   3m25s
```

Also, verify that the necessary `Addon` has been created,

```bash
$ kubectl get addons
NAME          AGE
pvc-addon      3m25s
```

Now, you can view all `BackupTasks` and `RestoreTasks` of the `pvc-addon` addon using the following command

```bash
$ kubectl get addon pvc-addon -o=jsonpath='{.spec.backupTasks[*].name}'| tr ' ' '\n'
logical-backup
volume-snapshot
```

```bash
$ kubectl get addon pvc-addon -o=jsonpath='{.spec.restoreTasks[*].name}'| tr ' ' '\n'
logical-backup-restore
volume-clone
volume-snapshot-restore
```


## Prepare Volume

At first, let's prepare our desired PVC. Here, we are going to create a [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) (PV) that will use an NFS server as storage. Then, we are going to create a PVC that will bind with the PV. Then, we are going to mount this PVC in two different pods. Each pod will generate a sample file into the PVC.

**Create PersistentVolume:**

We have deployed an NFS server in `storage` namespace and it is accessible through a Service named `nfs-service`. Now, we are going to create a PV that uses the NFS server as storage.

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

Notice the `spec.csi` section. Here, we have added `csi` driver information which represents that this storage is managed by an external CSI volume driver.

Let's create the PV we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/pvc/examples/pv.yaml
persistentvolume/nfs-pv created
```

**Create PersistentVolumeClaim:**

Now, create a PVC to bind with the PV we have just created. Below, is the YAML of the PVC that we are going to create,

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-pvc
  namespace: demo
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 1Gi
  volumeName: nfs-pv
```

Notice the `spec.accessModes` section. We are using `ReadWriteMany` access mode so that multiple pods can use this PVC simultaneously. Without this access mode, KubeStash will fail to backup the volume if any other pod mount it during backup.

Also, notice the `spec.volumeName` section. We have specified `nfs-pv` as the PV that we have created earlier, which will be claimed by above PVC.

Let's create the PVC we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/pvc/examples/pvc.yaml
persistentvolumeclaim/nfs-pvc created
```

Verify that the PVC has bounded with our desired PV,

```bash
$ kubectl get pvc -n demo nfs-pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc   Bound    nfs-pv   1Gi        RWX                           32s
```

Here, we can see that the PVC `nfs-pvc` has been bounded with PV `nfs-pv`.

**Deploy Workload:**

Now, we are going to deploy two sample pods `demo-pod-1` and `demo-pod-2` that will mount `pod-1/data` and `pod-2/data` [subPath](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath) of the `nfs-pvc` respectively. Each of the pods will generate a sample file named `hello.txt` with some demo data. We are going to backup the entire PVC using KubeStash that contains the sample files.

Below, is the YAML of the first pod that we are going to deploy,

```yaml
kind: Pod
apiVersion: v1
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/pvc/examples/pod-1.yaml
pod/demo-pod-1 created
```

Verify that the sample data has been generated into `/sample/data/` directory,

```bash
$ kubectl exec -n demo demo-pod-1 -- cat /sample/data/hello.txt
hello from pod 1.
```

Below is the YAML of the second pod that we are going to deploy,

```yaml
kind: Pod
apiVersion: v1
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/pvc/examples/pod-2.yaml
pod/demo-pod-2 created
```

Verify that the sample data has been generated into `/sample/data/` directory,

```bash
$ kubectl exec -n demo demo-pod-2 -- cat /sample/data/hello.txt
hello from pod 2.
```

## Backup

Now, we are going to backup the PVC `nfs-pvc` in a GCS bucket using KubeStash. We have to create a Secret and a `BackupStorage` object with access credentials and backend information respectively.

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
      secretName: gcs-secret
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true 
  deletionPolicy: WipeOut
```

Let's create the `BackupStorage` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/pvc/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-repo created
```

Now, we also have to create another secret with an encryption key `RESTIC_PASSWORD` for Restic. This secret will be used by Restic for both encrypting and decrypting the backup data during backup & restore.

**Create Restic Repository Secret:**

Let's create a Secret named `encry-secret` with an encryption key `RESTIC_PASSWORD` for Restic.

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encryption-secret \
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/pvc/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

We are ready to start taking backup. Now, we have to create a `BackupConfiguration` custom resource targeting `nfs-pvc`.

**Create BackupConfiguration:**

Below is the YAML of the `BackupConfiguration` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: nfs-pvc-backup
  namespace: demo
spec:
  target:
    apiGroup:
    kind: PersistentVolumeClaim
    name:  nfs-pvc
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
            name: encry-secret
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
backupconfiguration.core.kubestash.com/nfs-pvc-backup created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be in `Ready` state. The `Ready` phase indicates that the backup setup is successful. Let's check the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME             PHASE   PAUSED   AGE
nfs-pvc-backup   Ready            19s
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
trigger-nfs-pvc-backup-frequent-backup   */5 * * * *   False     0        5s              40s
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` crd using the following command,

```bash
$ watch -n 1 kubectl get backupsession -n demo -l=kubestash.com/invoker-name=nfs-pvc-backup

Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-name=nfs-pvc-backup                                                                                           workstation: Wed Jan  3 17:26:00 2024

NAME                                        INVOKER-TYPE          INVOKER-NAME     PHASE       DURATION   AGE
nfs-pvc-backup-frequent-backup-1704281100   BackupConfiguration   nfs-pvc-backup   Succeeded              60s

```

**Verify Backup:**

When backup session is created, kubestash operator creates `Snapshot` which represents the state of backup run for each `Repository` which are provided in `BackupConfiguration`.

Run the following command to check `Snapshot` phase,

```bash
$ kubectl get snapshots -n demo
NAME                                                       REPOSITORY       SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       VERIFICATION-STATUS   AGE
gcs-repository-nfs-pvc-backup-frequent-backup-1704281100   gcs-repository   frequent-backup   2024-01-03T11:25:13Z   Delete            Succeeded                         2m14s
```

When backup session is completed, KubeStash will update the respective `Repository` to reflect the latest state of backed up data.

Run the following command to check if a backup snapshot has been stored in the backend,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository   true        1                2.262 KiB   Ready   103s                     8m

```

From the output above, we can see that 1 snapshot has been stored in the backend specified by Repository `gcs-repository`.

If we navigate to `kubestash-qa/demo/pvc-backup-demo/repository/v1/frequent-backup/pvc` directory of our GCS bucket, we are going to see that the snapshot has been stored there.

<figure align="center">
  <img alt="Backed up data of a stand-alone PVC in GCS backend" src="images/pvc_repo.png">
  <figcaption align="center">Fig: Backed up data of a stand-alone PVC in GCS backend</figcaption>
</figure>

> KubeStash keeps all backup data encrypted. So, snapshot files in the bucket will not contain any meaningful data until they are decrypted.

## Restore

This section will show you how to restore the backed up data inside a stand-alone PVC using KubeStash. Here, we are going to restore the data we have backed up in the previous section.

**Simulate Disaster:**

At first, let's simulate a disaster scenario. Let's delete all the files from the PVC.

Delete the data of pod `demo-pod-1`:

```bash
# delete data
$ kubectl exec -n demo demo-pod-1 -- sh -c "rm /sample/data/*"

# verify that data has been removed successfully
$ kubectl exec -n demo demo-pod-1 -- ls /sample/data/
# empty output which means all the files have been deleted
```

Delete the data of pod `demo-pod-2`:

```bash
# delete data
$ kubectl exec -n demo demo-pod-2 -- sh -c "rm /sample/data/*"

# verify that data has been removed successfully
$ kubectl exec -n demo demo-pod-2 -- ls /sample/data/
# empty output which means all the files have been deleted
```

**Create RestoreSession:**

Now, we are going to create a `RestoreSession` object to restore the backed up data into the desired PVC. 
Below is the YAML of the `RestoreSession` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: nfs-pvc-restore
  namespace: demo
spec:
  target:
    apiGroup:
    kind: PersistentVolumeClaim
    name: nfs-pvc
    namespace: demo
  dataSource:
    repository: gcs-repository
    snapshot: latest
    encryptionSecret:
      name: encryption-secret
      namespace: demo
  addon:
    name: pvc-addon
    tasks:
      - name: logical-backup-restore
```

- `spec.target` refers to the targeted PVC where the data will be restored.
- `spec.dataSource.repository` specifies the name of the `Repository` from which the data will be restored.
- `spec.dataSource.snapshot` specifies that we want to restore the latest snapshot of the `gcs-repository`.
- `spec.dataSource.encryptionSecret` specifies the encryption secret for `Restic Repository` used during backup. It will be used to decrypting the backup data.
- `spec.addon.name` specifies the name of the `Addon` object that specifies addon configuration that will be used to perform restore of a stand-alone PVC.
- `spec.addon.tasks[*].name` specifies the name of the `Task` that holds the Function and their order of execution to perform restore of a stand-alone PVC.

Let's create the `RestoreSession` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/pvc/examples/restoresession.yaml
restoresession.core.kubestash.com/nfs-pvc-restore created
```

**Wait for RestoreSession to Succeed:**

Now, wait for the restore process to complete. You can watch the `RestoreSession` phase using the following command,

```bash
$ watch -n 1 kubectl get restoresession -n demo nfs-pvc-restore

Every 1.0s: kubectl get restoresession -n demo nfs-pvc...  workstation: Wed Jan  3 17:30:20 2024

NAME              REPOSITORY       FAILURE-POLICY   PHASE       DURATION   AGE
nfs-pvc-restore   gcs-repository                    Succeeded   10s        51s
```

From the output of the above command, we can see that restoration process has been completed successfully.

**Verify Restored Data:**

Let's verify if the deleted files have been restored successfully into the PVC. We are going to exec into individual pod and check whether the sample data exist or not.

Verify that the data of `demo-pod-1` has been restored:

```bash
$ kubectl exec -n demo demo-pod-1 -- cat /sample/data/hello.txt
hello from pod 1.
```

Verify that the data of `demo-pod-2` has been restored:

```bash
$ kubectl exec -n demo demo-pod-2 -- cat /sample/data/hello.txt
hello from pod 2.
```

So, we can see from the above output that the files we had deleted in **Simulate Disaster** section have been restored successfully.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash

kubectl delete backupconfiguration -n demo nfs-pvc-backup
kubectl delete restoresession -n demo nfs-pvc-restore

kubectl delete backupstorage -n demo gcs-storage
kubectl delete retentionPolicy -n demo demo-retention

kubectl delete secret -n demo gcs-secret
kubectl delete secret -n demo encryption-secret

kubectl delete pod -n demo demo-pod-1
kubectl delete pod -n demo demo-pod-2

kubectl delete pvc -n demo nfs-pvc
kubectl delete pv -n demo nfs-pv
```

If you would like to uninstall KubeStash operator, please follow the steps [here](/docs/setup/README.md).
