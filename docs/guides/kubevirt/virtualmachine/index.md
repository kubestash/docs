---
title: Backup and Restore a KubeVirt VirtualMachine | KubeStash
description: A step by step guide showing how to backup and restore a KubeVirt VirtualMachine.
menu:
  docs_{{ .version }}:
    identifier: kubevirt-virtualmachine
    name: Backup & Restore a VirtualMachine
    parent: kubevirt
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup and Restore a KubeVirt VirtualMachine

This guide will show you how to use KubeStash to backup and restore a KubeVirt `VirtualMachine`, along with the data on its disks.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).

- Install [KubeVirt](https://kubevirt.io/user-guide/cluster_workloads/installation/) and [CDI](https://github.com/kubevirt/containerized-data-importer) in your cluster.

- You should be familiar with the following `KubeStash` concepts:
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [Snapshot](/docs/concepts/crds/snapshot/index.md)

- If you want to backup a **running** `VirtualMachine`, your cluster additionally needs a CSI driver with snapshot support, a default `VolumeSnapshotClass` for the disks' `StorageClass`, and the KubeVirt `Snapshot` feature gate enabled. A stopped `VirtualMachine` needs none of these. See [How Backup and Restore works?](/docs/guides/kubevirt/overview/index.md) for the details.

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/kubevirt/virtualmachine/examples](/docs/guides/kubevirt/virtualmachine/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Backup a VirtualMachine

This section will show you how to use KubeStash to backup a `VirtualMachine`. Here, we are going to create a `VirtualMachine` with two disks and some sample data on them. Then, we are going to backup the VM and its disks using KubeStash.

### Prepare VirtualMachine

At first, we are going to create a `VirtualMachine` along with the objects it depends on. The sample VM has two `DataVolume`-backed disks, a cloud-init `Secret`, an SSH access-credential `Secret`, a `ConfigMap` and a `ServiceAccount`, so that we can see the whole dependency graph being captured.

Below is a part of the YAML of the `VirtualMachine` that we are going to create,

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: kubestash-demo-vm
  namespace: demo
spec:
  runStrategy: Halted
  template:
    spec:
      accessCredentials:
        - sshPublicKey:
            source:
              secret:
                secretName: kubestash-demo-sshkey
            propagationMethod:
              noCloud: {}
      domain:
        cpu:
          cores: 1
        resources:
          requests:
            memory: 256Mi
        devices:
          disks:
            - name: rootdisk
              disk:
                bus: virtio
            - name: datadisk
              disk:
                bus: virtio
            ...
      volumes:
        - name: rootdisk
          dataVolume:
            name: kubestash-demo-vm-root
        - name: datadisk
          dataVolume:
            name: kubestash-demo-vm-data
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: kubestash-demo-cloudinit
        - name: configdisk
          configMap:
            name: kubestash-demo-config
        - name: sadisk
          serviceAccount:
            serviceAccountName: kubestash-demo-sa
```

Let's create the `VirtualMachine` and its dependencies we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/virtualmachine/examples/vm.yaml
serviceaccount/kubestash-demo-sa created
configmap/kubestash-demo-config created
secret/kubestash-demo-cloudinit created
secret/kubestash-demo-sshkey created
datavolume.cdi.kubevirt.io/kubestash-demo-vm-root created
datavolume.cdi.kubevirt.io/kubestash-demo-vm-data created
virtualmachine.kubevirt.io/kubestash-demo-vm created
```

Now, wait for the `DataVolume`s to go into the `Succeeded` phase, which means CDI has finished provisioning the disks.

```bash
$ kubectl get datavolume -n demo
NAME                     PHASE       PROGRESS   RESTARTS   AGE
kubestash-demo-vm-data   Succeeded   N/A                   62s
kubestash-demo-vm-root   Succeeded   N/A                   62s
```

**Generate Sample Data:**

Now, let's write some sample data onto both disks so that we can verify the restore later. The VM is stopped, so its disk PVCs are free and a `Job` can mount them.

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/virtualmachine/examples/sample-data.yaml
job.batch/kubestash-demo-data-writer created
```

Let's note down the checksums of the sample data, so that we can compare them after the restore,

```bash
$ kubectl logs -n demo job/kubestash-demo-data-writer
bf07cb11e6dea27eb81b15480bbd093983cf68f8c619a2b32a3f07beab3d821e  /root-disk/sample.txt
117aeee770d409f8503f87a594cd2a7dc10b0b00aa26dcf481b5fcf5b7e45309  /data-disk/sample.txt
```

Once the `Job` has completed, delete it so that it does not hold a reference to the disk PVCs,

```bash
$ kubectl delete job -n demo kubestash-demo-data-writer
job.batch "kubestash-demo-data-writer" deleted
```

### Prepare Backend

We are going to store our backed up data into a GCS bucket. We have to create a Secret with necessary credentials and a `BackupStorage` CR to use this backend. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

**Create Secret:**

Let's create a secret called `gcs-secret` with access credentials to our desired GCS bucket,

```bash
$ echo -n '<your-project-id>' > GOOGLE_PROJECT_ID
$ cat /path/to/downloaded-sa-key.json > GOOGLE_SERVICE_ACCOUNT_JSON_KEY
$ kubectl create secret generic -n demo gcs-secret \
    --from-file=./GOOGLE_PROJECT_ID \
    --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
secret/gcs-secret created
```

**Create BackupStorage:**

Now, create a `BackupStorage` using this secret. Below is the YAML of `BackupStorage` CR we are going to create,

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

Let's create the BackupStorage we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/virtualmachine/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-storage created
```

**Create RetentionPolicy:**

Now, let's create a `RetentionPolicy` to specify how the old Snapshots should be cleaned up.

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
      from: All
```

Let’s create the above `RetentionPolicy`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/virtualmachine/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

### Backup

We have to create a `BackupConfiguration` CR targeting the `kubestash-demo-vm` `VirtualMachine` that we have created earlier. Then, KubeStash will create a `CronJob` for each session to take periodic backup of the VM.

At first, we need to create a secret with a Restic password for backup data encryption.

**Create Secret:**

Let's create a secret called `encrypt-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD \
secret "encrypt-secret" created
```

**Create BackupConfiguration:**

Below is the YAML of the `BackupConfiguration` CR that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-vm-backup
  namespace: demo
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: kubestash-demo-vm
    namespace: demo
  backends:
    - name: gcs-backend
      storageRef:
        name: gcs-storage
        namespace: demo
      retentionPolicy:
        name: demo-retention
        namespace: demo
  sessions:
    - name: demo-session
      scheduler:
        schedule: "*/5 * * * *"
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: gcs-vm-repo
          backend: gcs-backend
          directory: /vm
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: kubevirt
        tasks:
          - name: logical-backup
      retryConfig:
        maxRetry: 2
        delay: 1m
```

Here,

- `spec.target` refers to the `VirtualMachine` we are backing up. A `VirtualMachine` is a first-class KubeStash target, identified by the `kubevirt.io` API group.
- `spec.sessions[*].addon.tasks[*].name` is set to `logical-backup`, which backs up the VM's manifests and all of its disks in one task.

> The addon also exposes `manifest-backup` and `volume-backup` as separate tasks if you would rather run the two halves as separate task containers. The resulting `Snapshot` is the same either way.

Let's create the `BackupConfiguration` CR we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/virtualmachine/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-vm-backup created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME               PHASE   PAUSED   AGE
sample-vm-backup   Ready            21s
```

Additionally, we can verify that the `Repository` specified in the `BackupConfiguration` has been created using the following command,

```bash
$ kubectl get repo -n demo
NAME          INTEGRITY   SNAPSHOT-COUNT   SIZE   PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-vm-repo               0                0 B    Ready                            51s
```

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` CR.

Verify that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-sample-vm-backup-demo-session   */5 * * * *   False     0        <none>          79s
```

**Wait for BackupSession:**

Wait for the next schedule for backup. Run the following command to watch `BackupSession` CR,

```bash
$ kubectl get backupsession -n demo -w
NAME                                       INVOKER-TYPE          INVOKER-NAME       PHASE       DURATION   AGE
sample-vm-backup-demo-session-1787216260   BackupConfiguration   sample-vm-backup   Succeeded   87s        3m
```

We can see from the above output that the backup session has succeeded. Now, we are going to verify whether the backed up data has been stored in the backend.

**Verify Backup:**

Once a backup is complete, KubeStash will update the respective `Repository` CR to reflect the backup. Check that the repository `gcs-vm-repo` has been updated by the following command,

```bash
$ kubectl get repository -n demo gcs-vm-repo
NAME          INTEGRITY   SNAPSHOT-COUNT   SIZE          PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-vm-repo   true        1                47.374 KiB    Ready   2m11s                    6m
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-vm-repo
NAME                                                   REPOSITORY    SESSION        SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-vm-repo-sample-vm-backup-demo-session-1787216260   gcs-vm-repo   demo-session   2026-08-20T14:19:23Z   Delete            Succeeded   2m35s
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <target-kind>`
> - `kubestash.com/app-ref-name: <target-name>`
> - `kubestash.com/app-ref-namespace: <target-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
>
> These labels can be used to watch only the `Snapshot`s related to our desired `VirtualMachine` or `Repository`.

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the `VirtualMachine`.

```bash
$ kubectl get snapshots -n demo gcs-vm-repo-sample-vm-backup-demo-session-1787216260 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: VirtualMachine
    kubestash.com/app-ref-name: kubestash-demo-vm
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-vm-repo
  name: gcs-vm-repo-sample-vm-backup-demo-session-1787216260
  namespace: demo
spec:
  ...
status:
  components:
    manifest:
      driver: Restic
      integrity: true
      path: repository/v1/demo-session/manifest
      phase: Succeeded
      ...
    volume-kubestash-demo-vm-data:
      driver: Restic
      integrity: true
      path: repository/v1/demo-session/volume-kubestash-demo-vm-data
      phase: Succeeded
      ...
    volume-kubestash-demo-vm-root:
      driver: Restic
      integrity: true
      path: repository/v1/demo-session/volume-kubestash-demo-vm-root
      phase: Succeeded
      ...
  integrity: true
  phase: Succeeded
  totalComponents: 3
  ...
```

Here, the `manifest` component holds the manifests of the VM's dependency graph, and each `volume-<pvc-name>` component holds the data of one disk. The `Snapshot` reaches the `Succeeded` phase only when every component has succeeded, so a `Succeeded` `Snapshot` can never be missing a disk.

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/vm/repository/v1/demo-session` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/vm/snapshots` directory.

> Note: KubeStash stores all backed up data encrypted in the backup directory, meaning it remains unreadable until decrypted.

## Restore

In this section, we are going to show you how to restore the `VirtualMachine` from the backup we have just taken.

**Simulate Disaster:**

Now, let's simulate a disaster scenario. Here, we are going to delete the `VirtualMachine` along with everything it depends on.

```bash
$ kubectl delete -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/virtualmachine/examples/vm.yaml
serviceaccount "kubestash-demo-sa" deleted
configmap "kubestash-demo-config" deleted
secret "kubestash-demo-cloudinit" deleted
secret "kubestash-demo-sshkey" deleted
datavolume.cdi.kubevirt.io "kubestash-demo-vm-root" deleted
datavolume.cdi.kubevirt.io "kubestash-demo-vm-data" deleted
virtualmachine.kubevirt.io "kubestash-demo-vm" deleted
```

Verify that the `VirtualMachine` and its disks are gone,

```bash
$ kubectl get vm,datavolume,pvc -n demo
No resources found in demo namespace.
```

**Create RestoreSession:**

To restore the `VirtualMachine`, you have to create a `RestoreSession` object pointing to the `Repository` and `Snapshot` you want to restore from.

Here, is the YAML of the `RestoreSession` object that we are going to use for restoring our `kubestash-demo-vm` `VirtualMachine`.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: sample-vm-restore
  namespace: demo
spec:
  dataSource:
    repository: gcs-vm-repo
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret
      namespace: demo
  addon:
    name: kubevirt
    tasks:
      - name: logical-restore
```

Here,

- `spec.dataSource.snapshot` specifies to restore from latest `Snapshot`.
- `spec.addon.tasks[*].name` is set to `logical-restore`, which restores the disks first and then applies the manifests, in a single task.

> **Note:** A `RestoreSession` for a `VirtualMachine` must **not** set `spec.target`. The `VirtualMachine` does not exist yet at restore time, so a target pointing at a missing object would never be found. KubeStash learns the VM's identity from the backed up manifests instead.

Let's create the `RestoreSession` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/virtualmachine/examples/restoresession.yaml
restoresession.core.kubestash.com/sample-vm-restore created
```

Once, you have created the `RestoreSession` object, KubeStash will create restore Job(s). Run the following command to watch the phase of the `RestoreSession` object,

```bash
$ watch kubectl get restoresession -n demo
NAME                REPOSITORY    FAILURE-POLICY   PHASE       DURATION   AGE
sample-vm-restore   gcs-vm-repo                    Succeeded   51s        79s
```

The `Succeeded` phase means that the restore process has been completed successfully.

**Verify Restored VirtualMachine:**

Let's verify that the `VirtualMachine` and every object it depends on have come back,

```bash
$ kubectl get vm,datavolume,pvc,secret,configmap,serviceaccount -n demo | grep kubestash-demo
virtualmachine.kubevirt.io/kubestash-demo-vm   25s   Stopped   False
datavolume.cdi.kubevirt.io/kubestash-demo-vm-data   Succeeded   N/A   25s
datavolume.cdi.kubevirt.io/kubestash-demo-vm-root   Succeeded   N/A   25s
persistentvolumeclaim/kubestash-demo-vm-data   Bound   pvc-39036687-7270-4521-9207-ba5b113bbecc   1Gi   RWO
persistentvolumeclaim/kubestash-demo-vm-root   Bound   pvc-a1fb889b-6d30-4846-a0b0-7685070e3019   1Gi   RWO
secret/kubestash-demo-cloudinit    Opaque   1   25s
secret/kubestash-demo-sshkey       Opaque   1   25s
configmap/kubestash-demo-config    1        25s
serviceaccount/kubestash-demo-sa   25s
```

The `DataVolume`s are already in the `Succeeded` phase because KubeStash restored the disk data before applying them, and CDI adopted the populated PVCs instead of importing their contents again.

**Verify Restored Data:**

Now, let's verify that the data on the disks was restored, by comparing the checksums against the ones we noted before the backup,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/kubevirt/virtualmachine/examples/verify-data.yaml
job.batch/kubestash-demo-data-verify created

$ kubectl logs -n demo job/kubestash-demo-data-verify
kubestash-root-sample-data
kubestash-data-sample-data
bf07cb11e6dea27eb81b15480bbd093983cf68f8c619a2b32a3f07beab3d821e  /root-disk/sample.txt
117aeee770d409f8503f87a594cd2a7dc10b0b00aa26dcf481b5fcf5b7e45309  /data-disk/sample.txt
```

Hence, we can see from the above output that the checksums match the ones we noted before the backup, so the disk data has been restored successfully.

The restored `VirtualMachine` is stopped. Start it whenever you are ready,

```bash
$ kubectl patch vm -n demo kubestash-demo-vm --type merge -p '{"spec":{"runStrategy":"Always"}}'
virtualmachine.kubevirt.io/kubestash-demo-vm patched
```

> A restored `VirtualMachine` always cold-boots. KubeStash does not capture the guest's RAM or the `VirtualMachineInstance`, because KubeVirt recreates those from the `VirtualMachine` itself.

## Backup a Running VirtualMachine

You do not have to stop a `VirtualMachine` to back it up, and you do not have to configure anything differently — the same `BackupConfiguration` works. When the VM is running, KubeStash creates a `VirtualMachineSnapshot` and lets KubeVirt freeze the guest filesystem, take one CSI `VolumeSnapshot` per disk and unfreeze, then copies those snapshots to the backend and cleans them up.

You can watch that happen during a backup of a running VM,

```bash
$ kubectl get virtualmachinesnapshot,volumesnapshot -n demo
```

Install `qemu-guest-agent` in the guest to get a filesystem-consistent backup. Without it, KubeVirt cannot freeze the filesystem and the backup is crash-consistent — it still succeeds, and records a warning.

## Restore Selected Components

Since each disk is a separate component, you can restore a subset instead of the whole VM. Pass the component names to the `components` parameter of the restore task,

```yaml
  addon:
    name: kubevirt
    tasks:
      - name: logical-restore
        params:
          components: volume-kubestash-demo-vm-root
```

Restoring only the `manifest` component recreates the `VirtualMachine` and its `DataVolume`s without their data, letting CDI import the disks from their original sources again.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo backupconfiguration sample-vm-backup
kubectl delete -n demo restoresession sample-vm-restore
kubectl delete -n demo vm kubestash-demo-vm
kubectl delete -n demo datavolume --all
kubectl delete -n demo backupstorage gcs-storage
kubectl delete -n demo retentionpolicy demo-retention
kubectl delete -n demo secret gcs-secret encrypt-secret
```
