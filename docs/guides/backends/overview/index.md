---
title: Backend Overview | KubeStash
description: An overview of the backends used by KubeStash to store backed up data.
menu:
  docs_{{ .version }}:
    identifier: backend-overview
    name: What is Backend?
    parent: backend
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# KubeStash Backends

## BackupStorage

KubeStash supports various backends for use as a BackupStorage. It can be a cloud storage like GCS bucket, AWS S3, Azure Blob Storage etc. or a Kubernetes persistent volume like [HostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath), [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/volumes/#persistentvolumeclaim), [NFS](https://kubernetes.io/docs/concepts/storage/volumes/#nfs) etc.

The following diagram shows how kubestash operator and backup container accesses and backs up data into a backend.

<figure align="center">
	<img alt="KubeStash Backend Overview" src="images/kubestash_backend_overview.svg">
  <figcaption align="center">Fig: KubeStash Backend Overview</figcaption>
</figure>

The Backend process works in the following steps:

- At first user creates a [BackupStorage](/docs/concepts/crds/backupstorage/index.md) object that contains the backend information along with a `Secret` object containing the corresponding backend credentials required for accessing the backend.
- KubeStash operator watches for `BackupStorage` custom resources and `Secrets`. When it finds a `BackupStorage` object, it establishes a connection with the storage by uploading the `metadata.yaml` file into the target storage.

Below, a screenshot that shows initialization of a `BackupStorage` in a GCS bucket named `kubestash-qa`:

<figure align="center">
  <img alt="BackupStorage initialization in GCS Backend" src="./images/backupstorage-initialize.png">
  <figcaption align="center">Fig: BackupStorage initialization in GCS Backend</figcaption>
</figure>

Here, `kubestash-qa` serves as the bucket name, and the presence of `metadata.yaml` indicates the successful initialization of the BackupStorage.

## Repository

- Once the `BackupStorage` is initialized and in ready phase then the next steps is creating [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md).
- User creates a `BackupConfiguration` custom resource. The `BackupConfiguration` object specifies the `Repository` pointing to a `BackupStorage` that contains backend information, indicating where to upload backup data. Multiple `Repository` objects can refer to a single `BackupStorage`.
- Then KubeStash operator retrieves `Repository` information from it and create `Repository` object. This Repository object serves as a container for effectively managing and storing the backup data.
- KubeStash operator triggers a `Backup` job according to the schedule defined in the `BackupConfiguration`. This job accesses the `BackupStorage` object and `Secret` and uploads backup data into the storage.

Below, a screenshot the shows a `Repository` located in the directory path `kubestash-qa/demo/monodb-backup/`, and the presence of `repository.yaml` indicates the successful initialization of the Repository.

<figure align="center">
  <img alt="Repository under a BackupStorage" src="./images/gcs_repository.png">
  <figcaption align="center">Fig: Repository under a BackupStorage</figcaption>
</figure>


## Next Steps
- Learn how to configure `Kubernetes Volume` as backend from [here](/docs/guides/backends/local/index.md).
- Learn how to configure `AWS S3/Minio/Rook` backend from [here](/docs/guides/backends/s3/index.md).
- Learn how to configure `Google Cloud Storage (GCS)` backend from [here](/docs/guides/backends/gcs/index.md).
- Learn how to configure `Microsoft Azure Storage` backend from [here](/docs/guides/backends/azure/index.md).
