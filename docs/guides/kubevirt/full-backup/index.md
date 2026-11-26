---
title: Backup & Restore a KubeVirt VirtualMachine | KubeStash
description: A step by step guide on how to take full backups of a KubeVirt VirtualMachine and restore it using KubeStash.
menu:
  docs_{{ .version }}:
    identifier: kubevirt-full-backup
    name: Full Backup & Restore
    parent: kubevirt
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup & Restore a KubeVirt VirtualMachine using KubeStash

This guide will show you how to use KubeStash to take full backups of a [KubeVirt](https://kubevirt.io/) `VirtualMachine` (VM) and how to restore it.

## Before You Begin

- You need a Kubernetes cluster and a `kubectl` configured to talk to it.

- Install KubeStash in your cluster following the steps [here](/docs/setup/README.md).

- Install [KubeVirt](https://kubevirt.io/user-guide/cluster_admin/installation/) and [CDI](https://github.com/kubevirt/containerized-data-importer). KubeVirt needs the `Snapshot` feature gate to back up a running VM. Add it to the `KubeVirt` resource with `kubectl -n kubevirt edit kubevirt kubevirt`:

  ```yaml
  spec:
    configuration:
      developerConfiguration:
        featureGates:
        - Snapshot
  ```

  Then check it:

  ```bash
  $ kubectl -n kubevirt get kubevirt kubevirt -o jsonpath='{.spec.configuration.developerConfiguration.featureGates}{"\n"}'
  ["Snapshot"]
  ```

  Your list may contain other feature gates too; only `Snapshot` is needed for this guide.

  If your cluster has no hardware virtualization (for example, a test cluster in a VM), set `spec.configuration.developerConfiguration.useEmulation: true` in the `KubeVirt` resource.

- Ensure a `VolumeSnapshotClass` exists. Backing up a running VM needs a CSI driver that supports volume snapshots, with a `VolumeSnapshotClass` for it.

  ```bash
  $ kubectl get volumesnapshotclasses
  NAME                    DRIVER               DELETIONPOLICY   AGE
  longhorn-snapshot-vsc   driver.longhorn.io   Delete           7d22h
  ```

  If there is none, you can use [Longhorn](https://longhorn.io/docs/latest/deploy/install/) or any other CSI driver with snapshot support. To install Longhorn:

  ```bash
  $ helm repo add longhorn https://charts.longhorn.io
  $ helm repo update
  $ helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
  ...
  $ kubectl get pod -n longhorn-system
  ...
  ```

  Then create a `VolumeSnapshotClass` for it. Below is the YAML we are going to use:

  ```yaml
  apiVersion: snapshot.storage.k8s.io/v1
  kind: VolumeSnapshotClass
  metadata:
    name: longhorn-snapshot-vsc
    annotations:
      snapshot.storage.kubernetes.io/is-default-class: "true"
  driver: driver.longhorn.io
  deletionPolicy: Delete
  parameters:
    type: snap
  ```

  ```bash
  $ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/full-backup/examples/volumesnapshotclass.yaml
  volumesnapshotclass.snapshot.storage.k8s.io/longhorn-snapshot-vsc created
  ```

- You should be familiar with the following KubeStash concepts:
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)

To keep everything isolated, we are going to use a namespace called `demo` for the VM, its backup and the backend.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/kubevirt/full-backup/examples](/docs/guides/kubevirt/full-backup/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Prepare the VirtualMachine

We are going to back up a small [CirrOS](https://github.com/cirros-dev/cirros) VM with one 5Gi disk on the `longhorn` StorageClass. Below is the YAML of the VM:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vmtest
  namespace: demo
spec:
  runStrategy: Always
  dataVolumeTemplates:
  - metadata:
      name: vmtest-root
    spec:
      pvc:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 5Gi
        storageClassName: longhorn
      source:
        registry:
          url: "docker://quay.io/kubevirt/cirros-container-disk-demo:latest"
  template:
    spec:
      domain:
        cpu:
          cores: 1
        memory:
          guest: 512Mi
        machine:
          type: q35
        devices:
          disks:
          - name: rootdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
      networks:
      - name: default
        pod: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: rootdisk
        dataVolume:
          name: vmtest-root
```

The disk is created through `dataVolumeTemplates`, so it belongs to the VM and is deleted together with it.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/full-backup/examples/vm.yaml
virtualmachine.kubevirt.io/vmtest created

$ kubectl -n demo wait vm vmtest --for=condition=Ready --timeout=10m
virtualmachine.kubevirt.io/vmtest condition met

$ kubectl -n demo get vm,pvc
NAME                              AGE    STATUS    READY
virtualmachine.kubevirt.io/vmtest 3m5s   Running   True

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
persistentvolumeclaim/vmtest-root   Bound    pvc-79803ad3-3cd7-4b3e-91c8-16123ba1c75e   5Gi        RWO            longhorn
```

**Write some data into the VM:**

Now, let's write a file inside the VM so we can check it after the restore.

```bash
$ virtctl -n demo console vmtest
...
cirros login: cirros
Password: gocubsgo
$ echo "hello from the full-backup test" > /home/cirros/note.txt
$ sync
$ cat /home/cirros/note.txt
hello from the full-backup test
```

Press `Ctrl+]` to leave the console. The `sync` makes sure the file is on the disk before the backup runs.

## Prepare Backend

We are going to store the backed up data in an S3 bucket. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

**Create Secret:**

```bash
$ kubectl create secret generic -n demo s3-secret \
    --from-literal=AWS_ACCESS_KEY_ID='<your-access-key-id>' \
    --from-literal=AWS_SECRET_ACCESS_KEY='<your-secret-access-key>'
secret/s3-secret created
```

**Create BackupStorage:**

Below is the YAML of the `BackupStorage` CR we are going to create. The `usagePolicy` allows every namespace to use it:

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
      endpoint: https://s3.us-east-2.amazonaws.com
      region: us-east-2
      prefix: demo
      secretName: s3-secret
  usagePolicy:
    allowedNamespaces:
      from: All
  deletionPolicy: WipeOut
```

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/full-backup/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/s3-storage created
```

**Create RetentionPolicy:**

Now, let's create a `RetentionPolicy` to specify how the old Snapshots should be cleaned up. Below is the YAML of the `RetentionPolicy` we are going to create:

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: RetentionPolicy
metadata:
  name: demo-retention
  namespace: demo
spec:
  successfulSnapshots:
    last: 3
  failedSnapshots:
    last: 2
  usagePolicy:
    allowedNamespaces:
      from: All
```

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/full-backup/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

**Create the encryption Secret:**

KubeStash encrypts the repository with the password in this Secret. Keep it safe; you need the same password to restore.

```bash
$ kubectl -n demo create secret generic encryption-secret --from-literal=RESTIC_PASSWORD='<your-password>'
secret/encryption-secret created
```

## Backup

Below is the YAML of the `BackupConfiguration` we are going to create:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: vmtest-backup
  namespace: demo
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vmtest
    namespace: demo
  backends:
  - name: s3-backend
    storageRef:
      name: s3-storage
      namespace: demo
    retentionPolicy:
      name: demo-retention
      namespace: demo
  sessions:
  - name: daily
    scheduler:
      schedule: "0 2 * * *"
      jobTemplate:
        backoffLimit: 1
    repositories:
    - name: vmtest-repo
      backend: s3-backend
      directory: /vmtest
      encryptionSecret:
        name: encryption-secret
        namespace: demo
    addon:
      name: kubevirt
      tasks:
      - name: logical-backup
```

Here,

- `spec.target` is the VM to back up.
- `spec.sessions[*].scheduler.schedule` is a cron expression. Here, the backup runs every day at 02:00.
- `spec.sessions[*].addon.tasks` uses the `logical-backup` task, which backs up the manifest and every disk of the VM in one task.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/full-backup/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/vmtest-backup created
```

**Verify the BackupConfiguration:**

```bash
$ kubectl -n demo get backupconfiguration vmtest-backup -o jsonpath='{.status.phase}{"\n"}'
Ready

$ kubectl -n demo get cronjob
NAME                         SCHEDULE       TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-vmtest-backup-daily  0 2 * * *     <none>     False     0        <none>          5s
```

KubeStash creates a CronJob that triggers a `BackupSession` on each schedule. It also triggers **one backup immediately** when the `BackupConfiguration` becomes ready, so you do not have to wait for the first scheduled time.

**Watch the backup:**

```bash
$ watch kubectl get backupsessions.core.kubestash.com,snapshots.storage.kubestash.com -n demo
```

The first `BackupSession`, taken while the VM was running, finished like this:

```bash
$ kubectl get backupsessions.core.kubestash.com -n demo
NAME                                  INVOKER-TYPE          INVOKER-NAME    PHASE       DURATION   AGE
vmtest-backup-daily-1795083228        BackupConfiguration   vmtest-backup   Succeeded   1m42s      10m

$ kubectl get snapshots.storage.kubestash.com -n demo
NAME                                           REPOSITORY    SESSION   SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
vmtest-repo-vmtest-backup-daily-1795083228     vmtest-repo   daily     2026-11-19T10:13:58Z   Delete            Succeeded   10m
```

The `Snapshot` lists the two parts that were stored:

```bash
$ kubectl -n demo get snapshot vmtest-repo-vmtest-backup-daily-1795083228 -o jsonpath='{.status.components}'
```

| Component | What was read | Uploaded | Duration |
|---|---|---|---|
| `manifest` | 3.8 KiB | 6.7 KiB | 5.5s |
| `volume-vmtest-root` | 4.55 GiB | 38 MiB | 17.9s |

The disk is 4.55 GiB read, but only 38 MiB is stored, because most of a fresh CirrOS disk is empty.

## Restore

A restore recreates the VM (manifest and disks) from a `Snapshot` of the repository. With `snapshot: latest` it uses the newest one. You can also give a `Snapshot` name from `kubectl get snapshots -n demo`. Note that `latest` moves on: it always means the newest snapshot at the time you create the `RestoreSession`.

The `RestoreSession` can use one of two sets of tasks:

| Tasks | What it does |
|---|---|
| `volume-restore` and `manifest-restore` | Restores into a **new name**. The original VM is not touched. |
| `logical-restore` | Restores under the **original name**. The original VM must be deleted first. |

Do not set `spec.target` on the `RestoreSession`: the VM does not exist yet when the restore starts.

A restored VM comes back in the run state it had when it was backed up. If it was stopped, start it with `virtctl start`.

### Restore into a New VM

Below is the YAML of the `RestoreSession`. The new VM is named `vmtest-restored`; the original `vmtest` is left alone.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: vmtest-restore-new
  namespace: demo
spec:
  dataSource:
    repository: vmtest-repo
    snapshot: latest
    encryptionSecret:
      name: encryption-secret
      namespace: demo
  addon:
    name: kubevirt
    tasks:
    - name: volume-restore
      params:
        targetName: vmtest-restored
        targetNamespace: demo
    - name: manifest-restore
      params:
        targetName: vmtest-restored
        targetNamespace: demo
```

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/full-backup/examples/restoresession-new-vm.yaml
restoresession.core.kubestash.com/vmtest-restore-new created

$ kubectl get restoresessions.core.kubestash.com -n demo
NAME                 REPOSITORY    PHASE       DURATION   AGE
vmtest-restore-new   vmtest-repo   Succeeded   1m14s      91s
```

The new VM has its own disk, named after it. The disk reference in the VM was renamed too:

```bash
$ kubectl -n demo get vm,pvc
NAME                                      AGE   STATUS    READY
virtualmachine.kubevirt.io/vmtest         20m   Running   True
virtualmachine.kubevirt.io/vmtest-restored 25s  Running   True

NAME                                         STATUS   VOLUME                                     CAPACITY   STORAGECLASS
persistentvolumeclaim/vmtest-root            Bound    pvc-79803ad3-3cd7-4b3e-91c8-16123ba1c75e   5Gi        longhorn
persistentvolumeclaim/vmtest-restored-root   Bound    pvc-246936f3-3ad3-4938-b4a2-7d6a3c53d299   5Gi        longhorn
```

Log in to the restored VM and check the file:

```bash
$ virtctl -n demo console vmtest-restored
cirros login: cirros
Password: gocubsgo
$ cat /home/cirros/note.txt
hello from the full-backup test
```

### Restore Over the Original VM

To restore under the original name, delete the VM first. Its disk belongs to the VM through `dataVolumeTemplates`, so the PVC and DataVolume are deleted with it, and nothing is left to get in the way:

```bash
$ kubectl -n demo delete vm vmtest
$ kubectl -n demo get pvc,dv
NAME                                         STATUS   VOLUME      CAPACITY   STORAGECLASS
persistentvolumeclaim/vmtest-restored-root   Bound    pvc-2469... 5Gi        longhorn
...
```

Below is the YAML of the `RestoreSession`, which uses the single `logical-restore` task:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: vmtest-restore-inplace
  namespace: demo
spec:
  dataSource:
    repository: vmtest-repo
    snapshot: latest
    encryptionSecret:
      name: encryption-secret
      namespace: demo
  addon:
    name: kubevirt
    tasks:
    - name: logical-restore
```

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/full-backup/examples/restoresession-in-place.yaml
restoresession.core.kubestash.com/vmtest-restore-inplace created

$ kubectl get restoresessions.core.kubestash.com -n demo
NAME                     REPOSITORY    PHASE       DURATION   AGE
vmtest-restore-inplace   vmtest-repo   Succeeded   1m12s      106s

$ kubectl -n demo get vm,pvc,dv
NAME                                         AGE     STATUS     READY
virtualmachine.kubevirt.io/vmtest            2s      Starting   False
virtualmachine.kubevirt.io/vmtest-restored   6m28s   Running    True

NAME                                         STATUS   VOLUME                                     CAPACITY   STORAGECLASS
persistentvolumeclaim/vmtest-restored-root   Bound    pvc-aa02b27a-09f8-4e7c-b024-2b54fed052d1   5Gi        longhorn
persistentvolumeclaim/vmtest-root            Bound    pvc-22d6d1b9-974c-4556-b079-f7986568e424   5Gi        longhorn

NAME                                              PHASE       PROGRESS   RESTARTS   AGE
datavolume.cdi.kubevirt.io/vmtest-restored-root   Succeeded   N/A                   6m28s
datavolume.cdi.kubevirt.io/vmtest-root            Succeeded   N/A                   2s
```

The VM is back under its original name with a new disk. It was running when it was backed up, so it starts on its own. The `BackupConfiguration` stays `Ready` (`targetFound: true`), so scheduled backups continue for the restored VM without any change.

Log in and check the file:

```bash
$ virtctl -n demo console vmtest
cirros login: cirros
Password: gocubsgo
$ cat /home/cirros/note.txt
hello from the full-backup test
```

## Cleaning Up

```bash
kubectl delete -n demo backupconfiguration vmtest-backup
kubectl delete -n demo restoresession --all
kubectl delete -n demo vm --all
kubectl delete -n demo backupstorage s3-storage
kubectl delete -n demo retentionpolicy demo-retention
kubectl delete ns demo
```

> The `BackupStorage` in this guide has `deletionPolicy: WipeOut`. Deleting it removes the backed up data from the bucket, under the `prefix` you set.
