---
title: Non-Disruptive Restoration
description: Restore data without impacting application performance or response time.
menu:
  docs_{{ .version }}:
    identifier: capabilities-non-disruptive-restoration
    name: Non-Disruptive Restoration
    parent: capabilities
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# Non-Disruptive Restoration

KubeStash restores data without impacting application performance or response time. A `RestoreSession` does not have to overwrite the source. The default and recommended pattern is to restore into an alternate namespace, PVC, or even a separate cluster, validate integrity, and only then cut traffic over. Because the live production workload is never touched during recovery, response time and throughput remain at steady state. `BackupVerifier` runs scheduled, isolated restore drills against every backup, providing continuous evidence that any snapshot is fully recoverable, again without disturbing production.

## Related concepts

- [BackupConfiguration](../../crds/backupconfiguration/)
- [BackupBlueprint](../../crds/backupblueprint/)
- [RestoreSession](../../crds/restoresession/)
- [RetentionPolicy](../../crds/retentionpolicy/)
- [BackupVerifier](../../crds/backupverifier/)
