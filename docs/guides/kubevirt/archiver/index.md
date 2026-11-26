---
title: Continuous Backup of KubeVirt VirtualMachines | KubeStash
description: A step by step guide on how to continuously back up KubeVirt VirtualMachines with KubeVirtArchiver and restore them to a chosen point in time.
menu:
  docs_{{ .version }}:
    identifier: kubevirt-archiver
    name: KubeVirt Archiver
    parent: kubevirt
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Continuous Backup of KubeVirt VirtualMachines using KubeStash

This guide will show you how to use a KubeStash `KubeVirtArchiver` to back up KubeVirt `VirtualMachines` (VMs) continuously with changed block tracking (CBT), and how to restore a VM to a chosen checkpoint or point in time.

## Before You Begin

- You need a Kubernetes cluster and a `kubectl` configured to talk to it.

- Install KubeStash in your cluster following the steps [here](/docs/setup/README.md).

- Install [KubeVirt](https://kubevirt.io/user-guide/cluster_admin/installation/) and [CDI](https://github.com/kubevirt/containerized-data-importer). KubeVirt needs:
  - the `IncrementalBackup` feature gate, for changed block tracking and `VirtualMachineBackup`,
  - the `UtilityVolumes` feature gate, because the scratch volume of a backup is attached to the VM as a utility volume,
  - a label selector that tells it which VMs get changed block tracking. Here, VMs with the label `cbt: enabled` are selected.

  Add them to the `KubeVirt` resource with `kubectl -n kubevirt edit kubevirt kubevirt`:

  ```yaml
  spec:
    configuration:
      developerConfiguration:
        featureGates:
        - IncrementalBackup
        - UtilityVolumes
      changedBlockTrackingLabelSelectors:
        virtualMachineLabelSelector:
          matchLabels:
            cbt: enabled
  ```

  Then check it:

  ```bash
  $ kubectl -n kubevirt get kubevirt kubevirt -o jsonpath='{.spec.configuration.developerConfiguration.featureGates}{"\n"}{.spec.configuration.changedBlockTrackingLabelSelectors}{"\n"}'
  ["IncrementalBackup","UtilityVolumes"]
  {"virtualMachineLabelSelector":{"matchLabels":{"cbt":"enabled"}}}
  ```

  Your list may contain other feature gates too; only these two are needed for this guide.

  If your cluster has no hardware virtualization (for example, a test cluster in a VM), set `spec.configuration.developerConfiguration.useEmulation: true` in the `KubeVirt` resource.

- You should be familiar with the following KubeStash concepts:
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)
  - [Snapshot](/docs/concepts/crds/snapshot/index.md)
  - [Repository](/docs/concepts/crds/repository/index.md)

To keep everything isolated, we are going to use a namespace called `demo` for the VM, its backup and the backend.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/kubevirt/archiver/examples](/docs/guides/kubevirt/archiver/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Prepare the VirtualMachine

We are going to back up a small [CirrOS](https://github.com/cirros-dev/cirros) VM with one 5Gi disk. Below is the YAML of the VM:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vmtest
  namespace: demo
  labels:
    cbt: enabled
    archive: enabled
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
        storageClassName: local-path
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

Here,

- `cbt: enabled` matches the KubeVirt selector above, so KubeVirt tracks changed blocks of this VM.
- `archive: enabled` is the label our archiver will select.

> Changed block tracking is switched on when the VM starts. If you add the `cbt` label to a VM that is already running, it has to be restarted before it can be backed up.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/archiver/examples/vm.yaml
virtualmachine.kubevirt.io/vmtest created

$ kubectl -n demo wait vm vmtest --for=condition=Ready --timeout=10m
virtualmachine.kubevirt.io/vmtest condition met
```

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

Below is the YAML of the `BackupStorage` CR we are going to create:

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/archiver/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/s3-storage created
```

**Create RetentionPolicy:**

Below is the YAML of the `RetentionPolicy` we are going to create:

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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/archiver/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

**Create the encryption Secret:**

KubeStash encrypts the repository with the password in this Secret. Keep it safe; you need the same password to restore.

```bash
$ kubectl -n demo create secret generic encryption-secret --from-literal=RESTIC_PASSWORD='<your-password>'
secret/encryption-secret created
```

## Backup

Below is the YAML of the `KubeVirtArchiver` we are going to create:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: KubeVirtArchiver
metadata:
  name: vmtest-archiver
  namespace: demo
spec:
  virtualMachines:
    namespaces:
      from: Same
    selector:
      matchLabels:
        archive: enabled
  backupStorage:
    ref:
      name: s3-storage
      namespace: demo
  encryptionSecret:
    name: encryption-secret
    namespace: demo
  retentionPolicy:
    name: demo-retention
    namespace: demo
  fullBackup:
    driver: Restic
    scheduler:
      schedule: "0 2 * * *"
      jobTemplate: {}
  cbtBackup:
    interval: 3m
    scratchStorageClass: local-path
    scratchSize: 2Gi
    skipQuiesce: true
```

Here,

- `spec.virtualMachines` selects the VMs to archive: the VMs in the same namespace with the label `archive: enabled`. Every matching VM, now or later, is archived.
- `spec.fullBackup.scheduler.schedule` is when a full backup is taken, here every day at 02:00. A full backup is the starting point of a chain of incremental checkpoints.
- `spec.cbtBackup.interval` is how often an incremental checkpoint is taken between full backups. Each one stores only the blocks that changed since the previous checkpoint.
- `spec.cbtBackup.scratchStorageClass` and `scratchSize` define the scratch volume that KubeVirt needs while it exports a checkpoint.
- `spec.cbtBackup.skipQuiesce: true` skips freezing the guest filesystem. CirrOS has no QEMU guest agent, so the checkpoints are crash-consistent. With a guest agent, leave it `false` to get application-consistent checkpoints.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/archiver/examples/kubevirtarchiver.yaml
kubevirtarchiver.core.kubestash.com/vmtest-archiver created
```

**Verify the archiver:**

```bash
$ kubectl -n demo get kubevirtarchiver vmtest-archiver -o jsonpath='{.status.virtualMachines}{"\n"}'
[{"changedBlockTrackingReady":true,"changedBlockTrackingState":"Enabled","lastObservedTime":"2026-11-22T04:42:29Z","name":"vmtest","namespace":"demo"}]
```

`changedBlockTrackingReady: true` means the VM can be backed up. If it is `false`, `changedBlockTrackingState` and `reason` tell you why, for example `PendingRestart`.

For each matched VM, the archiver creates a `BackupConfiguration` and a `Repository`, and starts a full backup right away. It also starts a long-running Job that takes an incremental checkpoint every `interval`.

```bash
$ kubectl -n demo get kubevirtarchiver,backupconfiguration,backupsessions.core.kubestash.com,repository,snapshots.storage.kubestash.com
NAME                                                  PAUSED   VMS   LAST-INCREMENTAL       INCREMENTAL-FAILURES   AGE
kubevirtarchiver.core.kubestash.com/vmtest-archiver            1     2026-11-22T04:48:21Z                          5m56s

NAME                                                                                          PHASE   PAUSED   AGE
backupconfiguration.core.kubestash.com/vmtest-d1fa4628-605e-4543-921c-5c59921ed036-archiver   Ready            5m56s

NAME                                                                                               INVOKER-TYPE          INVOKER-NAME                                           PHASE       DURATION   AGE
backupsession.core.kubestash.com/vmtest-d1fa4628-605e-4543-921c-5archiver-full-backup-1795322549   BackupConfiguration   vmtest-d1fa4628-605e-4543-921c-5c59921ed036-archiver   Succeeded   1m52s      5m46s

NAME                                                                                INTEGRITY   SNAPSHOT-COUNT   SIZE         PHASE   LAST-SUCCESSFUL-BACKUP   AGE
repository.storage.kubestash.com/vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full   true        1                16.685 MiB   Ready   5m46s                    5m56s

NAME                                                                                             REPOSITORY                                         SESSION       SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
snapshot.storage.kubestash.com/vmtest-d1fa4628-605e-4543-921c-5archiver-full-backup-1795322549   vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full   full-backup   2026-11-22T04:42:39Z   Delete            Succeeded   5m46s
snapshot.storage.kubestash.com/vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt              vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full   full-backup   2026-11-22T04:44:28Z   Delete            Running     3m57s
```

Here,

- The Snapshot ending in `-full-backup-<number>` is the full backup. It is `Succeeded`.
- The Snapshot ending in `-full-cbt` holds the incremental checkpoints of the chain that starts at that full backup. It stays `Running` while the archiver is running, and a new checkpoint is added to it every `interval`. `LAST-INCREMENTAL` on the archiver shows when the last one was taken.

The `RetentionPolicy` applies to the full Snapshots. When an old full backup is removed, the checkpoints of its chain are removed with it.

## Write Some Data

Now, let's write a file inside the VM, so we can check it after the restore.

```bash
$ virtctl -n demo console vmtest
...
cirros login: cirros
Password: gocubsgo
$ echo "version 1" > /home/cirros/note.txt
$ sync
$ cat /home/cirros/note.txt
version 1
```

Press `Ctrl+]` to leave the console. Wait for the next checkpoint (up to `interval` plus about a minute) so that it contains this file. Then change the data:

```bash
$ virtctl -n demo console vmtest
$ echo "version 2" > /home/cirros/note.txt
$ echo "second file" > /home/cirros/note2.txt
$ sync
```

After another checkpoint, the archiver holds both states.

## Find the Recovery Points

A recovery point is a checkpoint that is stored completely, both its disk and its manifest. The `-full-cbt` Snapshot lists them. Each line below is one recovery point: the time its manifest finished and its checkpoint name.

```bash
$ kubectl -n demo get snapshots.storage.kubestash.com vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt \
    -o jsonpath='{range .status.components.manifest.resticCBTStats[*]}{.capturedAt}{"  "}{.checkpoint}{"\n"}{end}'
...
2026-11-22T04:52:11Z  kubestash-vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt-2026-11-22_04-51-31
2026-11-22T04:56:03Z  kubestash-vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt-2026-11-22_04-55-22
2026-11-22T04:59:55Z  kubestash-vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt-2026-11-22_04-59-13
2026-11-22T05:03:50Z  kubestash-vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt-2026-11-22_05-03-06
```

The checkpoint name ends with the time the checkpoint **started**. The time in front is when the recovery point was **complete**, and it is the time that a `targetTime` restore is compared with. In the example, we wrote `version 1` at 04:48 and `version 2` at 04:57. So the checkpoint started at 04:51:31 has `version 1`, and the one started at 04:59:13 has `version 2`.

## Restore

A `KubeVirtArchiverRestore` recreates the VM from a recovery point. It creates a **new** VM named `spec.target.name` in its own namespace and fails rather than overwrite an existing one, so the source VM is not touched.

`spec.source.repositoryRef.name` is the name of the `Repository` that holds the VM's backups:

```bash
$ kubectl -n demo get repository
NAME                                                  INTEGRITY   SNAPSHOT-COUNT   SIZE         PHASE   LAST-SUCCESSFUL-BACKUP   AGE
vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full      true        1                16.685 MiB   Ready   5m46s                    5m56s
```

The restore takes the recovery point from `spec.recoveryPoint`, which is either an exact `checkpoint` or a `targetTime`. If you leave `recoveryPoint` out, it takes the newest recovery point.

**Pause the archiver first.** A restored VM keeps the labels of the source VM. If the archiver is running, it selects the restored VM as soon as it is created and starts backing it up too, with its own `BackupConfiguration`, `Repository` and full backup. Pause the archiver before you restore:

```bash
$ kubectl -n demo patch kubevirtarchiver vmtest-archiver --type=merge -p '{"spec":{"pause":true}}'
kubevirtarchiver.core.kubestash.com/vmtest-archiver patched
```

While the archiver is paused, no full backups or checkpoints are taken for any VM. Resume it after the restore, as shown [below](#resume-the-archiver).

### Restore a Checkpoint

Below is the YAML of the restore for the checkpoint that has `version 1`. Use the repository name and the checkpoint name from your own cluster:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: KubeVirtArchiverRestore
metadata:
  name: vmtest-restore-checkpoint
  namespace: demo
spec:
  source:
    repositoryRef:
      name: vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full
  target:
    name: vmtest-restored
  recoveryPoint:
    checkpoint: kubestash-vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt-2026-11-22_04-51-31
```

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/archiver/examples/restore-checkpoint.yaml
kubevirtarchiverrestore.core.kubestash.com/vmtest-restore-checkpoint created

$ kubectl -n demo get kubevirtarchiverrestore
NAME                        PHASE       CHECKPOINT                                                                           AGE
vmtest-restore-checkpoint   Succeeded   kubestash-vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt-2026-11-22_04-51-31   2m29s
```

The restore has two stages, the disks and then the VM definition. Their progress is in the conditions:

```bash
$ kubectl -n demo get kubevirtarchiverrestore vmtest-restore-checkpoint -o jsonpath='{range .status.conditions[*]}{.type}{"  "}{.status}{"  "}{.message}{"\n"}{end}'
RecoveryPointSelected  True  selected checkpoint kubestash-vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt-2026-11-22_04-51-31 at 2026-11-22T04:52:11Z
VolumeRestoreSucceeded  True  volume RestoreSession completed
ManifestRestoreSucceeded  True  manifest RestoreSession completed
```

Check the data in the restored VM:

```bash
$ virtctl -n demo console vmtest-restored
cirros login: cirros
Password: gocubsgo
$ ls /home/cirros
note.txt
$ cat /home/cirros/note.txt
version 1
```

The VM has the data of that checkpoint: `version 1`, and no `note2.txt`.

### Restore to a Point in Time

With `targetTime`, the restore takes the newest recovery point that was complete at or before that time. Below is the YAML. The time `2026-11-22T05:00:00Z` is after the recovery point at 04:59:55 and before the one at 05:03:50.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: KubeVirtArchiverRestore
metadata:
  name: vmtest-restore-time
  namespace: demo
spec:
  source:
    repositoryRef:
      name: vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full
  target:
    name: vmtest-restored-time
  recoveryPoint:
    targetTime: "2026-11-22T05:00:00Z"
```

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/archiver/examples/restore-time.yaml
kubevirtarchiverrestore.core.kubestash.com/vmtest-restore-time created

$ kubectl -n demo get kubevirtarchiverrestore
NAME                        PHASE       CHECKPOINT                                                                           AGE
vmtest-restore-checkpoint   Succeeded   kubestash-vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt-2026-11-22_04-51-31   9m14s
vmtest-restore-time         Succeeded   kubestash-vmtest-d1fa4628-605e-4543-921c-5c59921ed036-full-cbt-2026-11-22_04-59-13   2m1s
```

The restore chose the checkpoint started at 04:59:13. Check the data:

```bash
$ virtctl -n demo console vmtest-restored-time
cirros login: cirros
Password: gocubsgo
$ ls /home/cirros
note.txt  note2.txt
$ cat /home/cirros/note.txt /home/cirros/note2.txt
version 2
second file
```

### Resume the Archiver

Before you resume the archiver, remove the labels from the restored VMs, so that it does not select them:

```bash
$ kubectl -n demo label vm vmtest-restored vmtest-restored-time archive- cbt-
virtualmachine.kubevirt.io/vmtest-restored unlabeled
virtualmachine.kubevirt.io/vmtest-restored-time unlabeled

$ kubectl -n demo patch kubevirtarchiver vmtest-archiver --type=merge -p '{"spec":{"pause":false}}'
kubevirtarchiver.core.kubestash.com/vmtest-archiver patched
```

If you want a restored VM to be backed up as well, keep its labels. The archiver then creates a separate `Repository` for it and starts a new chain with a full backup.

> **Note:** If a restored VM was already selected, for example because the archiver was not paused, removing its labels stops further backups, but the `Repository` the archiver created for it and its data in the backend are kept. Find it with `kubectl -n demo get repository` (its name starts with the VM name) and delete it if you do not need it.

## Cleaning Up

```bash
kubectl delete -n demo kubevirtarchiverrestore --all
kubectl delete -n demo vm --all
kubectl delete -n demo kubevirtarchiver vmtest-archiver
kubectl delete -n demo backupstorage s3-storage
kubectl delete -n demo retentionpolicy demo-retention
kubectl delete ns demo
```

> The `BackupStorage` in this guide has `deletionPolicy: WipeOut`. Deleting it removes the backed up data from the bucket, under the `prefix` you set.
