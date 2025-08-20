---
title: "Full Cluster Backup & Restore | KubeStash"
description: "Full Cluster Backup & Restore"
menu_name: docs_{{ .version }}
section_menu_id: guides
product_name: KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-full-cluster-backup-and-restore
    name: "Full Cluster Backup & Restore"
    parent: kubestash-cluster-resources
    weight: 35
---

## Full Cluster Resources Backup & Restore

In this tutorial, we will use two AKS clusters — **one for backup and another for restore**.

- Let’s say the first cluster is `aks-1`, which will be used for the **backup process**.

- The second cluster is `aks-2`, which will be used for the **restore process**.

In `aks-1`, **KubeDB** will be installed in the `kubedb` namespace, and a **MySQL** database will be running in the `db` namespace, provisioned by KubeDB.

Later, using the backed-up `snapshot`, we will restore both **KubeDB** and the **MySQL** database in `aks-2`. After the restore, we will verify that the database is up and running, provisioned by KubeDB, in the **completely new cluster** (`aks-2`).

--- 

### Backup Process in Cluster `aks-1`

#### Create Cluster `aks-1`: 

You can create an aks cluster using the azure cli commands. Follow the [official documentation](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli) for cluster creation. 

Example: 
```bash
export RG_NAME="yourResourceGroupName"
export AKS_NAME="aks-1"
az aks create -g $RG_NAME -n $AKS_NAME --enable-oidc-issuer --enable-workload-identity --node-count 1
```
---

#### Install `KubeDB` in aks-1 

Follow the `KubeDB` [official setup page](https://kubedb.com/docs/latest/setup/install/kubedb/) for getting a license and installing `KubeDB`.

After that make sure `KubeDB` is up and running in aks-1. 
```bash
$ kubectl get pods -n kubedb
NAME                                            READY   STATUS    RESTARTS   AGE
kubedb-kubedb-autoscaler-0                      1/1     Running   0          119m
kubedb-kubedb-ops-manager-0                     1/1     Running   0          119m
kubedb-kubedb-provisioner-0                     1/1     Running   0          119m
kubedb-kubedb-webhook-server-656d654875-hm2x5   1/1     Running   0          119m
kubedb-petset-7d7f6dccf7-75sjn                  1/1     Running   0          119m
kubedb-sidekick-944f4df5-nv67b                  1/1     Running   0          119m
```

#### Create a MySQL Database: 

First, create a dedicated namespace named db:
```bash 
$ kubectl create ns db 
namespace/db created
```

Apply the following manifest to create a MySQL database with Group Replication:
```yaml 
apiVersion: kubedb.com/v1
kind: MySQL
metadata:
  name: my-mysql
  namespace: db
spec:
  version: "8.1.0"  
  replicas: 3
  topology:
    mode: GroupReplication
  storageType: Durable
  storage:
    storageClassName: "default"
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  deletionPolicy: WipeOut
```
Let's create the `MySQL` database: 
```bash 
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/full-cluster-backup-and-restore/examples/mysql.yaml 
mysql.kubedb.com/my-mysql created
```

**Verify the database**: 
Check if the `MySQL` database is ready:
```bash 
$ kubectl get mysql.kubedb.com  -n db
NAME       VERSION   STATUS   AGE
my-mysql   8.1.0     Ready    40m
```

**Verify the pods**: 
Confirm that the `MySQL` pods are running:
```bash 
$ kubectl get pods -n db
NAME         READY   STATUS    RESTARTS   AGE
my-mysql-0   2/2     Running   0          38m
my-mysql-1   2/2     Running   0          37m
my-mysql-2   2/2     Running   0          37m
```

**Verify the PVCs**: 
Check that the PersistentVolumeClaims (PVCs) are successfully bound:
```bash 
$ kubectl get pvc -n db  
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-my-mysql-0   Bound    pvc-019b31b3-d08e-43b5-855d-9c152ce6c701   1Gi        RWO            default        <unset>                 39m
data-my-mysql-1   Bound    pvc-c36b8673-7ef1-404f-bf54-92accfc22c71   1Gi        RWO            default        <unset>                 39m
data-my-mysql-2   Bound    pvc-fdeccbac-397e-4f9b-bf84-97400f38dbe6   1Gi        RWO            default        <unset>                 38m
```

---

#### Configure Storage Backend and RBAC

#### Create BackupStorage: 

Please refer to the following [guide](/docs/guides/backends/azure/index.md) to configure Microsoft Azure Backend Storage.

Example of `BackupStorage`: 
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
      storageAccount: <your-storage-account>
      container: <name-of-container>
      prefix: <prefix-name>
      secretName: <secret-of-storage>
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true
  deletionPolicy: Delete
```
**Verify BackupStorage**: 
Check if `BackupStorage` is ready:
```bash 
$ kubectl get backupstorage.storage.kubestash.com -n demo azure-storage
NAME            PROVIDER   DEFAULT   DELETION-POLICY   TOTAL-SIZE   PHASE   AGE
azure-storage   azure      true      Delete                         Ready   8m2s
```

>Note: Set the `deletionPolicy` of `BackupStorage` to `Delete`. This ensures that snapshots remain accessible from other clusters even if the `BackupStorage` object is deleted.

Please refer to the following [guide](/docs/guides/cluster-resources/configure-storage-and-rbac/#create-encryption-secret) to create a secret called `encrypt-secret` with the Restic password.

Please refer to the following [guide](/docs/guides/cluster-resources/configure-storage-and-rbac/#create-rbac-for-backupconfiguration) to configure the necessary RBAC permissions for `BackupConfiguration`.

--- 

#### Create RetentionPolicy

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/full-cluster-backup-and-restore/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

**Verify RetentionPolicy**: 

```bash 
kubectl get retentionpolicy -n demo
NAME             MAX-RETENTION-PERIOD   DEFAULT   AGE
demo-retention   2mo                              15s
```
---

#### Create BackupConfiguration

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
              IncludeNamespaces: "kubedb,db"                   
              IncludeResources: "*"                            
        jobTemplate:                                           
          spec:                                                
            serviceAccountName: cluster-resource-reader-writter
```

Let's create the `BackupConfiguration` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/full-cluster-backup-and-restore/examples/backupconfiguration.yaml
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
NAME         INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
azure-repo   true        2                1.596 MiB   Ready   85s                      4m19s                         28s
```

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

NAME                                                  INVOKER-TYPE          
cluster-resources-backup-frequent-backup-1755596627   BackupConfiguration   cluster-resources-backup   Succeeded   21s        5m42s

```

**Verify Backup:**

When `BackupSession` is created, KubeStash operator creates `Snapshot` for each `Repository` listed in the respective session of the `BackupConfiguration`. Since we have only specified only one repository in the session, at this moment we should have one `Snapshot`. 

Run the following command to check the respective `Snapshot`,

```bash

arnab@nipun-pc ~> kubectl get snapshots.storage.kubestash.com -n demo 
NAME                                                             REPOSITORY   SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
azure-repo-cluster-resources-backup-frequent-backup-1755596627   azure-repo   frequent-backup   2025-08-19T09:43:47Z   Delete            Succeeded   4m26s
```

---

#### Download the YAMLs

KubeStash provides a [kubectl plugin](/docs/guides/cli/kubectl-plugin/index.md#download-snapshot) for making it easy to download a snapshot locally.

Now, let's download the latest Snapshot from our backed-up data into the `$HOME/Downloads/kubestash` folder of our local machine.

```bash
$ kubectl kubestash download --namespace=demo azure-repo-cluster-resources-backup-frequent-backup-1755596627 --destination=<path-to-download>
```

Now, lets use [tree](https://linux.die.net/man/1/tree) command to inspect downloaded YAMLs files.

```bash
~/Downloads/azure-repo-cluster-resources-backup-frequent-backup-1755596627/manifest/kubestash-tmp/manifest$ tree
...
# (output trancated)
├── replicasets.apps
│   └── namespaces
│       └── kubedb
│           ├── kubedb-kubedb-webhook-server-656d654875.yaml
│           ├── kubedb-petset-7d7f6dccf7.yaml
│           └── kubedb-sidekick-944f4df5.yaml
├── rolebindings.rbac.authorization.k8s.io
│   └── namespaces
│       ├── db
│       │   └── my-mysql.yaml
│       └── kubedb
│           ├── kubedb-kubedb-webhook-server:leader-election.yaml
│           ├── kubedb-petset:leader-election.yaml
│           └── kubedb-sidekick:leader-election.yaml
├── roles.rbac.authorization.k8s.io
│   └── namespaces
│       ├── db
│       │   └── my-mysql.yaml
│       └── kubedb
│           ├── kubedb-kubedb-webhook-server:leader-election.yaml
│           ├── kubedb-petset:leader-election.yaml
│           └── kubedb-sidekick:leader-election.yaml
├── schemaregistryversions.catalog.kubedb.com
│   └── cluster
│       ├── 2.5.11.final.yaml
│       └── 3.15.0.yaml
├── secrets
│   └── namespaces
│       ├── db
│       │   └── my-mysql-auth.yaml
│       └── kubedb
│           ├── kubedb-kubedb-autoscaler-license.yaml
│           ├── kubedb-kubedb-ops-manager-license.yaml
│           ├── kubedb-kubedb-provisioner-license.yaml
│           ├── kubedb-kubedb-webhook-server-cert.yaml
│           ├── kubedb-petset-cert.yaml
│           ├── kubedb-sidekick-cert.yaml
│           └── sh.helm.release.v1.kubedb.v1.yaml
├── serviceaccounts
│   └── namespaces
│       ├── db
│       │   ├── default.yaml
│       │   └── my-mysql.yaml
│       └── kubedb
│           ├── default.yaml
│           ├── kubedb-kubedb-autoscaler.yaml
│           ├── kubedb-kubedb-ops-manager.yaml
│           ├── kubedb-kubedb-provisioner.yaml
│           ├── kubedb-kubedb-webhook-server.yaml
│           ├── kubedb-petset.yaml
│           └── kubedb-sidekick.yaml
├── services
│   └── namespaces
│       ├── db
│       │   ├── my-mysql-pods.yaml
│       │   ├── my-mysql-standby.yaml
│       │   └── my-mysql.yaml
│       └── kubedb
│           ├── kubedb-kubedb-autoscaler-headless.yaml
│           ├── kubedb-kubedb-autoscaler.yaml
│           ├── kubedb-kubedb-ops-manager-headless.yaml
│           ├── kubedb-kubedb-ops-manager.yaml
│           ├── kubedb-kubedb-provisioner-headless.yaml
│           ├── kubedb-kubedb-provisioner.yaml
│           ├── kubedb-kubedb-webhook-server.yaml
│           ├── kubedb-petset.yaml
│           └── kubedb-sidekick.yaml
├── statefulsets.apps
│   └── namespaces
│       └── kubedb
│           ├── kubedb-kubedb-autoscaler.yaml
│           ├── kubedb-kubedb-ops-manager.yaml
│           └── kubedb-kubedb-provisioner.yaml

138 directories, 793 files
# (output trancated)
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

### Restore Process in Cluster `aks-2`

#### Create Cluster `aks-2`: 

You can create an aks cluster using the azure cli commands. Follow the [official documentation](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli) for cluster creation. 

Example: 
```bash
export RG_NAME="yourResourceGroupName"
export AKS_NAME="aks-2"
az aks create -g $RG_NAME -n $AKS_NAME --enable-oidc-issuer --enable-workload-identity --node-count 1
```
---

#### Install `KubeStash` in Cluster `aks-2`:

Since `aks-2` is a new cluster, you need to install `KubeStash` before using it. 

Follow the `KubeStash` [official setup page](/docs/setup/install/kubestash/) for getting a license and installing `KubeStash`.

**Verify KubeStash**: 

Verify that `KubeStash` pods are running in the `kubestash` namespace: 
```bash
$ kubectl get pods -n kubestash
NAME                                                           READY   STATUS    RESTARTS   AGE
kubestash-kubestash-operator-operator-d57486655-7p9c4          1/1     Running   0          5h43m
kubestash-kubestash-operator-webhook-server-6fb8f5cfb9-scrx8   1/1     Running   0          5h43m

```

---

#### Configure Storage Backend and RBAC in Cluster aks-2

#### Create BackupStorage: 

Please refer to the following [link](/docs/guides/backends/azure/index.md) to configure Microsoft Azure Backend Storage.

Example of `BackupStorage`: 
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
      storageAccount: <your-storage-account>
      container: <name-of-container>
      prefix: <prefix-name>
      secretName: <secret-of-storage>
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true
  deletionPolicy: Delete
```
---

**Verify BackupStorage**: 
Check if `BackupStorage` is ready:
```bash 
$ kubectl get backupstorage.storage.kubestash.com -n demo azure-storage
NAME            PROVIDER   DEFAULT   DELETION-POLICY   TOTAL-SIZE   PHASE   AGE
azure-storage   azure      true      Delete                         Ready   6m13s
```

---

**Verify Snapshots**: 
Check if the `snapshots` created in cluster `aks-1` are available in cluster `aks-2`: 
```bash 
$ kubectl get snapshots.storage.kubestash.com -n demo
NAME                                                             REPOSITORY   SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
azure-repo-cluster-resources-backup-frequent-backup-1755596627   azure-repo   frequent-backup   2025-08-19T09:43:47Z   Delete            Succeeded   7m13s
```

>Note: 
- The `deletionPolicy` of `BackupStorage` must be set to `Delete` to make snapshots accessible from other clusters.

- The `storageAccount`, `prefix`, and `container` values must match those of the `BackupStorage` used in cluster aks-1.

---

Please refer to the following [guide](/docs/guides/cluster-resources/configure-storage-and-rbac/#create-encryption-secret) to create a secret called `encrypt-secret` with the Restic password.

Please refer to the following [guide](/docs/guides/cluster-resources/configure-storage-and-rbac/#create-rbac-for-backupconfiguration) to configure the necessary RBAC permissions for `RestoreSession`.

--- 

Now apply `RestoreSession` to restore your target resources from snapshot.

#### Create RestoreSession

Below is the YAML for `RestoreSession` object we care going to use to restore the YAMLs and apply those YAMLs to create the lost/deleted cluster resources,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: cluster-restore
  namespace: demo
spec:
  addon:
    name: kubedump-addon
    tasks:
      - name: manifest-restore
        params:
          IncludeClusterResources: "true"
          IncludeNamespaces: "kubedb,db"
          IncludeResources: "*"
          ExcludeResources: "nodes.metrics.k8s.io,nodes,pods.metrics.k8s.io,endpointslices.discovery.k8s.io" 
    jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader-writter
```
---

Let's create the `RestoreSession` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/full-cluster-resource/examples/restoresession.yaml
restoresession.core.kubestash.com/cluster-resources-restore created
```

**Verify RestoreSession**: 
Check if the `RestoreSession` has completed successfully:
```bash
$ kubectl get restoresession -n demo
NAME              REPOSITORY   PHASE       DURATION   AGE
cluster-restore   azure-repo   Succeeded   31s        54s
```
> Note: Azure may restrict certain resources or API groups. If this happens, exclude them using the `ExcludeResources` parameter when applying the `RestoreSession`.


**Verify `KubeDB` restoration**: 

```bash 
$ kubectl get pods -n kubedb
NAME                                            READY   STATUS             RESTARTS        AGE
kubedb-kubedb-autoscaler-0                      0/1     CrashLoopBackOff   9 (3m44s ago)   24m
kubedb-kubedb-ops-manager-0                     0/1     CrashLoopBackOff   9 (3m24s ago)   24m
kubedb-kubedb-provisioner-0                     0/1     CrashLoopBackOff   9 (3m17s ago)   24m
kubedb-kubedb-webhook-server-656d654875-hm2x5   1/1     Running            1 (24m ago)     24m
kubedb-petset-7d7f6dccf7-75sjn                  1/1     Running            0               24m
kubedb-sidekick-944f4df5-nv67b                  1/1     Running            0               24m
```

>Note: Some of the KubeDB pods may appear in `CrashLoopBackOff` due to license restrictions. After upgrading the `license` for the new cluster, those pods will transition to the `Running` state.

**Verify the `MySQL` database: 
Check the status of the `MySQL` database: 
```bash 
$ kubectl get mysql.kubedb.com  -n db
NAME       VERSION   STATUS   AGE
my-mysql   8.1.0              19m
```

>Note: The `STATUS` field may appear empty initially. After upgrading the license for the new cluster, the database will transition to the `Ready` state.

#### Upgrade the KubeDB License for the New Cluster (aks-2): 

Follow the `KubeDB` [official setup page](https://kubedb.com/docs/latest/setup/install/kubedb/) for getting a license and upgrading the `KubeDB` with that license.

```bash 
$ export LICENSE_FILE=/home/arnab/Downloads/kubedb-license-aks-2.txt
$ helm upgrade kubedb oci://ghcr.io/appscode-charts/kubedb \
        --version <kubedb-version> \
        --namespace kubedb  \
        --set-file global.license=$LICENSE_FILE \
        --wait --burst-limit=10000 --debug
```
---

**Verify KubeDB Pods**:
After upgrading the license, ensure that all `KubeDB` pods are running in `aks-2`:
```bash 
$ kubectl get pods -n kubedb
NAME                                            READY   STATUS    RESTARTS   AGE
kubedb-kubedb-autoscaler-0                      1/1     Running   0          52m
kubedb-kubedb-ops-manager-0                     1/1     Running   0          52m
kubedb-kubedb-provisioner-0                     1/1     Running   0          52m
kubedb-kubedb-webhook-server-57fcd55fb6-c8nnk   1/1     Running   0          52m
kubedb-petset-7ddcf965c4-ltw49                  1/1     Running   0          52m
kubedb-sidekick-69cf56c67f-kxk6s                1/1     Running   0          52m
```
---

**Verify the MySQL database**: 
Check if the `MySQL` database is ready:
```bash 
kubectl get mysql.kubedb.com  -n db 
NAME       VERSION   STATUS   AGE
my-mysql   8.1.0     Ready    90m
```

**Verify the pods**: 
Check if the `MySQL` pods are running:
```bash 
$ kubectl get pods -n db
NAME         READY   STATUS    RESTARTS   AGE
my-mysql-0   2/2     Running   0          105m
my-mysql-1   2/2     Running   0          105m
my-mysql-2   2/2     Running   0          105m
```

**Verify the PVCs**: 
Check if the PersistentVolumeClaims (PVCs) are successfully bound:
```bash 
$ kubectl get pvc -n db  
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-my-mysql-0   Bound    pvc-d9c88f86-6265-4494-abc4-37771b90708b   1Gi        RWO            default        <unset>                 103m
data-my-mysql-1   Bound    pvc-50d7c598-af6d-444d-9c63-a0d7fd97a04c   1Gi        RWO            default        <unset>                 103m
data-my-mysql-2   Bound    pvc-34530543-4b06-4c36-900f-75cccaa643f0   1Gi        RWO            default        <unset>                 103m
```

>Note: All PVCs are in `Bound` state, which indicates that `KubeDB` and the `MySQL` database have been successfully restored.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n db mysql.kubedb.com my-mysql
kubectl delete ns db
```

Follow the `KubeDB` [official setup page](https://kubedb.com/docs/latest/setup/install/kubedb/) to uninstall `KubeDB`.

Follow the `KubeStash` [official setup page](/docs/setup/uninstall/kubestash/) to uninstall `KubeStash`.

 