---
title: Welcome | KubeStash
description: Welcome to KubeStash
menu:
  docs_{{ .version }}:
    identifier: readme-kubestash
    name: Readme
    parent: welcome
    weight: -1
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: welcome
url: /docs/{{ .version }}/welcome/
aliases:
  - /docs/{{ .version }}/
  - /docs/{{ .version }}/README/
---

# KubeStash

[KubeStash](https://kubestash.com) by AppsCode is a cloud native data backup and recovery solution for Kubernetes workloads. If you are running production workloads in Kubernetes, you might want to take backup of your disks, databases etc. Traditional tools are too complex to set up and maintain in a dynamic compute environment like Kubernetes. KubeStash is a Kubernetes operator that uses [restic](https://github.com/restic/restic) or Kubernetes CSI Driver VolumeSnapshotter functionality to address these issues. Using KubeStash, you can backup Kubernetes volumes mounted in workloads, stand-alone volumes and databases. Users may even extend KubeStash via [addons](https://kubestash.com/docs/{{< param "info.version" >}}/guides/addons/overview/) for any custom workload.

Here, we are going to give you an overview of KubeStash documentation structure.

## [Concepts](/docs/concepts/)

Concept explains some significant aspect of KubeStash. This is where you can learn about what KubeStash does and how it does it.

- [KubeStash Overview](/docs/concepts/what-is-kubestash/overview/index.md) Provides an introduction to KubeStash and gives an overview of the features it provides.
- [KubeStash Architecture](/docs/concepts/what-is-kubestash/architecture/index.md) Provides an overview of KubeStash architecture, and it's core components.
- [KubeStash API](/docs/concepts/crds/backupstorage/index.md) Introduces KubeStash CRDs.

## [Setup](/docs/setup/)

Setup contains instruction for installing, uninstalling, and upgrading KubeStash.

- **Install KubeStash:** Provides installation instructions for KubeStash and its various components.
  - [KubeStash](/docs/setup/install/kubestash/index.md): Provides installation instructions for KubeStash.
  - [kubeStash kubectl Plugin](/docs/setup/install/kubectl-plugin/index.md): Provides installation instructions for KubeStash `kubectl` plugin.
  - [Troubleshooting](/docs/setup/install/troubleshooting/index.md): Provides troubleshooting guide for various installation problems.
- **Uninstall KubeStash:** Provides uninstallation instructions for KubeStash and its various components.
  - [KubeStash](/docs/setup/uninstall/kubestash/index.md): Provides uninstallation instructions for KubeStash.
  - [KubeStash kubectl Plugin](/docs/setup/uninstall/kubectl-plugin/index.md): Provides uninstallation instructions for KubeStash `kubectl` plugin.
- [Upgrade KubeStash](/docs/setup/upgrade/index.md): Provides instruction for updating KubeStash license and upgrading between various KubeStash versions.

## [Guides](/docs/guides/)

Guides show how to perform different operations with KubeStash.

- [Supported Backends](/docs/guides/backends/overview/index.md): Describes how to configure different storage for storing backed up data.
- [Workload Volume Backup](/docs/guides/workloads/overview/index.md): Shows how to use KubeStash to backup and restore volumes of a workload (i.e. `Deployment`, `StatefulSet`, `DaemonSet`, etc).
- [Stand-alone Volume Backup](/docs/guides/volumes/overview/index.md): Shows how to use KubeStash to backup and restore stand-alone volumes(i.e. `PersistentVolumeClaim`).
- [Auto Backup](/docs/guides/auto-backup/overview/index.md): Shows how to configure automatic backup of any stateful workload in your cluster.
- [Volume Snapshot](/docs/guides/volumesnapshot/overview/index.md): Shows how KubeStash takes snapshot of `PersistentVolumeClaim`s and restore them from snapshot using Kubernetes `VolumeSnapshot` API.
- [Platforms](/docs/guides/platforms/eks-irsa/index.md): Shows how to use KubeStash to backup and restore volumes of a Kubernetes workload running in different platforms.
- [Hooks](/docs/guides/hooks/overview/index.md): Shows how to execute different actions before/after the backup/restore process.
- [CLI](/docs/guides/cli/kubectl-plugin/index.md): Shows how to manage KubeStash objects quickly and easily using KubeStash `kubectl` plugin.
- [Security](/docs/guides/security/rbac/index.md): Describes different built-in cluster security support by KubeStash.

We're always looking for help improving our documentation, so please don't hesitate to [file an issue](https://github.com/kubestash/project/issues/new) if you see some problem.
