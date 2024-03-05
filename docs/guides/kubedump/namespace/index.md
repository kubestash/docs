---
title: Backup resource YAMLs of a Namespace | KubeStash
description: Take backup of resources YAMLs of a Namespace using KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-kubedump-namespace
    name: Namespace Backup
    parent: kubestash-kubedump
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup resource YAMLs of a Namespace using KubeStash

KubeStash `{{< param "info.version" >}}` supports taking backup of the resource YAMLs using `kubedump` plugin. This guide will show you how you can take a backup of the resource YAMLs of a particular namespace using KubeStash.

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
$ kubectl create ns demo
namespace/demo created
```

> Note: YAML files used in this tutorial are stored [here](https://github.com/kubestash/docs/tree/{{< param "info.version" >}}/docs/guides/kubedump/namespace/examples).

### Prepare for Backup

In this section, we are going to configure a backup for all the resource YAMLs of `kube-system` namespace.


#### Prepare Backend

Now, we are going to store our backed-up data into a GCS bucket using KubeStash. We have to create a Secret and a `BackupStorage` object with access credentials and backend information respectively. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

> For GCS backend, if the bucket does not exist, KubeStash needs `Storage Object Admin` role permissions to create the bucket. For more details, please check the following [guide](/docs/guides/backends/gcs/index.md).

**Create Secret:**

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/namespace/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-repo created
```

**Create RetentionPolicy:**

Now, we have to create a `RetentionPolicy` custom resource which specifies how the old `Snapshots` should be cleaned up.

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/namespace/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

#### Create RBAC

The `kubedump` plugin requires read permission for all the resources of the desired namespace. By default, KubeStash does not grant such permissions. We have to provide the necessary permissions manually.

Here, is the YAML of the `ServiceAccount`, `ClusterRole`, and `ClusterRoleBinding` that we are going to use for granting the necessary permissions.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-resource-reader
  namespace: demo
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
  namespace: demo
roleRef:
  kind: ClusterRole
  name: cluster-resource-reader
  apiGroup: rbac.authorization.k8s.io
```

Here, we have give permission to read all the cluster resources. You can restrict this permission to a particular namespace only.

Let's create the RBAC resources we have shown above,

```bash
❯ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/namespace/examples/rbac.yaml
serviceaccount/cluster-resource-reader created
clusterrole.rbac.authorization.k8s.io/cluster-resource-reader created
clusterrolebinding.rbac.authorization.k8s.io/cluster-resource-reader created
```

Now, we are ready for backup. In the next section, we are going to schedule a backup for our cluster resources.

### Backup

To schedule a backup, we have to create a `BackupConfiguration` object.

we also have to create another `Secret` with an encryption key `RESTIC_PASSWORD` for `Restic`. This secret will be used by `Restic` for both encrypting and decrypting the backup data during backup & restore.

**Create Secret:**

Let's create a secret named `encry-secret` with the Restic password.

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encryption-secret \
    --from-file=./RESTIC_PASSWORD 
secret/encryption-secret created
```

#### Create BackupConfiguration

Below is the YAML for `BackupConfiguration` object we care going to use to backup the YAMLs of the cluster resources,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: kube-system-backup
  namespace: demo
spec:
  target:
    apiGroup: ""
    kind: Namespace
    name: kube-system
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
          directory: /kube-system-manifests
          encryptionSecret:
            name: encryption-secret
            namespace: demo
          deletionPolicy: WipeOut
      addon:
        name: kubedump-addon
        tasks:
          - name: manifest-backup
        jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader
```

Here,

- `spec.sessions[*].addon.name` specifies the name of the `Addon` object that specifies addon configuration that will be used to perform backup of a stand-alone PVC.
- `spec.sessions[*].addon.tasks[*].name` specifies the name of the `Task` that holds the `Function` and their order of execution to perform backup of a stand-alone PVC.

Let's create the `BackupConfiguration` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/namespace/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/kube-system-backup created
```

#### Verify Backup Setup Successful

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the BackupConfiguration,

```bash
❯ kubectl get backupconfiguration -n demo
NAME                              PHASE   PAUSED   AGE
kube-system-backup                Ready            79s

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
trigger-kube-system-backup-frequent-backup            */5 * * * *   False     0        <none>          45s
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` crd using the following command,

```bash
$ Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-name=kube-system-backup                                               anisur: Fri Feb 23 18:34:53 2024

NAME                                            INVOKER-TYPE          INVOKER-NAME         PHASE       DURATION   AGE
kube-system-backup-frequent-backup-1708691400   BackupConfiguration   kube-system-backup   Succeeded              4m53s

```

**Verify Backup:**

When backup session is created, kubestash operator creates `Snapshot` which represents the state of backup run for each `Repository` which are provided in `BackupConfiguration`.

Run the following command to check `Snapshot` phase,

```bash
$ kubectl get snapshots -n demo
NAME                                                           REPOSITORY       SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-repository-kube-system-backup-frequent-backup-1708691400   gcs-repository   frequent-backup   2024-02-23T12:30:00Z   Delete            Succeeded   5m9s
```

When backup session is completed, KubeStash will update the respective `Repository` to reflect the latest state of backed up data.

Run the following command to check if a backup snapshot has been stored in the backend,

```bash
$ kubectl get repositories -n demo
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository   true        1                2.262 KiB   Ready   103s                     8m
```

From the output above, we can see that `1` snapshot has been stored in the backend specified by Repository `gcs-repository`.

Now, if we navigate to the GCS bucket, we will see the backed up data has been stored in `/manifest/namespace/kube-system` directory as specified by `.spec.backend.gcs.prefix` field of the `Repository` object.

<figure align="center">
  <img alt="Backup data in GCS Bucket" src="/docs/guides/kubedump/namespace/images/namespace_manifests_backup.png">
  <figcaption align="center">Fig: Backup data in GCS Bucket</figcaption>
</figure>

> Note: KubeStash keeps all the backed-up data encrypted. So, data in the backend will not make any sense until they are decrypted.

## Restore

KubeStash does not provide any automatic mechanism to restore the cluster resources from the backed-up YAMLs. Your application might be managed by Helm or by an operator. In such cases, just applying the YAMLs is not enough to restore the application. Furthermore, there might be an order issue. Some resources must be applied before others. It is difficult to generalize and codify various application-specific logic.

Therefore, it is the user's responsibility to download the backed-up YAMLs and take the necessary steps based on his application to restore it properly.

### Download the YAMLs

KubeStash provides a [kubectl plugin](/docs/guides/cli/cli/#download-snapshots) for making it easy to download a snapshot locally.

Now, let's download the latest Snapshot from our backed-up data into the `$HOME/Downloads/kubestash/namespace/kube-system` folder of our local machine.

```bash
❯ kubectl kubestash download --namespace=demo gcs-repository-kube-system-backup-frequent-backup-1708691400  --destination=$HOME/Downloads/kubestash/namespace/kube-system/
```

Now, lets use [tree](https://linux.die.net/man/1/tree) command to inspect downloaded YAMLs files.

```bash
❯ tree /home/anisur/Downloads/kubestash/namespace/kube-system
/home/anisur/Downloads/kubestash/namespace/kube-system
└── gcs-repository-kube-system-backup-frequent-backup-1708691400
    └── manifest
        └── tmp
            └── manifest
                ├── ConfigMap
                │   ├── coredns.yaml
                │   ├── extension-apiserver-authentication.yaml
                │   ├── kubeadm-config.yaml
                ├── ControllerRevision
                │   ├── kindnet-685dcc8c4c.yaml
                │   └── kube-proxy-7ccf78d585.yaml
                ├── DaemonSet
                │   ├── kindnet.yaml
                │   └── kube-proxy.yaml
                ├── Deployment
                │   └── coredns.yaml
                ├── Endpoints
                │   └── kube-dns.yaml
                ├── EndpointSlice
                │   └── kube-dns-nv9px.yaml
                ├── Lease
                │   ├── kube-controller-manager.yaml
                │   └── kube-scheduler.yaml
                ├── Pod
                │   ├── coredns-565d847f94-78r86.yaml
                │   ├── coredns-565d847f94-zdtcs.yaml
                │   ├── etcd-kind-control-plane.yaml
                │   ├── kindnet-qb7b8.yaml
                ├── ReplicaSet
                │   └── coredns-565d847f94.yaml
                ├── Role
                │   ├── extension-apiserver-authentication-reader.yaml
                │   ├── kubeadm:kubelet-config.yaml
                │   ├── kubeadm:nodes-kubeadm-config.yaml
                │   ├── kube-proxy.yaml
                ├── RoleBinding
                │   ├── kubeadm:kubelet-config.yaml
                │   ├── kubeadm:nodes-kubeadm-config.yaml
                │   ├── kube-proxy.yaml
                │   ├── system:controller:bootstrap-signer.yaml
                │   ├── system:controller:cloud-provider.yaml
                ├── Service
                │   └── kube-dns.yaml
                └── ServiceAccount
                    ├── attachdetach-controller.yaml
                    ├── bootstrap-signer.yaml
                    ├── certificate-controller.yaml
                    ├── cluster-resource-reader.yaml
                    ├── clusterrole-aggregation-controller.yaml
                    ├── coredns.yaml

17 directories, 80 files
```

Here, the resources has been grouped under the respective Kind folder.

Let's inspect the YAML of `coredns.yaml` file under ConfigMap folder,

```yaml
❯ cat /home/anisur/Downloads/kubestash/namespace/kube-system/gcs-repository-kube-system-backup-frequent-backup-1708691400/manifest/tmp/manifest/ConfigMap/coredns.yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system

```

Now, you can use these YAML files to re-create your desired application.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo backupconfiguration kube-system-backup
kubectl delete -n kubestash serviceaccount cluster-resource-reader
kubectl delete clusterrole cluster-resource-reader
kubectl delete clusterrolebinding cluster-resource-reader
kubectl delete retentionPolicy -n demo demo-retention
kubectl delete -n demo backupstorage gcs-storage
kubectl delete secret -n demo encryption-secret
kubectl delete secret -n demo gcs-secret
```
