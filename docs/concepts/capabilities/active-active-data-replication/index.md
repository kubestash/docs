---
title: Active-Active Data Replication between Primary and DR Sites
description: Real-time, secure data replication between primary DC and DR sites in active-active mode.
menu:
  docs_{{ .version }}:
    identifier: capabilities-active-active-data-replication
    name: Active-Active Data Replication between Primary and DR Sites
    parent: capabilities
    weight: 40
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# Active-Active Data Replication between Primary and DR Sites

KubeStash supports real-time and secure data replication between the primary data center (DC) and the disaster recovery (DR) site operating in active-active mode. Backup snapshots are replicated across multiple `BackupStorage` targets simultaneously, so the same logical snapshot is materialized in DC, DR, and any Near-DR site. For the live data path, KubeStash works together with KubeDB, which provides engine-native synchronous multi-master replication (MySQL Group Replication, Galera, MongoDB cross-region replica sets, PostgreSQL Patroni stretched clusters). Both sites remain authoritative and writable, with RPO near zero. Data in transit is encrypted end to end.

## Related concepts

- [BackupConfiguration](../../crds/backupconfiguration/)
- [BackupBlueprint](../../crds/backupblueprint/)
- [RestoreSession](../../crds/restoresession/)
- [RetentionPolicy](../../crds/retentionpolicy/)
- [BackupVerifier](../../crds/backupverifier/)
