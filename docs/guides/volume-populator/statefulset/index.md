---
title: Populate Volumes of a StatefulSets | KubeStash
description: An step by step guide to show how to populate volumes of a StatefulSet
menu:
  docs_{{ .version }}:
    identifier: volume-populate-statefulset
    name: Populate Volumes of a StatefulSet
    parent: volume-populator
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Populating volumes of a StatefulSet

This guide will show you how to use KubeStash to populate the volumes of a `StatefulSet`. Within this guide, we will demonstrate the process of backing up volumes of a `StatefulSet` and subsequently restoring it using Kubernetes native approach, facilitated by KubeStash.

## Before You Begin
- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).
- You should be familiar with the following `KubeStash` concepts:
    - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
    - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
    - [BackupSession](/docs/concepts/crds/backupsession/index.md)
    - [Snapshot](/docs/concepts/crds/snapshot/index.md)
    - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
    - [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/volume-populator/statefulset/examples](/docs/guides/volume-populator/statefulset/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Prepare Workload

At first, We are going to deploy a `StatefulSet` with a PVC. This `StatefulSet` will automatically generate sample data in `/source/data` directory.

Below is the YAML of the StatefulSet that we are going to create,

```yaml
apiVersion: v1
kind: Service
metadata:
  name: busybox
  namespace: demo
spec:
  ports:
    - name: http
      port: 80
      targetPort: 0
  selector:
    app: demo-busybox
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sample-sts
  namespace: demo
  labels:
    app: demo-busybox
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-busybox
  serviceName: busybox
  template:
    metadata:
      labels:
        app: demo-busybox
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh", "-c","echo $(POD_NAME) > /source/data/data.txt && sleep 3000"]
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          volumeMounts:
            - name: source-data
              mountPath: "/source/data"
          imagePullPolicy: IfNotPresent
  volumeClaimTemplates:
    - metadata:
        name: source-data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 256Mi
```

Let's create the `StatefulSet` we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/statefulset/examples/statefulset.yaml
service/busybox created
statefulset.apps/sample-sts created
```

Now, wait for the pods of the `StatefulSet` to go into the `Running` state.

```bash
$ kubectl get pod -n demo
NAME           READY   STATUS    RESTARTS   AGE
sample-sts-0   1/1     Running   0          42s
sample-sts-1   1/1     Running   0          40s
sample-sts-2   1/1     Running   0          36s
```

Verify that the sample data has been generated in `/source/data` directory for `sample-sts-0` , `sample-sts-1` and `sample-sts-2` pod respectively using the following commands,

```bash
$ kubectl exec -n demo sample-sts-0 -- cat /source/data/data.txt
sample-sts-0
$ kubectl exec -n demo sample-sts-1 -- cat /source/data/data.txt
sample-sts-1
$ kubectl exec -n demo sample-sts-2 -- cat /source/data/data.txt
sample-sts-2
```

### Prepare Backend

Now, we are going to backup the StatefulSet `sample-sts` to a GCS bucket using KubeStash. For this, we have to create a `Secret` with  necessary credentials and a `BackupStorage` object. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/statefulset/examples/backupstorage.yaml
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
      from: Same
```

Notice the `spec.usagePolicy` that allows referencing the `RetentionPolicy` from all namespaces.For more details on configuring it for specific namespaces, please refer to the following [RetentionPolicy usage policy](/docs/concepts/crds/retentionpolicy/index.md).

Let's create the `RetentionPolicy` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/statefulset/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

### Backup

Now, we have to create a `BackupConfiguration` custom resource targeting the `sample-tst` StatefulSet that we have created earlier.

We also have to create another `Secret` with an encryption key `RESTIC_PASSWORD` for `Restic`. This secret will be used by `Restic` for both encrypting and decrypting the backup data during backup & restore.

**Create Secret:**

Let's create a secret called `encrypt-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD \
secret "encrypt-secret" created
```

**Create BackupConfiguration:**

Below is the YAML of the `BackupConfiguration` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-backup-sts
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: StatefulSet
    name: sample-sts
    namespace: demo
  backends:
    - name: gcs-backend
      storageRef:
        name: gcs-storage
        namespace: demo
      retentionPolicy:
        name: demo-retention
        namespace: demo
  sessions:
    - name: demo-session
      scheduler:
        schedule: "*/5 * * * *"
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: gcs-demo-repo
          backend: gcs-backend
          directory: /sample-sts
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: workload-addon
        tasks:
          - name: logical-backup
            params:
              paths: /source/data
              exclude: /source/data/lost+found
      retryConfig:
        maxRetry: 2
        delay: 1m
```

Let's create the `BackupConfiguration` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/statefulset/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-backup-sts created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be in `Ready` state. The `Ready` phase indicates that the backup setup is successful.

Let's check the `Phase` of the BackupConfiguration

```bash
$ kubectl get backupconfiguration -n demo
NAME                PHASE   PAUSED   AGE
sample-backup-sts   Ready            2m50s
```

**Verify Repository:**

Verify that the Repository specified in the BackupConfiguration has been created using the following command,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE   PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository                                       Ready                            28s
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the Repository YAML stored in the `kubestash-qa/demo/sample-sts` directory.

**Verify CronJob:**

Verify that KubeStash has created a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` object.

Check that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                     SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-sample-backup-sts-demo-session   */5 * * * *             0        2m45s           3m25s
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` CR using the following command,

```bash
$ watch -n 1 kubectl get backupsession -n demo -l=kubestash.com/invoker-name=sample-backup-sts

Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-name=sample-backup-sts                                                                                           workstation: Wed Jan  3 17:26:00 2024

NAME                                            INVOKER-TYPE          INVOKER-NAME     PHASE       DURATION   AGE
sample-backup-sts-demo-session-1706015400       BackupConfiguration   pvc-backup       Succeeded              60s

```
Here, the phase `Succeeded` means that the backup process has been completed successfully.

**Verify Backup:**

When backup session is complete, KubeStash will update the respective `Repository` object to reflect the backup. Check that the repository `gcs-repository` has been updated by the following command,

```bash
$ kubectl get repository -n demo gcs-demo-repo
NAME              INTEGRITY   SNAPSHOT-COUNT   SIZE    PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository    true        1                806 B   Ready   8m27s                    9m18s
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot`.

Verify created `Snapshot` object by the following command,


```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-repository
NAME                                                          REPOSITORY      SESSION        SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-repository-sample-backup-sts-frequent-backup-1706015400   gcs-demo-repo   demo-session   2024-01-23T13:10:54Z   Delete            Succeeded   16h
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot`.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-demo-repo
NAME                                                      REPOSITORY      SESSION        SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-demo-repo-sample-backup-sts-demo-session-1706015400   gcs-demo-repo   demo-session   2024-01-23T13:10:54Z   Delete            Succeeded   16h
```

> When a backup is triggered according to schedule, KubeStash will create a `Snapshot` with the following labels  `kubestash.com/app-ref-kind: <workload-kind>`, `kubestash.com/app-ref-name: <workload-name>`, `kubestash.com/app-ref-namespace: <workload-namespace>` and `kubestash.com/repo-name: <repository-name>`. We can use these labels to watch only the `Snapshot` of our desired Workload or `Repository`.

Now, lets retrieve the YAML for the `Snapshot`, and inspect the `spec.status` section to see the backup up components of the StatefulSet.

```bash
$ kubectl get snapshots -n demo gcs-demo-repo-sample-backup-sts-demo-session-1706015400 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: StatefulSet
    kubestash.com/app-ref-name: sample-sts
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-demo-repo
  name: gcs-demo-repo-sample-backup-sts-demo-session-1706015400
  namespace: demo
spec:
  ...
status:
  components:
    dump-pod-0:
      driver: Restic
      duration: 1.61162906s
      integrity: true
      path: repository/v1/demo-session/dump-pod-0
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        id: 4e881fdd20afb49e1baab37654cc18d440dc2f90ad61c9077956ea4561bd41dd
        size: 13 B
        uploaded: 1.046 KiB
      size: 803 B
    dump-pod-1:
      driver: Restic
      duration: 1.597963671s
      integrity: true
      path: repository/v1/demo-session/dump-pod-1
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        id: 16a414187d554e1713c0a6363d904837998dc7f7d600d7c635a04c61dc1b5467
        size: 13 B
        uploaded: 1.046 KiB
      size: 803 B
    dump-pod-2:
      driver: Restic
      duration: 1.52695046s
      integrity: true
      path: repository/v1/demo-session/dump-pod-2
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        id: 9dc9efd5e9adfd0154eca48433cc57aa09bca018d970e9530769326c9783905c
        size: 13 B
        uploaded: 1.046 KiB
      size: 798 B
  ...
```

> For StatefulSet, KubeStash takes backup from every pod of the StatefulSet. Since we are using three replicas, three components have been taken backup. The component name is `dump-pod-<ordinal-value>`. The ordinal value in the component's name represents the ordinal value of the StatefulSet pod ordinal.

## Populate Volumes

This section will show you how to populate the volumes of a `StatefulSet` with data from the `Snapshot` of the previous backup using KubeStash.

**Deploy StatefulSet :**

Now, we need to create a new `StatefulSet` along with a PersistentVolumeClaim (PVC) using `VolumeClaimTemplates`. This PVC configure with `spec.dataSourceRef` pointing to our `Snapshot` object. KubeStash will populate volume with the restored data from pointing snapshot and attach it to corresponding PVCs. As a result, this PVCs will contain the data that has been restored.

Below is the YAML of the restored StatefulSet,

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sample-restored-sts
  namespace: demo
  labels:
    app: demo-busybox
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-busybox
  serviceName: busybox
  template:
    metadata:
      labels:
        app: demo-busybox
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh", "-c","sleep 3000"]
          volumeMounts:
            - name: restored-source-data
              mountPath: "/source/data"
          imagePullPolicy: IfNotPresent
  volumeClaimTemplates:
    - metadata:
        name: restored-source-data
        annotations:
          populator.kubestash.com/app-name: sample-restored-sts
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 256Mi
        dataSourceRef:
          apiGroup: storage.kubestash.com
          kind: Snapshot
          name: gcs-demo-repo-sample-backup-sts-demo-session-1707900900
```
Here,
- `spec.dataSourceRef` specifies that which `snapshot` we want to use for restoring and populating the volume. We have referenced the `Snapshot` object that was backed up in the previous section.
- `metadata.annotations.populator.kubestash.com/app-name` field is mandatory for any volume population of a StatefulSet through KubeStash.
  - This field denotes the StatefulSet that will be attached those volumes via mount paths. The volume population will only be successful if the mount path of this volume matches the mount paths of the backup StatefulSet.

Let's create the StatefulSet we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/volume-populator/statefulset/examples/restore-statefulset.yaml
statefulset.apps/sample-restored-sts created
```

**Wait for Populate Volume:**

When `StatefulSet` create `PVC` with `spec.dataSourceRef` that refers our `Snapshot` object for each replica, KubeStash automatically creates a populator Job. Now, just wait for the volume population process to finish.

You can watch the `PVCs` status using the following command,

```bash
$ watch kubectl get pvc -n demo 

Every 2.0s: kubectl get pvc -n demo                                                                   anisur: Tue Feb 13 18:37:26 2024

NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
restored-source-data-sample-restored-sts-0   Bound    pvc-1fb1df97-6c36-4dd3-9b30-be5307abba60   1Gi        RWO            standard-rwo   2m46s
restored-source-data-sample-restored-sts-1   Bound    pvc-4726726d-7f9d-4c67-8aa2-57615b88a989   1Gi        RWO            standard-rwo   2m25s
restored-source-data-sample-restored-sts-2   Bound    pvc-df60d50f-9697-4d2f-a6e2-9e9c7ea89524   1Gi        RWO            standard-rwo   2m3s
```

The output of the command shows the `PVCs` status as Bound, indicating successful completion of the volume population.

**Verify Restored Data :**

We are going to exec any pod of `sample-restored-sts` StatefulSet to verify whether the volume population with the backed up data has been restored successfully.

Now, wait for the StatefulSet pod to go into the `Running` state.

```bash
$ kubectl get pods -n demo 
NAME                    READY   STATUS    RESTARTS   AGE
sample-restored-sts-0   1/1     Running   0          95s
sample-restored-sts-1   1/1     Running   0          65s
sample-restored-sts-2   1/1     Running   0          41s
```

Verify that the backed up data has been restored into `/source/data` directory of above pod using the following command,

```bash
$ kubectl exec -it -n demo sample-restored-sts-0  -- cat /source/data/data.txt
sample-sts-0

$ kubectl exec -it -n demo sample-restored-sts-1  -- cat /source/data/data.txt
sample-sts-1

$ kubectl exec -it -n demo sample-restored-sts-2  -- cat /source/data/data.txt
sample-sts-2
```

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete backupconfiguration -n demo sample-backup-sts
kubectl delete backupstorage -n demo gcs-storage
kubectl delete retentionPolicy -n demo demo-retention
kubectl delete secret -n demo gcs-secret
kubectl delete secret -n demo encrypt-secret
kubectl delete statefulset -n demo sample-sts
kubectl delete statefulset -n demo sample-restored-sts
kubectl delete pvc -n demo --all
```