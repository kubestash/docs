---
title: Snapshot Deployment Volumes | KubeStash
description: An step by step guide showing how to snapshot the volumes of a Deployment
menu:
  docs_{{ .version }}:
    identifier: volume-snapshot-deployment
    name: Snapshot Deployment Volumes
    parent: volume-snapshot
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Snapshotting the volumes of a Deployment

This guide will show you how to use KubeStash to snapshot the volumes of a `Deployment` and restore them from the snapshots using Kubernetes [VolumeSnapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) API. In this guide, we are going to backup the volumes in Google Cloud Platform with the help of [GCE Persistent Disk CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver).

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/deployment/examples/storageclass.yaml
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

Let's create the `volumeSnapshotClass` we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/deployment/examples/volumesnapshotclass.yaml
volumesnapshotclass.snapshot.storage.k8s.io/csi-snapshot-class created
```

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```
> Note: YAML files used in this tutorial are stored in [/docs/guides/volumesnapshot/pvc/examples](/docs/guides/volumesnapshot/statefulset/examples/) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Prepare Workload

Here, we are going to deploy a `Deployment` with two PVCs and generate some sample data in it.

**Create PersistentVolumeClaim :**

At first, let's create two sample PVCs. We are going to mount these PVCs in our targeted `Deployment`.

Below is the YAML of the sample PVCs,

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: source-data
  namespace: demo
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: csi-standard
  resources:
    requests:
      storage: 1Gi
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: source-config
  namespace: demo
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: csi-standard
  resources:
    requests:
      storage: 1Gi
```

Let's create the PVCs we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/deployment/examples/pvcs.yaml
persistentvolumeclaim/source-data created
persistentvolumeclaim/source-config created
```

**Deploy Deployment :**

Now, we are going to deploy a `Deployment` that uses the above PVCs. This `Deployment` will automatically create `data.txt` and `config.cfg` file in `/source/data` and `/source/config` directory.

Below is the YAML of the Deployment that we are going to create,

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubestash-demo
  name: kubestash-demo
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubestash-demo
  template:
    metadata:
      labels:
        app: kubestash-demo
      name: busybox
    spec:
      containers:
      - args: ["echo sample_data > /source/data/data.txt; echo sample_config > /source/config/config.cfg  && sleep 3000"]
        command: ["/bin/sh", "-c"]
        image: busybox
        imagePullPolicy: IfNotPresent
        name: busybox
        volumeMounts:
        - mountPath: /source/data
          name: source-data
        - mountPath: /source/config
          name: source-config
      restartPolicy: Always
      volumes:
      - name: source-data
        persistentVolumeClaim:
         claimName: source-data
      - name: source-config
        persistentVolumeClaim:
          claimName: source-config
```

Let's create the `Deployment` we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/deployment/examples/deployment.yaml
deployment.apps/kubestash-demo created
```

Now, wait for the pod of the `Deployment` to go into the `Running` state.

```bash
$ kubectl get pod -n demo
NAME                              READY   STATUS    RESTARTS   AGE
kubestash-demo-5f7fcd67c8-4z766   1/1     Running   0          79s
```

Verify that the sample data has been created in `/source/data` and `/source/config` directory using the following command,

```bash
$ kubectl exec -n demo kubestash-demo-5f7fcd67c8-4z766 -- cat /source/data/data.txt
sample_data
$ kubectl exec -n demo kubestash-demo-5f7fcd67c8-4z766 -- cat /source/config/config.cfg
config_data
```

## Prepare Backend

Now, we are going to take `VolumeSnapshot` of the Deployment `kubestash-deployment` PVCs using KubeStash. For this, we have to create a `Secret` with  necessary credentials and a `BackupStorage` object. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/statefulset/examples/backupstorage.yaml
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumes/statefulset/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

## Backup

Now, we have to create a `BackupConfiguration` custom resource targeting the Deployment that we have created earlier.

**Create BackupConfiguration :**

Below is the YAML of the `BackupConfiguration` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: deployment-volume-snapshot
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name:  kubestash-demo
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
          directory: /deployment-volume-snapshot-repo
          deletionPolicy: WipeOut
      addon:
        name: workload-addon
        tasks:
          - name: volume-snapshot
            params:
              volumeSnapshotClassName: csi-snapshot-class
```
Here,
- `spec.sessions[*].addon.tasks[*].params.volumeSnapshotClassName` indicates the [VolumeSnapshotClass](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/) to be used for volume snapshotting. it should match with the `VolumeSnapshotClass` we created earlier. If we don't provide any then KubeStash use default `volumeSnapshotClass` for volume snapshotting.

Let's create the `BackupConfiguration` object we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/deployment/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/deployment-volume-snapshot created
```

**Verify Backup Setup Successful**:

If everything goes well, the phase of the `BackupConfiguration` should be in `Ready` state. The `Ready` phase indicates that the backup setup is successful.

Let's check the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME                          PHASE   PAUSED   AGE
deployment-volume-snapshot    Ready            12s
```

**Verify Repository:**

Verify that the Repository specified in the BackupConfiguration has been created using the following command,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE   PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository                                       Ready                            28s
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the Repository YAML stored in the `demo/deployment-volume-snapshot-repo` directory.

**Verify CronJob:**

Verify that KubeStash has created a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` object.

Check that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                                 SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-deployment-volume-snapshot-frequent-backup   */5 * * * *   False     0        <none>          20s
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` CR using the following command,

```bash
$ watch -n 1 kubectl get backupsession -n demo -l=kubestash.com/invoker-name=deployment-volume-snapshot

Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-name=deployment-volume-snapshot                                       anisur: Wed Jan 17 15:20:09 2024

NAME                                                    INVOKER-TYPE          INVOKER-NAME                 PHASE     DURATION   AGE
deployment-volume-snapshot-frequent-backup-1705483201   BackupConfiguration   deployment-volume-snapshot   Running              9s
```

Here, the phase `Succeeded` means that the backup process has been completed successfully.

**Verify Backup:**

When backup session is complete, KubeStash will update the respective `Repository` object to reflect the backup. Check that the repository `gcs-repository` has been updated by the following command,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository   true        1                2.262 KiB   Ready   103s                     72s

```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot`.

Verify created `Snapshot` object by the following command,

```bash
$ watch -n 1 kubectl get snapshot -n demo -l=kubestash.com/repo-name=gcs-repository

Every 1.0s: kubectl get snapshot -n demo -l=kubestash.com/repo-name=gcs-repository                                                                                                                            anisur: Wed Jan 17 15:20:56 2024

NAME                                                              REPOSITORY       SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       VERIFICATION-STATUS   AGE
gcs-repository-deployment-volumeshot-frequent-backup-1705483201   gcs-repository   frequent-backup   2024-01-17T09:20:04Z   Delete            Succeeded                         56s
```

> When a backup is triggered according to schedule, KubeStash will create a `Snapshot` with the following labels  `kubestash.com/app-ref-kind: Deployment`, `kubestash.com/app-ref-name: <volume-name>`, `kubestash.com/app-ref-namespace: <volume-namespace>` and `kubestash.com/repo-name: <repository-name>`. We can use these labels to watch only the `Snapshot` of our desired Workload or `Repository`.

Now, lets retrieve the YAML for the `Snapshot`, and inspect the `spec.status` section to see the backup information of the targeted PVC.

```bash
$ kubectl get snapshots  -n demo gcs-repository-deployment-volumeshot-frequent-backup-1705483201 -o yaml
```
```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  name: gcs-repository-deployment-volumeshot-frequent-backup-1705483201
  namespace: demo
spec:
  ---
status:
  components:
    volumesnapshot:
      driver: VolumeSnapshotter
      phase: Succeeded
      volumeSnapshotterStats:
        - hostPath: /source/data
          pvcName: source-data
          volumeSnapshotName: source-data-1705483201
        - hostPath: /source/config
          pvcName: source-config
          volumeSnapshotName: source-config-1705483201
  conditions:
   ---
  phase: Succeeded
  totalComponents: 1
```

Here,
> For volume snapshot backup KubeStash uses `volumesnapshot` as the component name for the Deployment `PVCs`.

- `volumeSnapshotterStats.pvcName` specifies the name of the targeted `PVC`.
- `volumeSnapshotterStats.hostPath` specifies the mount path for the targeted `PVC` within the workload.
- `volumeSnapshotterStats.volumeSnapshotName` specifies the name of the `VolumeSnapshot` created for the targeted PVC.

KubeStash keeps the backup for Snapshot YAMLs. If we navigate to the GCS bucket, we will see the Snapshot YAML stored in the `<kubestash-qa/demo/deployment-volume-snapshot-repo/snapshots>` directory.

**Verify Volume Snapshot :**

Once a `BackupSession` CR is created, KubeStash operator creates a volume snapshotter `Job`. Then the `Job` creates a `VolumeSnapshot` CR for the targeted PVC.

Run the following command to check that the `VolumeSnapshot` has been created Successfully.

```bash
$ kubectl get volumesnapshot -n demo
NAME                       READYTOUSE   SOURCEPVC       SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS        SNAPSHOTCONTENT                                    CREATIONTIME   AGE
source-config-1705483201   true         source-config                           1Gi           csi-snapshot-class   snapcontent-3fa30373-e2f7-4288-8bee-4ff2c7720117   3m1s           3m3s
source-data-1705483201     true         source-data                             1Gi           csi-snapshot-class   snapcontent-b3b3118e-318d-4876-b69b-f124ce868d24   3m1s           3m3s
```

Let's find out the actual `VolumeSnapshotContent` that has been saved in the Google Cloud by the following command,

```bash
kubectl get volumesnapshot source-data-1705483201 -n demo -o yaml
```

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  creationTimestamp: "2024-01-17T09:20:04Z"
  finalizers:
    - snapshot.storage.kubernetes.io/volumesnapshot-as-source-protection
    - snapshot.storage.kubernetes.io/volumesnapshot-bound-protection
  generation: 1
  name: source-data-1705483201
  namespace: demo
  resourceVersion: "8875"
  uid: b3b3118e-318d-4876-b69b-f124ce868d24
spec:
  source:
    persistentVolumeClaimName: source-data
  volumeSnapshotClassName: csi-snapshot-class
status:
  boundVolumeSnapshotContentName: snapcontent-b3b3118e-318d-4876-b69b-f124ce868d24
  creationTime: "2024-01-17T09:20:06Z"
  readyToUse: true
  restoreSize: 1Gi

```

Here, `spec.status.boundVolumeSnapshotContentName` field specifies the name of the `VolumeSnapshotContent` object. It also represents the actual snapshot name that has been saved in Google Cloud. If we navigate to the `Snapshots` tab in the GCP console, we are going to see snapshot `snapcontent-f4a199c2-eed5-4438-aa09-e9c9683556ef` has been stored successfully.

<figure align="center">
  <img alt="Snapshots in GCP" src="/docs/guides/volumesnapshot/statefulset/images/gcp.png">
<figcaption align="center">Fig: Snapshots in GCP</figcaption>
</figure>


## Restore

This section will show you how to restore the PVCs from the snapshots we have taken in the previous section.

**Create RestoreSession :**

Now, we are going to create a `RestoreSession` custom resource to restore all backed-up `PVC` volumes individually from their respective `VolumeSnapshot`. Below is the YAML of the `RestoreSession` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: restore-deployment-pvc
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
                name: restore-data
              spec:
                accessModes: [ "ReadWriteOnce" ]
                storageClassName: "csi-standard"
                resources:
                  requests:
                    storage: 1Gi
            - metadata:
                name: restore-config
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


Let's create the `RestoreSession` crd we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/deployment/examples/restoresession.yaml
restoresession.core.kubestash.com/restore-deployment-pvc created
```

Once, you have created the `RestoreSession` object, KubeStash will create a job to restore. We can watch the `RestoreSession` phase to check if the restore process has been succeeded or not.

Run the following command to watch RestoreSession phase,

```bash
$ watch -n 1 kubectl get -n demo restoresession 

Every 1.0s: kubectl get -n demo restoresession                  workstation: Wed Jan 17 15:28:39 2024

NAME                     REPOSITORY       FAILURE-POLICY   PHASE       DURATION   AGE
restore-deployment-pvc   gcs-repository                    Running     27s        62s
restore-deployment-pvc   gcs-repository                    Succeeded   27s        62s
```

So, we can see from the output of the above command that the restore process succeeded.

**Verify Restored PVC :**

Once the restore process is complete, we are going to see that new PVCs with the name `restore-data` and `restore-config` have been created.

Verify that the PVCs have been created by the following command,

```bash
$ kubectl get pvc -n demo
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
restore-config   Bound    pvc-f52515f0-f8a2-4939-8d38-fd625725ce7c   1Gi        RWO            csi-standard   104s
restore-data     Bound    pvc-b45ef4e3-bed3-4305-aabf-580300fc3db8   1Gi        RWO            csi-standard   104s
```

Notice the `STATUS` field. It indicates that the respective PV has been provisioned and initialized from the respective VolumeSnapshot by CSI driver and the PVC has been bound with the PV.

> The [volumeBindingMode](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode) field controls when volume binding and dynamic provisioning should occur. Kubernetes allows `Immediate` and `WaitForFirstConsumer` modes for binding volumes. The `Immediate` mode indicates that volume binding and dynamic provisioning occurs once the PVC is created and `WaitForFirstConsumer` mode indicates that volume binding and provisioning does not occur until a pod is created that uses this PVC. By default `volumeBindingMode` is `Immediate`.

**Verify Restored Data :**

We are going to create a new `Deployment` with the restored PVCs to verify whether the backed up data has been restored.

Below is the YAML of the `Deployment` that we are going to create,

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: restore-demo
  name: restore-demo
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: restore-demo
  template:
    metadata:
      labels:
        app: restore-demo
      name: busybox
    spec:
      containers:
      - args:
        - sleep
        - "3600"
        image: busybox
        imagePullPolicy: IfNotPresent
        name: busybox
        volumeMounts:
        - mountPath: /restore/data
          name: restore-data
        - mountPath: /restore/config
          name: restore-config
      restartPolicy: Always
      volumes:
      - name: restore-data
        persistentVolumeClaim:
          claimName: restore-data
      - name: restore-config
        persistentVolumeClaim:
          claimName: restore-config
```

Let's create the `Deployment` we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volumesnapshot/deployment/examples/restored-deployment.yaml
deployment.apps/restore-demo created
```

Now, wait for the pod of the `Deployment` to go into the `Running` state.

```bash
$ kubectl get pod -n demo
NAME                                                              READY   STATUS      RESTARTS   AGE
restore-demo-8bf47f8cb-4flpf                                      1/1     Running     0          30s
```

Verify that the backed up data has been restored in `/restore/data` and `/restore/config` directory using the following command,

```bash
$ kubectl exec -n demo restore-demo-8bf47f8cb-4flpf -- cat /restore/data/config.txt
config_data
$ kubectl exec -n demo restore-demo-8bf47f8cb-4flpf -- cat /restore/data/data.txt
sample_data
```

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo deployment kubestash-demo
kubectl delete -n demo deployment restore-demo
kubectl delete -n demo restoresession restore-deployment-pvc
kubectl delete -n demo backupconfiguration deployment-volume-snapshot
kubectl delete -n demo retentionPolicy demo-retention
kubectl delete -n demo backupstorage gcs-storage
kubectl delete -n demo pvc --all
kubectl delete -n demo volumesnapshot --all
kubectl delete -n demo storageclass csi-standard
kubectl delete -n demo volumesnapshotclass csi-snapshot-class
```
