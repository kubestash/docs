---
title: "Filtering Demonstration | KubeStash"
description: "Demonstration of filtering operations in KubeStash"
menu_name: docs_{{ .version }}
section_menu_id: guides
product_name: KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-cluster-resources-filtering-demonstration
    name: "Filtering Demonstration"
    parent: kubestash-cluster-resources
    weight: 30
---


## Backup and Restore Cluster Resources Using Filtering Operations

This guide will demonstrate how to use filtering operations to take backup and restore cluster resources.

### Create Resources

We are going to use separate namespaces `demo`, `demo-a` and `demo-b` throughout this tutorial. 

Create the namespaces.

```bash
$ kubectl create ns demo
namespace/demo created
$ kubectl create ns demo-a 
namespace/demo-a created
$ kubectl create ns demo-b
namespace/demo-b created
```

We need to create some resources both namespace scoped and cluster scoped to demonstrate our backup and restore process. For simplification we will be using two `labels` to demonstarte separation of the resources.   


For label `app=my-app` all of the resources will be either in `demo-a` namespace or cluster scoped.     

#### Create Resources Having Label `app=my-app` 

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
        - name: storage-volume-a`
          persistentVolumeClaim:
            claimName: my-pvc-a
```

Let's create the objects having label `app:my-app` we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/filtering-demonstration/examples/resources-a.yaml
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

#### Create Resources Having Label `app=my-sts` 

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/filtering-demonstration/examples/resources-b.yaml
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

### Configure Storage Backend and RBAC

Please refer to the following [link](https://github.com/kubestash/docs/latest/guides/cluster-resources/configure-storage-and-rbac/index.md) to Configure Storage Backend and RBAC.

--- 

### Create RetentionPolicy

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

>Notice the `spec.usagePolicy` that allows referencing the `RetentionPolicy` from all namespaces.For more details on configuring it for specific namespaces, please refer to the following [link](https://github.com/kubestash/docs/latest/concepts/crds/retentionpolicy/index.md).

Let's create the `RetentionPolicy` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/filtering-demonstration/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

---

### Create BackupConfiguration

Below is the YAML for `BackupConfiguration` object we care going to use to backup the YAMLs of the cluster resources,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: cluster-resources-backup
  namespace: demo
spec:
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

Let's create the `BackupConfiguration` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/filtering-demonstration/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/cluster-resources-backup created
```

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

### Download the YAMLs

KubeStash provides a [kubectl plugin](https://github.com/kubestash/docs/latest/guides/cli/kubectl-plugin/index.md#download-snapshot) for making it easy to download a snapshot locally.

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

Now, you can use these YAML files to **re-create your desired cluster resources**.

## Restore

Let's demostrate an accidental situation and assume that the resources are gone from the cluster. 

`Pause` the running `BackupConfiguration` using kubestash cli command: 

```fish 
$ kubectl kubestash pause cluster-resources-backup -n demo
I0710 15:33:36.497053 1445161 pause.go:51] BackupConfiguration demo/cluster-resources-backup has been paused successfully
```

**Delete all the resources having label `app=my-app`:** 
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

**Verify deletion:**  
```bash 
$ kubectl get  replicaset,secret,configmap,clusterrole,clusterrolebinding,persistentvolumeclaim,service,serviceaccount,deployment,pod -n demo-a -l app=my-app
No resources found
```

**Delete all the resources having label `app=my-sts`:** 
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

**Verify deletion:** 
```bash 
$ kubectl get replicaset,secret,configmap,
statefulset,clusterrole,clusterrolebinding,
persistentvolume,persistentvolumeclaim,service,serviceaccount,deployment,pod 
-n demo-b -l app=my-sts
No resources found
```

---

Now apply `RestoreSession` to restore your target resources from `snapshot`.

### Create RestoreSession

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
          ExcludeResources: "nodes.metrics.k8s.io,nodes,pods.metrics.k8s.io,endpointslices.discovery.k8s.io"
    jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader-writer
```

Let's create the `RestoreSession` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/filtering-demonstration/examples/restoresession.yaml
restoresession.core.kubestash.com/cluster-resources-restore created
```

**Verify the recovery of cluster resources:** 
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

**Inspect the manifest of statefulset:**
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
      matchLabels:docs
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

As we can see all the targeted resources are up and running again our manifest restoring was successfull. 

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