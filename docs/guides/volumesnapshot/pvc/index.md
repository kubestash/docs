---
title: Snapshot Stand-alone PVC | KubeStash
description: An step by step guide showing how to snapshot a stand-alone PVC
menu:
  docs_{{ .version }}:
    identifier: volume-snapshot-pvc
    name: Snapshot Stand-alone PVC
    parent: volume-snapshot
    weight: 40
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Snapshotting a Standalone PVC

This guide will show you how to use KubeStash to take volumeSnapshot of a standalone PersistentVolumeClaims and restore it from the volumeSnapshot using Kubernetes [VolumeSnapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) API. In this guide, we are going to backup the volumes in Google Cloud Platform with the help of [GCE Persistent Disk CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver).

## Before You Begin

- At first, you need to be familiar with the [GCE Persistent Disk CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver).
- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).
- If you don't know how VolumeSnapshot works in KubeStash, please visit [here](/docs/guides/volumesnapshot/overview/index.md).

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

- `driver` field to point to the respective CSI driver that is responsible for taking snapshot. As we are using [GCE Persistent Disk CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver), we are going to use `pd.csi.storage.gke.io` in this field.

Let's create the `volumeSnapshotClass` we have shown above,

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


## Take Volume Snapshot

Here, we are going to create a PVC and mount it with a pod and we are going to also generate some sample data on it. Then, we are going to take volumeSnapshot of this PVC using KubeStash.

**Create PersistentVolumeClaim:**

At first, let's create a PVC. We are going to mount this PVC in a pod.

Below is the YAML of the sample PVC,

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

Let's create the PVC we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/pvc/examples/source-pvc.yaml
persistentvolumeclaim/source-pvc created
```

**Create Pod :**

Now, we are going to deploy a pod that uses the above PVC. This pod will automatically create `data.txt` file in `/source/data`  directory and write some sample data in it and also mounted the desired PVC in `/source/data`  directory.

Below is the YAML of the pod that we are going to create,

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

Let's create the Pod we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/pvc/examples/source-pod.yaml
pod/source-pod created
```

Now, wait for the Pod to go into the `Running` state.

```bash
$ kubectl get pod -n demo
NAME         READY   STATUS    RESTARTS   AGE
source-pod   1/1     Running   0          11s
```

Verify that the sample data has been created in `/source/data` directory for `source-pod` pod
using the following command,

```bash
$ kubectl exec -n demo source-pod -- cat /source/data/data.txt
sample_data
```

## Backup

Now, we are going to taking `VolumeSnapshot` of the PVC `source-pvc` using KubeStash. We have to create a `Secret` and a `BackupStorage` object with access credentials and backend information respectively.

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
backupstorage.storage.kubestash.com/gcs-storage created
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

We are ready. Now, we have to create a `BackupConfiguration` custom resource targeting PVC `source-pvc`.

**Create BackupConfiguration :**

Now, create a `BackupConfiguration` custom resource to take `volumeSnapshot` of the `source-pvc` PVC.

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
- `spec.target` specifies the backup target. `apiGroup`, `kind`, `namespace` and `name` refers to the `apiGroup`, `kind`, `namespace` and `name` of the targeted workload respectively.
- `spec.sessions[*].scheduler.schedule` specifies a [cron expression](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/#schedule) indicates that `BackupSession` will be created at 5 minute interval.
- `spec.sessions[*].repositories[*].name` specifies the name of the `Repository` object that holds the backend information.
- `spec.sessions[*].repositories[*].backend` specifies the name of the backend that holds the `BackupStorage` information.
- `spec.sessions[*].repositories[*].directory` specifies the path of the `Repository` where the backed up data will be stored. 
- `spec.sessions[*].addon.name` specifies the name of the `Addon` object that specifies addon configuration that will be used to perform backup of a stand-alone PVC.
- `spec.sessions[*].addon.tasks[*].name` specifies the name of the `Task` that holds the `Function` and their order of execution to perform backup of a stand-alone PVC.
- `spec.sessions[*].addon.tasks[*].params.volumeSnapshotClassName` indicates the [VolumeSnapshotClass](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/) to be used for volume snapshotting. If we don't provide any then kubestash use default `volumeSnapshotClass` for volume snapshotting.

Let's create the `BackupConfiguration` crd we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/pvc/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/pvc-volume-snapshot created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be in `Ready` state. The `Ready` phase indicates that the backup setup is successful. Let's check the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME                  PHASE   PAUSED   AGE
pvc-volume-snapshot   Ready            17s
```
**Verify Repository:**

Verify that KubeStash has created `Repositories` that holds the `BackupStorage` information by the following command,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE   PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository                                       Ready                            35s
```

**Verify CronJob:**

Verify that KubeStash has created a CronJob to trigger a periodic backup of the targeted PVC by the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                          SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-pvc-volume-snapshot-frequent-backup   */5 * * * *   False     0        <none>          50s
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` object using the following command,

```bash
$ watch -n 1 kubectl get backupsession -n demo -l=kubestash.com/invoker-name=pvc-volume-snapshot

Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-name=pvc-volume-snapshot                                                         anisur: Tue Jan 16 17:01:06 2024

NAME                                             INVOKER-TYPE          INVOKER-NAME          PHASE       DURATION   AGE
pvc-volume-snapshot-frequent-backup-1705402801   BackupConfiguration   pvc-volume-snapshot   Succeeded              66s
```

**Verify Snapshot:**

Once a `BackupSession` object is created, it creates a `Snapshot` custom resource for each `Repository`. This snapshot `spec.status.components` section represents the backup information of the targeted backup components.

Verify created `Snapshot` object by the following command,

```bash
$ watch kubectl get snapshots.storage.kubestash.com -n demo -l=kubestash.com/repo-name=gcs-repository

Every 2.0s: kubectl get snapshots.storage.kubestash.com -n demo -l=kubestash.com/repo-name=gcs-repository                                               anisur: Tue Jan 16 17:03:06 2024

NAME                                                            REPOSITORY       SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       VERIFICATION-STATUS   AGE
gcs-repository-pvc-volume-snapshot-frequent-backup-1705402801   gcs-repository   frequent-backup   2024-01-16T11:00:08Z   Delete            Succeeded                         3m6s
```

Now, retrieve the YAML representation of the above created `Snapshot`, and inspect the `spec.status` section to see the backup information of the targeted backup components.

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
- `spec.status.components` specifies the backup information of the backup target components.
- `spec.status.components.[<component-name>].volumeSnapshotterStats` specifies the information about the `VolumeSnapshotter` driver. In the case of a standalone `PVC`, the component name is designated as `volumesnapshot`.
  - `volumeSnapshotterStats.hostPath` specifies the corresponding path of `PVC` volume for which `volumeSnapshot` has created.
  - `volumeSnapshotterStats.pvcName` specifies the name of the backed-up target `PVC` volume.
  - `volumeSnapshotterStats.volumeSnapshotName` specifies the created `VolumeSnapshot` object name corresponding to the target `PVC` volume.

**Verify Volume Snapshot :**

Once a `BackupSession` object is created, it creates volume snapshotter `Job`. Then the `Job` creates `VolumeSnapshot` object for the targeted PVC. We can see above that the backup session has succeeded. Now, we are going to verify that the `VolumeSnapshot` has been created and the snapshot have been stored in the respective backend.

Check that the `VolumeSnapshot` has been created Successfully.

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

## Restore PVC from VolumeSnapshot

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

- `spec.dataSource.repository` specifies the name of the `Repository` from which the data will be restored.
- `spec.dataSource.snapshot` specifies that we want to restore the latest snapshot of the `gcs-repository`. KubeStash retrieves `VolumeSnapshot` information from the `status.components[*]` field of this snapshot.  
- `spec.addon.name` specifies the name of the `Addon` object that specifies addon configuration that will be used to perform restore of a stand-alone PVC.
- `spec.addon.tasks[*].name` specifies the name of the `Task` that holds the `Function` and their order of execution to perform restore of a stand-alone PVC.
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

>If you use `volumeBindingMode: WaitForFirstConsumer`, respective PVC will be initialized from respective VolumeSnapshot after you create a workload with that PVC. In this case, KubeStash will mark the restore session as completed with phase `Unknown`.

**Verify Restored Data :**

We are going to create a new pod with the restored `PVC` to verify whether the backed up data has been restored.

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
      claimName: restore-source-pvc
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
