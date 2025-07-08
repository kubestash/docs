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

# Backup resource YAMLs of entire cluster using KubeStash

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

To keep things isolated, we are going to use a separate namespace called `demo` throughout this tutorial. Create the `demo` namespace if you haven't created it already.

```bash
$ kubectl create ns demo
namespace/demo created
```

> Note: YAML files used in this tutorial are stored [here](https://github.com/kubestash/docs/tree/{{< param "info.version" >}}/docs/guides/kubedump/cluster/examples).

### Prepare for Backup

We are going to configure a backup for all the resource of our cluster.

#### Prepare Backend

Now, we are going backup of the YAMLs of entire cluster to a S3 bucket using KubeStash. For this, we have to create a `Secret` with  necessary credentials and a `BackupStorage` object. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

> For S3 backend, if the bucket does not exist, KubeStash needs `Storage Object Admin` role permissions to create the bucket. For more details, please check the following [guide](/docs/guides/backends/s3/index.md).

**Create Secret:**

Let's create a Secret named `s3-secret` with access credentials of our desired S3 backend,

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

Notice the `spec.usagePolicy` that allows referencing the `RetentionPolicy` from all namespaces.For more details on configuring it for specific namespaces, please refer to the following [link](/docs/concepts/crds/retentionpolicy/index.md).

Let's create the `RetentionPolicy` object that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/clusterResourcesBackup/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```


#### Create RBAC

To take backup of the resource YAMLs of entire cluster KubeStash creates a backup `Job`. This `Job` requires read permission for all the cluster resources. By default, KubeStash does not grant such cluster-wide permissions. We have to provide the necessary permissions manually.

Here, is the YAML of the `ServiceAccount`, `ClusterRole`, and `ClusterRoleBinding` that we are going to use for granting the necessary permissions.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-resource-reader-writter
  namespace: demo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-resource-reader-writter
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-resource-reader-writter
subjects:
- kind: ServiceAccount
  name: cluster-resource-reader-writter
  namespace: demo
roleRef:
  kind: ClusterRole
  name: cluster-resource-reader-writter
  apiGroup: rbac.authorization.k8s.io
```

Let's create the RBAC resources we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/kubedump/clusterResourcesBackup/examples/rbac.yaml
serviceaccount/cluster-resource-reader-writter created
clusterrole.rbac.authorization.k8s.io/cluster-resource-reader-writter created
clusterrolebinding.rbac.authorization.k8s.io/cluster-resource-reader-writter created
```

Now, we are ready for backup. In the next section, we are going to schedule a backup for our cluster resources.

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
          directory: /kubedump-manifests19
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
              IncludeNamespaces: "demo"
              #IncludeResources: "secrets,configmaps,deployments"
              ANDedLabelSelectors: "app:my-app"
              #ORedLabelSelectors: "app1:my-app1,app2:my-app2"
        jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader-writter
```

Here,
- `spec.sessions[*].addon.name` specifies the name of the `Addon`.
- `spec.sessions[*].addon.tasks[*].name` specifies the name of the backup task.
- `spec.sessions[*].addon.jobTemplate.spec.serviceAccountName`specifies the ServiceAccount name that we have created earlier with cluster-wide resource reading permission.

Let's create the `BackupConfiguration` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubedump/cluster/examples/backupconfiguration.yaml
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
NAME             INTEGRITY   SNAPSHOT-COUNT   SIZE   PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repository                                       Ready                            28s
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the Repository YAML stored in the `demo/cluster-manifests` directory.

**Verify CronJob:**

Verify that KubeStash has created a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` object.

Check that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                                        SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-cluster-resources-backup-frequent-backup            */5 * * * *   False     0        <none>          45s
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. You can watch for `BackupSession` CR using the following command,

```bash
$ watch -n 1 kubectl get backupsession -n demo -l=kubestash.com/invoker-name=cluster-resources-backup                                        anisur: Fri Feb 23 19:46:11 2024

NAME                                                  INVOKER-TYPE          INVOKER-NAME               PHASE       DURATION   AGE
cluster-resources-backup-frequent-backup-1708694700   BackupConfiguration   cluster-resources-backup   Succeeded              21m
```

**Verify Backup:**

When `BackupSession` is created, KubeStash operator creates `Snapshot` for each `Repository` listed in the respective session of the `BackupConfiguration`. Since we have only specified one repository in the session, at this moment we should have one `Snapshot`. 

Run the following command to check the respective `Snapshot`,

```bash
$ kubectl get snapshots -n demo
NAME                                                              REPOSITORY       SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-repository-cluster-resourcesckup-frequent-backup-1708694700   gcs-repository   frequent-backup   2024-02-23T13:25:00Z   Delete            Succeeded   22m
```

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/cluster-manifests/repository/v1/frequent-backup/manifest` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the` demo/cluster-manifests/repository/snapshots` directory.

<figure align="center">
  <img alt="Backup data in GCS Bucket" src="/docs/guides/kubedump/cluster/images/cluster_manifests_backup.png">
  <figcaption align="center">Fig: Backup data in GCS Bucket</figcaption>
</figure>

> Note: KubeStash stores all dumped data encrypted in backup directory, meaning it remains unreadable until decrypted.

## Restore

KubeStash does not provide any automatic mechanism to restore the cluster resources from the backed-up YAMLs. Your application might be managed by Helm or by an operator. In such cases, just applying the YAMLs is not enough to restore the application. Furthermore, there might be an order issue. Some resources must be applied before others. It is difficult to generalize and codify various application-specific logic.

Therefore, it is the user's responsibility to download the backed-up YAMLs and take the necessary steps based on his application to restore it properly.

### Download the YAMLs

KubeStash provides a [kubectl plugin](/docs/guides/cli/kubectl-plugin/index.md#download-snapshot) for making it easy to download a snapshot locally.

Now, let's download the latest Snapshot from our backed-up data into the `$HOME/Downloads/kubestash` folder of our local machine.

```bash
$  kubectl kubestash download --namespace=demo gcs-repository-cluster-resourcesckup-frequent-backup-1708694700  --destination=$HOME/Downloads/kubestash/cluster
```

Now, lets use [tree](https://linux.die.net/man/1/tree) command to inspect downloaded YAMLs files.

```bash
$ /home/anisur/Downloads/kubestash/cluster/gcs-repository-cluster-resourcesckup-frequent-backup-1708694700
└── manifest
    └── tmp
        └── manifest
            ├── global
            │   ├── Addon
            │   │   ├── kubedump-addon.yaml
            │   │   ├── pvc-addon.yaml
            │   │   └── workload-addon.yaml
            │   ├── APIService
            │   │   ├── v1.admissionregistration.k8s.io.yaml
            └── namespaces
                ├── default
                │   ├── ConfigMap
                │   │   └── kube-root-ca.crt.yaml
                │   ├── Endpoints
                │   │   └── kubernetes.yaml
                │   ├── EndpointSlice
                │   │   └── kubernetes.yaml
                │   ├── Service
                │   │   └── kubernetes.yaml
                │   └── ServiceAccount
                │       └── default.yaml
                ├── demo
                │   ├── BackupConfiguration
                │   │   └── cluster-resources-backup.yaml
                │   ├── BackupSession
                │   │   └── cluster-resources-backup-frequent-backup-1708694700.yaml
                │   ├── BackupStorage
                │   │   └── gcs-storage.yaml
                │   ├── ConfigMap
                │   │   └── kube-root-ca.crt.yaml
                │   ├── ReplicaSet
                │   │   └── coredns-565d847f94.yaml
                │   ├── Role
                │   │   ├── extension-apiserver-authentication-reader.yaml
                │   │   ├── kubeadm:kubelet-config.yaml
                │   ├── RoleBinding
                │   │   ├── kubeadm:kubelet-config.yaml
                │   │   ├── kubeadm:nodes-kubeadm-config.yaml
                │   ├── Service
                │   │   └── kube-dns.yaml
                │   └── ServiceAccount
                │       ├── attachdetach-controller.yaml
                │       ├── bootstrap-signer.yaml
                │       ├── certificate-controller.yaml
                │       ├── cluster-resource-reader.yaml
                └── local-path-storage
                    ├── ConfigMap
                    │   ├── kube-root-ca.crt.yaml
                    │   └── local-path-config.yaml
                    ├── Deployment
                    │   └── local-path-provisioner.yaml

84 directories, 405 files
```

Here, the non-namespaced resources have been grouped under the `global` directory and the namespaced resources have been grouped inside the namespace specific folder under the `namespaces` directory.

Let's inspect the YAML of `kubeadm-config.yaml` file under `kube-system` namespace.

```yaml
$ cat /home/anisur/Downloads/kubestash/cluster/gcs-repository-cluster-resourcesckup-frequent-backup-1708694700/manifest/tmp/manifest/namespaces/kube-system/ConfigMap/kubeadm-config.yaml
apiVersion: v1
data:
  ClusterConfiguration: |
    apiServer:
      certSANs:
      - localhost
      - 127.0.0.1
      extraArgs:
        authorization-mode: Node,RBAC
        runtime-config: ""
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta3
    certificatesDir: /etc/kubernetes/pki
    clusterName: kind
    controlPlaneEndpoint: kind-control-plane:6443
    controllerManager:
      extraArgs:
        enable-hostpath-provisioner: "true"
    dns: {}
    etcd:
      local:
        dataDir: /var/lib/etcd
    imageRepository: registry.k8s.io
    kind: ClusterConfiguration
    kubernetesVersion: v1.25.2
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.244.0.0/16
      serviceSubnet: 10.96.0.0/16
    scheduler: {}
kind: ConfigMap
metadata:
  name: kubeadm-config
  namespace: kube-system
```

Now, you can use these YAML files to re-create your desired application.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo backupconfiguration cluster-resources-backup
kubectl delete -n kubestash serviceaccount cluster-resource-reader
kubectl delete clusterrole cluster-resource-reader
kubectl delete clusterrolebinding cluster-resource-reader
kubectl delete retentionPolicy -n demo demo-retention
kubectl delete -n demo backupstorage gcs-storage
kubectl delete secret -n demo encrypt-secret
kubectl delete secret -n demo gcs-secret
```
