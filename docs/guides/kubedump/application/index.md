---
title: Backup resource YAMLs of an Application | KubeStash
description: Backup application manifests along with it's dependant using KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-kubedump-application
    name: Application Manifest Backup
    parent: kubestash-kubedump
    weight: 40
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup YAMLs of an Application using KubeStash

KubeStash `{{< param "info.version" >}}` supports taking backup of the resource YAMLs using `kubedump` plugin. This guide will show you how you can take a backup of the YAMLs of an application along with it's dependant using KubeStash.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster.
- Install KubeStash in your cluster following the steps [here](/docs/setup/install/kubestash/index.md).
- Install KubeStash `kubectl` plugin in your local machine following the steps [here](/docs/setup/install/kubectl-plugin/index.md).
- If you are not familiar with how KubeStash backup the resource YAMLs, please check the following guide [here](/docs/guides/kubedump/overview/index.md).

You have to be familiar with the following custom resources:

- [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
- [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
- [BackupSession](/docs/concepts/crds/backupsession/index.md)
- [SnapShot](/docs/concepts/crds/snapshot/index.md)
- [RestoreSession](/docs/concepts/crds/restoresession/index.md)
- [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)


To keep things isolated, we are going to use a separate namespace called `demo` throughout this tutorial. Create the `demo` namespace if you haven't created it already.

```bash
❯ kubectl create ns demo
namespace/demo created
```

> Note: YAML files used in this tutorial are stored [here](https://github.com/kubestash/docs/tree/{{< param "info.version" >}}/docs/addons/kubedump/application/examples).

#### Ensure `kubedump` Addon

**Verify necessary Addons and Functions:**

When you install KubeStash, it automatically creates the necessary `Addon` and `Function` to backup Kubernetes Resource YAML.

Let's verify that KubeStash has created the necessary `Function` to backup Kubernetes Resource YAML by the following command,

```bash
$ kubectl get functions | grep 'kubedump'
kubedump-backup          142m
```

Also, verify that the necessary `Addon` has been created,

```bash
❯ kubectl get addons | grep 'kubedump'
kubedump-addon   143m
```

Now, you can view all `BackupTasks` and `RestoreTasks` of the `pvc-addon` addon using the following command

```bash
❯ kubectl get addon kubedump-addon -o=jsonpath='{.spec.backupTasks[*].name}'| tr ' ' '\n'
manifest-backup
```

#### Prepare Backend

Now, we are going to store our backed-up data into a GCS bucket using KubeStash. We have to create a Secret and a `BackupStorage` object with access credentials and backend information respectively. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

> For GCS backend, if the bucket does not exist, KubeStash needs `Storage Object Admin` role permissions to create the bucket. For more details, please check the following [guide](/docs/guides/backends/gcs/index.md).

**Create Storage Secret:**

At first, let's create a secret called `gcs-secret` with access credentials to our desired GCS bucket,

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/application/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-repo created
```

Now, we also have to create another secret with an encryption key `RESTIC_PASSWORD` for Restic. This secret will be used by Restic for both encrypting and decrypting the backup data during backup & restore.

**Create Restic Repository Secret:**

Let's create a Secret named `encryption-secret` with an encryption key `RESTIC_PASSWORD` for Restic.

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encryption-secret \
    --from-file=./RESTIC_PASSWORD 
secret/encryption-secret created
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

Let's create the `RetentionPolicy` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/application/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

#### Create RBAC

The `kubedump` plugin requires read permission for the application resources. By default, KubeStash does not grant such permissions. We have to provide the necessary permissions manually.

Here, is the YAML of the `ServiceAccount`, `ClusterRole`, and `ClusterRoleBinding` that we are going to use for granting the necessary permissions.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-resource-reader
  namespace: kubestash
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-resource-reader
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-resource-reader
subjects:
  - kind: ServiceAccount
    name: cluster-resource-reader
    namespace: kubestash
roleRef:
  kind: ClusterRole
  name: cluster-resource-reader
  apiGroup: rbac.authorization.k8s.io
```

Here, we have give permission to read all the cluster resources. You can restrict this permission to a particular application resources only.

Let's create the RBAC resources we have shown above,

```bash
❯ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/application/examples/rbac.yaml
serviceaccount/cluster-resource-reader created
clusterrole.rbac.authorization.k8s.io/cluster-resource-reader created
clusterrolebinding.rbac.authorization.k8s.io/cluster-resource-reader created
```

Now, we are ready for backup. In the next section, we are going to schedule a backup for our cluster resources.

### Backup

To schedule a backup, we have to create a `BackupConfiguration` object. Then KubeStash will create a CronJob to periodically backup the database.

At first, lets list available Deployment in `kubestash` namespace.

```bash
❯ kubectl get deployments -n kubestash
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
kubestash-kubestash-operator   1/1     1            1           19h
```

Here, we are going to setup backup YAMLs for `kubestash-kubestash-operator` Deployment.

#### Create BackupConfiguration

Below is the YAML for `BackupConfiguration` object we care going to use to backup the YAMLs of the cluster resources,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: application-manifest-backup
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name:  kubestash-kubestash-operator
    namespace: kubestash
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
          directory: /deployment-manifests
          encryptionSecret:
            name: encryption-secret
            namespace: demo
          deletionPolicy: WipeOut
      addon:
        name: kubedump-addon
        tasks:
          - name: manifest-backup
            params:
              includeDependants: "true"
        jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader
```

Here,

- `spec.target` specifies the backup target. `apiGroup`, `kind`, `namespace` and `name` refers to the `apiGroup`, `kind`, `namespace` and `name` of the targeted workload respectively.
- `spec.sessions[*].scheduler.schedule` specifies a [cron expression](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/#schedule) indicates that `BackupSession` will be created at 5 minute interval.
- `spec.sessions[*].repositories[*].name` specifies the name of the `Repository` object that holds the backend information.
- `spec.sessions[*].repositories[*].backend` specifies the name of the backend that holds the `BackupStorage` information.
- `spec.sessions[*].repositories[*].directory` specifies the path of the `Repository` where the backed up data will be stored.
- `spec.sessions[*].repositories[*].encryptionSecret` specifies the encryption secret for `Restic Repository` which will be used to encrypting the backup data.
- `spec.sessions[*].addon.name` specifies the name of the `Addon` object that specifies addon configuration that will be used to perform backup of a stand-alone PVC.
- `spec.sessions[*].addon.tasks[*].name` specifies the name of the `Task` that holds the `Function` and their order of execution to perform backup of a stand-alone PVC.
- `spec.sessions[*].addon.jobTemplate.runtimeSettings.pod.serviceAccountName` specifies the ServiceAccount name that we have created earlier with cluster-wide resource reading permission.

Let's create the `BackupConfiguration` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/application/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/application-manifest-backup created
```

#### Verify Backup Setup Successful

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the BackupConfiguration,

```bash
❯ kubectl get backupconfiguration -n demo
NAME                              PHASE   PAUSED   AGE
application-manifest-backup       Ready            19s
```

**Verify Repository:**

Verify that KubeStash has created `Repositories` that holds the `BackupStorage` information by the following command,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE   PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository                                       Ready                            28s
```

**Verify CronJob:**

Verify that KubeStash has created a CronJob to trigger a periodic backup of the targeted PVC by the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                                  SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-application-manifest-backup-frequent-backup   */5 * * * *   False     0        <none>          45s
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` crd using the following command,

```bash
$ watch -n 1 kubectl get backupsession -n demo -l=kubestash.com/invoker-name=application-manifest-backup
Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-name=application-manifest-backup                                      anisur: Fri Feb 23 14:35:27 2024

NAME                                                     INVOKER-TYPE          INVOKER-NAME                  PHASE       DURATION   AGE
application-manifest-backup-frequent-backup-1708677300   BackupConfiguration   application-manifest-backup   Succeeded              27s

```

**Verify Backup:**

When backup session is created, kubestash operator creates `Snapshot` which represents the state of backup run for each `Repository` which are provided in `BackupConfiguration`.

Run the following command to check `Snapshot` phase,

```bash
$ kubectl get snapshots -n demo
NAME                                                              REPOSITORY       SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-repository-application-manifckup-frequent-backup-1708677300   gcs-repository   frequent-backup   2024-02-23T08:35:00Z   Delete            Succeeded   43s
```

When backup session is completed, KubeStash will update the respective `Repository` to reflect the latest state of backed up data.

Run the following command to check if a backup snapshot has been stored in the backend,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository   true        1                2.262 KiB   Ready   103s                     8m
```

From the output above, we can see that `1` snapshot has been stored in the backend specified by Repository `gcs-repository`.

If we navigate to `kubestash-qa/demo/deployment-manifests/repository/v1/frequent-backup/manifest` directory of our GCS bucket, we are going to see that the snapshot has been stored there.

<figure align="center">
  <img alt="Backup YAMLs data of an Application in GCS storage" src="/docs/guides/kubedump/application/images/application_manifest_backup.png">
  <figcaption align="center">Fig: Backup YAMLs data of an Application in GCS backend</figcaption>
</figure>

> KubeStash keeps all backup data encrypted. So, snapshot files in the bucket will not contain any meaningful data until they are decrypted.


## Restore

KubeStash does not provide any automatic mechanism to restore the cluster resources from the backed-up YAMLs. Your application might be managed by Helm or by an operator. In such cases, just applying the YAMLs is not enough to restore the application. Furthermore, there might be an order issue. Some resources must be applied before others. It is difficult to generalize and codify various application-specific logic.

Therefore, it is the user's responsibility to download the backed-up YAMLs and take the necessary steps based on his application to restore it properly.

### Download the YAMLs

KubeStash provides a [kubectl plugin](https://stash.run/docs/v2022.05.12/guides/cli/cli/#download-snapshots) for making it easy to download a snapshot locally.

Now, let's download the latest Snapshot from our backed-up data into the `$HOME/Downloads/stash/applications/kube-system/stash-enterprise` folder of our local machine.

```bash
❯ kubectl kubestash download gcs-repository-application-manifckup-frequent-backup-1708677300 --namespace=demo --destination=$HOME/Downloads/kubestash/applications/kubestash/kubestash-kubestash-operator
```

Now, lets use [tree](https://linux.die.net/man/1/tree) command to inspect downloaded YAMLs files.

```bash
❯ tree $HOME/Downloads/kubestash/applications/kubestash/kubestash-kubestash-operator
/home/anisur/Downloads/kubestash/applications/kubestash/kubestash-kubestash-operator
└── gcs-repository-application-manifckup-frequent-backup-1708677300
    └── manifest
        └── tmp
            └── manifest
                ├── kubestash-kubestash-operator.yaml
                └── ReplicaSet
                    └── kubestash-kubestash-operator-7bc7564b69
                        ├── kubestash-kubestash-operator-7bc7564b69.yaml
                        └── Pod
                            └── kubestash-kubestash-operator-7bc7564b69-2frcq
                                └── kubestash-kubestash-operator-7bc7564b69-2frcq.yaml

8 directories, 3 files

```

As you can see that the Deployment has been backed up along with it's ReplicaSet and Pods.

Let's inspect the YAML of `kubestash-kubestash-operator.yaml` file,

```yaml
❯ cat /home/anisur/Downloads/kubestash/applications/kubestash/kubestash-kubestash-operator/gcs-repository-application-manifckup-frequent-backup-1708677300/manifest/tmp/manifest/kubestash-kubestash-operator.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    meta.helm.sh/release-name: kubestash
    meta.helm.sh/release-namespace: kubestash
  labels:
    app.kubernetes.io/instance: kubestash
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubestash-operator
    app.kubernetes.io/version: v0.5.0
    helm.sh/chart: kubestash-operator-v0.5.0
  name: kubestash-kubestash-operator
  namespace: kubestash
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/instance: kubestash
      app.kubernetes.io/name: kubestash-operator
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      annotations:
        checksum/apiregistration.yaml: 62af9aba894e98a7dc849e63a31ef52d6c3b459df8d2242e71cc72e458553d11
      labels:
        app.kubernetes.io/instance: kubestash
        app.kubernetes.io/name: kubestash-operator
    spec:
      containers:
        - args:
            - run
            - --config=/var/config/config.yaml
            - --license-file=/var/run/secrets/appscode/license/key.txt
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
          image: ghcr.io/kubestash/kubestash:v0.5.0
          imagePullPolicy: IfNotPresent
          name: operator
          ports:
            - containerPort: 9443
              name: webhook-server
              protocol: TCP
          resources:
            requests:
              cpu: 100m
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 65534
          terminationMessagePolicy: File
          volumeMounts:
            - mountPath: /kubestash-tmp
              name: kubestash-tmp-volume
            - mountPath: /var/serving-cert
              name: serving-cert
              readOnly: true
            - mountPath: /var/config
              name: config
            - mountPath: /var/run/secrets/appscode/license
              name: license
        - args:
            - --secure-listen-address=0.0.0.0:8443
            - --upstream=http://127.0.0.1:8080/
            - --logtostderr=true
            - --v=10
          image: ghcr.io/appscode/kube-rbac-proxy:v0.11.0
          imagePullPolicy: IfNotPresent
          name: kube-rbac-proxy
          ports:
            - containerPort: 8443
              name: https
              protocol: TCP
          resources: {}
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 65534
          terminationMessagePolicy: File
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 65535
      serviceAccount: kubestash-kubestash-operator
      serviceAccountName: kubestash-kubestash-operator
      volumes:
        - emptyDir: {}
          name: kubestash-tmp-volume
        - name: serving-cert
          secret:
            defaultMode: 420
            secretName: kubestash-kubestash-operator-webhook-cert
        - configMap:
            defaultMode: 420
            name: kubestash-kubestash-operator-config
          name: config
        - name: license
          secret:
            defaultMode: 420
            secretName: kubestash-kubestash-operator-license
```

Now, you can use these YAML files to re-create your desired application.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo backupconfiguration application-manifest-backup
kubectl delete -n kubestash serviceaccount cluster-resource-reader
kubectl delete clusterrole cluster-resource-reader
kubectl delete clusterrolebinding cluster-resource-reader
kubectl delete retentionPolicy -n demo demo-retention
kubectl delete -n demo backupstorage gcs-storage
kubectl delete secret -n demo encryption-secret
kubectl delete secret -n demo gcs-secret
```
