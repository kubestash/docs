---
title: Complete Transaction History with Point-in-Time Recovery
description: Complete transaction history database with point-in-time recovery (PITR) to any second.
menu:
  docs_{{ .version }}:
    identifier: capabilities-complete-transaction-history-pitr
    name: Complete Transaction History with Point-in-Time Recovery
    parent: capabilities
    weight: 60
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# Complete Transaction History with Point-in-Time Recovery

KubeStash maintains a complete transaction history database for supported engines (PostgreSQL, MySQL, MongoDB, and equivalents) by continuously archiving WAL, binlog, and engine-equivalent transaction logs. Combined with periodic base snapshots, this enables Point-in-Time Recovery (PITR) to any second within the retention window. The transaction log archive itself is the complete, queryable transaction history of record, which satisfies audit and regulatory requirements for transaction-level traceability without standing up a separate CDC pipeline.

## Related concepts

- [BackupConfiguration](../../crds/backupconfiguration/)
- [BackupBlueprint](../../crds/backupblueprint/)
- [RestoreSession](../../crds/restoresession/)
- [RetentionPolicy](../../crds/retentionpolicy/)
- [BackupVerifier](../../crds/backupverifier/)
