---
title: Online, Offline, and End-of-Day Backup
description: Online, offline, and end-of-day backup from one declarative API.
menu:
  docs_{{ .version }}:
    identifier: capabilities-online-offline-end-of-day-backup
    name: Online, Offline, and End-of-Day Backup
    parent: capabilities
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# Online, Offline, and End-of-Day Backup

KubeStash provides comprehensive online, offline, and end-of-day backup capabilities for Kubernetes workloads, databases, and persistent volumes. A single `BackupConfiguration` can run hot online captures throughout the business day for low-RPO recovery, plus an end-of-day backup at a fixed cron expression for daily checkpoints. Snapshots are written to one or more `BackupStorage` targets, including immutable / WORM-compliant tiers such as S3 Object Lock, Azure Blob immutability, GCS bucket lock, and MinIO WORM, which makes the backup effectively offline and tamper-proof even though the storage itself remains online for fast recovery.

## Related concepts

- [BackupConfiguration](../../crds/backupconfiguration/)
- [BackupBlueprint](../../crds/backupblueprint/)
- [RestoreSession](../../crds/restoresession/)
- [RetentionPolicy](../../crds/retentionpolicy/)
- [BackupVerifier](../../crds/backupverifier/)
