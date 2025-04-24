---
title: Application level Backup and Restore | KubeStash
description: A step by step guide showing how to backup and restore an application along with related resources.
menu:
  docs_{{ .version }}:
    identifier: workload-statefulset
    name: Application level Backup & Restore
    parent: workload
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Application level Backup and Restore

This guide will show you how to use KubeStash to backup and restore an application along with related resources.

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

> **Note:** YAML files used in this tutorial are stored in [docs/guides/workloads/application/examples](/docs/guides/workloads/application/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Backup Application Along With Related Resources

This section will show you how to use KubeStash to backup backup an application along with related resources. Here, we are going to deploy a StatefulSet with a PVC and generate some sample data in it. Then, we are going to backup this sample data using KubeStash.

**Deploy The Application Along With Related Resources:**

At first, We are going to deploy an Application along with some related resources.

Below is the YAML of the Application that we are going to create,

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: demo
  labels:
    app: my-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  namespace: demo
  labels:
    app: my-app
data:
  app.properties: |
    greeting.message=Hello, World!
    app.version=1.0.0
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: demo
  labels:
    app: my-app
type: Opaque
data:
  username: dXNlcg==
  password: cGFzc3dvcmQ= 
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: demo
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-serviceaccount
  namespace: demo
  labels:
    app: my-app
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-role
  namespace: demo
  labels:
    app: my-app
rules:
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["roles","rolebindings"]
    verbs: ["create", "list", "get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-rolebinding
  namespace: demo
  labels:
    app: my-app
subjects:
  - kind: ServiceAccount
    name: my-serviceaccount
    namespace: demo
roleRef:
  kind: Role
  name: my-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-clusterrole
  labels:
    app: my-app
rules:
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["clusterroles", "clusterrolebindings"]
    verbs: ["create", "get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-clusterrolebinding
  labels:
    app: my-app
subjects:
  - kind: ServiceAccount
    name: my-serviceaccount
    namespace: demo
roleRef:
  kind: ClusterRole
  name: my-clusterrole
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-app
  namespace: demo
  labels:
    app: my-app
spec:
  serviceName: "my-service" 
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-serviceaccount
      containers:
        - name: my-container
          image: nginx:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
            - name: secret-volume
              mountPath: /etc/secret
            - name: storage-volume
              mountPath: /data
      volumes:
        - name: config-volume
          configMap:
            name: my-config
        - name: secret-volume
          secret:
            secretName: my-secret
        - name: storage-volume
          persistentVolumeClaim:
            claimName: my-pvc
  volumeClaimTemplates: 
    - metadata:
        name: storage-volume
        labels:
          app: my-app
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 5Gi 
```

Let's create the Application and all it's related resources we have shown above.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/application/examples/application.yaml
persistentvolumeclaim/my-pvc created
configmap/my-config created
secret/my-secret created
service/my-service created
serviceaccount/my-serviceaccount created
role.rbac.authorization.k8s.io/my-role created
rolebinding.rbac.authorization.k8s.io/my-rolebinding created
clusterrole.rbac.authorization.k8s.io/my-clusterrole created
clusterrolebinding.rbac.authorization.k8s.io/my-clusterrolebinding created
statefulset.apps/my-app created
```

Now, wait for the pods of the Application to go into the `Running` state.

```bash
$ kubectl get pods -n demo 
NAME       READY   STATUS    RESTARTS   AGE
my-app-0   1/1     Running   0          8m33s
my-app-1   1/1     Running   0          8m8s
```
```bash
$ kubectl get persistentvolumeclaim,configmap,secret,service,serviceaccount,role,rolebinding -n demo -l app=my-app
NAME                                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/my-pvc                    Bound    pvc-c8537568-5a81-43ff-a5d4-6b7efb011de0   5Gi        RWO            longhorn       <unset>                 4m38s
persistentvolumeclaim/storage-volume-my-app-0   Bound    pvc-f0e92133-e84e-45c5-8a97-4d2eeca0ca0d   5Gi        RWO            longhorn       <unset>                 4m38s
persistentvolumeclaim/storage-volume-my-app-1   Bound    pvc-c15dc622-5dc1-4b16-a451-de424445bf72   5Gi        RWO            longhorn       <unset>                 4m13s

NAME                  DATA   AGE
configmap/my-config   1      4m38s

NAME               TYPE     DATA   AGE
secret/my-secret   Opaque   2      4m38s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/my-service   ClusterIP   10.43.229.109   <none>        80/TCP    4m38s

NAME                               SECRETS   AGE
serviceaccount/my-serviceaccount   0         4m38s

NAME                                     CREATED AT
role.rbac.authorization.k8s.io/my-role   2025-04-22T08:37:59Z

NAME                                                   ROLE           AGE
rolebinding.rbac.authorization.k8s.io/my-rolebinding   Role/my-role   4m38s
```
```bash
$ kubectl get clusterrole,clusterrolebinding -l app=my-app
NAME                                                   CREATED AT
clusterrole.rbac.authorization.k8s.io/my-clusterrole   2025-04-22T08:37:59Z

NAME                                                                 ROLE                         AGE
clusterrolebinding.rbac.authorization.k8s.io/my-clusterrolebinding   ClusterRole/my-clusterrole   10m
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/application/examples/backupstorage.yaml
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
  maxRetentionPeriod: 2mo
  successfulSnapshots:
    last: 2
  failedSnapshots:
    last: 1
  usagePolicy:
    allowedNamespaces:
      from: All
```

Let’s create the above `RetentionPolicy`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/application/examples/retentionpolicy.yaml
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
  name: my-app-backupconfiguration
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: StatefulSet
    name: my-app
    namespace: demo
  backends:
    - name: gcs-storage
      storageRef:
        namespace: demo
        name: gcs-storage
      retentionPolicy:
        name: demo-retention
        namespace: demo
  sessions:
    - name: workload-backup
      sessionHistoryLimit: 3
      scheduler:
        schedule: "*/5 * * * *"
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: gcs-repo-for-app
          backend: gcs-storage
          directory: /manifest/statefulset
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: workload-addon
        tasks:
          - name: manifest-backup
            params:
              includeClusterResources: "true"
        jobTemplate:
          spec:
            serviceAccountName: permissions-serviceaccount
      retryConfig:
        maxRetry: 2
        delay: 1m
```

Let's create the `BackupConfiguration` CR we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/workloads/application/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/my-app-backup-configuration created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration.core.kubestash.com -n demo 
backupconfiguration.core.kubestash.com/my-app-backupconfiguration created
```

Additionally, we can verify that the `Repository` specified in the `BackupConfiguration` has been created using the following command,

```bash
$ kubectl get repository -n demo gcs-repo-for-app
NAME               INTEGRITY   SNAPSHOT-COUNT   SIZE         PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repo-for-app   true        3                27.456 KiB   Ready   4m58s                    7m25s

```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the `Repository` YAML stored in the `demo/demo/sample-sts` directory.

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` CR.

Verify that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo 
NAME                                                 SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-my-app-backupconfiguration-workload-backup   */5 * * * *   <none>     False     0        2m37s           4m54s

```

**Wait for BackupSession:**

Wait for the next schedule for backup. Run the following command to watch `BackupSession` CR,

```bash
$ kubectl get backupsession -n demo 
NAME                                                    INVOKER-TYPE          INVOKER-NAME                 PHASE       DURATION   AGE
my-app-backupconfiguration-workload-backup-1745487153   BackupConfiguration   my-app-backupconfiguration   Succeeded   38s        5m54s
my-app-backupconfiguration-workload-backup-1745487300   BackupConfiguration   my-app-backupconfiguration   Succeeded   42s        3m37s

```

We can see from the above output that the backup session has succeeded. Now, we are going to verify whether the backed up data has been stored in the backend.

**Verify Backup:**

Once a backup is complete, KubeStash will update the respective `Repository` CR to reflect the backup. Check that the repository `gcs-demo-repo` has been updated by the following command,

```bash
$ kubectl get repository -n demo gcs-repo-for-app
NAME               INTEGRITY   SNAPSHOT-COUNT   SIZE         PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repo-for-app   true        3                27.456 KiB   Ready   4m58s                    7m25s

```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots.storage.kubestash.com -n demo -l kubestash.com/repo-name=gcs-repo-for-app
NAME                                                              REPOSITORY         SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-repo-for-app-my-app-backupcotion-workload-backup-1745487300   gcs-repo-for-app   workload-backup   2025-04-24T09:35:00Z   Delete            Succeeded   7m49s
gcs-repo-for-app-my-app-backupcotion-workload-backup-1745487600   gcs-repo-for-app   workload-backup   2025-04-24T09:40:00Z   Delete            Succeeded   2m49s
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <target-kind>`
> - `kubestash.com/app-ref-name: <target-name>`
> - `kubestash.com/app-ref-namespace: <target-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
> 
> These labels can be used to watch only the `Snapshot`s related to our desired Workload or `Repository`. 

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the StatefulSet.

```bash
$ kubectl get snapshots.storage.kubestash.com -n demo gcs-repo-for-app-my-app-backupcotion-workload-backup-1745487600 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  creationTimestamp: "2025-04-24T09:40:00Z"
  finalizers:
  - kubestash.com/cleanup
  generation: 1
  labels:
    kubestash.com/app-ref-kind: StatefulSet
    kubestash.com/app-ref-name: my-app
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-repo-for-app
  name: gcs-repo-for-app-my-app-backupcotion-workload-backup-1745487600
  namespace: demo
  ownerReferences:
  - apiVersion: storage.kubestash.com/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: Repository
    name: gcs-repo-for-app
    uid: babf1352-4c2c-4821-a53d-cfb9617a8ac4
  resourceVersion: "306561"
  uid: c8d77b9a-452f-491a-a7be-2dd5022561dd
spec:
  appRef:
    apiGroup: apps
    kind: StatefulSet
    name: my-app
    namespace: demo
  backupSession: my-app-backupconfiguration-workload-backup-1745487600
  deletionPolicy: Delete
  repository: gcs-repo-for-app
  session: workload-backup
  snapshotID: 01JSKJP78E1P5695K7S3FFKF20
  type: FullBackup
  version: v1
status:
  components:
    manifest:
      driver: Restic
      duration: 12.029804059s
      integrity: true
      path: repository/v1/workload-backup/manifest
      phase: Succeeded
      resticStats:
      - hostPath: /kubestash-tmp/manifest
        id: 94695b794dce66038d4a6752b3ed92272b7a80e5f5de1eedcfb7ec66e3e2e55a
        size: 6.132 KiB
        uploaded: 9.883 KiB
      size: 27.429 KiB
  conditions:
  - lastTransitionTime: "2025-04-24T09:40:01Z"
    message: Recent snapshot list updated successfully
    reason: SuccessfullyUpdatedRecentSnapshotList
    status: "True"
    type: RecentSnapshotListUpdated
  - lastTransitionTime: "2025-04-24T09:40:28Z"
    message: Metadata uploaded to backend successfully
    reason: SuccessfullyUploadedSnapshotMetadata
    status: "True"
    type: SnapshotMetadataUploaded
  integrity: true
  phase: Succeeded
  size: 27.429 KiB
  snapshotTime: "2025-04-24T09:40:00Z"
  totalComponents: 1
  verificationStatus: NotVerified
```
> For StatefulSet, KubeStash takes backup from every pod of the StatefulSet. Since we are using three replicas, three components are present in the Snapshot. KubeStash uses `dump-pod-<ordinal-value>` as the component name for StatefulSet. The ordinal value in the component's name represents the ordinal value of the StatefulSet pod ordinal.

Now, if we navigate to GCS bucket, we will see the backed up data stored in the `demo/demo/sample-sts/repository/v1/demo-session/dump-pod-<ordinal-value>` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/demo/sample-sts/snapshots` directory.

> Note: KubeStash stores all dumped data encrypted in the backup directory, meaning it remains unreadable until decrypted.

**Download Snapshot and Verify Manifest Files**

To download a specific `Snapshot` you have to install `KubeStash Kubectl Plugin` from [here](/docs/setup/install/kubectl-plugin/). To know the format and flags available for `kubectl kubestash download` command explore [here](/docs/guides/cli/kubectl-plugin/). 

```bash 
 $ kubectl kubestash download cs-repo-for-app-my-app-backupcotion-workload-backup-1745487600 --namespace=demo --destination=/path/to/download

I0424 17:14:47.434803  882209 download.go:197] Component: manifest of Snapshot demo/gcs-repo-for-app-my-app-backupcotion-workload-backup-1745487600 restored in path /path/to/download
```

After downloading the snapshot navigate to it's directory and check all the manifests file related to your application. 

```bash 
$ ~/path/to/download/gcs-repo-for-app-my-app-backupcotion-workload-backup-1745492700> tree
.
└── manifest
    └── kubestash-tmp
        └── manifest
            ├── clusterrole
            │   └── my-clusterrole.yaml
            ├── clusterrolebinding
            │   └── my-clusterrolebinding.yaml
            ├── configmap
            │   └── my-config.yaml
            ├── pvc
            │   └── my-pvc.yaml
            ├── role
            │   └── my-role.yaml
            ├── rolebinding
            │   └── my-rolebinding.yaml
            ├── secret
            │   └── my-secret.yaml
            ├── service
            │   └── my-service.yaml
            ├── serviceaccount
            │   └── my-serviceaccount.yaml
            └── statefulset
                └── my-app.yaml

14 directories, 10 files
```

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
  name: my-app-restoresession
  namespace: demo
spec:
  manifestOptions:
    workload:
      restoreNamespace: demo
  dataSource:
    repository: gcs-repo-for-app
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret
      namespace: demo
  addon:
    name: workload-addon
    tasks:
      - name: manifest-restore
        params:
          includeClusterResources: "true"
          overrideResources: "true"
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