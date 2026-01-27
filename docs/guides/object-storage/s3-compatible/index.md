---
title: Backup & Restore of S3-Compatible Object Storage | KubeStash
description: A comprehensive guide showing how to backup and restore data from S3-compatible object storage buckets using KubeStash.
menu:
  docs_{{ .version }}:
    identifier: s3-compatible
    name: S3-Compatible Object Storage
    parent: object-storage
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup and Restore of S3-Compatible Object Storage

This tutorial demonstrates how to use KubeStash to backup and restore data from S3-compatible object storage buckets. Many cloud providers offer S3-compatible object storage solutions, such as DigitalOcean Spaces, MinIO, Wasabi, Linode Object Storage, and others. This guide shows you how to:

1. Mount an S3-compatible bucket prefix as a PersistentVolume using a CSI driver
2. Backup the data from the mounted volume to another cloud object storage backend using KubeStash
3. Restore the data from backup when needed

## Overview

In this guide, we will use DigitalOcean Spaces (an S3-compatible object storage) as our source data storage. We'll mount a bucket prefix as a Kubernetes volume, then backup its contents to a different object storage backend. The same approach works with any S3-compatible storage provider.

## Before You Begin

- You need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).

- You need access to an S3-compatible object storage service (e.g., DigitalOcean Spaces, MinIO, AWS S3, etc.) with:
  - Access credentials (Access Key ID and Secret Access Key)
  - An existing bucket
  - The endpoint URL and region information

- You should be familiar with the following `KubeStash` concepts:
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [Snapshot](/docs/concepts/crds/snapshot/index.md)
  - [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/object-storage/s3-compatible/examples](https://github.com/kubestash/docs/tree/{{< param "info.version" >}}/docs/guides/object-storage/s3-compatible/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Backup S3-Compatible Object Storage

This section demonstrates how to use KubeStash to backup data from an S3-compatible bucket. We'll walk through the complete process step by step.

### Install S3-Compatible CSI Driver

To mount an S3-compatible bucket as a Kubernetes volume, we need an S3 CSI driver. In this guide, we'll use the [k8s-csi-s3 driver](https://github.com/yandex-cloud/k8s-csi-s3), which supports various S3-compatible storage providers.

Install the CSI driver using Helm:

```bash
$ helm repo add yandex-s3 https://yandex-cloud.github.io/k8s-csi-s3/charts

$ helm install csi-s3 yandex-s3/csi-s3 \
      --set secret.accessKey=$AWS_ACCESS_KEY_ID \
      --set secret.secretKey=$AWS_SECRET_ACCESS_KEY \
      --set secret.endpoint="https://sfo3.digitaloceanspaces.com" \
      --set secret.region="us-sfo3" \
      --set storageClass.singleBucket="kubestash" \
      --namespace csi-s3 --create-namespace
```

**Parameter Explanation:**
- `secret.accessKey` - Your S3-compatible storage access key ID
- `secret.secretKey` - Your S3-compatible storage secret access key
- `secret.endpoint` - The endpoint URL of your S3-compatible storage (e.g., DigitalOcean Spaces endpoint)
- `secret.region` - The region where your bucket is located
- `storageClass.singleBucket` - The name of your existing bucket that will be used for mounting

> **Note:** For alternative installation methods, follow the official installation guide of the driver [here](https://github.com/yandex-cloud/k8s-csi-s3?tab=readme-ov-file#kubernetes-installation).

Verify that the driver is installed successfully:

```bash
$ kubectl get pods -n csi-s3
NAME                   READY   STATUS    RESTARTS   AGE
csi-s3-fxpck           2/2     Running   0          34s
csi-s3-provisioner-0   2/2     Running   0          34s
```

Verify that the `StorageClass` has been created successfully:

```bash
$ kubectl get storageclasses.storage.k8s csi-s3
NAME     PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
csi-s3   ru.yandex.s3.csi   Delete          Immediate           false                  50s
```

You can view the StorageClass details with:

```bash
$ kubectl get storageclasses.storage.k8s csi-s3 -o yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    meta.helm.sh/release-name: csi-s3
    meta.helm.sh/release-namespace: csi-s3
  creationTimestamp: "2025-12-23T09:01:02Z"
  labels:
    app.kubernetes.io/managed-by: Helm
  name: csi-s3
  resourceVersion: "1782"
  uid: dde5668e-8fdb-4025-9aa2-919cc2e93af3
parameters:
  bucket: kubestash
  csi.storage.k8s.io/controller-publish-secret-name: csi-s3-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: csi-s3
  csi.storage.k8s.io/node-publish-secret-name: csi-s3-secret
  csi.storage.k8s.io/node-publish-secret-namespace: csi-s3
  csi.storage.k8s.io/node-stage-secret-name: csi-s3-secret
  csi.storage.k8s.io/node-stage-secret-namespace: csi-s3
  csi.storage.k8s.io/provisioner-secret-name: csi-s3-secret
  csi.storage.k8s.io/provisioner-secret-namespace: csi-s3
  mounter: geesefs
  options: --memory-limit 1000 --dir-mode 0777 --file-mode 0666
provisioner: ru.yandex.s3.csi
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

### Prepare Sample Data

We will generate some sample files and upload them to an S3 bucket prefix. This data will be mounted as a volume and then backed up using KubeStash.

**Create S3 Configuration Secret:**

First, create a secret with your S3-compatible bucket credentials and configuration:

```bash
$ kubectl create secret generic s3-config \
             --from-literal=AWS_ACCESS_KEY_ID=<your_access_key_id> \
             --from-literal=AWS_SECRET_ACCESS_KEY=<your_secret_access_key> \
             --from-literal=AWS_REGION='us-sfo3' \
             --from-literal=S3_ENDPOINT='https://sfo3.digitaloceanspaces.com' \
             --from-literal=S3_BUCKET='kubestash' \
             --from-literal=PREFIX='fuse' \
             --namespace demo
secret/s3-config created
```

Replace the placeholder values with your actual credentials and configuration.

**Upload Sample Data:**

Now, create a pod that will generate random sample files and upload them to your S3 bucket prefix:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-uploader
  namespace: demo
spec:
  restartPolicy: Never
  containers:
    - name: uploader
      image: amazon/aws-cli:latest
      env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: s3-config
              key: AWS_ACCESS_KEY_ID
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: s3-config
              key: AWS_SECRET_ACCESS_KEY
        - name: AWS_REGION
          valueFrom:
            secretKeyRef:
              name: s3-config
              key: AWS_REGION
        - name: S3_ENDPOINT
          valueFrom:
            secretKeyRef:
              name: s3-config
              key: S3_ENDPOINT
        - name: S3_BUCKET
          valueFrom:
            secretKeyRef:
              name: s3-config
              key: S3_BUCKET
        - name: S3_PREFIX
          valueFrom:
            secretKeyRef:
              name: s3-config
              key: PREFIX
      command: ["/bin/sh", "-c"]
      args:
        - |
          set -eu
          mkdir -p /tmp/random

          for i in $(seq 1 5); do
            FILE="/tmp/random/file_${i}.txt"
            head -c 2048 /dev/urandom | base64 > "$FILE"
            echo "Generated: $(basename "$FILE")"

            KEY="${S3_PREFIX}/$(basename "$FILE")"
            if ! aws s3 ls "s3://${S3_BUCKET}/${KEY}" \
                --endpoint-url "${S3_ENDPOINT}" --region "${AWS_REGION}" >/dev/null 2>&1; then
              aws s3 cp "$FILE" "s3://${S3_BUCKET}/${KEY}" \
                --endpoint-url "${S3_ENDPOINT}" --region "${AWS_REGION}"
              echo "Uploaded: ${KEY}"
            else
              echo "Skip (exists): ${KEY}"
            fi
          done

          echo "🎉 Done uploading random files."
```

Let's create the pod we have shown above:

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/object-storage/s3-compatible/examples/random-data-uploader-pod.yaml
pod/random-uploader created
```

Verify that the pod has successfully uploaded the sample files to your S3 bucket prefix:

```bash
$ kubectl logs -n demo random-uploader
Generated: file_1.txt
upload: ../tmp/random/file_1.txt to s3://kubestash/fuse/file_1.txt
Uploaded: fuse/file_1.txt
Generated: file_2.txt
upload: ../tmp/random/file_2.txt to s3://kubestash/fuse/file_2.txt
Uploaded: fuse/file_2.txt
Generated: file_3.txt
upload: ../tmp/random/file_3.txt to s3://kubestash/fuse/file_3.txt
Uploaded: fuse/file_3.txt
Generated: file_4.txt
upload: ../tmp/random/file_4.txt to s3://kubestash/fuse/file_4.txt
Uploaded: fuse/file_4.txt
Generated: file_5.txt
upload: ../tmp/random/file_5.txt to s3://kubestash/fuse/file_5.txt
Uploaded: fuse/file_5.txt
🎉 Done uploading random files.
```

### Mount S3 Bucket Prefix as PersistentVolume

Now we need to create a PersistentVolume (PV) that mounts the S3 bucket at the specified prefix, and a PersistentVolumeClaim (PVC) that will bind to that PV.

**Create PersistentVolume:**

Below is the YAML of the PV that we are going to create:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: fuse-pv
  namespace: demo
spec:
  storageClassName: csi-s3
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  claimRef:
    namespace: demo
    name: fuse-pvc
  csi:
    driver: ru.yandex.s3.csi
    controllerPublishSecretRef:
      name: csi-s3-secret
      namespace: demo
    nodePublishSecretRef:
      name: csi-s3-secret
      namespace: csi-s3
    nodeStageSecretRef:
      name: csi-s3-secret
      namespace: csi-s3
    volumeAttributes:
      capacity: 10Gi
      mounter: geesefs
      options: --memory-limit 1000 --dir-mode 0777 --file-mode 0666
    volumeHandle: kubestash/fuse
```

**Important:** You must manually configure the `csi` section to mount an existing S3 bucket prefix path, and set the `claimRef` to point to the namespace where the PVC will be created.

**Field Explanations:**
- `claimRef` - Specifies the namespace and name of the PVC that will bind to this PV
- `csi.driver` - Specifies the CSI driver to use (`ru.yandex.s3.csi`)
- `csi.controllerPublishSecretRef`, `csi.nodePublishSecretRef`, `csi.nodeStageSecretRef` - Reference the secret containing S3 credentials
- `csi.volumeAttributes` - Contains additional volume attributes (capacity, mounter type, options)
- `csi.volumeHandle` - Specifies the S3 bucket and prefix to mount in the format `<bucket-name>/<prefix>`

Let's create the PV:

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/object-storage/s3-compatible/examples/pv.yaml
persistentvolume/fuse-pv created
```

**Create PersistentVolumeClaim:**

Now, create a PVC that will bind to the above PV. Below is the YAML of the PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fuse-pvc
  namespace: demo
spec:
  storageClassName: "csi-s3"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

Let's create the PVC:

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/object-storage/s3-compatible/examples/pvc.yaml
persistentvolumeclaim/fuse-pvc created
```

### Deploy Application with Mounted Volume

Now, we are going to deploy a Deployment that uses the above PVC. This deployment pod will mount the S3 bucket prefix to the `/source/data` directory where we can see the sample files.

Below is the YAML of the Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: fuse-demo
  name: fuse-demo
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fuse-demo
  template:
    metadata:
      labels:
        app: fuse-demo
      name: ubuntu
    spec:
      containers:
        - image: ubuntu:latest
          command: ["/bin/sh", "-c"]
          args:
            - |
              sleep infinity
          imagePullPolicy: IfNotPresent
          name: ubuntu
          volumeMounts:
            - mountPath: /source/data
              name: source-data
      restartPolicy: Always
      volumes:
        - name: source-data
          persistentVolumeClaim:
            claimName: fuse-pvc
```

Let's create the Deployment:

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/object-storage/s3-compatible/examples/deployment.yaml
deployment.apps/fuse-demo created
```

Wait for the pods of the Deployment to go into the `Running` state:

```bash
$ kubectl get pod -n demo
NAME                         READY   STATUS    RESTARTS   AGE
fuse-demo-7794cc7994-qwn9m   1/1     Running   0          2m52s
```

Verify that the sample files are present in the mounted directory `/source/data`:

```bash
$ kubectl exec -it -n demo fuse-demo-7794cc7994-qwn9m -- ls /source/data
file_1.txt  file_2.txt  file_3.txt  file_4.txt  file_5.txt
```

Great! The S3 bucket data is now successfully mounted to the Deployment pod. Now we're ready to backup this PVC using KubeStash.

### Prepare Backend Storage

We need a backend storage where KubeStash will store the backup snapshots. This should be different from your source S3 storage. We can use another S3-compatible storage or a different cloud provider.

We are going to use a GCS bucket. For this, we have to create a `Secret` with necessary credentials and a `BackupStorage` object. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

> For GCS backend, if the bucket does not exist, KubeStash needs `Storage Object Admin` role permissions to create the bucket. For more details, please check the following [guide](/docs/guides/backends/gcs/index.md).

**Create Secret:**

Let's create a Secret named `gcs-secret` with access credentials to our desired GCS bucket,

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
      prefix: fuse-backup
      secretName: gcs-secret
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true
  deletionPolicy: WipeOut
```

Let's create the `BackupStorage` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/object-storage/s3-compatible/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-storage created
```

Now, we are ready to backup our target volume to this backend.

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
      from: All
```

Notice the `spec.usagePolicy` that allows referencing the `RetentionPolicy` from all namespaces.For more details on configuring it for specific namespaces, please refer to the following [RetentionPolicy usage policy](/docs/concepts/crds/retentionpolicy/index.md).

Let's create the `RetentionPolicy` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/object-storage/s3-compatible/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

## Backup

Now, we have to create a `BackupConfiguration` custom resource targeting the PVC mounted with S3 bucket data.

We also have to create another `Secret` with an encryption key `RESTIC_PASSWORD` for `Restic`. This secret will be used by `Restic` for both encrypting and decrypting the backup data during backup & restore.

**Create Secret:**

Let's create a secret named `encrypt-secret` with the Restic password.

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD 
secret/encrypt-secret created
```

**Create BackupConfiguration:**

Now, we need to create a `BackupConfiguration` CR that specifies how to backup the PVC mounted with S3 bucket data.

Below is the YAML of the BackupConfiguration:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: s3-bucket-backup
  namespace: demo
spec:
  target:
    apiGroup:
    kind: PersistentVolumeClaim
    name: fuse-pvc
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
      scheduler:
        schedule: "*/5 * * * *"
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: gcs-repository
          backend: gcs-backend
          directory: /fuse-bucket-data
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: pvc-addon
        tasks:
          - name: logical-backup
```
Now, let's create the BackupConfiguration:

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/object-storage/s3-compatible/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/s3-bucket-backup created
```


**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be in `Ready` state. The `Ready` phase indicates that the backup setup is successful.

Let's check the `Phase` of the BackupConfiguration

```bash
➤ kubectl get backupconfiguration -n demo
NAME               PHASE      PAUSED   AGE
s3-bucket-backup   NotReady            12s

```


**Verify Repository:**

Verify that the Repository specified in the BackupConfiguration has been created using the following command,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE   PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository                                       Ready                            28s
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the Repository YAML stored in the `kubestash-qa/demo/fuse-bucket-data` directory.

**Verify CronJob:**

Verify that KubeStash has created a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` object.

Check that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                       SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-s3-bucket-backup-frequent-backup   */5 * * * *   <none>     False     0        <none>          2m5s

```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` CR using the following command,

```bash
$ watch -n 1 kubectl get backupsessions.core.kubestash.com -n demo -l=kubestash.com/invoker-name=s3-bucket-backup

NAME                                          INVOKER-TYPE          INVOKER-NAME       PHASE       DURATION   AGE
s3-bucket-backup-frequent-backup-1766732073   BackupConfiguration   s3-bucket-backup   Succeeded   40s        112s

```

Here, the phase `Succeeded` means that the backup process has been completed successfully.

**Verify Backup:**

When backup session is complete, KubeStash will update the respective `Repository` to reflect the latest state of backed up data.

```bash
$ kubectl get repositories.storage.kubestash.com -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository   true        1                2.262 KiB   Ready   103s                     8m
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot`.

Verify created `Snapshot` object by the following command,

```bash
$ kubectl get snapshots.storage.kubestash.com -n demo -l=kubestash.com/repo-name=gcs-repository
NAME                                                         REPOSITORY       SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-repository-s3-bucket-backup-frequent-backup-1766732073   gcs-repository   frequent-backup   2025-12-26T06:56:03Z   Delete            Succeeded   3m26s
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <target-kind>`
> - `kubestash.com/app-ref-name: <target-name>`
> - `kubestash.com/app-ref-namespace: <target-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
>
> These labels can be used to watch only the `Snapshot`s related to our desired Workload or `Repository`.


Now, lets retrieve the YAML for the `Snapshot`, and inspect the `spec.status` section to see the backup up components of the PVC.

```bash
$ kubectl get snapshots.storage.kubestash.com -n demo gcs-repository-s3-bucket-backup-frequent-backup-1766732073 -o yaml

apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  creationTimestamp: "2025-12-26T06:56:03Z"
  finalizers:
  - kubestash.com/cleanup
  generation: 1
  labels:
    kubestash.com/app-ref-kind: PersistentVolumeClaim
    kubestash.com/app-ref-name: fuse-pvc
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-repository
  name: gcs-repository-s3-bucket-backup-frequent-backup-1766732073
  namespace: demo
  ownerReferences:
  - apiVersion: storage.kubestash.com/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: Repository
    name: gcs-repository
    uid: fb2d4473-358b-48cd-8ff1-cb72ede8e674
  resourceVersion: "74128"
  uid: daa82dfb-aff5-49dc-8c0c-47918eebbf93
spec:
  appRef:
    kind: PersistentVolumeClaim
    name: fuse-pvc
    namespace: demo
  backupSession: s3-bucket-backup-frequent-backup-1766732073
  deletionPolicy: Delete
  repository: gcs-repository
  session: frequent-backup
  snapshotID: 01KDCQ2T476FP40ZQZYD1TXAJX
  type: FullBackup
  version: v1
status:
  components:
    dump:
      driver: Restic
      duration: 13.179104712s
      integrity: true
      path: repository/v1/frequent-backup/dump
      phase: Succeeded
      resticStats:
      - hostPath: /kubestash-data
        id: 32bf38606023ddf9808b7dc5f98cde1da49a71583b0f0ffac3b0cb367478c66b
        size: 13.516 KiB
        uploaded: 16.197 KiB
      size: 14.694 KiB
  conditions:
  - lastTransitionTime: "2025-12-26T06:56:03Z"
    message: Recent snapshot list updated successfully
    reason: SuccessfullyUpdatedRecentSnapshotList
    status: "True"
    type: RecentSnapshotListUpdated
  - lastTransitionTime: "2025-12-26T06:56:40Z"
    message: Metadata uploaded to backend successfully
    reason: SuccessfullyUploadedSnapshotMetadata
    status: "True"
    type: SnapshotMetadataUploaded
  integrity: true
  phase: Succeeded
  size: 14.693 KiB
  snapshotTime: "2025-12-26T06:56:03Z"
  totalComponents: 1
  verificationStatus: NotVerified
```


## Restore

This section will demonstrate how to restore the backed-up data from the snapshot to a stand-alone PVC. We'll simulate a disaster scenario by deleting the data, then restore it using KubeStash.

**Simulate Disaster:**

At first, let's simulate a disaster scenario. Let's delete the files from the S3 bucket:

```bash
$ kubectl exec -it -n demo fuse-demo-7794cc7994-qhqg6 -- rm -rf /source/data/
$ kubectl exec -it -n demo fuse-demo-7794cc7994-qhqg6 -- ls /source/data
# Empty output - all files deleted
```

Files are now deleted from the S3 bucket. Now, we will restore the data from the backup snapshot.

**Create RestoreSession:**

Now, we are going to create a `RestoreSession` object to restore the backed up data into the desired PVC.

Below is the YAML of the `RestoreSession` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: s3-bucket-restore
  namespace: demo
spec:
  target:
    apiGroup:
    kind: PersistentVolumeClaim
    name: fuse-pvc
    namespace: demo
  dataSource:
    repository: gcs-repository
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret
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

Let's create the `RestoreSession` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/object-storage/s3-compatible/examples/restoresession.yaml
restoresession.core.kubestash.com/s3-bucket-restore created
```

**Wait for RestoreSession to Succeed:**

Once, you have created the `RestoreSession` object, KubeStash will create restore Job. Wait for the restore process to complete.

You can watch the `RestoreSession` phase using the following command,

```bash
$ watch -n 1 kubectl get restoresession -n demo
Every 1.0s: kubectl get restoresession -n demo                                                                        anisur-pc: Wed Dec 31 12:44:50 2025

NAME                REPOSITORY       PHASE       DURATION   AGE
s3-bucket-restore   gcs-repository   Succeeded   7s         6m11s


```
From the output of the above command, the `Succeeded` phase indicates that the restore process has been completed successfully.


**Verify Restored Data:**

Let's verify if the deleted files have been restored successfully into the PVC. We are going to exec into individual pod and check whether the sample data exist or not.

```bash
$ kubectl exec -it -n demo fuse-demo-7794cc7994-qwn9m -- ls /source/data
file_1.txt  file_2.txt  file_3.txt  file_4.txt  file_5.txt
```

Excellent! All files have been successfully restored. Files are also restored in the S3 bucket.

## Cleanup

To cleanup the resources created in this tutorial:

```bash
$ kubectl delete restoresession -n demo s3-bucket-restore
$ kubectl delete backupconfiguration -n demo s3-bucket-backup
$ kubectl delete retentionpolicy -n demo demo-retention
$ kubectl delete backupstorage -n demo s3-storage
$ kubectl delete deployment -n demo fuse-demo
$ kubectl delete pvc -n demo fuse-pvc
$ kubectl delete pv fuse-pv
$ kubectl delete secret -n demo gcs-secret encrypt-secret
```