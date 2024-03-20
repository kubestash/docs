---
title: Backup and Restore Volumes of a Deployment | KubeStash
description: A step by step guide showing how to backup and restore volumes of a Deployment.
menu:
  docs_{{ .version }}:
    identifier: workload-deployment
    name: Backup & Restore Volumes of a Deployment
    parent: workload
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup and Restore Volumes of a Deployment

This guide will show you how to use KubeStash to backup and restore volumes of a Deployment.

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

> **Note:** YAML files used in this tutorial are stored in [docs/guides/workloads/deployment/examples](/docs/guides/workloads/deployment/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Backup Volumes of a Deployment

This section will show you how to use KubeStash to backup volumes of a Deployment. Here, we are going to deploy a Deployment with a PVC and generate some sample data in it. Then, we are going to backup this sample data using KubeStash.

### Prepare Workload

At first, we are going to create a PVC then we are going to create a Deployment that will use this PVC.

**Create PVC:**

Below is the YAML of the sample PVC that we are going to create,

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kubestash-pvc
  namespace: demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Let's create the PVC we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/deployment/examples/pvc.yaml
persistentvolumeclaim/kubestash-pvc created
```

**Deploy Deployment:**

Now, we are going to deploy a Deployment that uses the above PVC. This Deployment will automatically generate sample data (`text.txt` file) in `/source/data` directory where we have mounted the PVC.

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
  replicas: 3
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
        - image: busybox
          command: ["/bin/sh", "-c","echo dummy_data > /source/data/text.txt && sleep 3000"]
          imagePullPolicy: IfNotPresent
          name: busybox
          volumeMounts:
            - mountPath: /source/data
              name: source-data
      restartPolicy: Always
      volumes:
        - name: source-data
          persistentVolumeClaim:
            claimName: kubestash-pvc
```

Let's create the Deployment we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/deployment/examples/deployment.yaml
deployment.apps/kubestash-demo created
```

Now, wait for the pods of the Deployment to go into the `Running` state.

```bash
$ kubectl get pod -n demo
NAME                              READY   STATUS    RESTARTS   AGE
kubestash-demo-67c46977f5-2xzqn   1/1     Running   0          12s
kubestash-demo-67c46977f5-hnbcg   1/1     Running   0          12s
kubestash-demo-67c46977f5-xllz2   1/1     Running   0          12s
```

Verify that the sample data has been created in `/source/data` directory using the following command,

```bash
$ kubectl exec -n demo kubestash-demo-67c46977f5-xllz2 -- cat /source/data/text.txt
dummy_data
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/deployment/examples/backupstorage.yaml
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/deployment/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

### Backup

We have to create a `BackupConfiguration` CR targeting the `kubestash-demo` Deployment that we have deployed earlier. Then, KubeStash will create a `CronJob` for each session to take periodic backup of `/source/data` directory of the target.

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
  name: sample-backup-dep
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name: kubestash-demo
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
          directory: /dep
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/deployment/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-backup-dep created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME                PHASE   PAUSED   AGE
sample-backup-dep   Ready            2m50s
```

Additionally, we can verify that the `Repository` specified in the `BackupConfiguration` has been created using the following command,

```bash
kubectl get repo -n demo
NAME               INTEGRITY   SNAPSHOT-COUNT   SIZE     PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-demo-repo                  0                0 B      Ready                            3m
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the `Repository` YAML stored in the `demo/dep` directory.

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` CR.

Verify that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                     SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-sample-backup-dep-demo-session   */2 * * * *             0        2m45s           3m25s
```

**Wait for BackupSession:**

Wait for the next schedule for backup. Run the following command to watch `BackupSession` CR,

```bash
$ kubectl get backupsession -n demo -w

NAME                                        INVOKER-TYPE          INVOKER-NAME        PHASE       DURATION   AGE
sample-backup-dep-demo-session-1706015400   BackupConfiguration   sample-backup-dep   Succeeded              7m22s
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
gcs-demo-repo-sample-backup-dep-demo-session-1706015400   gcs-demo-repo   demo-session   2024-01-23T13:10:54Z   Delete            Succeeded   16h
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <workload-kind>`
> - `kubestash.com/app-ref-name: <workload-name>`
> - `kubestash.com/app-ref-namespace: <workload-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
>
> These labels can be used to watch only the `Snapshot`s related to our desired Workload or `Repository`.

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the Deployment.

```bash
$ kubectl get snapshots -n demo gcs-demo-repo-sample-backup-dep-demo-session-1706015400 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: Deployment
    kubestash.com/app-ref-name: kubestash-demo
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-demo-repo
  name: gcs-demo-repo-sample-backup-dep-demo-session-1706015400
  namespace: demo
spec:
  ...
status:
  components:
    dump:
      driver: Restic
      duration: 7.534461497s
      integrity: true
      path: repository/v1/demo-session/dump
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        id: f28441a36b2167d64597d66d1046573181cad81aa8ff5b0998b64b31ce16f077
        size: 11 B
        uploaded: 1.049 KiB
      size: 806 B
  ...
```
> For Deployment, KubeStash takes backup from only one pod of the Deployment. So, only one component has been backed up. For logical backup, KubeStash uses `dump` as the component name for Deployment.

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/dep/repository/v1/demo-session/dump` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/dep/snapshots` directory.

> Note: KubeStash stores all dumped data encrypted in the backup directory, meaning it remains unreadable until decrypted.

## Restore

In this section, we are going to show you how to restore in the same Deployment which may be necessary when you have accidentally deleted any data.

**Simulate Disaster:**

Now, let's simulate an accidental deletion scenario. Here, we are going to exec into the Deployment pod `kubestash-demo-77f9c4cb8c-fc5qh` and delete the `text.txt` file from `/source/data`.

```bash
$ kubectl exec -it -n demo kubestash-demo-67c46977f5-xllz2 -- sh
/ # 
/ # rm /source/data/text.txt
/ # cat /source/data/text.txt
cat: can't open '/source/data/text.txt': No such file or directory
/ # exit
```

**Create RestoreSession:**

To restore the Deployment, you have to create a `RestoreSession` object pointing to the Deployment.

Here, is the YAML of the `RestoreSession` object that we are going to use for restoring our `kubestash-demo` Deployment.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: sample-restore
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name: kubestash-demo
    namespace: demo
  dataSource:
    repository: gcs-demo-repo
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret
      namespace: demo
  addon:
    name: workload-addon
    tasks:
      - name: logical-backup-restore
```

Here,

- `spec.dataSource.snapshot` specifies to restore from latest `Snapshot`.

Let's create the `RestoreSession` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/deployment/examples/restoresession.yaml
restoresession.core.kubestash.com/sample-restore created
```

Once, you have created the `RestoreSession` object, KubeStash will create restore Job(s). Run the following command to watch the phase of the `RestoreSession` object,

```bash
$ watch kubectl get restoresession -n demo
Every 2.0s: kubectl get restores... AppsCode-PC-03: Wed Jan 10 17:13:18 2024

NAME             REPOSITORY        FAILURE-POLICY   PHASE       DURATION   AGE
sample-restore   gcs-demo-repo                      Succeeded   3s         53s
```

The `Succeeded` phase means that the restore process has been completed successfully.

**Verify Restored Data:**

Now, lets exec into the Deployment pod and verify whether actual data was restored or not,

```bash
$ kubectl exec -n demo kubestash-demo-67c46977f5-xllz2 -- cat /source/data/text.txt
dummy_data
```

Hence, we can see from the above output that the deleted data has been restored successfully from the backup.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo deployment kubestash-demo
kubectl delete -n demo backupconfiguration sample-backup-dep
kubectl delete -n demo restoresession sample-restore
kubectl delete -n demo backupstorage gcs-storage
kubectl delete -n demo pvc --all
```
