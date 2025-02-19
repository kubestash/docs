---
title: Monitoring Overview | KubeStash
description: A general overview of monitoring KubeStash
menu:
  docs_{{ .version }}:
    identifier: monitoring-overview
    name: Overview
    parent: monitoring
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# Monitoring KubeStash

KubeStash has native support for monitoring via [Prometheus](https://prometheus.io/). You can use builtin [Prometheus](https://github.com/prometheus/prometheus) scraper or [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator) to monitor KubeStash. This tutorial will show you how Prometheus monitoring works with KubeStash, what metrics KubeStash exports, and how to enable monitoring.

## How Prometheus monitoring works

KubeStash monitoring metrics comes from [Panopticon](https://blog.byte.builders/post/introducing-panopticon/) which is a generic state metric exporter for Kubernetes developed by AppsCode. It watches KubeStash CRDs and export necessary metrics.

The following diagram shows the logical structure of the KubeStash monitoring flow.

<figure align="center">
  <img alt="Stash Monitoring Flow" src="/docs/guides/monitoring/overview/images/monitoring-structure.svg">
<figcaption align="center">Fig: Monitoring process in KubeStash</figcaption>
</figure>

The Panopticon tool runs as a separate workload. It watches for KubeStash CRDs and exports relevant metrics.

## Available Metrics

Panopticon watches the BackupConfiguration, BackupSession, BackupStorage, Repository, RestoreSession, Snapshot CRDs and export necessary metrics. This section will list the metrics exported by KubeStash for different CRDs.

**BackupConfiguration Metrics:**

| Metric Name                                         | Usage                                                                        |
|-----------------------------------------------------|------------------------------------------------------------------------------|
| `core_kubestash_com_backupconfiguration_created`    | Unix creation timestamp of this BackupConfiguration object                   |
| `core_kubestash_com_backupconfiguration_info`       | General information about this BackupConfiguration                           |
| `core_kubestash_com_backupconfiguration_sessions`   | List of sessions of this BackupConfiguration                                 |
| `core_kubestash_com_backupconfiguration_phase`      | BackupConfiguration object current phase                                     |
| `core_kubestash_com_backupconfiguration_conditions` | Current conditions of this BackupConfiguration                               |

**BackupSession Metrics:**

| Metric Name                                         | Usage                                                         |
|-----------------------------------------------------|---------------------------------------------------------------|
| `core_kubestash_com_backupsession_created`          | Unix creation timestamp of this BackupSession object          |
| `core_kubestash_com_backupsession_info`             | General information about this BackupSession                  | 
| `core_kubestash_com_backupsession_snapshots`        | General information about the snapshots of this BackupSession | 
| `core_kubestash_com_backupsession_phase`            | BackupSession object current phase                            | 
| `core_kubestash_com_backupsession_duration_seconds` | Time required to complete this backup process                 |
| `core_kubestash_com_backupsession_conditions`       | Current conditions of this BackupSession                      |


**BackupStorage Metrics:**

| Metric Name                                                 | Usage                                                        | 
|-------------------------------------------------------------|--------------------------------------------------------------|
| `storage_kubestash_com_backupstorage_created`               | Unix creation timestamp of this BackupStorage object         | 
| `storage_kubestash_com_backupstorage_info`                  | General information about this BackupStorage                 | 
| `storage_kubestash_com_backupstorage_repository_size_bytes` | Total backed up data in the repository of this BackupStorage | 
| `storage_kubestash_com_backupstorage_phase`                 | BackupStorage object current phase                           | 
| `storage_kubestash_com_backupstorage_size_bytes`            | Total backed up data size in this BackupStorage              | 
| `storage_kubestash_com_backupstorage_conditions`            | Current conditions of this BackupStorage                     |

**Repository Metrics:**

| Metric Name                                               | Usage                                                 | 
|-----------------------------------------------------------|-------------------------------------------------------| 
| `storage_kubestash_com_repository_created`                | Unix creation timestamp of this Repository object     |
| `storage_kubestash_com_repository_info`                   | General information about this Repository             |
| `storage_kubestash_com_repository_last_successful_backup` | Last successful backup stored in this Repository      | 
| `storage_kubestash_com_repository_size_bytes`             | Total backed up data stored in this Repository        | 
| `storage_kubestash_com_repository_snapshot_count_total`   | Number of current snapshots stored in this Repository |
| `storage_kubestash_com_repository_conditions`             | Current conditions of this Repository                 |

**RestoreSession Metrics:**

| Metric Name                                | Usage                                                  | 
|--------------------------------------------|--------------------------------------------------------|
| `core_kubestash_com_created`               | Unix creation timestamp of this RestoreSession object  |
| `core_kubestash_com_info`                  | General information about this RestoreSession          |
| `core_kubestash_com_phase`                 | RestoreSession object current phase                    |
| `core_kubestash_com_duration_seconds`      | The total time taken to complete the restore process   |
| `core_kubestash_com_component_count_total` | The number of total components for this RestoreSession |
| `core_kubestash_com_conditions`            | Current conditions of this RestoreSession              |

**Snapshot Metrics:**

| Metric Name                                            | Usage                                            |
|--------------------------------------------------------|--------------------------------------------------| 
| `storage_kubestash_com_snapshot_created`               | Unix creation timestamp of this Snapshot object  |
| `storage_kubestash_com_snapshot_info`                  | General information about this Snapshot          | 
| `storage_kubestash_com_snapshot_phase`                 | Snapshot object current phase                    | 
| `storage_kubestash_com_snapshot_size_bytes`            | Size of this Snapshot                            | 
| `storage_kubestash_com_snapshot_time_seconds`          | The time when this Snapshot was taken            | 
| `storage_kubestash_com_snapshot_component_count_total` | The number of total components for this Snapshot | 
| `storage_kubestash_com_snapshot_conditions`            | Current conditions of this Snapshot              | 

## How to Enable Monitoring

During the installation of KubeStash, all the necessary `MetricsConfigurations` are created. You can find these `MetricsConfigurations` using the following command:

```bash
$ kubectl get metricsconfigurations
NAME                                         APIVERSION                       KIND                  AGE
kubestash-appscode-com-backupconfiguration   core.kubestash.com/v1alpha1      BackupConfiguration   113m
kubestash-appscode-com-backupsession         core.kubestash.com/v1alpha1      BackupSession         113m
kubestash-appscode-com-backupstorage         storage.kubestash.com/v1alpha1   BackupStorage         113m
kubestash-appscode-com-repository            storage.kubestash.com/v1alpha1   Repository            113m
kubestash-appscode-com-restoresession        core.kubestash.com/v1alpha1      RestoreSession        113m
kubestash-appscode-com-snapshot              storage.kubestash.com/v1alpha1   Snapshot              113m
```

Next, you need to install `Panopticon`. You can monitor KubeStash by using either the built-in [Prometheus](https://github.com/prometheus/prometheus) scraper or [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator).