---
title: Seamless Source/Target Switchover
description: Seamless source/target role switching with minimal performance impact.
menu:
  docs_{{ .version }}:
    identifier: capabilities-seamless-source-target-switchover
    name: Seamless Source/Target Switchover
    parent: capabilities
    weight: 50
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# Seamless Source/Target Switchover

KubeStash supports seamless source/target role switching with minimal performance impact. Switching primary and secondary roles between DC and DR is a coordinated act across the data layer, the control plane, and the traffic plane. KubeDB owns intra-cluster failover for supported database engines. KubeStash provides the verified, restore-tested data layer underneath: `BackupVerifier` proves the DR copy is restorable before the switch, and a `RestoreSession` can promote DR data on demand. The result is a smooth, scripted switchover with minimal performance impact on either side.

## Related concepts

- [BackupConfiguration](../../crds/backupconfiguration/)
- [BackupBlueprint](../../crds/backupblueprint/)
- [RestoreSession](../../crds/restoresession/)
- [RetentionPolicy](../../crds/retentionpolicy/)
- [BackupVerifier](../../crds/backupverifier/)
