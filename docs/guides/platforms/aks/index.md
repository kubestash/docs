---
title: AKS | KubeStash
description: Using KubeStash in Azure AD Workload Identity
menu:
  docs_{{ .version }}:
    identifier: platforms-aks
    name: AKS
    parent: platforms
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Using KubeStash with Azure AD Workload Identity

This guide will show you how to use KubeStash to backup and restore volumes of a Kubernetes workload running in [Azure Kubernetes Service (AKS)](https://azure.microsoft.com/en-us/services/kubernetes-service/) with Azure AD Workload Identity. Here, we are going to backup a volume of a Deployment into [Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/). Then, we are going to show how to restore this backed up data.

## Before You Begin

- At first, you need to have an AKS cluster with [Azure AD Workload Identity](https://azure.github.io/azure-workload-identity/docs/introduction.html) enabled. If you don't already have a cluster, create one from [here](https://azure.microsoft.com/en-us/services/kubernetes-service/).

- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md). You need to provide some labels and annotations during installation, see [here](#prepare-kubestash-operator).

- You should be familiar with the following `KubeStash` concepts:
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
- You will need access to [Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/) to store the backup snapshots.

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

## Create User-assigned Managed Identity

You need to create an AAD application or user-assigned managed identity and grant permissions to access the Azure Blob Storage. To create a user-assigned managed identity run the following command:

```bash
$ export RESOURCE_GROUP=<resource-group-name>
$ export USER_ASSIGNED_IDENTITY_NAME=<user-assigned-identity-name>
$ az identity create --name $USER_ASSIGNED_IDENTITY_NAME --resource-group $RESOURCE_GROUP
```

We need to assign `Storage Blob Data Contributor` role to user-assigned managed identity by running the following commands,

```bash
$ export STORAGE_NAME=<your-blob-storage-name>
$ export STORAGE_ID=(az storage account show --name $STORAGE_NAME --resource-group $RESOURCE_GROUP --query id -o tsv)
$ export USER_ASSIGNED_IDENTITY_CLIENT_ID=(az identity show --name $USER_ASSIGNED_IDENTITY_NAME --resource-group $RESOURCE_GROUP --query 'clientId' -otsv)
$ az role assignment create \
   --assignee $USER_ASSIGNED_IDENTITY_CLIENT_ID \
   --role 'Storage Blob Data Contributor' \
   --scope $STORAGE_ID
```

## Prepare KubeStash Operator

During KubeStash installation in Azure AD Workload Identity cluster, you need to provide some labels and annotations described [here](https://azure.github.io/azure-workload-identity/docs/topics/service-account-labels-and-annotations.html). Here, we have installed KubeStash providing the required pod label `azure.workload.identity/use: "true"` and service account annotation `azure.workload.identity/client-id: <user-assigned-managed-identity-client-ID>` by running the following commands:

```bash
$ export USER_ASSIGNED_IDENTITY_CLIENT_ID=(az identity show --name $USER_ASSIGNED_IDENTITY_NAME --resource-group $RESOURCE_GROUP --query 'clientId' -otsv)
$  helm install kubestash oci://ghcr.io/appscode-charts/kubestash \
      --version <kubestash-version> \
      --namespace <kubestash-namespace> --create-namespace \
      --set-file global.license=/path/to/the/license.txt \
      --set-string kubestash-operator.podLabels."azure\\.workload\\.identity/use"="true" \
      --set-string kubestash-operator.serviceAccount.annotations."azure\\.workload\\.identity/client-id"=$USER_ASSIGNED_IDENTITY_CLIENT_ID \
      --wait --burst-limit=10000 --debug
```

Now, we are going to create identity federated credential for our KubeStash operator's service account,

```bash
$ export SERVICE_ACCOUNT_ISSUER=(az aks show -n $CLUSTER_NAME -g $RESOURCE_GROUP --query "oidcIssuerProfile.issuerUrl" -otsv)
$ az identity federated-credential create \
     --name "operator-cred" \
     --identity-name $USER_ASSIGNED_IDENTITY_NAME \
     --resource-group $RESOURCE_GROUP \
     --issuer $SERVICE_ACCOUNT_ISSUER \
     --subject system:serviceaccount:kubestash:kubestash-kubestash-operator
```

## Prepare Deployment

Here, we are going to deploy a Deployment with a PVC. This Deployment will automatically generate some sample data into the PVC. Then, we are going to backup this sample data using KubeStash.

### Create Deployment

At first, let's deploy the workload whose volumes we are going to backup. Here, we are going create a PVC and deploy a Deployment with this PVC.

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/aks/examples/pvc.yaml
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/aks/examples/deployment.yaml
deployment.apps/kubestash-demo created
```

Now, wait for the pod of the Deployment to go into the `Running` state.

```bash
$ kubectl get pod -n demo
NAME                              READY   STATUS    RESTARTS   AGE
kubestash-demo-77f9c4cb8c-4l26t   1/1     Running   0          3m25s
```

To verify that the sample data has been created in `/source/data` directory, use the following command:

```bash
$ kubectl exec -it -n demo kubestash-demo-77f9c4cb8c-4l26t -- cat /source/data/text.txt
dummy_data
```

From the above, we can see the sample data is set successfully.

## Prepare Backup

In this section, we are going to prepare the necessary resources before backup.

### Prepare ServiceAccount

We are going create a Kubernetes service account and attach annotation `azure.workload.identity/client-id: <user-assigned-managed-identity-client-ID>` to the service account.

Let's create a `ServiceAccount` in the `demo` namespace,

```bash
$ kubectl create serviceaccount -n demo bucket-user
serviceaccount/bucket-user created
```

Now, lets attach the annotation,

```bash
$ kubectl annotate sa -n demo bucket-user azure.workload.identity/client-id=$USER_ASSIGNED_IDENTITY_CLIENT_ID
```

Now, we are going to create identity federated credential for our newly created service account,

```bash
$ az identity federated-credential create \
     --name "demo-cred" \
     --identity-name $USER_ASSIGNED_IDENTITY_NAME \
     --resource-group $RESOURCE_GROUP \
     --issuer $SERVICE_ACCOUNT_ISSUER \
     --subject system:serviceaccount:demo:bucket-user
```

### Prepare Backend

Now we are going to store our backed up data into an [Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/). As we are using workload identity enabled cluster, we don't need the storage secret to access the azure blob storage.

**Create BackupStorage:**

Now, let's create a `BackupStorage` with the information of our desired azure blob storage. Below is the YAML of `BackupStorage` crd we are going to create,

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: BackupStorage
metadata:
  name: azure-storage
  namespace: demo
spec:
  storage:
    provider: azure
    azure:
      prefix: demo
      container: ishtiaq
      storageAccount: kubestash
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true 
  deletionPolicy: WipeOut 
  runtimeSettings:
    pod:
      podLabels: 
        azure.workload.identity/use: "true"
      serviceAccountName: bucket-user
```

Notice the `spec.runtimeSettings`, here we have to provide the label `azure.workload.identity/use: "true"` in `podLabes` and have to provide the `serviceAccountName`. These values will be set in the cleaner job created by the KubeStash operator. To learn more about `spec.runtimeSettings`, visit [here](/docs/concepts/crds/backupstorage#runtimeSettings).

Let's create the `BackupStorage` we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/aks/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/azure-storage created
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/aks/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

## Backup

To schedule a backup, we have to create a `BackupConfiguration` object targeting the respective Deployment. Then KubeStash will create a CronJob for each session to periodically backup the Deployment.

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

Below is the `YAML` for BackupConfiguration object we are going to use to backup the `kubestash-demo` Deployment we have deployed earlier,

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
    - name: azure-backend
      storageRef:
        name: azure-storage
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
        - name: azure-demo-repo
          backend: azure-backend
          directory: /dep
          encryptionSecret:
            name: encry-secret
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
          metadata:
            labels:
              azure.workload.identity/use: "true"
          spec:
            serviceAccountName: bucket-user
      retryConfig:
        maxRetry: 2
        delay: 1m
```

Here, `spec.sessions[*].addon.jobTemplate.spec.serviceAccountName` refers to the name of the `ServiceAccount` to use in the backup job and `spec.sessions[*].addon.jobTemplate.metadata.labels` refers to the labels to use in the backup job.

Let's create the `BackupConfiguration` crd we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/aks/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-backup-dep configured
```

**Verify Backup Setup Successful:**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let’s verify the Phase of the `BackupConfiguration`,

```bash
$ kubectl get backupconfiguration -n demo
NAME                PHASE   PAUSED   AGE
sample-backup-dep   Ready            3m
```

**Wait for BackupSession:**

Now, wait for a schedule to appear. Run the following command to watch for a `BackupSession` object,

```bash
$ watch kubectl get backupsession -n demo
Every 2.0s: kubectl get backupsession -n demo                           AppsCode-PC-03: Wed Jan 10 16:52:25 2024

NAME                                        INVOKER-TYPE          INVOKER-NAME        PHASE       DURATION   AGE
sample-backup-dep-demo-session-1705907281   BackupConfiguration   sample-backup-dep   Succeeded              6m
```

Here, the phase `Succeeded` means that the backup process has been completed successfully.

**Verify Backup:**

Now, we are going to verify whether the backed up data is present in the backend or not. Once a backup is completed, KubeStash will update the respective `Repository` object to reflect the backup completion. Check that the repository `azure-demo-repo` has been updated by the following command,

```bash
$ kubectl get repository -n demo azure-demo-repo
NAME              INTEGRITY   SNAPSHOT-COUNT   SIZE    PHASE   LAST-SUCCESSFUL-BACKUP   AGE
azure-demo-repo   true        1                801 B   Ready   8m                       9m
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run to a particular `Repository`.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=azure-demo-repo
NAME                                                        REPOSITORY        SESSION        SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
azure-demo-repo-sample-backup-dep-demo-session-1705907281   azure-demo-repo   demo-session   2024-01-22T07:08:07Z   Delete            Succeeded   29m
```

> When a backup is triggered according to schedule, KubeStash will create a `Snapshot` with the following labels  `kubestash.com/app-ref-kind: <workload-kind>`, `kubestash.com/app-ref-name: <workload-name>`, `kubestash.com/app-ref-namespace: <workload-namespace>` and `kubestash.com/repo-name: <repository-name>`. We can use these labels to watch only the `Snapshot` of our desired Workload or `Repository`.

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the Deployment.

```bash
$ kubectl get snapshots -n demo azure-demo-repo-sample-backup-dep-demo-session-1705907281 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: Deployment
    kubestash.com/app-ref-name: kubestash-demo
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: azure-demo-repo
  name: azure-demo-repo-sample-backup-dep-demo-session-1705907281
  namespace: demo
spec:
  ...
status:
  components:
    dump:
      driver: Restic
      duration: 1.474354769s
      integrity: true
      path: repository/v1/demo-session/dump
      phase: Succeeded
      resticStats:
        - hostPath: /source/data
          id: c5a88e95e476161b3594ffb2630513a4d3a59007419f618e2aae62995e118eca
          size: 11 B
          uploaded: 1.041 KiB
      size: 801 B
  ...
```
> For Deployment, KubeStash takes backup from only one pod of the Deployment. So, only one component has been taken backup.

Now, if we navigate to `dep/repository/v1/demo-session/` directory of our [Azure Blob Container](https://azure.microsoft.com/en-us/services/storage/blobs/), we are going to see that the component of the Deployment has been stored there.

> KubeStash keeps all backup data encrypted. So, the files in the bucket will not contain any meaningful data until they are decrypted.

## Restore

In this section, we are going to show you how to restore in the same Deployment which may be necessary when you have accidentally deleted any data.

**Simulate Disaster:**

Now, let's simulate an accidental deletion scenario. Here, we are going to exec into the Deployment pod `kubestash-demo-77f9c4cb8c-fc5qh` and delete the `text.txt` file from `/source/data`.

```bash
$ kubectl exec -it -n demo kubestash-demo-77f9c4cb8c-fc5qh -- sh
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
    repository: azure-demo-repo
    snapshot: latest
    encryptionSecret:
      name: encry-secret
      namespace: demo
  addon:
    name: workload-addon
    tasks:
      - name: logical-backup-restore
    jobTemplate:
      metadata:
        labels:
          azure.workload.identity/use: "true"
      spec:
        serviceAccountName: bucket-user
```

Here,

- `spec.addon.jobTemplate.spec.serviceAccountName` refers to the name of the `ServiceAccount` to use in the restore job(s).
- `spec.addon.jobTemplate.metadata.labels` refers to the labels to use in the restore job(s).
- `spec.dataSource.snapshot` specifies to restore from latest `Snapshot`.

Let's create the `RestoreSession` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/platforms/aks/examples/restoresession.yaml
restoresession.core.kubestash.com/sample-restore created
```

Once, you have created the `RestoreSession` object, KubeStash will create restore Job(s). Run the following command to watch the phase of the `RestoreSession` object,

```bash
$ watch kubectl get restoresession -n demo
Every 2.0s: kubectl get restores... AppsCode-PC-03: Wed Jan 10 17:13:18 2024

NAME             REPOSITORY        FAILURE-POLICY   PHASE       DURATION   AGE
sample-restore   azure-demo-repo                    Succeeded   3s         53s
```

The `Succeeded` phase means that the restore process has been completed successfully.

**Verify Restored Data:**

Now, lets exec into the Deployment pod and verify whether actual data was restored or not,

```bash
$ kubectl exec -it -n demo kubestash-demo-77f9c4cb8c-fc5qh -- cat /source/data/text.txt
dummy_data
```

Hence, we can see from the above output that the deleted data has been restored successfully from the backup.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo deployment kubestash-demo
kubectl delete -n demo backupconfiguration sample-backup-dep
kubectl delete -n demo restoresession sample-restore
kubectl delete -n demo backupstorage azure-storage
kubectl delete -n demo secret encry-secret
kubectl delete -n demo pvc --all
```

## Next Steps

1. See a step-by-step guide to backup/restore volumes of a StatefulSet [here](/docs/guides/workloads/statefulset/index.md).
2. See a step-by-step guide to backup/restore volumes of a DaemonSet [here](/docs/guides/workloads/daemonset/index.md).
3. See a step-by-step guide to Backup/restore Stand-alone PVC [here](/docs/guides/volumes/pvc/index.md)
