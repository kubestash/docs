---
title: Using Workload Identity with KubeStash on GKE
description: A guide on how to use GKE workload identity with KubeStash
menu:
  docs_{{ .version }}:
    identifier: platforms-gke
    name: GKE Workload Identity
    parent: platforms
    weight: 18
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Using Workload Identity with Stash on Google Kubernetes Engine (GKE)

This guide will show you how to use workload identity of [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/) with KubeStash. Here, we are going to backup a StatefulSet and store the backed up data into a [GCS Bucket](https://cloud.google.com/storage/). Then, we are going to show how to restore this backed up data.

## Before You Begin

- At first, you need to have a Kubernetes cluster in the Google Cloud Platform with [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) enabled. If you don’t already have a cluster, create one from [here](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#console).

- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).
- You should be familiar with the following `KubeStash` concepts:
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [Snapshot](/docs/concepts/crds/snapshot/index.md)
- Install Google Cloud CLI following the steps [here](https://cloud.google.com/sdk/docs/install).
- You will need a [GCS Bucket](https://console.cloud.google.com/storage/).

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

## Prepare IAM Service Account

At first, let's create a IAM service account which will contain the roles for accessing GCS Bucket,

```bash
$ gcloud iam service-accounts create bucket-accessor \
    --project=sample-project
```

Let's add the required roles to this service account for accessing the GCS bucket.

```bash
$ gcloud projects add-iam-policy-binding sample-project \
    --member "serviceAccount:bucket-accessor@sample-project.iam.gserviceaccount.com" \
    --role "roles/storage.objectAdmin"
```

```bash
$ gcloud projects add-iam-policy-binding sample-project \
    --member "serviceAccount:bucket-accessor@sample-project.iam.gserviceaccount.com" \
    --role "roles/storage.admin"
```

Also ensure that the required IAM roles for Workload Identity is added.

## Prepare KubeStash Operator

In this section, we are going to bind the Kubernetes service account for KubeStash operator and the IAM service account `bucket-accessor` that we have created earlier. This binding allows the Kubernetes service account to act as the IAM service account.

We have installed KubeStash in `kubestash` namespace. Let’s add the IAM annotations to the `ServiceAccount`,

```bash
$ kubectl annotate sa -n kubestash kubestash-kubestash-operator iam.gke.io/gcp-service-account="bucket-accessor@sample-project.iam.gserviceaccount.com" 
serviceaccount/kubestash-kubestash-operator annotated
```

Now, let's bind it with the IAM service account,

```bash
$ gcloud iam service-accounts add-iam-policy-binding bucket-accessor@sample-project.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:sample-project.svc.id.goog[kubestash/kubestash-kubestash-operator]"
```

> For a Standard cluster, you might require a `nodeSelector` for the pod that uses workload identity to connect to the backend. However, for Autopilot clusters, there's no need to set the nodeSelector. Autopilot rejects this because all nodes utilize Workload Identity.
>
> To set the `nodeSelector` for the operator pod, you can use the flag `--set-string kubestash-operator.nodeSelector."iam.gke.io/gke-metadata-server-enabled"="true"` when installing or upgrading KubeStash via Helm.

## Prepare StatefulSet 

In this section, we are going to create a StatefulSet with three replicas. We are going to configure the StatefulSet to generate sample data in each replica.

### Create StatefulSet

At first, let’s create a StatefulSet named `sample-sts`,

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
          command: ["/bin/sh", "-c","echo $(POD_NAME) > /source/data/pod-name.txt && sleep 3000"]
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

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/gke/examples/statefulset.yaml
service/busybox created
statefulset.apps/sample-sts created
```

Now, wait for the StatefulSet pod `sample-sts-0` to go into Running state,

```bash
$ kubectl get pod -n demo sample-sts-0
NAME               READY   STATUS    RESTARTS   AGE
sample-sts-0       1/1     Running   0          29m
```

Once the pod is in Running state, verify that the StatefulSet pod has sample data in it.

```bash
$ kubectl exec -it -n demo sample-sts-0 -- cat /source/data/pod-name.txt
sample-sts-0
```

From the above, we can see the sample data is set successfully.

## Prepare Backup

In this section, we are going to prepare the necessary resources before backup.

### Prepare ServiceAccount

We are going create a Kubernetes service account and bind it with the IAM service account `bucket-accessor` that we have created earlier. This binding allows the Kubernetes service account to act as the IAM service account.

Let's create a `ServiceAccount` in the `demo` namespace,

```bash
$ kubectl create serviceaccount -n demo bucket-user
serviceaccount/bucket-user created
```

Let's add the IAM annotations to the `ServiceAccount`,

```bash
$ kubectl annotate sa -n demo bucket-user iam.gke.io/gcp-service-account="bucket-accessor@sample-project.iam.gserviceaccount.com"
serviceaccount/bucket-user annotated
```

Now Let's bind it with the IAM service account,

```bash
$ gcloud iam service-accounts add-iam-policy-binding bucket-accessor@sample-project.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:sample-project.svc.id.goog[demo/bucket-user]"
```

> If `BackupStorage`, `BackupConfiguration` and `RestoreSession` are in different namespaces, we need to create a Kubernetes service account in each namespace and bind those with the IAM service account `bucket-accessor` that we have created earlier. Here we are using `demo` namespace for all of these objects, so only one service account is enough.

### Prepare Backend

Now we are going to store our backed up data into a [GCS bucket](https://cloud.google.com/storage/). As we are using workload identity enabled cluster, we don't need the storage secret to access the GCS bucket.

**Create BackupStorage:**

Now, let's create a `BackupStorage` with the information of our desired GCS bucket. Below is the YAML of `BackupStorage` CR we are going to create,

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
      bucket: kubestash
      prefix: demo
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true
  deletionPolicy: WipeOut
  runtimeSettings:
    pod:
      serviceAccountName: bucket-user
```

Let's create the `BackupStorage` we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/gke/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-storage created
```

Now, we are ready to backup our sample data into this backend.

> For a Standard cluster, you might require a `nodeSelector` for the backend cleaner jobs. You can set this in the `spec.runtimeSettings.nodeSelector` field of the `BackupStorage` object.

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/gke/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

## Backup

To schedule a backup, we have to create a `BackupConfiguration` object targeting the respective StatefulSet. Then KubeStash will create a CronJob for each session to periodically backup the StatefulSet.

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

Below is the `YAML` for BackupConfiguration object we are going to use to backup the `sample-sts` StatefulSet we have deployed earlier,

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
        jobTemplate:
          spec:
            serviceAccountName: bucket-user
      retryConfig:
        maxRetry: 2
        delay: 1m
```

Here, `spec.sessions[*].addon.jobTemplate.spec.serviceAccountName` refers to the name of the `ServiceAccount` to use in the backup job(s).

Let's create the `BackupConfiguration` CR we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/gke/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-backup-sts created
```

> For a Standard cluster, you might require a `nodeSelector` for the Backup job. You can set this in the `spec.sessions[*].addon.jobTemplate.spec.nodeSelector` field of the `BackupConfiguration`.

**Verify Backup Setup Successful:**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let’s verify the Phase of the `BackupConfiguration`,

```bash
$ kubectl get backupconfiguration -n demo
NAME                PHASE   PAUSED   AGE
sample-backup-sts   Ready            53m
```
Additionally, we can verify that the `Repository` specified in the `BackupConfiguration` has been created using the following command,

```bash
kubectl get repo -n demo
NAME               INTEGRITY   SNAPSHOT-COUNT   SIZE     PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-demo-repo                  0                0 B      Ready                            3m
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the `Repository` YAML stored in the `demo/demo/sample-sts` directory.

**Wait for BackupSession:**

Now, wait for a schedule to appear. Run the following command to watch for a `BackupSession` object,

```bash
$ watch kubectl get backupsession -n demo
Every 2.0s: kubectl get backupsession -n demo                           AppsCode-PC-03: Wed Jan 10 16:52:25 2024

NAME                                        INVOKER-TYPE          INVOKER-NAME        PHASE       DURATION   AGE
sample-backup-sts-demo-session-1704880082   BackupConfiguration   sample-backup-sts   Succeeded              63m
```

Here, the phase `Succeeded` means that the backup process has been completed successfully.

**Verify Backup:**

Now, we are going to verify whether the backed up data is present in the backend or not. Once a backup is completed, KubeStash will update the respective `Repository` object to reflect the backup completion. Check that the repository `gcs-demo-repo` has been updated by the following command,

```bash
$ kubectl get repository -n demo gcs-demo-repo
NAME            INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-demo-repo   true        1                2.348 KiB   Ready   65m                      66m
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-demo-repo
NAME                                                      REPOSITORY      SESSION        SNAPSHOT-TIME          DELETION-POLICY   PHASE       VERIFICATION-STATUS   AGE
gcs-demo-repo-sample-backup-sts-demo-session-1704880082   gcs-demo-repo   demo-session   2024-01-10T09:48:09Z   Delete            Succeeded                         68m
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
$ kubectl get snapshots -n demo gcs-demo-repo-sample-backup-sts-demo-session-1704880082 -oyaml
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
  name: gcs-demo-repo-sample-backup-sts-demo-session-1704880082
  namespace: demo
spec:
  ...
status:
  components:
    dump-pod-0:
      driver: Restic
      duration: 1.61162906s
      integrity: true
      path: repository/v1/demo-session/pod-0
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
      path: repository/v1/demo-session/pod-1
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
      path: repository/v1/demo-session/pod-2
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        id: 9dc9efd5e9adfd0154eca48433cc57aa09bca018d970e9530769326c9783905c
        size: 13 B
        uploaded: 1.046 KiB
      size: 798 B
  ...
```
> For StatefulSet, KubeStash takes backup from every pod of the StatefulSet. Since we are using three replicas, three components have been backed up. For logical backup, KubeStash uses `dump-pod-<ordinal-value>` as the component name where `<ordinal-value>` corresponds to the pod's ordinal number for the StatefulSet.

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/demo/sample-sts/repository/v1/demo-session/dump-pod-<ordinal-value>` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/demo/sample-sts/snapshots` directory.

> Note: KubeStash stores all dumped data encrypted in the backup directory, meaning it remains unreadable until decrypted.

## Restore

In this section, we are going to show you how to restore in the same StatefulSet which may be necessary when you have accidentally deleted any data.

**Simulate Disaster:**

Now, let's simulate an accidental deletion scenario. Here, we are going to exec into the StatefulSet pod `sample-sts-0` and delete the `pod-name.txt` file from `/source/data`.

```bash
$ kubectl exec -it -n demo sample-sts-0 -- sh
/ # 
/ # rm /source/data/pod-name.txt
/ # cat /source/data/pod-name.txt
cat: can't open '/source/data/pod-name.txt': No such file or directory
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
    jobTemplate:
      spec:
        serviceAccountName: bucket-user
```

Here,

- `spec.addon.jobTemplate.spec.serviceAccountName` refers to the name of the `ServiceAccount` to use in the restore job(s).
- `spec.dataSource.snapshot` specifies to restore from latest `Snapshot`.
- `spec.dataSource.components` refers to the components that we want to restore. Here we want to restore data to `pod-0`, as we only deleted data from `sample-sts-0`.

> For a Standard cluster, you might require a `nodeSelector` for the Backup job. You can set this in the `spec.addon.jobTemplate.spec.nodeSelector` field of the `RestoreSession`.

Let's create the `RestoreSession` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/gke/examples/restoresession.yaml
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
$ kubectl exec -it -n demo sample-sts-0 -- cat /source/data/pod-name.txt
sample-sts-0
```

Hence, we can see from the above output that the deleted data has been restored successfully from the backup.

### Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo backupconfiguration sample-backup-sts
kubectl delete -n demo restoresession sample-restore
kubectl delete -n demo secret encrypt-secret
kubectl delete -n demo backupstorage gcs-storage
kubectl delete -n demo sts sample-sts
kubectl delete -n demo sa bucket-user
kubectl delete ns demo
```
