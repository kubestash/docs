---
title: Populate Volumes of a Deployment | KubeStash
description: An step by step guide to show how to populate volumes of a Deployment
menu:
  docs_{{ .version }}:
    identifier: volume-populate-deployment
    name: Populate Volumes of a Deployment
    parent: volume-populator
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Populating volumes of a Deployment

This guide will show you how to use KubeStash to populate the volumes of a `Deployment`. Within this guide, we will demonstrate the process of backing up volumes of a `Deployment` and subsequently restoring it using Kubernetes native approach, facilitated by KubeStash.

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

## Prepare Workload

Here, we are going to deploy a `Deployment` with two PVCs and generate some sample data in it. Then, we are going to backup of these PVCs using KubeStash.

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
  resources:
    requests:
      storage: 1Gi
```

Let's create the PVCs we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/deployment/examples/backup-pvcs.yaml
persistentvolumeclaim/source-data created
persistentvolumeclaim/source-config created
```

**Deploy Deployment :**

Now, we are going to deploy a `Deployment` that uses the above PVCs. This `Deployment` will automatically create `data.txt` and `config.cfg` file in `/source/data` and `/source/config` directory.

Below is the YAML of the `Deployment` that we are going to create,

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubestash-demo
  name: kubestash-deployment
  namespace: demo
spec:
  replicas: 2
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

Let's create the deployment that we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/deployment/examples/backup-deployment.yaml
deployment.apps/kubestash-deployment created
```

Now, wait for the pod of the `Deployment` to go into the `Running` state.

```bash
$ kubectl get pods -n demo
NAME                              READY   STATUS    RESTARTS   AGE
kubestash-deployment-745d49c4cd-vsj8m   1/1     Running   0          62s
kubestash-deployment-745d49c4cd-wwj8w   1/1     Running   0          62s
```

Verify that the sample data has been created in `/source/data` and `/source/config` directory using the following command,

```bash
$ kubectl exec -n demo kubestash-deployment-745d49c4cd-vsj8m -- cat /source/data/data.txt
sample_data
$ kubectl exec -n demo kubestash-deployment-745d49c4cd-vsj8m -- cat /source/config/config.cfg
config_data
```

### Prepare Backend

Now, we are going to backup the `kubestash-deployment` Deployment in a GCS bucket using KubeStash. We have to create a `Secret` with  necessary credentials and a `BackupStorage` object to use this backend. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

> For GCS backend, if the bucket does not exist, KubeStash needs `Storage Object Admin` role permissions to create the bucket. For more details, please check the following [guide](/docs/guides/backends/gcs/index.md).

**Create Secret:**

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/deployment/examples/backupstorage.yaml
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
Notice the `spec.usagePolicy` that allows referencing the `RetentionPolicy` from all namespaces. To allow specific namespaces, we can configure it accordingly by following [RetentionPolicy usage policy](/docs/concepts/crds/retentionpolicy/index.md#retentionpolicy-spec).

Let's create the `RetentionPolicy` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/deployment/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

### Backup

We have to create a `BackupConfiguration` object targeting the `kubestash-demo` Deployment that we have deployed earlier.

we also have to create another `Secret` with an encryption key `RESTIC_PASSWORD` for `Restic`. This secret will be used by `Restic` for both encrypting and decrypting the backup data during backup & restore.

**Create Secret:**

Let's create a secret called `encry-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encryption-secret \
    --from-file=./RESTIC_PASSWORD \
secret "encryption-secret" created
```

**Create BackupConfiguration :**

Below is the YAML of the `BackupConfiguration` that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-backup-dep
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name:  kubestash-deployment
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
          directory: /dep
          encryptionSecret:
            name: encryption-secret # some addon may not support encryption
            namespace: demo
          deletionPolicy: WipeOut
      addon:
        name: workload-addon
        tasks:
          - name: logical-backup
            params:
              paths: /source/data,/source/config
              exclude: /source/data/lost+found,/source/config/lost+found
```

Let's create the `BackupConfiguration` object we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/deployment/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-backup-dep created
```

**Verify Backup Setup Successful**:

If everything goes well, the phase of the `BackupConfiguration` should be in `Ready` state. The `Ready` phase indicates that the backup setup is successful. 

Let's check the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME                PHASE   PAUSED   AGE
sample-backup-dep   Ready            2m50s
```
**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` object.

Check that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                                 SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-sample-backup-dep-frequent-backup               */5 * * * *   False     0        <none>          20s
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` object using the following command,

```bash
$ watch -n 1 kubectl get backupsession -n demo -l=kubestash.com/invoker-name=sample-backup-dep

Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-name=sample-backup-dep                                       anisur: Wed Jan 17 15:20:09 2024

NAME                                                    INVOKER-TYPE          INVOKER-NAME                 PHASE     DURATION   AGE
sample-backup-dep-frequent-backup-1705483201            BackupConfiguration   sample-backup-dep            Running              9s
```

**Verify Backup:**

When backup session is complete, KubeStash will update the respective `Repository` object to reflect the backup. Check that the repository `gcs-repository` has been updated by the following command,

```bash
$ kubectl get repository -n demo gcs-demo-repo
NAME              INTEGRITY   SNAPSHOT-COUNT   SIZE    PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository    true        1                806 B   Ready   8m27s                    9m18s
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run to a particular `Repository`.

Verify created `Snapshot` object by the following command,

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-repository
NAME                                                          REPOSITORY      SESSION        SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-repository-sample-backup-dep-frequent-backup-1706015400   gcs-demo-repo   demo-session   2024-01-23T13:10:54Z   Delete            Succeeded   16h
```

## Populate Volumes

This section will guide you through the process of populating volumes of a `Deployment` by restoring previously backed up data using KubeStash.

Now, we need to create new Persistent Volume Claims (PVCs) with the `spec.dataSourceRef` set to reference our `Snapshot` object. These PVCs will contain the restored data.

Below is the YAML of the restored PVCs,

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: restored-source-data
  namespace: demo
  annotations:
        populator.kubestash.com/app-name: kubestash-restored-deployment
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  dataSourceRef:
    apiGroup: storage.kubestash.com
    kind: Snapshot
    name: gcs-repository-sample-backup-dep-frequent-backup-1706015400
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: restored-source-data
  namespace: demo
  annotations:
        populator.kubestash.com/app-name: kubestash-restored-deployment
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  dataSourceRef:
    apiGroup: storage.kubestash.com
    kind: Snapshot
    name: gcs-repository-sample-backup-dep-frequent-backup-1706015400
```
Here,
- `spec.dataSourceRef` specifies that which `snapshot` we want to restore into our populate volume. we are referencing a `snapshot` object that we have backed up in the previous section.
- `metadata.annotations.populator.kubestash.com/app-name` field is mandatory  for any volume population of a deployment through KubeStash.
  - This field denotes the deployment that will be attached those volumes via mount paths. The volume population will only be successful if the mount path of this volume matches the mount paths of the backup deployment.
  - For example, you backed up a deployment with volumes named `source-data` and `source-config`, each with corresponding mount paths `/source/data` and `/source/config`. Now you wish to populate volumes named `restore-source-data` and `restore-source-config` attached to a deployment named `restored-kubestash-deployment`, then this deployment must have mount paths set to `/source/data` and `/source/config`, respectively.

Let's create the PVCs YAMl that we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/deployment/examples/populated-pvcs.yaml
persistentvolumeclaim/restored-source-data created
persistentvolumeclaim/restored-source-config created
```

**Deploy Deployment :**

Now, we are going to deploy a `Deployment` that uses the above PVCs. This `Deployment` has mount paths `/source/data` and `/source/config` corresponding to volumes named `restored-source-data` and `restored-source-config` respectively.

Below is the YAML of the `Deployment` that we are going to create,

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubestash-demo
  name: restored-kubestash-deployment
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
      - command: ["/bin/sh", "-c"]
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
      - name: restored-source-data
        persistentVolumeClaim:
         claimName: source-data
      - name: restored-source-config
        persistentVolumeClaim:
          claimName: source-config
```

Let's create the deployment we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/deployment/examples/restore-deployment.yaml
deployment.apps/restored-kubestash-deployment created
```

**Wait for Populate Volume:**

When you create two `PVCs` with `spec.dataSourceRef` set and refers our `Snapshot` object, KubeStash will create a restore Job. Now, wait for the volume populate process to complete. 

You can watch the `PVCs` status using the following command,

```bash
$ watch kubectl get pvc -n demo 

Every 2.0s: kubectl get pvc -n demo                                                                   anisur: Tue Feb 13 18:37:26 2024

NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
restored-source-config   Bound    pvc-27a19e86-0ed5-4b18-9263-c8a66459aeb0   1Gi        RWO            standard-rwo   2m50s
restored-source-data     Bound    pvc-d210787e-5137-40b7-89e7-73d982716b40   1Gi        RWO            standard-rwo   2m51s

```

From the output of the above command, we can see that `PVCs` status is `Bound` that means populate volumes has been completed successfully.

**Verify Restored Data :**

We are going to exec a pod of `restored-dep` deployment to verify whether the volume population with the backed up data has been restored successfully or not.

Now, wait for the deployment pod to go into the `Running` state.

```bash
$ kubectl get pods -n demo 
NAME                                             READY   STATUS    RESTARTS   AGE
restored-kubestash-deployment-84b974dfb5-4jzjd   1/1     Running   0          2m48s

```

Verify that the backed up data has been restored into `/source/data` and `/source/config` directory of above pod using the following command,

```bash
$ kubectl exec -it -n demo restored-kubestash-deployment-84b974dfb5-4jzjd -- cat /source/data/data.txt
sample_data

$ kubectl exec -it -n demo restored-kubestash-deployment-84b974dfb5-4jzjd -- cat /source/config/config.cfg
sample_config
```

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete backupconfiguration -n demo sample-backup-dep
kubectl delete backupstorage -n demo gcs-storage
kubectl delete retentionPolicy -n demo demo-retention
kubectl delete secret -n demo gcs-secret
kubectl delete secret -n demo encryption-secret
kubectl delete deploy -n demo kubestash-deployment restored-kubestash-deployment
kubectl delete pvc -n demo --all
```