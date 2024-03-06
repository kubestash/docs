---
title: Backup and Restore Volumes of a StatefulSet | KubeStash
description: A step by step guide showing how to backup and restore volumes of a StatefulSet.
menu:
  docs_{{ .version }}:
    identifier: workload-statefulset
    name: Backup & Restore Volumes of a StatefulSet
    parent: workload
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup and Restore Volumes of a StatefulSet

This guide will show you how to use KubeStash to backup and restore volumes of a StatefulSet.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).

- You should be familiar with the following `KubeStash` concepts:
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [Snapshot](/docs/concepts/crds/snapshot/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/workloads/statefulset/examples](/docs/guides/workloads/statefulset/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Backup Volumes of a StatefulSet

This section will show you how to use KubeStash to backup volumes of a StatefulSet. Here, we are going to deploy a StatefulSet with a PVC and generate some sample data in it. Then, we are going to backup this sample data using KubeStash.

**Deploy StatefulSet:**

At first, We are going to deploy a StatefulSet. This StatefulSet will automatically generate sample data in `/source/data` directory.

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

Let's create the StatefulSet we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/statefulset/examples/statefulset.yaml
service/busybox created
statefulset.apps/sample-sts created
```

Now, wait for the pods of the StatefulSet to go into the `Running` state.

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

We are going to store our backed up data into a GCS bucket. We have to create a Secret with necessary credentials and a `BackupStorage` CR to use this backend. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

**Create Secret:**

Let's create a secret called `gcs-secret` with access credentials to our desired GCS bucket,

```bash
$ echo -n '<your-project-id>' > GOOGLE_PROJECT_ID
$ cat /path/to/downloaded-sa-key.json > GOOGLE_SERVICE_ACCOUNT_JSON_KEY
$ kubectl create secret generic -n demo gcs-secret \
    --from-file=./GOOGLE_PROJECT_ID \
    --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
secret/gcs-secret created
```

**Create BackupStorage:**

Now, create a `BackupStorage` using this secret. Below is the YAML of `BackupStorage` CR we are going to create,

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

Let's create the BackupStorage we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/statefulset/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-storage created
```

Now, we are ready to backup our sample data into this backend.

**Create RetentionPolicy:**

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/statefulset/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

### Backup

We have to create a `BackupConfiguration` CR targeting the `sample-sts` StatefulSet that we have deployed earlier. KubeStash will create a `CronJob` for each session to take periodic backup of `/source/data` directory of the target.

At first, we need to create a secret with a Restic password for backup data encryption.

**Create Secret:**

Let's create a secret called `encrypt-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD \
secret "encrypt-secret" created
```

**Create BackupConfiguration:**

Below is the YAML of the `BackupConfiguration` CR that we are going to create,

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
          directory: /demo/sample-sts
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: workload-addon
        tasks:
          - name: logical-backup
            targetVolumes:
              volumeMounts:
                - name: source-data
                  mountPath: /source/data
            params:
              paths: /source/data
              exclude: /source/data/lost+found
      retryConfig:
        maxRetry: 2
        delay: 1m
```

Let's create the `BackupConfiguration` CR we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/statefulset/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-backup-sts created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME                PHASE   PAUSED   AGE
sample-backup-sts   Ready            2m50s
```

Additionally, we can verify that the `Repository` specified in the `BackupConfiguration` has been created using the following command,

```bash
kubectl get repo -n demo
NAME               INTEGRITY   SNAPSHOT-COUNT   SIZE     PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-demo-repo                  0                0 B      Ready                            3m
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the `Repository` YAML stored in the `demo/demo/sample-sts` directory.

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` CR.

Verify that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                     SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-sample-backup-sts-demo-session   */2 * * * *             0        2m45s           3m25s
```

**Wait for BackupSession:**

Wait for the next schedule for backup. Run the following command to watch `BackupSession` CR,

```bash
$ kubectl get backupsession -n demo -w
NAME                                        INVOKER-TYPE          INVOKER-NAME        PHASE       DURATION   AGE
sample-backup-sts-demo-session-1706015400   BackupConfiguration   sample-backup-sts   Succeeded              7m22s
```

We can see from the above output that the backup session has succeeded. Now, we are going to verify whether the backed up data has been stored in the backend.

**Verify Backup:**

Once a backup is complete, KubeStash will update the respective `Repository` CR to reflect the backup. Check that the repository `gcs-demo-repo` has been updated by the following command,

```bash
$ kubectl get repository -n demo gcs-demo-repo
NAME              INTEGRITY   SNAPSHOT-COUNT   SIZE    PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-demo-repo     true        1                806 B   Ready   8m27s                    9m18s
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-demo-repo
NAME                                                      REPOSITORY      SESSION        SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-demo-repo-sample-backup-sts-demo-session-1706015400   gcs-demo-repo   demo-session   2024-01-23T13:10:54Z   Delete            Succeeded   16h
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <workload-kind>`
> - `kubestash.com/app-ref-name: <workload-name>`
> - `kubestash.com/app-ref-namespace: <workload-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
> 
> These labels can be used to watch only the `Snapshot`s related to our desired Workload or `Repository`. 

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the StatefulSet.

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
> For StatefulSet, KubeStash takes backup from every pod of the StatefulSet. Since we are using three replicas, three components are present in the Snapshot. KubeStash uses `dump-pod-<ordinal-value>` as the component name for StatefulSet. The ordinal value in the component's name represents the ordinal value of the StatefulSet pod ordinal.

Now, if we navigate to GCS bucket, we will see the backed up data stored in the `demo/demo/sample-sts/repository/v1/demo-session/dump-pod-<ordinal-value>` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/demo/sample-sts/snapshots` directory.

> KubeStash keeps all the dumped data encrypted in the backup directory meaning the dumped files won't contain any readable data until decryption.

## Restore

In this section, we are going to show you how to restore in the same StatefulSet which may be necessary when you have accidentally deleted any data.

**Simulate Disaster:**

Now, let's simulate an accidental deletion scenario. Here, we are going to exec into the StatefulSet pod `sample-sts-0` and delete the `data.txt` file from `/source/data`.

```bash
$ kubectl exec -it -n demo sample-sts-0 -- sh
/ # 
/ # rm /source/data/data.txt
/ # cat /source/data/data.txt
cat: can't open '/source/data/data.txt': No such file or directory
/ # exit
```

**Create RestoreSession:**

To restore the StatefulSet, you have to create a `RestoreSession` object pointing to the StatefulSet.

Here, is the YAML of the `RestoreSession` object that we are going to use for restoring our `sample-sts` StatefulSet.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: sample-restore
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: StatefulSet
    name: sample-sts
    namespace: demo
  dataSource:
    repository: gcs-demo-repo
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret
      namespace: demo
    components:
      - dump-pod-0
  addon:
    name: workload-addon
    tasks:
      - name: logical-backup-restore
```

Here,

- `spec.dataSource.snapshot` specifies to restore from latest `Snapshot`.
- `spec.dataSource.components` refers to the components that we want to restore. Here we want to restore data to `dump-pod-0`, as we only deleted data from `sample-sts-0`.

Let's create the `RestoreSession` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/statefulset/examples/restoresession.yaml
restoresession.core.kubestash.com/sample-restore created
```

Once, you have created the `RestoreSession` object, KubeStash will create restore Job(s). Run the following command to watch the phase of the `RestoreSession` object,

```bash
$ watch kubectl get restoresession -n demo
Every 2.0s: kubectl get restores... AppsCode-PC-03: Wed Jan 10 17:13:18 2024

NAME             REPOSITORY      FAILURE-POLICY   PHASE       DURATION   AGE
sample-restore   gcs-demo-repo                    Succeeded   3s         53s
```

The `Succeeded` phase means that the restore process has been completed successfully.

**Verify Restored Data:**

Now, lets exec into the StatefulSet pod and verify whether actual data was restored or not,

```bash
$ kubectl exec -it -n demo sample-sts-0 -- cat /source/data/data.txt
sample-sts-0
```

Hence, we can see from the above output that the deleted data has been restored successfully from the backup.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo statefulset sample-sts
kubectl delete -n demo backupconfiguration sample-backup-sts
kubectl delete -n demo restoresession sample-restore
kubectl delete -n demo backupstorage gcs-storage
```
