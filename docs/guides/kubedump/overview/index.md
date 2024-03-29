---
title: KubeDump Backup Overview | KubeStash
description: How KubeDump Backup Works in KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-kubedump-overview
    name: How does it work?
    parent: kubestash-kubedump
    weight: 10
product_name: KubeStash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# How KubeStash Backups Kubernetes Resources

KubeStash `{{< param "info.version" >}}` supports taking backup of Kubernetes resource YAMLs. You can backup the YAML definition of the resources of entire cluster, a particular namespace, or only an application. In this guide, we are going to show you how Kubernetes resource backup works in KubeStash.

## How Backup Works

The following diagram shows how KubeStash takes backup of the Kubernetes resources. Open the image in a new tab to see the enlarged version.

<figure align="center">
  <img alt="KubeDump Backup Overview" src="/docs/guides/kubedump/overview/images/kubedump-backup.svg">
  <figcaption align="center">Fig: KubeDump Backup Overview</figcaption>
</figure>

The backup process consists of the following steps:

1. At first, a user creates a `Secret`. This secret holds the credentials to access the backend where the backed up data will be stored.

2. Then, she creates a `BackupStorage` custom resource that specifies the backend information, along with the `Secret` containing the credentials needed to access the backend.

3. KubeStash operator watches for `BackupStorage` custom resources. When it finds a `BackupStorage` object, it initializes the `BackupStorage` by uploading the `metadata.yaml` file to the target storage.

4. Then, she creates a `BackupConfiguration` custom resource that specifies the target, the Addon info with a specified task, etc. It also provides information about one or more repositories, each indicating a path and a `BackupStorage` for storing the backed-up data.

5. KubeStash operator watches for `BackupConfiguration` custom resources.

6. Once the KubeStash operator finds a `BackupConfiguration` object, it creates `Repository` with the information specified in the BackupConfiguration.

7. KubeStash operator watches for `Repository` custom resources. When it finds the `Repository` object, it Initializes `Repository` by uploading `repository.yaml` file into the `spec.sessions[*].repositories[*].directory` path specified in `BackupConfiguration`.

8. Then, it creates a `CronJob` for each session with the schedule specified in `BackupConfiguration` to trigger backup periodically.

9. On the next scheduled slot, the `CronJob` triggers a backup by creating a `BackupSession` custom resource.

10. KubeStash operator watches for `BackupSession` custom resources.

11. When it finds a `BackupSession` object, it creates a `Snapshot` custom resource for each `Repository` specified in the respective session of the `BackupConfiguration`.

12. Then, it resolves the respective `Addon` and `Function` and prepares a backup `Job` definition.

13. Then, it creates the `Job` to backup the desired resource YAMLs.

14. Then, the Job dumps the Kubernetes Resource YAML, and store them temporarily in a directory. Then, it uploads the content of the directory file to the target storage.

15. After the backup process is completed, the backup `Job` updates the `status.components[*]` field of the `Snapshot` resources.

## Next Steps

- Backup the YAMLs of a particular application using KubeStash following the guide from [here](/docs/guides/kubedump/application/index.md).
- Backup the YAMLs of an entire Namespace using KubeStash following the guide from [here](/docs/guides/kubedump/namespace/index.md).
- Backup the YAMLs of your entire Kubernetes cluster using KubeStash following the guide from [here](/docs/guides/kubedump/cluster/index.md).
