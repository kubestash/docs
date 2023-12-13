---
title: KubeStash Overview
description: KubeStash Overview
menu:
  docs_{{ .version }}:
    identifier: overview-concepts
    name: Overview
    parent: what-is-kubestash
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# KubeStash

[KubeStash](https://kubestash.Com) by AppsCode is a cloud native data backup and recovery solution for Kubernetes workloads. If you are running production workloads in Kubernetes, you might want to take backup of your disks, databases etc. Traditional tools are too complex to setup and maintain in a dynamic compute environment like Kubernetes. KubeStash is a Kubernetes operator that uses [restic](https://github.com/restic/restic) or Kubernetes CSI Driver VolumeSnapshotter functionality to address these issues. Using KubeStash, you can backup Kubernetes volumes mounted in workloads, stand-alone volumes and databases. User may even extend KubeStash via [addons](https://stash.run/docs/{{< param "info.version" >}}/guides/addons/overview/) for any custom workload.

## Features

| Features                                                                                | Scope                                                                                                                                  |
|-----------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Backup & Restore Stand-alone Volume (PVC)                                               | PersistentVolumeClaim, PersistentVolume                                                                                                |
| Schedule Backup, Instant Backup                                                         | Schedule through [cron expression](https://en.wikipedia.org/wiki/Cron) or trigger instant backup using KubeStash Kubernetes plugin     |
| Backup & Restore subset of files                                                        | Only backup/restore the files that matches the provided patterns                                                                       |
| Backup & Restore databases                                                              | PostgreSQL, MySQL, MongoDB, Elasticsearch, Redis                                                                                       |
| Pause Backup                                                                            | No new backup when paused.                                                                                                             |
| Cleanup old `Snapshot`s automatically                                                   | Cleanup old `Snapshot`s according to `RetentionPolicy`.                                                                                |
| Encryption, Deduplication (send only diff)                                              | Encrypt backed up data with AES-256. Stash only sends the changes since last backup.                                                   |
| [CSI Driver Integration](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) | VolumeSnapshot for Kubernetes workloads. Supported for Kubernetes v1.17.0+.                                                            |
| Security                                                                                | Built-in support for RBAC and Network Policy                                                                                           |
| CLI                                                                                     | `kubectl` plugin (for Kubernetes 1.12+)                                                                                                |
| Extensibility and Customizable                                                          | Write addons for bespoke applications and customize currently supported workloads                                                      |
| Hooks	                                                                                  | Execute `httpGet`, `httpPost`, `tcpSocket` and `exec` hooks before and after of backup or restore process according to `HookTemplate`. |
| Cloud Storage as Backend                                                                | Stores backup data in AWS S3, Minio, GCS and Azure                                                                                     |
| On-prem Storage as Backend                                                              | Stores backup data in any locally mounted Kubernetes Volumes such as NFS, etc.                                                         |
| Auto Backup                                                                             | Share backup configuration across workloads using templates. Enable backup for a target application via annotation.                    | 
| Point-In-Time Recovery (PITR)                                                           | Restore a set of files from a time in the past.                                                                                        |
| Role Based Access Control (RBAC)                                                        |                                                                                                                                        |
