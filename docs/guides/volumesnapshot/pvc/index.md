---
title: Snapshot Stand-alone PVC | KubeStash
description: An step by step guide showing how to snapshot a stand-alone PVC
menu:
  docs_{{ .version }}:
    identifier: volume-snapshot-pvc
    name: Snapshot Stand-alone PVC
    parent: volume-snapshot
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Snapshotting a Standalone PVC

This guide will show you how to use KubeStash to snapshot of a standalone PersistentVolumeClaims (PVC) and restore it from the volumeSnapshot using Kubernetes [VolumeSnapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) API. In this guide, we are going to backup the volumes in Google Cloud Platform with the help of [GCE Persistent Disk CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver).

## Before You Begin

- At first, you need to be familiar with the [GCE Persistent Disk CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver).
- If you don't know how VolumeSnapshot works in KubeStash, please visit [here](/docs/guides/volumesnapshot/overview/index.md).
- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).
- You should be familiar with the following `KubeStash` concepts:
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [Snapshot](/docs/concepts/crds/snapshot/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)

## Prepare for VolumeSnapshot

Here, we are going to create `StorageClass` that uses [GCE Persistent Disk CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver).

Below is the YAML of the  `StorageClass` we are going to use,

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-standard
parameters:
  type: pd-standard
provisioner: pd.csi.storage.gke.io
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

Let's create the `StorageClass` we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/pvc/examples/storageclass.yaml
storageclass.storage.k8s.io/csi-standard created
```

We also need a `VolumeSnapshotClass`. Below is the YAML of the `VolumeSnapshotClass` we are going to use,

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapshot-class
driver: pd.csi.storage.gke.io
deletionPolicy: Delete
```

Here,

- `driver` specifies the respective CSI driver that is responsible for taking snapshot. As we are using [GCE Persistent Disk CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver), we are going to use `pd.csi.storage.gke.io` in this field.

Let's create the `VolumeSnapshotClass` we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/pvc/examples/volumesnapshotclass.yaml
volumesnapshotclass.snapshot.storage.k8s.io/csi-snapshot-class created
```

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> Note: YAML files used in this tutorial are stored in [/docs/guides/volumesnapshot/pvc/examples](/docs/guides/volumesnapshot/pvc/examples/) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.


## Prepare Volume

At first, let's prepare our desired `PVC`. Here, we are going to create a `PVC`. Then, we are going to mount this `PVC` with a `Pod`. `Pod` will generate a sample file into that `PVC`.

**Create PersistentVolumeClaim:**

Below, is the YAML of the PVC that we are going to create,

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: source-pvc
  namespace: demo
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: csi-standard
  resources:
    requests:
      storage: 1Gi
```

Let's create the `PVC` we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/pvc/examples/source-pvc.yaml
persistentvolumeclaim/source-pvc created
```

**Deploy Workload:**

Now, we are going to deploy a sample pod `source-pod` that will mount `/sample/data` path of the `source-pvc`. This pod will generate a sample file named `data.txt` with some demo data inside that file.

Below is the YAML of the `Pod` that we are going to create,

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: source-pod
  namespace: demo
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["/bin/sh", "-c"]
    args: ["echo sample_data > /source/data/data.txt && sleep 3000"]
    volumeMounts:
    - name: source-data
      mountPath: /source/data
  volumes:
  - name: source-data
    persistentVolumeClaim:
      claimName: source-pvc
      readOnly: false
```

Let's create the `Pod` we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/pvc/examples/source-pod.yaml
pod/source-pod created
```

Now, wait for the `Pod` to go into the `Running` state.

```bash
$ kubectl get pod -n demo
NAME         READY   STATUS    RESTARTS   AGE
source-pod   1/1     Running   0          11s
```

Verify that the sample data has been created in `/source/data` directory for `source-pod` pod using the following command,

```bash
$ kubectl exec -n demo source-pod -- cat /source/data/data.txt
sample_data
```

## Prepare Backend

Now, we are going to take `VolumeSnapshot` of the PVC `source-pvc` using KubeStash. For this, we have to create a `Secret` with  necessary credentials and a `BackupStorage` object. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

> For GCS backend, if the bucket does not exist, KubeStash needs `Storage Object Admin` role permissions to create the bucket. For more details, please check the following [guide](/docs/guides/backends/gcs/index.md).

**Create Secret:**

Let's create a `Secret` named `gcs-secret` with access credentials of our desired GCS backend,

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
backupstorage.storage.kubestash.com/gcs-storage created
```

Now, we are ready to backup our target volume into this backend.

**Create RetentionPolicy:**

Now, we have to create a `RetentionPolicy` object to specify how the old `Snapshots` should be cleaned up.

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

Notice the `spec.usagePolicy` that allows referencing the `RetentionPolicy` from all namespaces.For more details on configuring it for specific namespaces, please refer to the following [RetentionPolicy usage policy](/docs/concepts/crds/retentionpolicy/index.md).

Let's create the `RetentionPolicy` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/pvc/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

## Backup

Now, we have to create a `BackupConfiguration` custom resource targeting the PVC that we have created earlier.

**Create BackupConfiguration :**

Below is the YAML of the `BackupConfiguration` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: pvc-volume-snapshot
  namespace: demo
spec:
  target:
    apiGroup:
    kind: PersistentVolumeClaim
    name:  source-pvc
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
          directory: /pvc-volume-snapshot-repo
          deletionPolicy: WipeOut
      addon:
        name: pvc-addon
        tasks:
          - name: volume-snapshot
            params:
              volumeSnapshotClassName: csi-snapshot-class
```
Here,
- `spec.sessions[*].addon.tasks[*].params.volumeSnapshotClassName` indicates the [VolumeSnapshotClass](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/) to be used for volume snapshotting. it should match with the `VolumeSnapshotClass` we created earlier. If we don't provide any then KubeStash use default `volumeSnapshotClass` for volume snapshotting.

Let's create the `BackupConfiguration` object we that have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/pvc/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/pvc-volume-snapshot created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be in `Ready` state. The `Ready` phase indicates that the backup setup is successful.

Let's check the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME                  PHASE   PAUSED   AGE
pvc-volume-snapshot   Ready            17s
```

**Verify Repository:**

Verify that the Repository specified in the BackupConfiguration has been created using the following command,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE   PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository                                       Ready                            28s
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the Repository YAML stored in the `kubestash-qa/demo/pvc-volume-snapshot-repo` directory.

**Verify CronJob:**

Verify that KubeStash has created a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` object.

```bash
$ kubectl get cronjob -n demo
NAME                                          SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-pvc-volume-snapshot-frequent-backup   */5 * * * *   False     0        <none>          50s
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` CR using the following command,

```bash
$ watch -n 1 kubectl get backupsession -n demo -l=kubestash.com/invoker-name=pvc-volume-snapshot

Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-name=pvc-volume-snapshot                                                         anisur: Tue Jan 16 17:01:06 2024

NAME                                             INVOKER-TYPE          INVOKER-NAME          PHASE       DURATION   AGE
pvc-volume-snapshot-frequent-backup-1705402801   BackupConfiguration   pvc-volume-snapshot   Succeeded              66s
```

Here, the phase `Succeeded` means that the backup process has been completed successfully.

**Verify Backup:**

Once a backup is complete, KubeStash will update the respective `Repository` object to reflect the backup. Check that the repository `gcs-repository` has been updated by the following command,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository   true        1                2.262 KiB   Ready   103s                     72s

```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot`.

Verify created `Snapshot` object by the following command,

```bash
$ watch kubectl get snapshots.storage.kubestash.com -n demo -l=kubestash.com/repo-name=gcs-repository

Every 2.0s: kubectl get snapshots.storage.kubestash.com -n demo -l=kubestash.com/repo-name=gcs-repository                                   workstation: Tue Jan 16 17:03:06 2024

NAME                                                            REPOSITORY       SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       VERIFICATION-STATUS   AGE
gcs-repository-pvc-volume-snapshot-frequent-backup-1705402801   gcs-repository   frequent-backup   2024-01-16T11:00:08Z   Delete            Succeeded                         3m6s
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <target-kind>`
> - `kubestash.com/app-ref-name: <target-name>`
> - `kubestash.com/app-ref-namespace: <target-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
>
> These labels can be used to watch only the `Snapshot`s related to our desired Workload or `Repository`.

Now, lets retrieve the YAML for the `Snapshot`, and inspect the `spec.status` section to see the backup information of the targeted PVC.

```bash
$ kubectl get snapshots  -n demo gcs-repository-pvc-volume-snapshot-frequent-backup-1705402801 -o yaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  name: gcs-repository-pvc-volume-snapshot-frequent-backup-1705402801
  namespace: demo
spec:
  ---
status:
  components:
    volumesnapshot:
      driver: VolumeSnapshotter
      phase: Succeeded
      volumeSnapshotterStats:
        - pvcName: source-pvc
          volumeSnapshotName: source-pvc-1705402801
  conditions:
    ---
  phase: Succeeded
  snapshotTime: "2024-01-16T11:00:08Z"
  totalComponents: 1
```

Here,
> For volume snapshot backup KubeStash uses `volumesnapshot` as the component name for Standalone PVC.

- `volumeSnapshotterStats.pvcName` specifies the name of the targeted `PVC`.
- `volumeSnapshotterStats.hostPath` specifies the mount path for the targeted `PVC` within the workload.
- `volumeSnapshotterStats.volumeSnapshotName` specifies the name of the `VolumeSnapshot` created for the targeted PVC.

KubeStash keeps the backup for Snapshot YAMLs. If we navigate to the GCS bucket, we will see the Snapshot YAML stored in the `<kubestash-qa/demo/pvc-volume-snapshot-repo/snapshots>` directory.

**Verify Volume Snapshot :**

Once a `BackupSession` CR is created, KubeStash operator creates a volume snapshotter `Job`. Then the `Job` creates a `VolumeSnapshot` CR for the targeted PVC.

Run the following command to check that the `VolumeSnapshot` has been created Successfully.

```bash
$ kubectl get volumesnapshot -n demo
NAME                    READYTOUSE   SOURCEPVC    SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS        SNAPSHOTCONTENT                                    CREATIONTIME   AGE
source-pvc-1705402801   true         source-pvc                           1Gi           csi-snapshot-class   snapcontent-b5692a5b-8834-48dc-9185-56ed9a2fa124   7m20s          7m22s
```

Let's find out the `VolumeSnapshotContent` that has been saved in the Google Cloud by the following command,

```bash
kubectl get volumesnapshot source-pvc-fnbwz  -n demo -o yaml
```

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  creationTimestamp: "2024-01-16T11:00:08Z"
  finalizers:
    - snapshot.storage.kubernetes.io/volumesnapshot-as-source-protection
    - snapshot.storage.kubernetes.io/volumesnapshot-bound-protection
  generation: 1
  name: source-pvc-1705402801
  namespace: demo
  resourceVersion: "11593"
  uid: b5692a5b-8834-48dc-9185-56ed9a2fa124
spec:
  source:
    persistentVolumeClaimName: source-pvc
  volumeSnapshotClassName: csi-snapshot-class
status:
  boundVolumeSnapshotContentName: snapcontent-b5692a5b-8834-48dc-9185-56ed9a2fa124
  creationTime: "2024-01-16T11:00:10Z"
  readyToUse: true
  restoreSize: 1Gi
```

Here, `spec.status.boundVolumeSnapshotContentName` field specifies the name of the `VolumeSnapshotContent` object. It also represents the actual snapshot name that has been saved in Google Cloud. If we navigate to the `Snapshots` tab in the GCP console, we are going to see snapshot `snapcontent-1fa0a06c-80bb-4e7c-b584-579ddceb649d` has been stored successfully.

<figure align="center">
  <img alt="Snapshots in GCP console" src="/docs/guides/volumesnapshot/pvc/images/gcp.png">
<figcaption align="center">Fig: Snapshots in GCP </figcaption>
</figure>

## Restore

This section will show you how to restore the PVC from the `VolumeSnapshot` we have taken in the earlier section.

**Create RestoreSession :**

Now, we are going to create a `RestoreSession` object to restore the `PVC` from respective `VolumeSnapshot`. Below is the YAML of the `RestoreSession` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: restore-pvc
  namespace: demo
spec:
  dataSource:
    repository: gcs-repository
    snapshot: latest
  addon:
    name: pvc-addon
    tasks:
      - name: VolumeSnapshotRestore
        targetVolumes:
          volumeClaimTemplates:
            - metadata:
                name: restore-source-pvc
              spec:
                accessModes: [ "ReadWriteOnce" ]
                storageClassName: "csi-standard"
                resources:
                  requests:
                    storage: 1Gi
```

Here,
- `spec.dataSource.repository` specifies name of the `Repository` from which the data will be restored.
- `spec.dataSource.snapshot` specifies the name of the `Snapshot` that will be restored.
- `spec.addon.targetVolumes.volumeClaimTemplates[*]`:
  - `metadata.name` is a template for the name of the restored `PVC`. KubeStash will create `PVC` with the specified name. 

Let's create the `RestoreSession` object we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/pvc/examples/restoresession.yaml
restoresession.core.kubestash.com/restore-pvc created
```

Once, you have created the `RestoreSession` object, KubeStash will create a job to restore. We can watch the `RestoreSession` phase to check if the restore process has succeeded or not.

Run the following command to watch RestoreSession phase,

```bash
$ watch -n 1 kubectl get restore -n demo

Every 1.0s: kubectl get restore -n demo                       anisur: Tue Jan 16 17:22:34 2024

NAME          REPOSITORY       FAILURE-POLICY   PHASE       DURATION   AGE
restore-pvc   gcs-repository                    Succeeded   25s        28s
```

So, we can see from the output of the above command that the restore process succeeded.

**Verify Restored PVC :**

Once the restore process is complete, we are going to see that new `PVC` with the name `restore-source-pvc` has been created.

To verify that the `PVC` has been created, run by the following command,
 
```bash
$ kubectl get pvc -n demo
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
restore-source-pvc   Bound    pvc-60890df8-ff1c-448c-9310-14d8ed9c06f8   1Gi        RWO            csi-standard   69s
```

Notice the `STATUS` field. It indicates that the respective `PV` has been provisioned and initialized from the respective `VolumeSnapshot` by CSI driver and the `PVC` has been bound with the `PV`.

>The [volumeBindingMode](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode) field controls when volume binding and dynamic provisioning should occur. Kubernetes allows `Immediate` and `WaitForFirstConsumer` modes for binding volumes. The `Immediate` mode indicates that volume binding and dynamic provisioning occurs once the PVC is created and `WaitForFirstConsumer` mode indicates that volume binding and provisioning does not occur until a pod is created that uses this PVC. By default `volumeBindingMode` is `Immediate`.

**Verify Restored Data :**

We are going to create a new `Pod` mount with the restored `PVC` to verify whether the backed up data has been successfully restored.

Below, the YAML for the pod we are going to create.

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
      claimName: restore-source-pvc
      readOnly: false
```
Let's create the pod we have shown above.

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
$ kubectl exec -n demo restored-pod -- cat /restore/data/data.txt
sample_data
```

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo pod source-pod
kubectl delete -n demo pod restored-pod
kubectl delete -n demo backupconfiguration pvc-volume-snapshot
kubectl delete -n demo restoresession restore-pvc
kubectl delete -n demo retentionpolicy demo-retention
kubectl delete -n demo backupstorage gcs-storage
kubectl delete -n demo pvc --all
kubectl delete -n demo volumesnapshots --all
kubectl delete -n demo storageclass csi-standard
kubectl delete -n demo volumesnapshotclass csi-snapshot-class
```
