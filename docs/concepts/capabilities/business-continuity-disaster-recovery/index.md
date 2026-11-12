---
title: Business Continuity and Disaster Recovery
description: Business Continuity Plan (BCP) and Disaster Recovery (DR) with online DR readiness.
menu:
  docs_{{ .version }}:
    identifier: capabilities-business-continuity-disaster-recovery
    name: Business Continuity and Disaster Recovery
    parent: capabilities
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# Business Continuity and Disaster Recovery

KubeStash provides a complete Business Continuity Plan (BCP) and Disaster Recovery (DR) solution with online DR readiness for Kubernetes workloads and databases. `BackupBlueprint` standardizes how every workload is protected, `RetentionPolicy` enforces the long-term retention windows that BCP frameworks require, and `BackupVerifier` proves recoverability on a schedule. A `RestoreSession` can target a separate cluster, so the DR site is online and ready to be promoted at any time. Combined with engine-level high availability from KubeDB, the result is an always-ready DR posture with defined RPO and RTO.

## Related concepts

- [BackupConfiguration](../../crds/backupconfiguration/)
- [BackupBlueprint](../../crds/backupblueprint/)
- [RestoreSession](../../crds/restoresession/)
- [RetentionPolicy](../../crds/retentionpolicy/)
- [BackupVerifier](../../crds/backupverifier/)
