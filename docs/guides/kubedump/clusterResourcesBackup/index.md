---
title: Backup resource YAMLs of entire cluster | KubeStash
description: Take backup of cluster resources YAMLs using KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-kubedump-cluster
    name: Cluster Backup
    parent: kubestash-kubedump
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backing Up and Restoring Cluster Resource Manifests Using KubeStash

This guide will show you how you can take a backup of the resource YAMLs of your entire cluster using KubeStash.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster.
- Install KubeStash in your cluster following the steps [here](/docs/setup/install/kubestash/index.md).
- Install KubeStash `kubectl` plugin in your local machine following the steps [here](/docs/setup/install/kubectl-plugin/index.md).
- If you are not familiar with how KubeStash backup the resource YAMLs, please check the following guide [here](/docs/guides/kubedump/overview/index.md).

You have to be familiar with the following custom resources:

- [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
- [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
- [BackupSession](/docs/concepts/crds/backupsession/index.md)
- [Snapshot](/docs/concepts/crds/snapshot/index.md)
- [RestoreSession](/docs/concepts/crds/restoresession/index.md)
- [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)

> Note: YAML files used in this tutorial are stored [here](https://github.com/kubestash/docs/tree/{{< param "info.version" >}}/docs/guides/kubedump/clusterResourcesBackup/examples).

---

## Create Resources

To keep things isolated, we are going to use separate namespaces `demo`, `demo-a` and `demo-b` throughout this tutorial. Create the namespaces.

```bash
$ kubectl create ns demo
namespace/demo created
$ kubectl create ns demo-a 
namespace/demo-a created
$ kubectl create ns demo-b
namespace/demo-b created
```

We need to create some resources both namespace scoped and cluster scoped to demonstrate our backup and restore process. For simplification we will be using two `labels` to demonstarte separation of the resources.   

---

For label `app=my-app` all of the resources will be either in `demo-a` namespace or cluster scoped.     

--- 

#### Resource Having Label `app=my-app` 

Below is the YAML of the resources: 

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc-a
  namespace: demo-a
  labels:
    app: my-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: "longhorn"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config-a
  namespace: demo-a
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
  name: my-secret-a
  namespace: demo-a
  labels:
    app: my-app
type: Opaque
data:
  username: <your_username>
  password:  <your_password>
---
apiVersion: v1
kind: Service
metadata:
  name: my-service-a
  namespace: demo-a
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
  name: my-serviceaccount-a
  namespace: demo-a
  labels:
    app: my-app
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-clusterrole-a
  labels:
    app: my-app
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-clusterrolebinding-a
  labels:
    app: my-app
subjects:
  - kind: ServiceAccount
    name: my-serviceaccount-a
    namespace: demo-a
roleRef:
  kind: ClusterRole
  name: my-clusterrole-a
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-a
  namespace: demo-a
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-serviceaccount-a
      containers:
        - name: my-container-a
          image: nginx:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: config-volume-a
              mountPath: /etc/config
            - name: secret-volume-a
              mountPath: /etc/secret
            - name: storage-volume-a
              mountPath: /data
      volumes:
        - name: config-volume-a
          configMap:
            name: my-config-a
        - name: secret-volume-a
          secret:
            secretName: my-secret-a
        - name: storage-volume-a
          persistentVolumeClaim:
            claimName: my-pvc-a
```

Let's create the objects having label `app:my-app` we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/clusterResourcesBackup/examples/resources-a.yaml
persistentvolumeclaim/my-pvc-a created
configmap/my-config-a created
secret/my-secret-a created
service/my-service-a created
serviceaccount/my-serviceaccount-a created
clusterrole.rbac.authorization.k8s.io/my-clusterrole-a created
clusterrolebinding.rbac.authorization.k8s.io/my-clusterrolebinding-a created
deployment.apps/my-deployment-a created
```

**Verify Resource Creation:**

Verify that the resources with label `app=my-app` have been created using the following command,

```bash
Every 2.0s: kubectl get replicaset,secret,configmap,clusterrole,clusterrolebinding,persistentvolumeclaim,service,serviceaccount,deployment,pod -n demo-a -l app=my-app   nipun-pc: Thu Jul 10 15:39:45 2025

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/my-deployment-a-6bbd894c5   3         3         3       4m47s

NAME                TYPE     DATA   AGE
secret/my-secret-a  Opaque   2      4m47s

NAME                   DATA   AGE
configmap/my-config-a  1      4m47s

NAME                                                  CREATED AT
clusterrole.rbac.authorization.k8s.io/my-clusterrole-a   2025-07-10T09:34:58Z

NAME                                                                 ROLE                            AGE
clusterrolebinding.rbac.authorization.k8s.io/my-clusterrolebinding-a   ClusterRole/my-clusterrole-a   4m47s

NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/my-pv-a 5Gi        RWO            Retain           Bound    demo-a/my-pvc-a     <unset>        <unset>                          4m47s

NAME                            STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/my-pvc-a Bound    my-pv-a    5Gi        RWO            <unset>        <unset>                 4m47s

NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/my-service-a ClusterIP  10.43.81.18   <none>        80/TCP    4m47s

NAME                                SECRETS   AGE
serviceaccount/my-serviceaccount-a  0         4m47s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-deployment-a   3/3     3             3         4m47s

AME                                  READY   STATUS    RESTARTS   AGE
pod/my-deployment-a-6bbd894c5-b9ks5   1/1     Running   0          4m47s
pod/my-deployment-a-6bbd894c5-p992t   1/1     Running   0          4m47s
pod/my-deployment-a-6bbd894c5-s72fw   1/1     Running   0          4m47s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/my-deployment-a-6bbd894c5   3         3         3       4m47s
```

--- 

For label `app=my-sts` all of the resources will be either in `demo-b` namespace or cluster scoped.   

---

#### Resource Having Label `app=my-sts` 

Below is the YAML of the resources: 

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc-b
  namespace: demo-b
  labels:
    app: my-sts
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: "longhorn"  
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config-b
  namespace: demo-b
  labels:
    app: my-sts
data:
  app.properties: |
    greeting.message=Hello, World!
    app.version=1.0.0
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret-b
  namespace: demo-b
  labels:
    app: my-sts
type: Opaque
data:
  username: <your_username>
  password: <your_password>
---
apiVersion: v1
kind: Service
metadata:
  name: my-service-b
  namespace: demo-b
  labels:
    app: my-sts
spec:
  selector:
    app: my-sts
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-serviceaccount-b
  namespace: demo-b
  labels:
    app: my-sts
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-clusterrole-b
  labels:
    app: my-sts
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-clusterrolebinding-b
  labels:
    app: my-sts
subjects:
  - kind: ServiceAccount
    name: my-serviceaccount-b
    namespace: demo-b
roleRef:
  kind: ClusterRole
  name: my-clusterrole-b
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-statefulset
  namespace: demo-b
  labels:
    app: my-sts
spec:
  serviceName: "my-service-b"
  replicas: 3
  selector:
    matchLabels:
      app: my-sts
  template:
    metadata:
      labels:
        app: my-sts
    spec:
      serviceAccountName: my-serviceaccount-b
      containers:
        - name: my-container-b
          image: nginx:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: config-volume-b
              mountPath: /etc/config
            - name: secret-volume-b
              mountPath: /etc/secret
            - name: storage-volume-b
              mountPath: /data
      volumes:
        - name: config-volume-b
          configMap:
            name: my-config-b
        - name: secret-volume-b
          secret:
            secretName: my-secret-b
        - name: storage-volume-b
          persistentVolumeClaim:
            claimName: my-pvc-b 
```

Let's create the objects of having label `app:my-sts` we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/clusterResourcesBackup/examples/resources-b.yaml
persistentvolume/my-pv-b created
persistentvolumeclaim/my-pvc-b created
configmap/my-config-b created
secret/my-secret-b created
service/my-service-b created
serviceaccount/my-serviceaccount-b created
clusterrole.rbac.authorization.k8s.io/my-clusterrole-b created
clusterrolebinding.rbac.authorization.k8s.io/my-clusterrolebinding-b created
statefulset.apps/my-statefulset created
```

**Verify Resource Creation:**

Verify that the resources with label `app=my-sts` have been created using the following command,

```bash
Every 2.0s: kubectl get secret,configmap,statefulset,clusterrole,clusterrolebinding,persistentvolume,persistentvolumeclaim,service,serviceaccount,pod -n demo-b -l app=my-sts   nipun-pc: Thu Jul 10 15:40:14 2025

NAME                 TYPE     DATA   AGE
secret/my-secret-b   Opaque   2      80m

NAME                    DATA   AGE
configmap/my-config-b   1      80m

NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/my-service-b   ClusterIP   10.43.63.79   <none>        80/TCP    80m

NAME                                                     CREATED AT
clusterrole.rbac.authorization.k8s.io/my-clusterrole-b   2025-07-18T08:52:52Z

NAME                                                                   ROLE                           AGE
clusterrolebinding.rbac.authorization.k8s.io/my-clusterrolebinding-b   ClusterRole/my-clusterrole-b   80m

NAME                                 SECRETS   AGE
serviceaccount/my-serviceaccount-b   0         80m

NAME                              READY   AGE
statefulset.apps/my-statefulset   3/3     80m

NAME                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/my-pvc-b   Bound    pvc-aa924f87-e580-4dcd-9873-904e5c18c194   5Gi        RWO            longhorn       <unset>                 80m

NAME                   READY   STATUS    RESTARTS   AGE
pod/my-statefulset-0   1/1     Running   0          80m
pod/my-statefulset-1   1/1     Running   0          80m
pod/my-statefulset-2   1/1     Running   0          80m
```

---

## Prepare for Backup

We are going to configure a backup for some of the specific resources of our cluster.

### Prepare Backend

Now, we are going backup of the YAMLs of entire cluster to a S3 bucket using KubeStash. For this, we have to create a `Secret` with  necessary credentials and a `BackupStorage` object. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

> For S3 backend, if the bucket does not exist, KubeStash needs `Storage Object Admin` role permissions to create the bucket. For more details, please check the following [guide](/docs/guides/backends/s3/index.md).

**Create Secret:**

Let's create a Secret named `aws-s3-secret` with access credentials of our desired S3 backend,

```bash
$ kubectl create secret generic aws-s3-secret \
    --from-literal=AWS_ACCESS_KEY_ID=<your-aws-access-key-id> \
    --from-literal=AWS_SECRET_ACCESS_KEY=<your-aws-secret-access-key>
secret/aws-s3-secret created
```

**Create BackupStorage:**

Now, create a `BackupStorage` custom resource specifying the desired bucket, and directory inside the bucket where the backed up data will be stored.

Below is the YAML of `BackupStorage` object that we are going to create,

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: BackupStorage
metadata:
  name: s3-storage
  namespace: demo
spec:
  storage:
    provider: s3
    s3:
      bucket: kubestash-qa
      region: us-east-2
      endpoint: http://s3.us-east-2.amazonaws.com
      secretName: aws-s3-secret
      prefix: nipun    
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true 
  deletionPolicy: WipeOut
```

Let's create the `BackupStorage` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/clusterResourcesBackup/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/s3-storage created
```

Now, we are ready to backup our cluster yaml resources.

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
    last: 1
  maxRetentionPeriod: 2mo
  successfulSnapshots:
    last: 2
  usagePolicy:
    allowedNamespaces:
      from: All
```

>Notice the `spec.usagePolicy` that allows referencing the `RetentionPolicy` from all namespaces.For more details on configuring it for specific namespaces, please refer to the following [link](/docs/concepts/crds/retentionpolicy/index.md).

Let's create the `RetentionPolicy` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/clusterResourcesBackup/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

---

### Create RBAC

To take backup of the resource YAMLs of entire cluster KubeStash creates a backup `Job`. This `Job` requires read permission for all the cluster resources. By default, KubeStash does not grant such cluster-wide permissions. We have to provide the necessary permissions manually.

Here, is the YAML of the `ServiceAccount`, `ClusterRole`, and `ClusterRoleBinding` that we are going to use for granting the necessary permissions.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-resource-reader-writer
  namespace: demo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-resource-reader-writer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-resource-reader-writer
subjects:
- kind: ServiceAccount
  name: cluster-resource-reader-writer
  namespace: demo
roleRef:
  kind: ClusterRole
  name: cluster-resource-reader-writer
  apiGroup: rbac.authorization.k8s.io
```

Let's create the RBAC resources we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/kubedump/clusterResourcesBackup/examples/rbac.yaml
serviceaccount/cluster-resource-reader-writer created
clusterrole.rbac.authorization.k8s.io/cluster-resource-reader-writer created
clusterrolebinding.rbac.authorization.k8s.io/cluster-resource-reader-writer created
```

---

### Backup

To schedule a backup, we have to create a `BackupConfiguration` object.

**Create Secret:**

We also have to create another `Secret` with an encryption key `RESTIC_PASSWORD` for `Restic`. This secret will be used by `Restic` for encrypting the backup data.

Let's create a secret named `encry-secret` with the Restic password.

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD 
secret/encrypt-secret created
```

#### Create BackupConfiguration

Below is the YAML for `BackupConfiguration` object we care going to use to backup the YAMLs of the cluster resources,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: cluster-resources-backup
  namespace: demo
spec:
  backends:
    - name: s3-backend
      storageRef:
        namespace: demo
        name: s3-storage
      retentionPolicy:
        name: demo-retention
        namespace: demo
  sessions:
    - name: frequent-backup
      sessionHistoryLimit: 1
      scheduler:
        schedule: "*/5 * * * *"
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: s3-repo
          backend: s3-backend
          directory: /cluster-manifests
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
          deletionPolicy: Retain
      addon:
        name: kubedump-addon
        tasks:
          - name: manifest-backup
            params:
              IncludeClusterResources: "true"
              IncludeNamespaces: "demo-a,demo-b"
              ExcludeNamespaces: "kube-system,longhorn-system"
              IncludeResources: "*"
              ORedLabelSelectors: "app:my-app,app:my-sts"
        jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader-writer
```

Here,
- `spec.sessions[*].addon.name` specifies the name of the `Addon`.
- `spec.sessions[*].addon.tasks[*].name` specifies the name of the backup task.
- `spec.sessions[*].addon.jobTemplate.spec.serviceAccountName`specifies the ServiceAccount name that we have created earlier with cluster-wide resource reading permission.

---

### Flags in `manifest-backup` task in KubeDump 

We have introduced some flags for filtering resources while taking backup.  

``` yaml 
- ANDedLabelSelectors
  Usage: A set of labels, all of which need to be matched
  to filter the resources.
  Default: ""
  Required: false
  Example: "app:my-app,db:postgres,db"

- ORedLabelSelectors
  Usage: A set of labels, at least one of which need to 
  be matched to filter the resources. 
  Default: ""
  Required: false
  Example: "app:nginx,app:redis,app"

- IncludeClusterResources
  Usage: Specify whether to restore
  cluster scoped resources.
  Default: "false"
  Required: false
  Example: "true" 
  
- IncludeNamespaces
  Usage: Namespaces to include in backup.
  Default: "*"
  Required: false
  Example: "demo,kubedb,kubestash"

- ExcludeNamespaces
  Usage: Namespaces to exclude from backup.
  Default: ""
  Required: false
  Example: "default,kube-system"

- IncludeResources
  Usage: Resource types to include in backup.
  Default: "*"
  Required: false
  Example: "secrets,configmaps,deployments"
  
- ExcludeResources
  Usage: Resource types to exclude from backup
  Default: ""
  Required: false
  Example: "persistentvolumeclaims,persistentvolumes"
```
These flags are independent. That means when a resource is backed up all the parameters are checked
to pass the filter operation.

For example: 

Consider a deployment named as `my-deployment` in `demo-a` namespace having label `app=my-app`. It will pass the 
filter if the flags are set as followed: 
1. `IncludeResources` contain `deployments` in the list or set to default value `*`.    
2. `ExcludeResources` do not contain `deployments` in the list or set to default value `""`.
3. `IncludeNamespaces` contain `demo-a` in the list or set to default value `*`.
4. `ExcludeNamespaces` do not contain `demo-a` in the list or set to default value `""`.
5. `ANDedLabelSelectors` contain only `app:my-app` in the list or set to default value `""`.
6. `ORedLabelSelectors` contain `app:my-app` in the list or set to default value `""`.
7. `IncludeClusterResources` flag doesn't matter here as `deployments` are not cluster scoped resources. 

Conventions that're followed in the parameters: 
1. Resource types have to be in `plural` form for `IncludeResources` or `ExcludeResources` flag. 
2. Asterisk `*` indicates `all` and `""` indicates `empty`. 

---

Let's create the `BackupConfiguration` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/clusterResourcesBackup/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/cluster-resources-backup created
```
---

#### Verify Backup Setup Successful

If everything goes well, the phase of the `BackupConfiguration` should be in `Ready` state. The `Ready` phase indicates that the backup setup is successful.

Let's check the `Phase` of the BackupConfiguration

```bash
$ kubectl get backupconfiguration -n demo
NAME                                    PHASE   PAUSED   AGE
cluster-resources-backup                Ready            79s

```

**Verify Repository:**

Verify that the Repository specified in the BackupConfiguration has been created using the following command,

```bash
$ kubectl get repositories -n demo
NAME      INTEGRITY   SNAPSHOT-COUNT   SIZE         PHASE   LAST-SUCCESSFUL-BACKUP   AGE
s3-repo   true        2                13.856 KiB   Ready   98s                      19h                                   Ready                            28s
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the S3 bucket, we will see the Repository YAML stored in the `nipun/cluster-manifests` directory where nipun is the `prefix` of the `bucket`.

**Verify CronJob:**

Verify that KubeStash has created a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` object.

Check that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                               SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-cluster-resources-backup-frequent-backup   */5 * * * *   <none>     False     0        <none>          2m10s

```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` CR using the following command,

```bash
$ watch -n 1 kubectl get backupsession -n demo -l=kubestash.com/invoker-name=cluster-resources-backup                                        

Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-name=cluster-resources-backup          nipun-pc: Wed Jul  9 12:43:07 2025

NAME                                                           INVOKER-TYPE         INVOKER-NAME                PHASE       DURATION   AGE
cluster-resources-backup-1752043200                           BackupConfiguration  cluster-resources-backup    Succeeded   35s        3m7s
cluster-resources-backup-frequent-backup-1752043049           BackupConfiguration  cluster-resources-backup    Succeeded   35s        5m38s

```

**Verify Backup:**

When `BackupSession` is created, KubeStash operator creates `Snapshot` for each `Repository` listed in the respective session of the `BackupConfiguration`. Since we have only specified only one repository in the session, at this moment we should have one `Snapshot`. 

Run the following command to check the respective `Snapshot`,

```bash
$ kubectl get snapshots.storage.kubestash.com -n demo
NAME                                                          REPOSITORY   SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
s3-repo-cluster-resources-backup-1752139800                   s3-repo      frequent-backup   2025-07-10T09:30:00Z   Delete            Succeeded   91m
s3-repo-cluster-resources-backup-frequent-backup-1752139981   s3-repo      frequent-backup   2025-07-10T09:33:01Z   Delete            Succeeded   88m
```
---

Now, if we navigate to the S3 bucket, we will see the backed up data stored in the `nipun/cluster-manifests/repository/v1/frequent-backup/manifest` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the` nipun/cluster-manifests/repository/snapshots` directory.

<figure align="center">
  <img alt="Backup data in S3 Bucket" src="/docs/guides/kubedump/clusterResourcesBackup/images/s3-snapshots.png">
  <figcaption align="center">Fig: Backup data in S3 Bucket</figcaption>
</figure>

> Note: KubeStash stores all dumped data encrypted in backup directory, meaning it remains unreadable until decrypted.

---

### Download the YAMLs

KubeStash provides a [kubectl plugin](/docs/guides/cli/kubectl-plugin/index.md#download-snapshot) for making it easy to download a snapshot locally.

Now, let's download the latest Snapshot from our backed-up data into the `$HOME/Downloads/kubestash` folder of our local machine.

```bash
$ kubectl kubestash download --namespace=demo s3-repo-cluster-resources-backup-frequent-backup-1752139981 --destination=$/home/arnab/Downloads
```

Now, lets use [tree](https://linux.die.net/man/1/tree) command to inspect downloaded YAMLs files.

```bash
arnab@nipun-pc:~/Downloads/azure-repo-cluster-resources-backup-frequent-backup-1752836554/manifest/kubestash-tmp/manifest$ tree
.
├── clusterrolebindings.rbac.authorization.k8s.io
│   └── cluster
│       ├── my-clusterrolebinding-a.yaml
│       └── my-clusterrolebinding-b.yaml
├── clusterroles.rbac.authorization.k8s.io
│   └── cluster
│       ├── my-clusterrole-a.yaml
│       └── my-clusterrole-b.yaml
├── configmaps
│   └── namespaces
│       ├── demo-a
│       │   └── my-config-a.yaml
│       └── demo-b
│           └── my-config-b.yaml
├── controllerrevisions.apps
│   └── namespaces
│       └── demo-b
│           └── my-statefulset-7bc9c486fc.yaml
├── deployments.apps
│   └── namespaces
│       └── demo-a
│           └── my-deployment-a.yaml
├── endpoints
│   └── namespaces
│       ├── demo-a
│       │   └── my-service-a.yaml
│       └── demo-b
│           └── my-service-b.yaml
├── endpointslices.discovery.k8s.io
│   └── namespaces
│       ├── demo-a
│       │   └── my-service-a-crqk5.yaml
│       └── demo-b
│           └── my-service-b-lv4jh.yaml
├── persistentvolumeclaims
│   └── namespaces
│       ├── demo-a
│       │   └── my-pvc-a.yaml
│       └── demo-b
│           └── my-pvc-b.yaml
├── pods
│   └── namespaces
│       ├── demo-a
│       │   ├── my-deployment-a-6bbd894c5-b9ks5.yaml
│       │   ├── my-deployment-a-6bbd894c5-p992t.yaml
│       │   └── my-deployment-a-6bbd894c5-s72fw.yaml
│       └── demo-b
│           ├── my-statefulset-0.yaml
│           ├── my-statefulset-1.yaml
│           └── my-statefulset-2.yaml
├── replicasets.apps
│   └── namespaces
│       └── demo-a
│           └── my-deployment-a-6bbd894c5.yaml
├── secrets
│   └── namespaces
│       ├── demo-a
│       │   └── my-secret-a.yaml
│       └── demo-b
│           └── my-secret-b.yaml
├── serviceaccounts
│   └── namespaces
│       ├── demo-a
│       │   └── my-serviceaccount-a.yaml
│       └── demo-b
│           └── my-serviceaccount-b.yaml
├── services
│   └── namespaces
│       ├── demo-a
│       │   └── my-service-a.yaml
│       └── demo-b
│           └── my-service-b.yaml
└── statefulsets.apps
    └── namespaces
        └── demo-b
            └── my-statefulset.yaml
```

We followed this file structure for backing up manifests of resources: 

``` yaml
resources/
├── <groupResourceClusterScoped>/               # e.g., clusterroles.rbac.authorization.k8s.io
│   └── cluster/
│       └── <resource-name>.yaml

├── <groupResourceNamespaced>/                  # e.g., deployments.apps, configmaps 
│   └── namespaces/
│       ├── namespace-1/
│       │   ├── <resource-1>.yaml
│       │   ├── <resource-2>.yaml
│       │   └── ...
│       ├── namespace-2/
│       │   ├── <resource-1>.yaml
│       │   └── ...
│       └── namespace-n/
│           ├── ... 
│           └── <resource-n>.yaml

```

---

Let's inspect the YAML of `my-statefulset.yaml` file under `demo-b` namespace.

```yaml
arnab@nipun-pc:~/Downloads/s3-repo-cluster-resources-backup-frequent-backup-1752139981/manifest/kubestash-tmp/manifest$ cat statefulsets.apps/namespaces/demo-b/my-statefulset.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"StatefulSet","metadata":{"annotations":{},"labels":{"app":"my-sts"},"name":"my-statefulset","namespace":"demo-b"},"spec":{"replicas":3,"selector":{"matchLabels":{"app":"my-sts"}},"serviceName":"my-service-b","template":{"metadata":{"labels":{"app":"my-sts"}},"spec":{"containers":[{"image":"nginx:latest","name":"my-container-b","ports":[{"containerPort":8080}],"volumeMounts":[{"mountPath":"/etc/config","name":"config-volume-b"},{"mountPath":"/etc/secret","name":"secret-volume-b"},{"mountPath":"/data","name":"storage-volume-b"}]}],"serviceAccountName":"my-serviceaccount-b","volumes":[{"configMap":{"name":"my-config-b"},"name":"config-volume-b"},{"name":"secret-volume-b","secret":{"secretName":"my-secret-b"}},{"name":"storage-volume-b","persistentVolumeClaim":{"claimName":"my-pvc-b"}}]}}}}
  labels:
    app: my-sts
  name: my-statefulset
  namespace: demo-b
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
  podManagementPolicy: OrderedReady
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: my-sts
  serviceName: my-service-b
  template:
    metadata:
      labels:
        app: my-sts
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: Always
        name: my-container-b
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/config
          name: config-volume-b
        - mountPath: /etc/secret
          name: secret-volume-b
        - mountPath: /data
          name: storage-volume-b
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: my-serviceaccount-b
      serviceAccountName: my-serviceaccount-b
      volumes:
      - configMap:
          defaultMode: 420
          name: my-config-b
        name: config-volume-b
      - name: secret-volume-b
        secret:
          defaultMode: 420
          secretName: my-secret-b
      - name: storage-volume-b
        persistentVolumeClaim:
          claimName: my-pvc-b
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
status:
  availableReplicas: 3
  collisionCount: 0
  currentReplicas: 3
  currentRevision: my-statefulset-7bc9c486fc
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
  updateRevision: my-statefulset-7bc9c486fc
  updatedReplicas: 3
```

---

Now, you can use these YAML files to re-create your desired application.

## Restore

Let's demostrate an accidental situation and assume that the resources are gone from the cluster. 

Pause the running backupconfiguration using kubestash cli command: 

```fish 
$ kubectl kubestash pause cluster-resources-backup -n demo
I0710 15:33:36.497053 1445161 pause.go:51] BackupConfiguration demo/cluster-resources-backup has been paused successfully
```

Delete all the resources having label `app=my-app`
```bash 
$ kubectl delete replicaset,secret,configmap,statefulset,role,rolebinding,clusterrole,clusterrolebinding,persistentvolume,persistentvolumeclaim,service,serviceaccount,deployment,pod -n demo-a -l app=my-app
replicaset.apps "my-deployment-a-6bbd894c5" deleted
secret "my-secret-a" deleted
configmap "my-config-a" deleted
Warning: deleting cluster-scoped resources, not scoped to the provided namespace
clusterrole.rbac.authorization.k8s.io "my-clusterrole-a" deleted
clusterrolebinding.rbac.authorization.k8s.io "my-clusterrolebinding-a" deleted
persistentvolumeclaim "my-pvc-a" deleted
service "my-service-a" deleted
serviceaccount "my-serviceaccount-a" deleted
deployment.apps "my-deployment-a" deleted
pod "my-deployment-a-6bbd894c5-67lvb" deleted
pod "my-deployment-a-6bbd894c5-b9ks5" deleted
pod "my-deployment-a-6bbd894c5-jpm4x" deleted
pod "my-deployment-a-6bbd894c5-k8sjn" deleted
pod "my-deployment-a-6bbd894c5-p992t" deleted
pod "my-deployment-a-6bbd894c5-s72fw" deleted
```

Verify deletion:  

```bash 
$ kubectl get  replicaset,secret,configmap,clusterrole,clusterrolebinding,persistentvolumeclaim,service,serviceaccount,deployment,pod -n demo-a -l app=my-app
No resources found
```

---

Delete all the resources having label `app=my-sts`
```bash
$ kubectl delete  secret,configmap,statefulset,clusterrole,clusterrolebinding,persistentvolumeclaim,service,serviceaccount,pod -n demo-b -l app=my-sts
secret "my-secret-b" deleted
configmap "my-config-b" deleted
statefulset.apps "my-statefulset" deleted
Warning: deleting cluster-scoped resources, not scoped to the provided namespace
clusterrole.rbac.authorization.k8s.io "my-clusterrole-b" deleted
clusterrolebinding.rbac.authorization.k8s.io "my-clusterrolebinding-b" deleted
persistentvolumeclaim "my-pvc-b" deleted
service "my-service-b" deleted
serviceaccount "my-serviceaccount-b" deleted
pod "my-statefulset-0" deleted
pod "my-statefulset-1" deleted
pod "my-statefulset-2" deleted
``` 
Verify deletion: 
```bash 
$ kubectl get replicaset,secret,configmap,
statefulset,clusterrole,clusterrolebinding,
persistentvolume,persistentvolumeclaim,service,serviceaccount,deployment,pod 
-n demo-b -l app=my-sts
No resources found
```
---

Now apply `RestoreSession` to restore your target resources from snapshot.

#### Create RestoreSession

Below is the YAML for `RestoreSession` object we care going to use to restore the YAMLs and apply those YAMLs to create the lost/deleted cluster resources,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: cluster-resources-restore
  namespace: demo
spec:
...
  addon:
    name: kubedump-addon
    tasks:
      - name: manifest-restore
        params:
          IncludeClusterResources: "true"
          ExcludeNamespaces: "demo-a"
          ExcludeResources: "nodes.metrics.k8s.io,nodes,pods.metrics.k8s.io,metrics.k8s.io,endpointslices.discovery.k8s.io"
    jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader-writer
```
---

### Flags in `manifest-restore` task in KubeDump

We have introduced several flags to filter resources during the restore process.

``` yaml
- ANDedLabelSelectors
  Usage: A set of labels, all of which need to be matched
  to filter the resources.
  Default: ""
  Required: false
  Example: "app:my-app,db:mongo,db"

- ORedLabelSelectors
  Usage: A set of labels, at least one of which need to
  be matched to filter the resources.
  Default: ""
  Required: false
  Example: "app:nginx,app:redis"

- IncludeClusterResources
  Usage: Specify whether to restore
  cluster scoped resources.
  Default: "false"
  Required: false
  Example: "true"

- IncludeNamespaces
  Usage: Namespaces to include in backup.
  Default: "*"
  Required: false
  Example: "demo,kubedb,kubestash"

- ExcludeNamespaces
  Usage: Namespaces to exclude from backup.
  Default: ""
  Required: false
  Example: "default,kube-system"

- IncludeResources
  Usage: Resource types to include in backup.
  Default: "*"
  Required: false
  Example: "secrets,configmaps,deployments"
  
- ExcludeResources
  Usage: Resource types to exclude from backup
  Default: ""
  Required: false
  Example: "persistentvolumeclaims,persistentvolumes"
  
- OverrideResources
  Usage: Specify whether to override resources while restoring
  Default: "false"
  Required: false
  Example: "false"

- RestorePVs
  Usage: Specify whether to restore PersistentVolumes
  Default: "false"
  Required: false
  Example: "true"

- StorageClassMappings
  Usage: Mapping of old to new storage classes
  Default: ""
  Required: false
  Example: "gp2=ebs-sc,standard=fast-storage"
``` 

Let's create the `RestoreSession` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/clusterResourcesBackup/examples/restoresession.yaml
restoresession.core.kubestash.com/cluster-resources-restore created
```

Verify the recovery of cluster resources: 

```bash 
Every 2.0s: kubectl get replicaset,secret,configmap,statefulset,role,rolebinding,clusterrole,clusterrolebinding,persistentvolume,persistentvolumeclaim,service,serviceaccount,deployment,pod -n demo-b -l app=my-sts         nipun-pc: Thu Jul 10 11:47:50 2025

NAME                 TYPE     DATA   AGE
secret/my-secret-b   Opaque   2      52s

NAME                    DATA   AGE
configmap/my-config-b   1      52s

NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/my-service-b   ClusterIP   10.43.63.79   <none>        80/TCP    52s

NAME                                                     CREATED AT
clusterrole.rbac.authorization.k8s.io/my-clusterrole-b   2025-07-18T11:12:35Z

NAME                                                                   ROLE                           AGE
clusterrolebinding.rbac.authorization.k8s.io/my-clusterrolebinding-b   ClusterRole/my-clusterrole-b   52s

NAME                                 SECRETS   AGE
serviceaccount/my-serviceaccount-b   0         52s

NAME                              READY   AGE
statefulset.apps/my-statefulset   3/3     52s

NAME                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/my-pvc-b   Bound    pvc-d6aea112-c6d6-4764-8358-711baff501b0   5Gi        RWO            longhorn       <unset>                 52s

NAME                   READY   STATUS    RESTARTS   AGE
pod/my-statefulset-0   1/1     Running   0          52s
pod/my-statefulset-1   1/1     Running   0          36s
pod/my-statefulset-2   1/1     Running   0          33s
```

Inspect the manifest of statefulset: 

```bash 
$ kubectl get statefulset -n demo-b -oyaml
apiVersion: v1
items:
- apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"apps/v1","kind":"StatefulSet","metadata":{"annotations":{},"labels":{"app":"my-sts"},"name":"my-statefulset","namespace":"demo-b"},"spec":{"replicas":3,"selector":{"matchLabels":{"app":"my-sts"}},"serviceName":"my-service-b","template":{"metadata":{"labels":{"app":"my-sts"}},"spec":{"containers":[{"image":"nginx:latest","name":"my-container-b","ports":[{"containerPort":8080}],"volumeMounts":[{"mountPath":"/etc/config","name":"config-volume-b"},{"mountPath":"/etc/secret","name":"secret-volume-b"},{"mountPath":"/data","name":"storage-volume-b"}]}],"serviceAccountName":"my-serviceaccount-b","volumes":[{"configMap":{"name":"my-config-b"},"name":"config-volume-b"},{"name":"secret-volume-b","secret":{"secretName":"my-secret-b"}},{"name":"storage-volume-b","persistentVolumeClaim":{"claimName":"my-pvc-b"}}]}}}}
    creationTimestamp: "2025-07-18T11:12:35Z"
    generation: 1
    labels:
      app: my-sts
    name: my-statefulset
    namespace: demo-b
    resourceVersion: "16056"
    uid: 1a2fa288-a9f6-4d8e-be12-bf788b135c2f
  spec:
    persistentVolumeClaimRetentionPolicy:
      whenDeleted: Retain
      whenScaled: Retain
    podManagementPolicy: OrderedReady
    replicas: 3
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        app: my-sts
    serviceName: my-service-b
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: my-sts
      spec:
        containers:
        - image: nginx:latest
          imagePullPolicy: Always
          name: my-container-b
          ports:
          - containerPort: 8080
            protocol: TCP
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
          - mountPath: /etc/config
            name: config-volume-b
          - mountPath: /etc/secret
            name: secret-volume-b
          - mountPath: /data
            name: storage-volume-b
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        serviceAccount: my-serviceaccount-b
        serviceAccountName: my-serviceaccount-b
        terminationGracePeriodSeconds: 30
        volumes:
        - configMap:
            defaultMode: 420
            name: my-config-b
          name: config-volume-b
        - name: secret-volume-b
          secret:
            defaultMode: 420
            secretName: my-secret-b
        - name: storage-volume-b
          persistentVolumeClaim:
            claimName: my-pvc-b
    updateStrategy:
      rollingUpdate:
        partition: 0
      type: RollingUpdate
  status:
    availableReplicas: 3
    collisionCount: 0
    currentReplicas: 3
    currentRevision: my-statefulset-7bc9c486fc
    observedGeneration: 1
    readyReplicas: 3
    replicas: 3
    updateRevision: my-statefulset-7bc9c486fc
    updatedReplicas: 3
kind: List
metadata:
  resourceVersion: ""
```

---

## Intelligent and Automatic Cluster-Wide Resource Restoration

KubeStash includes an **automatic, dependency-aware mechanism** to restore entire cluster resources from backed-up YAMLs — now enhanced with **multi-iteration restores** and **owner reference fixes** for new clusters.

### Key Features

#### Built-in Priority and Ordering Logic
Resources are restored in a sequence that respects their dependencies.
Critical building blocks like **CRDs**, **Namespaces**, **StorageClasses**, **PVCs**, **ServiceAccounts**, **Secrets**, and **ConfigMaps** are applied **before** dependent workloads such as **Pods**, **ReplicaSets**, and **Services**.

#### Multi-Iteration Restore Process
Instead of a single-pass restore, KubeStash uses **multiple iterations**.
This allows dependent resources that initially fail (due to missing prerequisites) to succeed in later passes as their dependencies become available.

#### Automatic Owner Reference Fixing
To avoid broken references in a fresh cluster (where resource UIDs differ from the original backup):

1. All resources are initially restored as **orphans** (no owner references)
2. Once all resources are created, KubeStash **patches valid owner references** with the correct UIDs in the new cluster

This ensures workloads and their dependent resources remain correctly linked without conflicts.

#### No Manual Intervention Required
No need to write custom scripts or manually define restore orders — everything is handled automatically.

---

### Why This Matters
These improvements ensure a **robust, repeatable, and reliable** cluster restoration process.
Whether restoring a simple setup or a complex production-grade environment, resource dependencies and ownership metadata are handled **safely and automatically** — letting you restore your entire cluster with **confidence and ease**.

---

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo backupconfiguration cluster-resources-backup
kubectl delete -n demo restoresession cluster-resources-restore
kubectl delete -n demo serviceaccount cluster-resource-reader-writer
kubectl delete clusterrole cluster-resource-reader-writer
kubectl delete clusterrolebinding cluster-resource-reader-writer
kubectl delete retentionPolicy -n demo demo-retention
kubectl delete -n demo backupstorage azure-storage
kubectl delete secret -n demo encrypt-secret
kubectl delete secret -n demo azure-secret
kubectl delete secrets,configmaps,services,clusterroles,clusterrolebindings,serviceaccounts,deployments,persistentvolumeclaims,persistentvolumes,pods,replicasets -n demo-a -l app=my-app
kubectl delete secrets,configmaps,services,clusterroles,clusterrolebindings,serviceaccounts,statefulsets,persistentvolumeclaims,persistentvolumes,pods,replicasets -n demo-b -l app=my-sts
kubectl delete ns demo
kubectl delete ns demo-a
kubectl delete ns demo-b
```
