---
title: Comprehensively Documented Backup and Restoration Procedures
description: All backup and restoration procedures comprehensively documented through declarative CRDs.
menu:
  docs_{{ .version }}:
    identifier: capabilities-documented-backup-restoration-procedures
    name: Comprehensively Documented Backup and Restoration Procedures
    parent: capabilities
    weight: 70
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# Comprehensively Documented Backup and Restoration Procedures

KubeStash documents all backup and restoration procedures comprehensively, by collapsing procedure and configuration into the same artifact: a Kubernetes Custom Resource Definition (CRD). Every backup schedule, restore plan, retention rule, pre and post hook, verification cadence, and disaster recovery drill is expressed as a `BackupConfiguration`, `RestoreSession`, `RetentionPolicy`, `HookTemplate`, `BackupVerifier`, or `BackupBlueprint` object. These YAML manifests live in Git, are reviewed in pull requests, and are applied via GitOps. The documentation is the configuration, which means it cannot drift from reality.

## Related concepts

- [BackupConfiguration](../../crds/backupconfiguration/)
- [BackupBlueprint](../../crds/backupblueprint/)
- [RestoreSession](../../crds/restoresession/)
- [RetentionPolicy](../../crds/retentionpolicy/)
- [BackupVerifier](../../crds/backupverifier/)
