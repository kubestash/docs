---
title: kubectl Plugin | KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-cli
    name: KubeStash kubectl Plugin
    parent: cli
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# KubeStash kubectl Plugin

KubeStash gives you kubectl plugin support named `kubectl kubestash` cli. `kubectl kubestash` cli can be used to manage KubeStash objects quickly and easily. It performs various operations like coping KubeStash objects, cloning PVC, unlock Repository, triggering an instant backup, etc. To install KubeStash kubectl plugin on your workstation, follow the steps [here](/docs/setup/README.md).

## Available Command

Available command for `kubectl kubestash` cli are:

| Main Command                                | Uses                                                                       |
|---------------------------------------------|----------------------------------------------------------------------------|
| [copy secret](#copy-secret)                 | Copy `Secret` from source namespace to destination namespace.              |
| [copy volumesnapshot](#copy-volumesnapshot) | Copy `VolumeSnapshot` from source namespace to destination namespace.      |
| [clone pvc](#clone-pvc)                     | Clone a PVC from source namespace to destination namespace.                |
| [download](#download-snapshot)              | Download components of a snapshot from backend into your local repository. |
| [trigger](#trigger-an-instant-backup)       | Take an instant backup.                                                    |
| [unlock](#unlock-repository)                | Unlock Restic Repositories                                                 |
| [pause backup](#pause-backup)               | Pause KubeStash backup.                                                    |
| [resume backup](#resume-backup)             | Resume KubeStash backup.                                                   |
| [password add](#password-add)               | Add a new password (key) to restic repositories.                           |
| [password list](#password-list)             | List the passwords (keys) for restic repositories.                         |
| [password remove](#password-remove)         | Remove passwords (keys) from restic repositories                           |
| [password update](#password-update)         | Update current password (key) for restic repositories                      |
| [debug backup](#debug-backup)               | Debug KubeStash backup issues.                                             |
| [debug restore](#debug-restore)             | Debug KubeStash restore issues.                                            |
| [debug operator](#debug-operator)           | Debug KubeStash operator issues.                                           |
| [convert](#convert)                         | Convert Stash resources yaml into equivalent KubeStash resources yaml      |

## Copy Command

`kubectl kubestash cp` command is used to copy objects like, `Secret` and `VolumeSnapshot`, from one namespace to another namespace.

### Copy Secret

To copy a Secret, you need to provide `Secret` name and destination namespace. The available flags are:

| Flag             | Description                                                            |
|------------------|------------------------------------------------------------------------|
| `--namespace`    | Indicates the namespace of the respective `Secret`.                    |
| `--to-namespace` | Indicates the destination namespace where the `Secret` will be copied. |

**Format:**

```bash
kubectl kubestash cp secret <secret-name> [flags]
```

**Example:**

```bash
$ kubectl kubestash cp secret my-secret --namespace=demo --to-namespace=demo1
```

### Copy VolumeSnapshot

To copy a VolumeSnapshot, you need to provide `VolumeSnapshot` name and destination namespace. The available flags are:

| Flag             | Description                                                                    |
|------------------|--------------------------------------------------------------------------------|
| `--namespace`    | Indicates the namespace of the respective `VolumeSnapshot`.                    |
| `--to-namespace` | Indicates the destination namespace where the `VolumeSnapshot` will be copied. |

**Example:**

```bash
$ kubectl kubestash cp volumesnapshot my-vol-snap --namespace=demo --to-namespace=demo1
```

## Clone PVC

`kubectl kubestash clone pvc` command is used to clone PVC from one namespace to another namespace. When we run the command the cloning process consists of the following steps:

- At first, It creates a BackupStorage in the source namespace (if no storage is provided).
- It creates a RetentionPolicy in the source namespace.
- Using these BackupStorage and RetentionPolicy, it creates a BackupConfiguration targeting the PVC to take backup.
- After the backup process succeeded, It creates a RestoreSession in the source namespace with the target in destination namespace.
- finally, It restores the backed up data into VolumeClaimTemplate in the destination namespace.

To clone a PVC, you can provide the reference of the existing BackupStorage if you already have one; otherwise, you have to provide the backend information. Additionally, you need to provide the encryption secret reference. The available flags are:

| Flag                         | Description                                                           |
|------------------------------|-----------------------------------------------------------------------|
| `--namespace`                | Indicates namespace of the respective pvc.                            |
| `--to-namespace`             | Indicates the destination namespace where the PVC will be cloned.     |
| `--provider`                 | Specify the Backend provider (i.e. gcs, s3, azure etc).               |
| `--bucket`                   | Specify the name of the cloud bucket/container.                       |
| `--endpoint`                 | Specify the endpoint for s3 or s3 compatible backend.                 |
| `--region`                   | Specify the region for s3 or s3 compatible backend.                   |
| `--max-connections`          | Specify maximum concurrent connections for GCS, Azure and B2 backend. |
| `--storage-secret`           | Specify the name of the storage secret.                               |
| `--encrypt-secret`           | Specify the name of the encryption secret                             |
| `--encrypt-secret-namespace` | Specify the namespace of the encryption secret.                       |
| `--prefix`                   | Specify the directory inside the backend.                             |
| `--storage-name`             | Specify the name of the BackupStorage.                                |
| `--storage-namespace`        | Specify the namespace of the BackupStorage.                           |

If you want to use existing BackupStorage, then provide `storage-name` and `storage-namespace` flags and there is no need to provide the `provider`, `bucket`, `endpoint`, `region`, `max-connections`, `storage-secret` and `prefix` flags.

**Format:**

```bash
kubectl kubestash clone pvc <pvc-name> [flags]
```

**Example:**

```bash
$ kubectl kubestash clone pvc my-pvc -n demo --to-namespace=demo-1 --storage-secret=<storage-secret> --bucket=<bucket> --prefix=<prefix> --provider=<provider> --encrypt-secret=<encryption-secret> --encrypt-secret-namespace=<encryption-secret-namespace>
```

## Download Snapshot

`kubectl kubestash download` command is used to download the components of a snapshot from backend repository into your local machine.
To download the snapshot you have to provide `Snapshot` name and download directory. The available flags are:

| Flag            | Description                                                                                                                              |
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------|
| `--namespace`   | Indicates the namespace of the respective `Snapshot`.                                                                                    |
| `--destination` | Indicates the local directory where the components of the snapshot will be downloaded.                                                   |
| `--components`  | Specifies the list of components to download. (If this flag is not provided, then all the components of the snapshot will be downloaded) |
| `--exclude`     | Specifies the list of pattern for directory/file to ignore during download.                                                              |
| `--include`     | Specifies the list of pattern for directory/file to download.                                                                            |

**Format:**

```bash
kubectl kubestash download <snapshot-name> [flags]
```

**Example:**

```bash
$ kubectl kubestash download demo-storage-sample-backup-frequent-backup-1699355040 --namespace=demo --destination=/home/user/downloads/ 
```

## Trigger an Instant Backup

`kubectl kubestash trigger` command is used to take an instant backup in KubeStash.
To trigger an instant backup, you need to have a BackupConfiguration in your cluster. You need to provide the BackupConfiguration name. You can also provide the namespace by using the `--namespace` flag. This flag indicates the namespace where the trigger backup will be occurred. 
To trigger backup for specific sessions, you need to provide the session names (comma seperated) by using the `--sessions` flag. If you want to trigger backup for all the sessions present in the BackupConfiguration, then this flag should be omitted.

**Format:**

```bash
$ kubectl kubestash trigger <backupconfig-name> [flags]
```

**Example:**

```bash
$ kubectl kubestash trigger my-config --namespace=demo --sessions=frequent-backup,everyday-backup
```

## Unlock Repository

`kubectl kubestash unlock` is used to remove lock from the backend repository.

To unlock the Repository, you need to provide a Repository name. You can also provide the namespace by using the `--namespace` flag. This flag indicates the Repository namespace.

A Repository can have multiple components. To unlock the repository for a specific component, you need to provide the component paths (comma seperated) by using the `--paths` flag. You can find the paths in the `status.componentPaths` section of the respective Repository. If you want to unlock all the components of the respective Repository, then this flag should be omitted.

**Format:**

```bash
kubectl kubestash unlock <repository-name> [flags]
```

**Example:**

```bash
$ kubectl kubestash unlock demo-repo --namespace=demo --paths=repository/v1/frequent-backup/pod-0,repository/v1/frequent-backup/pod-1
```

## Pause Backup

`kubectl kubestash pause` command is used to pause KubeStash backup temporarily.
To pause a backup you have to provide the `BackupConfiguration` name. You can also provide the namespace by using the `--namespace` flag. This flag indicates the BackupConfiguration namespace.

**Format:**

```bash
kubectl kubestash pause <backupconfiguration-name> [flags]
```

**Example:**

```bash
$ kubectl kubestash pause sample-backupconfig -n demo
```

## Resume Backup
To resume a backup you have to provide the `BackupConfiguration` name. You can also provide the namespace by using the `--namespace` flag. This flag indicates the BackupConfiguration namespace.

**Format:**

```bash
kubectl kubestash resume <backupconfiguration-name> [flags]
```

**Example:**

```bash
$ kubectl kubestash resume sample-backupconfig -n demo
```

## Debug Command
`kubectl kubestash debug` command is used to debug KubeStash resources. This command describes the necessary resources and shows logs from the related pods which makes the debugging process quicker and easier.

### Debug Backup
To debug a backup you have to provide the BackupConfiguration name. You can also provide the namespace by using the `--namespace` flag. This flag indicates the BackupConfiguration namespace.

**Format:**

```bash
kubectl kubestash debug backup <backupconfiguration-name> [flags]
```

**Example:**

```bash
$ kubectl kubestash debug backup sample-backupconfig --namespace=demo
```

### Debug Restore
To debug a restore you have to provide the `RestoreSession` name. You can also provide the namespace by using the `--namespace` flag. This flag indicates the RestoreSession namespace.

**Format:**

```bash
kubectl kubestash debug restore <restoresession-name> [flags]
```

**Example:**

```bash
$ kubectl kubestash debug restore sample-restore --namespace=demo 
```

### Debug Operator
To debug the KubeStash operator you need to run the following command:

```bash
$ kubectl kubestash debug operator
```

## Password Command
`kubectl kubestash pw` command is used to manage passwords (keys) of restic repositories.

### Password Add
This command is used to add a new password (key) to restic repositories. You have to provide the information of the new password by using flags. The available flags are:

| Flag                  | Description                                                                          |
|-----------------------|--------------------------------------------------------------------------------------|
| `--namespace`         | Indicates the namespace of the respective `Repository`.                              |
| `--host`              | Indicates the host of the new password.                                              |
| `--user`              | Specifies the user of the new password.                                              |
| `--new-password-file` | Specifies the file from which to read the new password.                              |
| `--paths`             | Specifies the list of component paths (restic repositories) to add the new password. |

**Format**

```bash
kubectl kubestash pw add <repository-name> [flags]
```

**Example**

```bash
kubectl kubestash pw add demo-repo --namespace=demo --user=root --host=demo-host --new-password-file=pass.txt
```

### Password List
This command is used to list restic passwords (keys) of restic repositories. You can also provide the list of component paths (restic repositories) for which to list the passwords by using `--paths` flag. You can also provide the namespace by using the `--namespace` flag. This flag indicates the Repository namespace.

**Format**

```bash
kubectl kubestash pw list <repository-name> [flags]
```

**Example**

```bash
kubectl kubestash pw list demo-repo --namespace=demo --paths=repository/v1/frequent-backup/pod-0,repository/v1/frequent-backup/pod-1
```

### Password Update
This command is used to update the current password (key) of restic repositories. The available flags for this command are:

| Flag                  | Description                                                                         |
|-----------------------|-------------------------------------------------------------------------------------|
| `--namespace`         | Indicates the namespace of the respective `Repository`.                             |
| `--new-password-file` | Specifies the file from which to read the new password.                             |
| `--paths`             | Specifies the list of component paths (restic repositories) to update the password. |

**Format**

```bash
kubectl kubestash pw update <repository-name> [flags]
```

**Example**

```bash
kubectl kubestash pw update demo-repo --namespace=demo --paths=repository/v1/frequent-backup/pod-0,repository/v1/frequent-backup/pod-1 --new-password-file=pass.txt
```

### Password Remove
This command is used to remove passwords (keys) from restic repositories. The available flags for this command are:

| Flag          | Description                                                                                                                                                                                                                   |
|---------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `--namespace` | Indicates the namespace of the respective `Repository`.                                                                                                                                                                       |
| `--id-paths`  | Specifies the list of restic password ID and corresponding component path (restic repository) pairs (seperated by `:`). The specified passwords, associated with the given IDs, will be removed from the restic repositories. |

**Format**

```bash
kubectl kubestash pw remove <repository-name> [flags]
```

**Example**

```bash
kubectl kubestash pw remove demo-repo --namespace=demo --id-paths=a30343d8:repository/v1/frequent-backup/pod-2
```

### Convert
This command is used to convert [Stash](https://stash.run/) resources into their corresponding KubeStash equivalents. This allows users to seamlessly migrate from Stash to KubeStash by converting existing YAML configurations. The following Stash resources are converted into their KubeStash counterparts:

| Stash Resource        | KubeStash Equivalent                                     |
|-----------------------|----------------------------------------------------------|
| `Repository`          | `BackupStorage`                                          |
| `BackupConfiguration` | `BackupConfiguration`, `HookTemplate`, `RetentionPolicy` |
| `RestoreSession`      | `RestoreSession`, `HookTemplate`                         |

The available flags for this command are:

| Flag           | Description                                                                             |
|----------------|-----------------------------------------------------------------------------------------|
| `--source-dir` | Specifies the directory containing Stash resource YAML files that need to be converted. |
| `--target-dir` | Defines the directory where the converted KubeStash resource YAML files will be stored. |

**Format**

```bash
kubectl kubestash convert [flags]
```

**Example**

To convert Stash resource files stored in `/home/user/stash-resources` and save the converted KubeStash resources to `/home/user/kubestash-resources`, use the following command:

```bash
kubectl kubestash convert --source-dir=/home/user/stash-resources --target-dir=/home/user/kubestash-resources 
```

> After the conversion process, certain fields in the generated KubeStash resource files require manual input. These fields are marked with placeholders in the format `### Set Valid <field-name> ###`, indicating that users need to replace them with appropriate values before applying the resources.