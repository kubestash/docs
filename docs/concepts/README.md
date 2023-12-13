---
title: Concepts | KubeStash
menu:
  docs_{{ .version }}:
    identifier: concepts-readme
    name: README
    parent: concepts
    weight: -1
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
url: /docs/{{ .version }}/concepts/
aliases:
  - /docs/{{ .version }}/concepts/README/
---

# Concepts

Concepts help you to learn about the different parts of the KubeStash and the abstractions it uses.

This concept section is divided into the following modules:

## What is KubeStash?
- [Overview](/docs/concepts/what-is-kubestash/overview/index.md) provides an introduction to KubeStash. It also gives an overview of the features it provides.
- [Architecture](/docs/concepts/what-is-kubestash/architecture/index.md) provides a visual representation of KubeStash architecture. It also provides a brief overview of the components it uses.

## Declarative API
- [BackupStorage](/docs/concepts/crds/backupstorage/index.md) introduces the concept of `BackupStorage` crd that holds the backend information in a Kubernetes native way where the backed up data of different applications will be stored.
- [Repository](/docs/concepts/crds/repository/index.md) introduces the concept of `Repository` crd that holds information about the targeted application that has been backed up and the `BackupStorage` where the backed up data is being stored.
- [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md) introduces the concept of `BackupConfiguration` crd that is used to configure backup for a target application in a Kubernetes native way.
- [BackupSession](/docs/concepts/crds/backupsession/index.md) introduces the concept of `BackupSession` crd that represents a backup run of a target application for the respective `BackupConfiguration` object.
- [RestoreSession](/docs/concepts/crds/restoresession/index.md) introduces the concept of `RestoreSession` crd that represents a restore run of a target application.
- [HookTemplate](/docs/concepts/crds/hooktemplate/index.md) introduces the concept of `HookTemplate` crd that represents a template for some action that will be executed before or/and after backup/restore process.
- [Addon](/docs/concepts/crds/addon/index.md) introduces the concept of `Addon` crd which represents the backup and restore tasks that can be performed by a particular addon.
- [Function](/docs/concepts/crds/function/index.md) introduces the concept of `Function` crd that represents a step of a backup or restore process or any action.
- [BackupBlueprint](/docs/concepts/crds/backupblueprint/index.md) introduces the concept of `BackupBlueprint` crd that represents a blueprint for `BackupConfiguration` object which provides an option to share `BackupConfiguration` across similar targets.
- [Snapshot](/docs/concepts/crds/snapshot/index.md) introduces the concept of `Snapshot` crd that represents the state of a backup run to a particular `Repository`.
- [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md) introduces the concept of `RetentionPolicy` crd that represents how the old `Snapshots` should be cleaned up.
