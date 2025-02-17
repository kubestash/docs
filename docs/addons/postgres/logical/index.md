---
title: Backup & Restore PostgreSQL | KubeStash
description: Backup an externally managed PostgreSQL database using KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-postgresql-logical-backup
    name: PostgreSQL Database Backup
    parent: kubestash-postgresql
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
---

{{< notice type="warning" message="If you are using a KubeDB-managed PostgreSQL database, please refer to the following [guide](https://kubedb.com/docs/latest/guides/postgres/backup/kubestash/overview/). This guide covers backup and restore procedures for externally managed PostgreSQL databases." >}}

# Backup and Restore PostgreSQL database using KubeStash

KubeStash supports backups for `PostgreSQL` instances across different configurations, including Standalone and HA Cluster setups. In this demonstration, we’ll focus on a `PostgreSQL` database using HA cluster configuration. The backup and restore process is similar for Standalone configuration.

This guide will give you how you can take backup and restore your externally managed `PostgreSQL` databases using `Kubestash`.


## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using `Minikube` or `Kind`.
- Install `kubedb-kubestash-catalog` in your cluster following the steps [here](https://github.com/kubedb/installer/tree/master/charts/kubedb-kubestash-catalog).
- Install `KubeStash` in your cluster following the steps [here](https://kubestash.com/docs/latest/setup/install/kubestash).
- Install KubeStash `kubectl` plugin following the steps [here](https://kubestash.com/docs/latest/setup/install/kubectl-plugin/).
- If you are not familiar with how KubeStash backup and restore PostgreSQL databases, please check the following guide [here](/docs/addons/postgres/overview/index.md).

You should be familiar with the following `KubeStash` concepts:

- [BackupStorage](https://kubestash.com/docs/latest/concepts/crds/backupstorage/)
- [BackupConfiguration](https://kubestash.com/docs/latest/concepts/crds/backupconfiguration/)
- [AppBinding](https://kubedb.com/docs/v2024.9.30/guides/postgres/concepts/appbinding/)
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

> **Note:** YAML files used in this tutorial are stored in [docs/addons/postgres/logical/examples](/docs/addons/postgres/logical/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Backup PostgreSQL

KubeStash supports backups for `PostgreSQL` instances across different configurations, including Standalone and HA Cluster setups.

In this demonstration, we’ll focus on a DigitalOcean-managed `PostgreSQL` database configured in Standalone mode. The backup and restore process is similar for HA Cluster configurations as well.

### Create a Sample PostgreSQL Database

Let's create a sample `PostgreSQL` database in DigitalOcean and insert some data into it.


<figure align="center">
  <img alt="Sample database in DigitalOcean" src="/docs/addons/postgres/logical/images/sample-postgres-database.png">
  <figcaption align="center">Fig: Sample database in DigitalOcean</figcaption>
</figure>

Here’s what we’ve done so far:
- Created a sample `PostgreSQL` database named `kubestash-test`.
- The image also displays the necessary connection details for this database.

**Create Secret:**

Now, create a `Secret` that contains the authentication username and password.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-auth-secret
  namespace: demo
type: Opaque
stringData:
  username: doadmin # replace with your authentication username
  password: "" # replace with your authentication password
```

**Create AppBinding:**
Next, we need to manually create an `AppBinding` custom resource (CR) in the same namespace as the database secret. This `AppBinding` will contain the necessary connection details for the database we created earlier.

```yaml
apiVersion: appcatalog.appscode.com/v1alpha1
kind: AppBinding
metadata:
  name: postgres-appbinding
  namespace: demo
spec:
  clientConfig:
    url: postgres://kubestash-test-do-user-165729-0.m.db.ondigitalocean.com:25060/defaultdb?ssl-mode=REQUIRED
  secret:
    name: postgres-auth-secret
  type: postgres
  version: "17.2"
```

Here,
- `.spec.clientConfig.url` Specifies the connection URL for the target database. You can construct the URL as follows:
    - `postgres://<host>:<port>/<primary_database>?ssl-mode=<sslmode_value>`
- `.spec.secret` Specifies the name of the secret containing the authentication credentials. In this case, we’ll use the secret we created earlier.
- `.spec.version` Specifies the version of targeted database.

**Insert Sample Data:**

Now, connect to the database using the `postgres` client. Once connected, create a new database and table, then insert some sample data into it.

```bash

$ docker run -it --rm postgres:latest psql -h kubestash-test-do-user-165729-0.m.docker run -it \
  psql -h kubestash-test-do-user-165729-0.m.db.ondigitalocean.com -p 25060 -U doadmin -d defaultdb
Password for user doadmin:
psql (17.2 (Debian 17.2-1.pgdg120+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
Type "help" for help.

# list available databases
defaultdb=> \l
List of databases
Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges   
------------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
_dodb      | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =T/postgres          +
|          |          |                 |             |             |        |           | postgres=CTc/postgres
defaultdb  | doadmin  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           |
playground | doadmin  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           |
template0  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
|          |          |                 |             |             |        |           | postgres=CTc/postgres
template1  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
|          |          |                 |             |             |        |           | postgres=CTc/postgres
(5 rows)

# create a database named "demo"
defaultdb=> create database demo;
CREATE DATABASE

# verify that the "demo" database has been created
defaultdb=> \l
List of databases
Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges   
------------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
_dodb      | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =T/postgres          +
|          |          |                 |             |             |        |           | postgres=CTc/postgres
defaultdb  | doadmin  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           |
demo       | doadmin  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           |
playground | doadmin  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           |
template0  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
|          |          |                 |             |             |        |           | postgres=CTc/postgres
template1  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
|          |          |                 |             |             |        |           | postgres=CTc/postgres
(6 rows)

# connect to the "demo" database
defaultdb=> \c demo
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
You are now connected to database "demo" as user "doadmin".

# create a sample table
demo=> CREATE TABLE COMPANY( NAME TEXT NOT NULL, EMPLOYEE INT NOT NULL);
CREATE TABLE

# verify that the table has been created
demo=> \d
List of relations
Schema |  Name   | Type  |  Owner  
--------+---------+-------+---------
public | company | table | doadmin
(1 row)

# insert multiple rows of data into the table
demo=> INSERT INTO COMPANY (NAME, EMPLOYEE) VALUES ('TechCorp', 100), ('InnovateInc', 150), ('AlphaTech', 200);
INSERT 0 3

# verify the data insertion
demo=> SELECT * FROM COMPANY;
name     | employee
-------------+----------
TechCorp    |      100
InnovateInc |      150
AlphaTech   |      200
(3 rows)

demo=> \q
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/postgres/logical/examples/backupstorage.yaml
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/postgres/logical/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

### Backup

We have to create a `BackupConfiguration` targeting the respective `postgres-appbinding` AppBinding custom resource. This AppBinding resource contains all necessary connection information for the target `PostgreSQL` database. Then, KubeStash will create a `CronJob` for each session to take periodic backup of that database.

At first, we need to create a secret with a Restic password for backup data encryption.

Let's create a secret called `encrypt-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD
secret "encrypt-secret" created
```

**Create BackupConfiguration:**

Below is the YAML for `BackupConfiguration` CR to backup the `kubestash-test` externally managed `PostgreSQL` database that we have created earlier,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-postgres-backup
  namespace: demo
spec:
  target:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: postgres-appbinding
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
      scheduler:
        schedule: "*/5 * * * *"
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: gcs-postgres-repo
          backend: gcs-backend
          directory: /postgres
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: postgres-addon
        tasks:
          - name: logical-backup
            params:
              backupCmd: pg_dump
              args: demo
```

- `.spec.sessions[*].schedule` specifies that we want to backup the database at `5 minutes` interval.
- `.spec.target` refers to the `postgres-appbinding` AppBinding custom resource, Which contains all necessary connection information for the target PostgreSQL database.
- `.spec.sessions[].addon.tasks[].params.backupCmd` refers the command that we want to use during backup.
- `.spec.sessions[].addon.tasks[].params.args` refers the targeted backup database list.

Let's create the `BackupConfiguration` CR that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/postgres/logical/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-postgres-backup created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME                     PHASE   PAUSED   AGE
sample-postgres-backup   Ready            51s
```

Additionally, we can verify that the `Repository` specified in the `BackupConfiguration` has been created using the following command,

```bash
$ kubectl get repo -n demo
NAME                  INTEGRITY   SNAPSHOT-COUNT   SIZE     PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-postgres-repo                 0                0 B      Ready                            3m
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the `Repository` YAML stored in the `demo/postgres` directory.

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` CR.

Verify that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-sample-postgres-backup-frequent-backup   */5 * * * *             0        2m45s           3m25s
```

**Verify BackupSession:**

KubeStash triggers an instant backup as soon as the `BackupConfiguration` is ready. After that, backups are scheduled according to the specified schedule.

```bash
$ kubectl get backupsession -n demo -w
NAME                                                INVOKER-TYPE          INVOKER-NAME             PHASE       DURATION   AGE
sample-postgres-backup-frequent-backup-1738580539   BackupConfiguration   sample-postgres-backup   Succeeded   42s        7m22s
```

We can see from the above output that the backup session has succeeded. Now, we are going to verify whether the backed up data has been stored in the backend.


**Verify Backup:**

Once a backup is complete, KubeStash will update the respective `Repository` CR to reflect the backup. Check that the repository `sample-postgres-backup` has been updated by the following command,

```bash
$ kubectl get repository -n demo gcs-postgres-repo
NAME                       INTEGRITY   SNAPSHOT-COUNT   SIZE    PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-postgres-repo          true        1                806 B   Ready   8m27s                    9m18s
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-postgres-repo
NAME                                                              REPOSITORY          SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-postgres-repo-sample-postgreckup-frequent-backup-1738580539   gcs-postgres-repo   frequent-backup   2025-02-03T11:03:09Z   Delete            Succeeded   3m26s
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
$ kubectl get snapshots -n demo gcs-postgres-repo-sample-postgreckup-frequent-backup-1738580539 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  creationTimestamp: "2025-02-03T11:03:09Z"
  finalizers:
    - kubestash.com/cleanup
  generation: 1
  labels:
    kubestash.com/app-ref-kind: AppBinding
    kubestash.com/app-ref-name: postgres-appbinding
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-postgres-repo
  name: gcs-postgres-repo-sample-postgreckup-frequent-backup-1738580539
  namespace: demo
  ownerReferences:
    - apiVersion: storage.kubestash.com/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: Repository
      name: gcs-postgres-repo
      uid: 7d3713f4-6123-454d-9738-14a55c9e51ee
  resourceVersion: "11226"
  uid: 093ed3e8-862f-4428-92f5-909e4e3a209b
spec:
  appRef:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: postgres-appbinding
    namespace: demo
  backupSession: sample-postgres-backup-frequent-backup-1738580539
  deletionPolicy: Delete
  repository: gcs-postgres-repo
  session: frequent-backup
  snapshotID: 01JK5QMYX43N4G3XM8N8JW1PP8
  type: FullBackup
  version: v1
status:
  components:
    dump:
      driver: Restic
      duration: 21.570479502s
      integrity: true
      path: repository/v1/frequent-backup/dump
      phase: Succeeded
      resticStats:
        - endTime: "2025-02-03T11:03:40Z"
          hostPath: dumpfile.sql
          id: f6cd7e1827fd70ba1ac2256a3df14c673949f04381aea78abd4cbf308d9afea5
          size: 973 B
          startTime: "2025-02-03T11:03:18Z"
          uploaded: 1.241 KiB
      size: 781 B
  conditions:
    - lastTransitionTime: "2025-02-03T11:03:09Z"
      message: Recent snapshot list updated successfully
      reason: SuccessfullyUpdatedRecentSnapshotList
      status: "True"
      type: RecentSnapshotListUpdated
    - lastTransitionTime: "2025-02-03T11:03:48Z"
      message: Metadata uploaded to backend successfully
      reason: SuccessfullyUploadedSnapshotMetadata
      status: "True"
      type: SnapshotMetadataUploaded
  integrity: true
  phase: Succeeded
  size: 781 B
  snapshotTime: "2025-02-03T11:03:09Z"
  totalComponents: 1
  verificationStatus: NotVerified
```

> KubeStash uses the `pg_dump` command to take backups of target PostgreSQL databases. Therefore, the component name for logical backups is set as `dump`.

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/postgres/repository/v1/frequent-backup/dump` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/postgres/snapshots` directory.

> Note: KubeStash stores all dumped data encrypted in the backup directory, meaning it remains unreadable until decrypted.

## Restore

In this section, we are going to restore the database from the backup we have taken in the previous section. We are going to delete the backed-up database table and initialize it from the backup.

Now, we have to delete the previously backed-up table `company` of `demo` database by connecting with the 'kubestash-test' PostgreSQL database using the `postgres` client.

```bash
$ docker run -it --rm postgres:latest psql -h kubestash-test-do-user-165729-0.m.docker run -it \
  psql -h kubestash-test-do-user-165729-0.m.db.ondigitalocean.com -p 25060 -U doadmin -d defaultdb
Password for user doadmin: 
psql (17.2 (Debian 17.2-1.pgdg120+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
Type "help" for help.

# list available databases
defaultdb=> \l
                                                     List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges   
-----------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
 _dodb     | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =T/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
 defaultdb | doadmin  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
 demo      | doadmin  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
(5 rows)

# connect to the "demo" database
defaultdb=> \c demo
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
You are now connected to database "demo" as user "doadmin".

# drop table "company" from "demo" database
demo=> DROP TABLE company;
DROP TABLE

# verify that the table has been dropped
demo=> \dt
Did not find any relations.

demo=> exit 
```

Above shows that the `company` table of `demo` database has been deleted successfully.

#### Create RestoreSession:

Now, we need to create a `RestoreSession` CR pointing to targeted `AppBinding` of any externally managed `PostgreSQL` database .

Below, is the contents of YAML file of the `RestoreSession` object,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: restore-sample-postgres
  namespace: demo
spec:
  target:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: postgres-appbinding
    namespace: demo
  dataSource:
    repository: gcs-postgres-repo
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret
      namespace: demo
  addon:
    name: postgres-addon
    tasks:
      - name: logical-backup-restore
        params:
          args: --dbname=demo
```

Here,

- `.spec.target` refers to the `postgres-appbinding` AppBinding custom resource, Which contains all necessary connection information for the target `PostgreSQL` database.
- `.spec.dataSource.repository` specifies the Repository object that holds the backed up data.
- `.spec.dataSource.snapshot` specifies to restore from latest `Snapshot`.
- `.spec.addon.tasks[].params.args` specifies the additional psql arguments. In this case, `--dbname=demo` indicates that the backup data will be restored into the demo database.

Let's create the RestoreSession CRD object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/addons/postgres/logical/examples/restoresession.yaml
restoresession.core.kubestash.com/sample-postgres-restore created
```

Once, you have created the `RestoreSession` object, KubeStash will create restore Job. Run the following command to watch the phase of the `RestoreSession` object,

```bash
$ watch kubectl get restoresession -n demo
Every 2.0s: kubectl get restores... AppsCode-PC-03: Wed Aug 21 10:44:05 2024
NAME             REPOSITORY        FAILURE-POLICY   PHASE       DURATION   AGE
sample-restore   gcs-demo-repo                      Succeeded   3s         53s
```

The `Succeeded` phase means that the restore process has been completed successfully.


#### Verify Restored Data:

In this section, we are going to verify whether the desired data has been restored successfully. We are going to connect to the database server and check whether the deleted table and data of `demo` database we backed-up earlier are successfully restored or not.

Now, connect to the database using the `postgres` client. Once connected, check the database, table, and sample data existence.

```bash
$ docker run -it --rm postgres:latest psql -h kubestash-test-do-user-165729-0.m.docker run -it \
  psql -h kubestash-test-do-user-165729-0.m.db.ondigitalocean.com -p 25060 -U doadmin -d defaultdb
Password for user doadmin: 
psql (17.2 (Debian 17.2-1.pgdg120+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
Type "help" for help.

# list available databases
defaultdb=> \l
                                                     List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges   
-----------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
 _dodb     | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =T/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
 defaultdb | doadmin  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
 demo      | doadmin  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
(5 rows)

# connect to the "demo" database
defaultdb=> \c demo
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
You are now connected to database "demo" as user "doadmin".

# verify that the table has been restored
demo=> \d
List of relations
Schema |  Name   | Type  |  Owner  
--------+---------+-------+---------
public | company | table | doadmin
(1 row)

# verify the data has been restored
demo=> SELECT * FROM COMPANY;
name     | employee
-------------+----------
TechCorp    |      100
InnovateInc |      150
AlphaTech   |      200
(3 rows)

demo=> exit 
```

From the above output, we can confirm that the `COMPANY` table, along with all the backup data of `demo` database we created earlier, has been successfully restored.


## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete backupconfigurations.core.kubestash.com  -n demo sample-postgres-backup
kubectl delete restoresessions.core.kubestash.com -n demo restore-sample-postgres
kubectl delete retentionpolicies.storage.kubestash.com -n demo demo-retention
kubectl delete backupstorage -n demo gcs-storage
kubectl delete secret -n demo gcs-secret
kubectl delete secret -n demo encrypt-secret
```