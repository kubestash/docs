---
title: Backup and Restore Volumes of a DaemonSet | KubeStash
description: A step by step guide showing how to backup and restore volumes of a DaemonSet.
menu:
  docs_{{ .version }}:
    identifier: workload-daemonset
    name: Backup & Restore Volumes of a DaemonSet
    parent: workload
    weight: 40
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup and Restore Volumes of a DaemonSet

This guide will show you how to use KubeStash to backup and restore volumes of a DaemonSet.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).

- You should be familiar with the following `KubeStash` concepts:
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/workloads/daemonset/examples](/docs/guides/workloads/daemonset/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Backup Volumes of a DaemonSet

This section will show you how to use KubeStash to backup volumes of a DaemonSet. Here, we are going to deploy a DaemonSet and generate some sample data in it. Then, we are going to backup this sample data using KubeStash.

**Deploy DaemonSet:**

At first, we are going to deploy a DaemonSet. This DaemonSet will automatically generate sample data (`data.txt` file) in `/source/data` directory.

Below is the YAML of the DaemonSet that we are going to create,

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: ks-demo
  name: ks-demo
  namespace: demo
spec:
  selector:
    matchLabels:
      app: ks-demo
  template:
    metadata:
      labels:
        app: ks-demo
      name: busybox
    spec:
      containers:
        - args: ["echo sample_data > /source/data/data.txt && sleep 3000"]
          command: ["/bin/sh", "-c"]
          image: busybox
          imagePullPolicy: IfNotPresent
          name: busybox
          volumeMounts:
            - mountPath: /source/data
              name: source-data
      restartPolicy: Always
      volumes:
        - name: source-data
          hostPath:
            path: /kubestash/source/data
```

Let's create the DaemonSet we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/daemonset/examples/daemon.yaml
daemonset.apps/ks-demo created
```

Now, wait for the pod of the DaemonSet to go into the `Running` state.

```bash
$ kubectl get pod -n demo
NAME             READY   STATUS    RESTARTS   AGE
ks-demo-xrbfg    1/1     Running   0          39s
```

Verify that the sample data has been created in `/source/data` directory using the following command,

```bash
$ kubectl exec -n demo ks-demo-xrbfg -- cat /source/data/data.txt
sample_data
```

### Prepare Backend

We are going to store our backed up data into a GCS bucket. We have to create a Secret with necessary credentials and a BackupStorage crd to use this backend. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

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

Now, create a `BackupStorage` using this secret. Below is the YAML of `BackupStorage` crd we are going to create,

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/daemonset/examples/backupstorage.yaml
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
Notice the `spec.usagePolicy` that allows referencing the `RetentionPolicy` from all namespaces. To allow specific namespaces, we can configure it accordingly by following [RetentionPolicy usage policy](/docs/concepts/crds/retentionpolicy/index.md#retentionpolicy-spec).

Let’s create the above `RetentionPolicy`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/daemonset/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

### Backup

We have to create a `BackupConfiguration` crd targeting the `ks-demo` DaemonSet that we have deployed earlier. KubeStash will create a `CronJob` for each session to take periodic backup of `/source/data` directory of the target.

At first, we need to create a secret with a Restic password for backup data encryption.

**Create Secret:**

Let's create a secret called `encry-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encry-secret \
    --from-file=./RESTIC_PASSWORD \
secret "encry-secret" created
```

**Create BackupConfiguration:**

Below is the YAML of the `BackupConfiguration` crd that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-backup-daemon
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: DaemonSet
    name: ks-demo
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
          directory: /data/ks-demo
          encryptionSecret:
            name: encry-secret
            namespace: demo
      addon:
        name: workload-addon
        tasks:
          - name: logical-backup
            params:
              paths: /source/data
      retryConfig:
        maxRetry: 2
        delay: 1m
```

Let's create the `BackupConfiguration` crd we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/daemonset/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-backup-daemon created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME                   PHASE   PAUSED   AGE
sample-backup-daemon   Ready            2m50s
```

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` crd.

Verify that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                        SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-sample-backup-daemon-demo-session   */2 * * * *             0        2m45s           3m25s
```

**Wait for BackupSession:**

Wait for the next schedule for backup. Run the following command to watch `BackupSession` crd,

```bash
$ kubectl get backupsession -n demo -w
NAME                                           INVOKER-TYPE          INVOKER-NAME           PHASE       DURATION   AGE
sample-backup-daemon-demo-session-1706852520   BackupConfiguration   sample-backup-daemon   Succeeded              7m22s
```

We can see from the above output that the backup session has succeeded. Now, we are going to verify whether the backed up data has been stored in the backend.

**Verify Backup:**

Once a backup is complete, KubeStash will update the respective `Repository` crd to reflect the backup. Check that the repository `gcs-demo-repo` has been updated by the following command,

```bash
$ kubectl get repository -n demo gcs-demo-repo
NAME              INTEGRITY   SNAPSHOT-COUNT   SIZE    PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-demo-repo     true        1                806 B   Ready   8m27s                    9m18s
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run to a particular `Repository`.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-demo-repo
NAME                                                         REPOSITORY      SESSION        SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-demo-repo-sample-backup-daemon-demo-session-1706852520   gcs-demo-repo   demo-session   2024-01-23T13:10:54Z   Delete            Succeeded   16h
```

> When a backup is triggered according to schedule, KubeStash will create a `Snapshot` with the following labels  `kubestash.com/app-ref-kind: <workload-kind>`, `kubestash.com/app-ref-name: <workload-name>`, `kubestash.com/app-ref-namespace: <workload-namespace>` and `kubestash.com/repo-name: <repository-name>`. We can use these labels to watch only the `Snapshot` of our desired Workload or `Repository`.

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the DaemonSet.

```bash
$ kubectl get snapshots -n demo gcs-demo-repo-sample-backup-daemon-demo-session-1706852520 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: DaemonSet
    kubestash.com/app-ref-name: ks-demo
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-demo-repo
  name: gcs-demo-repo-sample-backup-daemon-demo-session-1706852520
  namespace: demo
spec:
  ...
status:
  components:
    dump-kind-control-plane:
      driver: Restic
      duration: 5.831892988s
      integrity: true
      path: repository/v1/demo-session/dump-kind-control-plane
      phase: Succeeded
      resticStats:
        - hostPath: /source/data
          id: 7f0d8dee7e27e869ffa3f4e2b4cf095d8e485d9c12dbceb5d8f5933e034cce0e
          size: 12 B
          uploaded: 1.043 KiB
      size: 805 B
  ...
```
> For DaemonSet, KubeStash takes backup from every pod of the DaemonSet. Since we have one node in our cluster, only one component has been taken backup. The component's name represents the nodes name.

Now, if we navigate to `demo/ks-demo/repository/v1/demo-session/` directory of our GCS bucket, we are going to see that the components of the DaemonSet have been stored there.

> KubeStash keeps all backup data encrypted. So, the files in the bucket will not contain any meaningful data until they are decrypted.

## Restore

This section will show you how to restore the backed up data from the backend we have taken in the earlier section.

**Deploy DaemonSet:**

We are going to create a new DaemonSet named `ks-recovered` and restore the backed up data inside it.

Below is the YAML of the DaemonSet that we are going to create,

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: rs-demo
  name: rs-demo
  namespace: demo
spec:
  selector:
    matchLabels:
      app: rs-demo
  template:
    metadata:
      labels:
        app: rs-demo
      name: busybox
    spec:
      containers:
        - image: busybox
          args:
            - sleep
            - "3600"
          imagePullPolicy: IfNotPresent
          name: busybox
          volumeMounts:
            - mountPath: /source/data
              name: source-data
      restartPolicy: Always
      volumes:
        - name: source-data
          hostPath:
            path: /kubestash/recovered/data
```

Let's create the DaemonSet we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/daemonset/examples/recovered_daemon.yaml
daemonset.apps/rs-demo created
```

**Create RestoreSession:**

Now, we need to create a `RestoreSession` crd targeting the `rs-demo` DaemonSet to restore the backed up data inside it.

Below is the YAML of the `RestoreSesion` crd that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: sample-restore
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: DaemonSet
    name: rs-demo
    namespace: demo
  dataSource:
    repository: gcs-demo-repo
    snapshot: latest
    encryptionSecret:
      name: encry-secret
      namespace: demo
  addon:
    name: workload-addon
    tasks:
      - name: logical-backup-restore
```

Let's create the `RestoreSession` crd we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/daemonset/examples/restoresession.yaml
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

Now, lets exec into the DaemonSet pod and verify whether actual data was restored or not,

```bash
$ kubectl exec -it -n demo rs-demo-hkxcd -- cat /source/data/data.txt
sample_data
```

Hence, we can see from the above output that the data has been restored successfully from the backup.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo daemonset ks-demo
kubectl delete -n demo daemonset rs-demo
kubectl delete -n demo backupconfiguration sample-backup-daemon
kubectl delete -n demo restoresession sample-restore
kubectl delete -n demo backupstorage gcs-storage
```
