---
title: GCS Backend | KubeStash
description: Configure KubeStash to use Google Cloud Storage (GCS) as Backend.
menu:
  docs_{{ .version }}:
    identifier: backend-gcs
    name: Google Cloud Storage
    parent: backend
    weight: 50
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Google Cloud Storage (GCS)

KubeStash supports [Google Cloud Storage(GCS)](https://cloud.google.com/storage/) as a backend. This tutorial will show you how to use this backend.

In order to use Google Cloud Storage as backend, you have to create a `Secret` and a `BackupStorage` object pointing to the desired GCS bucket.

> If the bucket already exists, the Google Cloud service account you provide to KubeStash only needs `Storage Object Creator` role permission. However, if the bucket does not exist, KubeStash  will create the bucket during the first backup. In this case, the Google Cloud service account key used for KubeStash must have `Storage Object Admin` role permission. To avoid giving this elevated level of permission to KubeStash, create the bucket manually (either from GCP console or gcloud cli) ahead of time.

#### Create Storage Secret

To configure storage secret for this backend, following secret keys are needed:

|                Key                |    Type    |                         Description                         |
| --------------------------------- | ---------- | ----------------------------------------------------------- |
| `GOOGLE_PROJECT_ID`               | `Required` | Google Cloud project ID.                                    |
| `GOOGLE_SERVICE_ACCOUNT_JSON_KEY` | `Required` | Google Cloud service account json key.                      |

Create storage secret as below,

```bash
$ echo -n '<your-project-id>' > GOOGLE_PROJECT_ID
$ mv downloaded-sa-json.key GOOGLE_SERVICE_ACCOUNT_JSON_KEY
$ kubectl create secret generic -n demo gcs-secret \
    --from-file=./GOOGLE_PROJECT_ID \
    --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
secret/gcs-secret created
```

### Create BackupStorage

Now, you have to create a `BackupStorage` crd. You have to provide the storage secret that we have created earlier in `spec.storage.gcs.SecretName` field.

Following parameters are available for `gcs` backend.

| Parameter            |    Type    | Description                                                                                                                                                                                                                                            |
|----------------------| ---------- |--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `gcs.bucket`         | `Required` | Name of Bucket. If the bucket does not exist yet, it will be created in the default location (US). It is not possible at the moment for KubeStash to create a new bucket in a different location, so you need to create it using Google cloud console. |
| `gcs.secretName`     | `Required` | Specify the name of the Secret that contains the access credential for this storage.                                                                                                                                                                   |
| `gcs.prefix`         | `Optional` | Path prefix inside the bucket where backed up data will be stored.                                                                                                                                                                                     |
| `gcs.maxConnections` | `Optional` | Maximum number of parallel connections to use for uploading backup data. By default, KubeStash will use maximum 5 parallel connections.                                                                                                                |

Below, the YAML of a sample `BackupStorage` crd that uses a GCS bucket as a backend.

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: BackupStorage
metadata:
  name: gcs-storage
  namespace: demo
spec:
  storage:
    provider: gcs
    gcs:
      bucket: kubestash-demo
      prefix: demo
      secretName: gcs-secret
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true # Use this BackupStorage as the default
  deletionPolicy: WipeOut # One of: WipeOut, Delete
```

Create the `BackupStorage` we have shown above using the following command,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/backends/gcs/examples/gcs.yaml
backupstorage.storage.kubestash.com/gcs-storaqge created

```

Now, we are ready to use this backend to backup our desired data using KubeStash.

## Next Steps

- Learn how to use KubeStash to backup workloads data from [here](/docs/guides/workloads/overview/index.md).
- Learn how to use KubeStash to backup stand-alone PVC from [here](/docs/guides/volumes/overview/index.md).
