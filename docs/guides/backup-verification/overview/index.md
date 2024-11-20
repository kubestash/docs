---
title: Backup Verification Overview | KubeStash
description: A overview of how backup verification works in KubeStash
menu:
  docs_{{ .version }}:
    identifier: verification-overview
    name: Overview
    parent: verification
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# KubeStash Backup Verification

KubeStash supports verifying backups using different types of verification strategies.

## Before You Begin

- You should be familiar with the following `KubeStash` concepts:
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupVerifier](/docs/concepts/crds/backupverifier/index.md)
  - [BackupVerificationSession](/docs/concepts/crds/backupverificationsession/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [Repository](/docs/concepts/crds/repository/index.md)

## How Backup Verification Process Works

The following diagram shows how KubeStash verifies backups. Open the image in a new tab to see the enlarged version.

<figure align="center">
   <img alt="KubeStash Backup Verification Flow" src="images/backup_verification_overview.svg">
    <figcaption align="center">Fig: Backup Verification process in KubeStash</figcaption>
</figure>

The backup process consists of the following steps:

1. At first, a user creates a `Secret`. This secret holds the credentials to access the backend where the backed up data will be stored.

2. Then, she creates a `BackupStorage` custom resource that specifies the backend information, along with the `Secret` containing the credentials needed to access the backend.

3. KubeStash operator watches for `BackupStorage` custom resources. When it finds a `BackupStorage` object, it initializes the `BackupStorage` by uploading the `metadata.yaml` file into the target storage.

4. Then, she creates a `BackupConfiguration` custom resource that specifies the targeted application, the Addon info with a specified task, etc. It also provides information about one or more repositories, each indicating a path and a `BackupStorage` for storing the backed-up data. Each repository refers to `BackupVerifier` which contains the configuration for verification.

5. KubeStash operator watches for `BackupConfiguration` objects.

6. Once the KubeStash operator finds a `BackupConfiguration` object, it creates `Repository` with the information specified in the `BackupConfiguration`.

7. KubeStash operator watches for `Repository` custom resources. When it finds the `Repository` object, it Initializes `Repository` by uploading `repository.yaml` file into the `spec.sessions[*].repositories[*].directory` path specified in `BackupConfiguration`.

8. Then, it creates a `CronJob` for each repository referring to a `BackupVerifier` with the schedule specified in `BackupVerifier` to trigger backup verification periodically.

9. On the next scheduled slot, the `CronJob` triggers a backup verification by creating a `BackupVerificationSession` custom resource.

10. KubeStash operator watches for `BackupVerificationSession` custom resources.

11. When it finds a `BackupVerificationSession` object, it creates a `RestoreSession` custom resource referring to the latest `Snapshot` as datasource.

12. Then it resolves the respective `Addon` and `Function` and prepares restore `Job`(s) definition.

13. After a successful restore, it creates a verifier `Job` depending on the verification type. 

14. The `Job` runs queries or script to verify backup.

15. After the backup verification is completed, the backup verifier `Job` updates the status field of `BackupVerificationSession`.

## Types of Verification Strategies

Currently, we have the following types of backup verification strategies:

- **RestoreOnly :** KubeStash operator will initiate a `RestoreSession` using the addon information specified in `BackupVerifier`. The verification of the backup will rely on the status of the `RestoreSession` phase; if the restore completes successfully, the backup is considered verified.
- **Query :** At first, KubeStash operator will initiate a `RestoreSession` and after successful restore, it will create a verifier job to run the queries provided in `BackupVerifier`.
- **Script :** At first, KubeStash operator will initiate a `RestoreSession` and after successful restore, it will create a verifier job to run the script provided in `BackupVerifier`.

