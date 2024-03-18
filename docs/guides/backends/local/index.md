---
title: Local Backend | KubeStash
description: Configure KubeStash to Use Local Backend.
menu:
  docs_{{ .version }}:
    identifier: backend-local
    name: Kubernetes Volumes
    parent: backend
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Local Backend

### What is Local Backend

KubeStash supports any Kubernetes supported [volumes](https://kubernetes.io/docs/concepts/storage/volumes/) such as [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/volumes/#persistentvolumeclaim), [HostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath), [EmptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) (for testing only), [NFS](https://kubernetes.io/docs/concepts/storage/volumes/#nfs),  [gcePersistentDisk](https://kubernetes.io/docs/concepts/storage/volumes/#gcepersistentdisk) etc. as local backend.

> Unlike other backend options that allow the KubeStash operator to interact directly with storage, it cannot do so with the local backend because the backend volume is not mounted in the operator pod. Therefore, it needs to execute jobs to initialize the BackupStorage and Repository, as well as upload Snapshot metadata to the local backend.

### Create BackupStorage

Now, In this section, we are going to create `BackupStorage` object that uses Kubernetes volumes as a backend.

Following parameters are available for `Local` backend.

| Parameter            | Type       | Description                                                                                        |
| -------------------- | ---------- |----------------------------------------------------------------------------------------------------|
| `local.mountPath`    | `Required` | Path where this volume will be mounted inside the backup job container. Example: `/safe/data`.     |
| `local.subPath`      | `Optional` | Sub-path inside the referenced volume where the backed up data will be stored instead of its root. |
| `local.VolumeSource` | `Required` | Any Kubernetes volume. Can be specified inlined. Example: `hostPath`.                              |

> By default, KubeStash runs an initializer job with the user `65534` for the local backend. However, this user might lack write permissions to the backend. To address this, you can specify a different `fsGroup` or `runAsUser` in the `.spec.runtimeSettings.pod.securityContext` section of the BackupStorage. 

Here, we are going to show some sample `BackupStorage` objects that uses different Kubernetes volume as a backend.

### HostPath volume as Backend

Below, the YAML of a sample `BackupStorage` object that uses a `hostPath` volume as a backend.

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: BackupStorage
metadata:
  name: local-storage-with-hostpath
  namespace: demo
spec:
  storage:
    provider: local
    local:
      mountPath: /safe/data
      hostPath:
        path: /data/kubestash-test/storage
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true
  deletionPolicy: WipeOut
  runtimeSettings:
    pod:
      securityContext:
        runAsUser: 0
```

Create the `BackupStoage` we have shown above using the following command,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/backends/local/examples/hostPath.yaml
backupstorage.storage.kubestash.com/local-storage-with-hostpath created
```

> Since a `hostPath` volume is typically writable only by the root user, you'll need to either run the initializer job as the `root` user or modify permissions directly on the host filesystem to enable non-root write access.

### PersistentVolumeClaim as Backend

Below, the YAML of a sample `BackupStorage` crd that uses a `PersistentVolumeClaim` as a backend.

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: BackupStorage
metadata:
  name: local-storage-with-pvc
  namespace: demo
spec:
  storage:
    provider: local
    local:
      mountPath: /safe/data
      persistentVolumeClaim:
        claimName: storage-pvc
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true
  deletionPolicy: WipeOut
  runtimeSettings:
    pod:
      securityContext:
        fsGroup: 65534
```

Create the `BackupStorage` we have shown above using the following command,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/backends/local/examples/pvc.yaml
backupstorage.storage.kubestash.com/local-storage-with-pvc created
```

### NFS volume as Backend

Below, the YAML of a sample `BackupStorage` crd that uses an `NFS` volume as a backend.

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: BackupStorage
metadata:
  name: local-storage-with-nfs
  namespace: demo
spec:
  storage:
    provider: local
    local:
      mountPath: /safe/data
      nfs:
        server: "nfs-service.storage.svc.cluster.local" # use you own NFS server address
        path: "/" # this path is relative to "/exports" path of NFS server
  usagePolicy:
    allowedNamespaces:
      from: All
  default: false
  deletionPolicy: WipeOut
  runtimeSettings:
    pod:
      securityContext:
        fsGroup: 65534
```

Create the `BackupStorage` we have shown above using the following command,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/backends/local/examples/nfs.yaml
backupstorage.storage.kubestash.com/local-storage-with-nfs created
```

>For network volumes such as NFS, KubeStash needs to deploy a helper network volume accessor deployment in the same namespace as the BackupStorage. This deployment mounts the NFS volume, allowing the CLI to interact with the backend. You can configure the network volume accessor by following the instructions [here](/docs/setup/install/troubleshooting/index.md#configuring-network-volume-accessor).

## Next Steps

- Learn how to use KubeStash to backup workloads data from [here](/docs/guides/workloads/overview/index.md).
- Learn how to use KubeStash to backup stand-alone PVC from [here](/docs/guides/volumes/overview/index.md).
