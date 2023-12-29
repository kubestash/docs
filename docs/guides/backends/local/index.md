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

`Local` backend refers to a local path inside `kubestash` backup container. Any Kubernetes supported [persistent volume](https://kubernetes.io/docs/concepts/storage/volumes/) such as [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/volumes/#persistentvolumeclaim), [HostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath), [EmptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) (for testing only), [NFS](https://kubernetes.io/docs/concepts/storage/volumes/#nfs),  [gcePersistentDisk](https://kubernetes.io/docs/concepts/storage/volumes/#gcepersistentdisk) etc. can be used as local backend.

### Create BackupStorage

Now, In this section, we are going to create `BackupStorage` object that uses Kubernetes volumes as a backend.

Following parameters are available for `Local` backend.

| Parameter            | Type       | Description                                                                                        |
| -------------------- | ---------- |----------------------------------------------------------------------------------------------------|
| `local.mountPath`    | `Required` | Path where this volume will be mounted inside the backup job container. Example: `/safe/data`.     |
| `local.subPath`      | `Optional` | Sub-path inside the referenced volume where the backed up data will be stored instead of its root. |
| `local.VolumeSource` | `Required` | Any Kubernetes volume. Can be specified inlined. Example: `hostPath`.                              |


> Note that by default, for local backend KubeStash run an initializer job, which doesn’t have file write permission. So, in order to achieve
that you must give file system group permission, achieved by specifying `spec.securityContext.pod.fsGroup` in the BackupStorage configuration.

Here, we are going to show some sample `BackupStorage` objects that uses different Kubernetes volume as a backend.

##### HostPath volume as Backend

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

> Note that by default, Kubestash run backupStorage initializer job with a `non-root` user. `hostPath` volume is writable only for the `root` user.
So, in order to use `hostPath` volume as a backend, you must either run initializer job as the `root` user, achieved by specifying 
`spec.securityContext.pod.runAsUser` in the BackupStorage configuration, or adjust the permissions of the `hostPath` to allow write access for `non-root` users.

##### PersistentVolumeClaim as Backend

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
        fsGroup: 65535
```

Create the `BackupStorage` we have shown above using the following command,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/backends/local/examples/pvc.yaml
backupstorage.storage.kubestash.com/local-storage-with-pvc created
```

##### NFS volume as Backend

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
        fsGroup: 65535
```

Create the `BackupStorage` we have shown above using the following command,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/backends/local/examples/nfs.yaml
backupstorage.storage.kubestash.com/local-storage-with-nfs created
```

>For NFS backend, KubeStash may have to run the network volume accessor deployments in privileged mode to provide Snapshot listing facility. In this case, please configure network volume accessors by following the instruction [here](/docs/setup/install/troubleshooting/index.md#configuring-network-volume-accessor).

## Next Steps

- Learn how to use Stash to backup workloads data from [here](/docs/guides/workloads/overview/index.md).
- Learn how to use Stash to backup databases from [here](/docs/guides/addons/overview/index.md).
- Learn how to use Stash to backup stand-alone PVC from [here](/docs/guides/volumes/overview/index.md).
