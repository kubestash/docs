---
title: Table of Contents | Guides
description: Table of Contents | Guides
menu:
  docs_{{ .version }}:
    identifier: guides-readme
    name: Readme
    parent: guides
    weight: -1
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
url: /docs/{{ .version }}/guides/
aliases:
  - /docs/{{ .version }}/guides/README/
---

# Guides

Guides show how to perform different operations with KubeStash. We have divided guides section into the following sub-sections:

- [Supported Backends](/docs/guides/backends/overview/index.md): Describes how to configure different storage for storing backed up data.
- [Workload Volume Backup](/docs/guides/workloads/overview/index.md): Shows how to use KubeStash to backup and restore volumes of a workload (i.e. `Deployment`, `StatefulSet`, `DaemonSet`, etc).
- [Stand-alone Volume Backup](/docs/guides/volumes/overview/index.md): Shows how to use KubeStash to backup and restore stand-alone volumes(i.e. `PersistentVolumeClaim`).
- [Auto Backup](/docs/guides/auto-backup/overview/index.md): Shows how to configure automatic backup of any stateful workload in your cluster.
- [Volume Snapshot](/docs/guides/volumesnapshot/overview/index.md): Shows how KubeStash takes snapshot of `PersistentVolumeClaim`s and restore them from snapshot using Kubernetes `VolumeSnapshot` API.
- [Platforms](/docs/guides/platforms/eks-irsa/index.md): Shows how to use KubeStash to backup and restore volumes of a Kubernetes workload running in different platforms.
- [Monitoring](/docs/guides/monitoring/overview/index.md): Shows how Prometheus monitoring works with KubeStash, what metrics KubeStash exports, and how to enable monitoring.
- [Hooks](/docs/guides/hooks/overview/index.md): Shows how to execute different actions before/after the backup/restore process.
- [CLI](/docs/guides/cli/kubectl-plugin/index.md): Shows how to manage KubeStash objects quickly and easily using KubeStash `kubectl` plugin.
- [Security](/docs/guides/security/rbac/index.md): Describes different built-in cluster security support by KubeStash.
