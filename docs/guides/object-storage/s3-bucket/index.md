---
title: Backup & Restore of S3 Object Storage | KubeStash
description: A comprehensive guide showing how to backup and restore data from S3 object storage buckets using KubeStash.
menu:
  docs_{{ .version }}:
    identifier: s3-bucket
    name: S3 Object Storage
    parent: object-storage
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup and Restore of S3 Object Storage

This tutorial demonstrates how to use KubeStash to backup and restore data from S3 object storage buckets. Many cloud providers offer S3-compatible object storage solutions, such as DigitalOcean Spaces, MinIO, Wasabi, Linode Object Storage, and others. This guide shows you how to:

1. Mount an S3 bucket prefix as a PersistentVolume using a CSI driver
2. Backup the data from the mounted volume to another cloud object storage backend using KubeStash
3. Restore the data from backup when needed

## Overview

In this guide, we will use DigitalOcean Spaces (an S3 object storage) as our source data storage. We'll mount a bucket prefix as a Kubernetes volume, then backup its contents to a different object storage backend. The same approach works with any S3-compatible storage provider.

## Before You Begin

- You need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).

- You need access to an S3 object storage service with:
    - Access credentials (Access Key ID and Secret Access Key)
    - An existing bucket
    - The endpoint URL and region information

- You should be familiar with the following `KubeStash` concepts:
    - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
    - [BackupSession](/docs/concepts/crds/backupsession/index.md)
    - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
    - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
    - [Snapshot](/docs/concepts/crds/snapshot/index.md)
    - [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/object-storage/s3--bucket/examples](https://github.com/kubestash/docs/tree/{{< param "info.version" >}}/docs/guides/object-storage/s3-compatible-bucket/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.
