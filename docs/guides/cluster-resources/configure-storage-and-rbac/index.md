---
title: Configure Storage Backend and RBAC | KubeStash
description: 
menu:
  docs_{{ .version }}:
    identifier: kubestash-cluster-resources-configure-storage
    name: Configure Storage Backend and RBAC
    parent: kubestash-cluster-resources
    weight: 40
product_name: KubeStash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

## Prepare Backend

Now, we are going backup of the YAMLs of entire cluster to a S3 bucket using KubeStash. For this, we have to create a `Secret` with  necessary credentials and a `BackupStorage` object. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

> For S3 backend, if the bucket does not exist, KubeStash needs `Storage Object Admin` role permissions to create the bucket. For more details, please check the following [guide](/docs/guides/backends/s3/index.md).

---

### Create Secret 

Let's create a Secret named `aws-s3-secret` with access credentials of our desired S3 backend,

```bash
$ kubectl create secret generic aws-s3-secret \
    --from-literal=AWS_ACCESS_KEY_ID=<your-aws-access-key-id> \
    --from-literal=AWS_SECRET_ACCESS_KEY=<your-aws-secret-access-key>
secret/aws-s3-secret created
```

---

### Create BackupStorage 

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/configure-storage-and-rbac/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/s3-storage created
```

---

### Create `RBAC` for `BackupConfiguration`

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/cluster-resources/configure-storage-and-rbac/examples/rbac.yaml
serviceaccount/cluster-resource-reader-writer created
clusterrole.rbac.authorization.k8s.io/cluster-resource-reader-writer created
clusterrolebinding.rbac.authorization.k8s.io/cluster-resource-reader-writer created
```

---

### Create Encryption Secret

We also have to create another `Secret` with an encryption key `RESTIC_PASSWORD` for `Restic`. This secret will be used by `Restic` for encrypting the backup data.

Let's create a secret named `encry-secret` with the Restic password.

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD 
secret/encrypt-secret created
```