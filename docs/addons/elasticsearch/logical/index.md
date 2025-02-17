---
title: Backup & Restore Elasticsearch | KubeStash
description: Backup an externally managed Elasticsearch database using KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-elasticsearch-logical-backup
    name: Elasticsearch Database Backup
    parent: kubestash-elasticsearch
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
---

{{< notice type="warning" message="If you are using a KubeDB-managed Elasticsearch database, please refer to the following [guide](https://kubedb.com/docs/latest/guides/elasticsearch/backup/kubestash/overview/). This guide covers backup and restore procedures for externally managed Elasticsearch databases." >}}

# Backup and Restore Elasticsearch database using KubeStash

KubeStash allows you to backup and restore `Elasticsearch` databases. KubeStash makes managing your `Elasticsearch` backups and restorations more straightforward and efficient.

This guide will give you how you can take backup and restore your externally managed `Elasticsearch` databases using `Kubestash`.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using `Minikube` or `Kind`.
- Install `kubedb-kubestash-catalog` in your cluster following the steps [here](https://github.com/kubedb/installer/tree/master/charts/kubedb-kubestash-catalog).
- Install `KubeStash` in your cluster following the steps [here](https://kubestash.com/docs/latest/setup/install/kubestash).
- Install KubeStash `kubectl` plugin following the steps [here](https://kubestash.com/docs/latest/setup/install/kubectl-plugin/).
- If you are not familiar with how KubeStash backup and restore Elasticsearch databases, please check the following guide [here](/docs/addons/elasticsearch/overview/index.md).

You should be familiar with the following `KubeStash` concepts:

- [BackupStorage](https://kubestash.com/docs/latest/concepts/crds/backupstorage/)
- [BackupConfiguration](https://kubestash.com/docs/latest/concepts/crds/backupconfiguration/)
- [AppBinding](https://kubedb.com/docs/v2024.9.30/guides/elasticsearch/concepts/appbinding/)
- [BackupSession](https://kubestash.com/docs/latest/concepts/crds/backupsession/)
- [RestoreSession](https://kubestash.com/docs/latest/concepts/crds/restoresession/)
- [Addon](https://kubestash.com/docs/latest/concepts/crds/addon/)
- [Function](https://kubestash.com/docs/latest/concepts/crds/function/)
- [Task](https://kubestash.com/docs/latest/concepts/crds/addon/#task-specification)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/addons/elasticsearch/logical/examples](/docs/addons/elasticsearch/logical/examples) directory of [kubedb/docs](https://github.com/kubedb/docs) repository.
## Backup Elasticsearch


In this demonstration, we’ll focus on a DigitalOcean-managed `Elasticsearch`.

### Create a Sample Elasticsearch Database

Let's create a sample `Elasticsearch` database in DigitalOcean and insert some data into it.

<figure align="center">
  <img alt="Sample database in DigitalOcean" src="/docs/addons/elasticsearch/logical/images/sample-elasticsearch-database.png">
  <figcaption align="center">Fig: Sample database in DigitalOcean</figcaption>
</figure>

Here’s what we’ve done so far:
- Created a sample `Elasticsearch` database named `kubestash-test`.
- The image also displays the necessary connection details for this database.

**Create Secret:**

Now, create a `Secret` that contains the authentication username and password.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-auth-secret
  namespace: demo
type: Opaque
stringData:
  username: doadmin
  password: ""
```
**Create AppBinding:**
Next, we need to manually create an `AppBinding` custom resource (CR) in the same namespace as the database secret. This `AppBinding` will contain the necessary connection details for the database we created earlier.

```yaml
apiVersion: appcatalog.appscode.com/v1alpha1
kind: AppBinding
metadata:
  name: elasticsearch-appbinding
  namespace: demo
spec:
  clientConfig:
    url: https://kubestash-es-test-do-user-165729-0.k.db.ondigitalocean.com:25060
  secret:
    name: elasticsearch-auth-secret
  type: elasticsearch
  version: "7.14.0"
```

Here,
- `.spec.clientConfig.url` Specifies the connection URL for the target database. You can construct the URL as follows:
    - `<connection scheme>://<host>:<port>`
- `.spec.secret` Specifies the name of the secret containing the authentication credentials. In this case, we’ll use the secret we created earlier.
- `.spec.version` Specifies the version of targeted database.

**Insert Sample Data:**

Now, connect to the database using the `elasticsearch` client. Once connected, create a new database and table, then insert some sample data into it.

```bash
$ curl -XPOST -k --user 'doadmin:<DB USER PASSWORD>' "https://kubestash-es-test-do-user-165729-0.k.db.ondigitalocean.com:25060/info/_doc?pretty" -H 'Content-Type: application/json' -d'
                                                   {
                                                       "Company": "AppsCode Inc",
                                                       "Product": "KubeDB"
                                                   }
                                                   '
{
  "_index" : "info",
  "_id" : "MUId0JQBGo7wpKKgemm8",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
$ curl -XGET -k --user 'doadmin:<YOUR USER PASSWORD>' "https://kubestash-es-test-do-user-165729-0.k.db.ondigitalocean.com:25060/_cat/indices?v&s=index&pretty"
health status index                     uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana_1                 qLaowo1nQCy9Ou-z969wdQ   1   0          0            0       208b           208b
green  open   .opendistro_security      VuAf3AVYS_WibezmyTwI0w   1   0         10            2     65.2kb         65.2kb
green  open   .opensearch-observability N1sihK-SSaODBsGq_D38oQ   1   0          0            0       208b           208b
green  open   .plugins-ml-config        y5MS3LMlR42wr66b4UpXgA   1   0          1            0      3.9kb          3.9kb
green  open   info                      Kj6hmC5bTgGN9faoHXof_Q   1   0          1            0      4.5kb          4.5kb
```

Now, we are ready to backup the database.


### Prepare Backend

We are going to store our backed up data into a GCS bucket. We have to create a Secret with necessary credentials and a `BackupStorage` CR to use this backend. If you want to use a different backend, please read the respective backend configuration doc from [here](https://kubestash.com/docs/latest/guides/backends/overview/).

**Create Secret:**

Let's create a secret called `gcs-secret` with access credentials to our desired GCS bucket,

```bash
$ echo -n '<your-project-id>' > GOOGLE_PROJECT_ID
$ cat /path/to/downloaded-sa-key.json > GOOGLE_SERVICE_ACCOUNT_JSON_KEY
$ kubectl create secret generic -n demo gcs-secret \
    --from-file=./GOOGLE_PROJECT_ID \
    --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
secret/gcs-secret created
```

**Create BackupStorage:**

Now, create a `BackupStorage` using this secret. Below is the YAML of `BackupStorage` CR we are going to create,

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
      bucket: kubestash-qa
      prefix: demo
      secretName: gcs-secret
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true
  deletionPolicy: Delete
```

Let's create the BackupStorage we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/elasticsearch/logical/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-storage created
```

Now, we are ready to backup our database to our desired backend.

**Create RetentionPolicy:**

Now, let's create a `RetentionPolicy` to specify how the old Snapshots should be cleaned up.

Below is the YAML of the `RetentionPolicy` object that we are going to create,

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: RetentionPolicy
metadata:
  name: demo-retention
  namespace: demo
spec:
  default: true
  failedSnapshots:
    last: 2
  maxRetentionPeriod: 2mo
  successfulSnapshots:
    last: 5
  usagePolicy:
    allowedNamespaces:
      from: All
```

Let’s create the above `RetentionPolicy`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/elasticsearch/logical/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

### Backup

We have to create a `BackupConfiguration` targeting the respective `elasticsearch-appbinding` AppBinding custom resource. This AppBinding resource contains all necessary connection information for the target Elasticsearch database. Then, KubeStash will create a `CronJob` for each session to take periodic backup of that database.

At first, we need to create a secret with a Restic password for backup data encryption.

Let's create a secret called `encrypt-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD
secret "encrypt-secret" created
```

**Create BackupConfiguration:**

Below is the YAML for `BackupConfiguration` CR to backup the `kubestash-test` externally managed Elasticsearch database that we have created earlier,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: elasticsearch-db-backup
  namespace: demo
spec:
  target:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: elasticsearch-appbinding
    namespace: demo
  backends:
    - name: gcs-backend
      storageRef:
        namespace: demo
        name: gcs-storage
      retentionPolicy:
        name: demo-retention
        namespace: demo
  sessions:
    - name: frequent-backup
      sessionHistoryLimit: 3
      scheduler:
        schedule: "*/5 * * * *"
        successfulJobsHistoryLimit: 1
        failedJobsHistoryLimit: 1
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: gcs-elasticsearch-repo
          backend: gcs-backend
          directory: /elasticsearch
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: elasticsearch-addon
        tasks:
          - name: logical-backup
            params:
              args: --match=^(?![.])(?!security-auditlog).+
```

- `.spec.sessions[*].schedule` specifies that we want to backup the database at `5 minutes` interval.
- `.spec.target` refers to the `elasticsearch-appbinding` AppBinding custom resource, Which contains all necessary connection information for the target Elasticsearch database.
- `.spec.sessions[].addon.tasks[].params.args` refers that backup configuration will take backup of all the indices except the indices starts with `.` and `security-auditlog`.

Let's create the `BackupConfiguration` CR that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/elasticsearch/logical/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/elasticsearch-db-backup created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the BackupConfiguration,

```bash
$ kubectl get bc -n demo 
NAME                      PHASE   PAUSED   AGE
elasticsearch-db-backup   Ready            13m
```

Additionally, we can verify that the `Repository` specified in the `BackupConfiguration` has been created using the following command,

```bash
$ kubectl get repo -n demo
NAME                     INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-elasticsearch-repo   true        5                6.021 KiB   Ready   2m11s                    16m
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the `Repository` YAML stored in the `demo/elasticsearch` directory.

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` CR.

Verify that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                              SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-elasticsearch-db-backup-frequent-backup   */2 * * * *   <none>     False     0        38s             12m
```

**Verify BackupSession:**

KubeStash triggers an instant backup as soon as the `BackupConfiguration` is ready. After that, backups are scheduled according to the specified schedule.

```bash
$ kubectl get backupsession -n demo
NAME                                                 INVOKER-TYPE          INVOKER-NAME              PHASE       DURATION   AGE
elasticsearch-db-backup-frequent-backup-1738660560   BackupConfiguration   elasticsearch-db-backup   Succeeded   56s        75s
```

We can see from the above output that the backup session has succeeded. Now, we are going to verify whether the backed up data has been stored in the backend.

**Verify Backup:**

Once a backup is complete, KubeStash will update the respective `Repository` CR to reflect the backup. Check that the repository `sample-elasticsearch-backup` has been updated by the following command,

```bash
$ kubectl get repo -n demo
NAME                     INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-elasticsearch-repo   true        5                6.022 KiB   Ready   2m15s                    18m
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-elasticsearch-repo
NAME                                                              REPOSITORY               SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-elasticsearch-repo-elasticseckup-frequent-backup-1738660680   gcs-elasticsearch-repo   frequent-backup   2025-02-04T09:18:00Z   Delete            Succeeded   48s
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <target-kind>`
> - `kubestash.com/app-ref-name: <target-name>`
> - `kubestash.com/app-ref-namespace: <target-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
>
> These labels can be used to watch only the `Snapshot`s related to our target Database or `Repository`.

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the Database.

```bash
$ kubectl get snapshots -n demo gcs-elasticsearch-repo-sample-elasticsearch-backup-frequent-backup-1731490567 -oyaml
```

```yaml
kubectl get snapshot -n demo gcs-elasticsearch-repo-elasticseckup-frequent-backup-1738660680 -oyaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  creationTimestamp: "2025-02-04T09:18:00Z"
  finalizers:
    - kubestash.com/cleanup
  generation: 1
  labels:
    kubestash.com/app-ref-kind: AppBinding
    kubestash.com/app-ref-name: elasticsearch-appbinding
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-elasticsearch-repo
  name: gcs-elasticsearch-repo-elasticseckup-frequent-backup-1738660680
  namespace: demo
  ownerReferences:
    - apiVersion: storage.kubestash.com/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: Repository
      name: gcs-elasticsearch-repo
      uid: 2283a678-05aa-4762-b381-bedd33938477
  resourceVersion: "16389"
  uid: 8028d8ee-1a53-4709-baed-188925f1d362
spec:
  appRef:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: elasticsearch-appbinding
    namespace: demo
  backupSession: elasticsearch-db-backup-frequent-backup-1738660680
  deletionPolicy: Delete
  repository: gcs-elasticsearch-repo
  session: frequent-backup
  snapshotID: 01JK8414XRVTXS9YDER2BDA4WR
  type: FullBackup
  version: v1
status:
  components:
    dump:
      driver: Restic
      duration: 6.984213094s
      integrity: true
      path: repository/v1/frequent-backup/dump
      phase: Succeeded
      resticStats:
        - hostPath: /kubestash-interim/data
          id: b3d8a033742a11d65277fc1d90d166e08623e4166f745564ddb0f227a92425e1
          size: 496 B
          uploaded: 2.088 KiB
      size: 6.017 KiB
  conditions:
    - lastTransitionTime: "2025-02-04T09:18:00Z"
      message: Recent snapshot list updated successfully
      reason: SuccessfullyUpdatedRecentSnapshotList
      status: "True"
      type: RecentSnapshotListUpdated
    - lastTransitionTime: "2025-02-04T09:18:35Z"
      message: Metadata uploaded to backend successfully
      reason: SuccessfullyUploadedSnapshotMetadata
      status: "True"
      type: SnapshotMetadataUploaded
  integrity: true
  phase: Succeeded
  size: 6.017 KiB
  snapshotTime: "2025-02-04T09:18:00Z"
  totalComponents: 1
  verificationStatus: NotVerified
```

> KubeStash uses the `multielasticdump` command to take backups of target Elasticsearch databases. Therefore, the component name for logical backups is set as `dump`.

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/elasticsearch/repository/v1/frequent-backup/dump` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/elasticsearch/snapshots` directory.

> Note: KubeStash stores all dumped data encrypted in the backup directory, meaning it remains unreadable until decrypted.

## Restore

In this section, we are going to restore the database from the backup we have taken in the previous section. We are going to delete the backed-up database and initialize it from the backup.

**Delete Backed-up Database:**

Now, we have to delete the previously backed-up 'info' database by connecting with the 'kubestash-es-test' Elasticsearch database using the `elasticsearch` client.

```bash
$ curl -XDELETE -k --user 'doadmin:<YOUR USER PASSWORD>' "https://kubestash-es-test-do-user-165729-0.k.db.ondigitalocean.com:25060/info"
{"acknowledged":true}
$ curl -XGET -k --user 'doadmin:<YOUR USER PASSWORD>' "https://kubestash-es-test-do-user-165729-0.k.db.ondigitalocean.com:25060/_cat/indices?v&s=index&pretty"
health status index                     uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana_1                 qLaowo1nQCy9Ou-z969wdQ   1   0          0            0       208b           208b
green  open   .opendistro_security      VuAf3AVYS_WibezmyTwI0w   1   0         10            2     65.2kb         65.2kb
green  open   .opensearch-observability N1sihK-SSaODBsGq_D38oQ   1   0          0            0       208b           208b
green  open   .plugins-ml-config        y5MS3LMlR42wr66b4UpXgA   1   0          1            0      3.9kb          3.9kb
```

Above shows that 'info' index has been deleted successfully.

#### Create RestoreSession:

Now, we need to create a `RestoreSession` CR pointing to targeted `Elasticsearch` database.

Below, is the contents of YAML file of the RestoreSession object,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: restore-db-elasticsearch
  namespace: demo
spec:
  target:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: elasticsearch-appbinding
    namespace: demo
  dataSource:
    repository: gcs-elasticsearch-repo
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret
      namespace: demo
  addon:
    name: elasticsearch-addon
    tasks:
      - name: logical-backup-restore
```

Here,

- `.spec.target` refers to the `elasticsearch-appbinding` AppBinding custom resource, Which contains all necessary connection information for the target Elasticsearch database.
- `.spec.dataSource.repository` specifies the Repository object that holds the backed up data.
- `.spec.dataSource.snapshot` specifies to restore from latest `Snapshot`.

Let's create the RestoreSession CRD object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/addons/elasticsearch/logical/examples/restoresession.yaml
restoresession.core.kubestash.com/restore-db-elasticsearch created
```

Once, you have created the `RestoreSession` object, KubeStash will create restore Job. Run the following command to watch the phase of the `RestoreSession` object,

```bash
$ kubectl get restoresession -n demo
NAME                           REPOSITORY               PHASE       DURATION   AGE
restore-db-elasticsearch   gcs-elasticsearch-repo   Succeeded   22s        92s
```

The `Succeeded` phase means that the restore process has been completed successfully.

#### Verify Restored Data:

In this section, we are going to verify whether the desired data has been restored successfully. We are going to connect to the database server and check whether the database and the table we backed-up earlier are successfully restored or not.

Now, connect to the database using the `elasticsearch` client. Once connected, check the database, index existence.

```bash
$ curl -XGET -k --user 'doadmin:<YOUR USER PASSWORD>' "https://kubestash-es-test-do-user-165729-0.k.db.ondigitalocean.com:25060/_cat/indices?v&s=index&pretty"
health status index                     uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana_1                 qLaowo1nQCy9Ou-z969wdQ   1   0          0            0       208b           208b
green  open   .opendistro_security      VuAf3AVYS_WibezmyTwI0w   1   0         10            2     65.2kb         65.2kb
green  open   .opensearch-observability N1sihK-SSaODBsGq_D38oQ   1   0          0            0       208b           208b
green  open   .plugins-ml-config        y5MS3LMlR42wr66b4UpXgA   1   0          1            0      3.9kb          3.9kb
green  open   info                      U7CAuZ4uSKqwAQ0XOAHxrg   1   0          1            0      4.5kb          4.5kb
```

So, from the above output, we can see that the `info` index we created earlier, has been restored successfully.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete backupconfigurations.core.kubestash.com  -n demo elasticsearch-db-backup
kubectl delete restoresessions.core.kubestash.com -n demo restore-db-elasticsearch
kubectl delete retentionpolicies.storage.kubestash.com -n demo demo-retention
kubectl delete backupstorage -n demo gcs-storage
kubectl delete secret -n demo gcs-secret
kubectl delete secret -n demo encrypt-secret