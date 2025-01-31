---
title: Backup & Restore MySQL | KubeStash
description: Backup an externally managed MySQL database using KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-mysql-logical-backup
    name: MySQL Database Backup
    parent: kubestash-mysql
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
---

{{< notice type="warning" message="If you are using a KubeDB-managed MySQL database, please refer to the following [guide](https://kubedb.com/docs/latest/guides/mysql/backup/kubestash/overview/). This guide covers backup and restore procedures for externally managed MySQL databases." >}}

# Backup and Restore MySQL database using KubeStash

KubeStash allows you to backup and restore `MySQL` databases. It supports backups for `MySQL` instances running in Standalone, Group Replication, and InnoDB cluster configurations. KubeStash makes managing your `MySQL` backups and restorations more straightforward and efficient.

This guide will give you how you can take backup and restore your externally managed `MySQL` databases using `Kubestash`.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using `Minikube` or `Kind`.
- Install `kubedb-kubestash-catalog` in your cluster following the steps [here](https://github.com/kubedb/installer/tree/master/charts/kubedb-kubestash-catalog).
- Install `KubeStash` in your cluster following the steps [here](https://kubestash.com/docs/latest/setup/install/kubestash).
- Install KubeStash `kubectl` plugin following the steps [here](https://kubestash.com/docs/latest/setup/install/kubectl-plugin/).
- If you are not familiar with how KubeStash backup and restore MySQL databases, please check the following guide [here](/docs/addons/mysql/overview/index.md).

You should be familiar with the following `KubeStash` concepts:

- [BackupStorage](https://kubestash.com/docs/latest/concepts/crds/backupstorage/)
- [BackupConfiguration](https://kubestash.com/docs/latest/concepts/crds/backupconfiguration/)
- [AppBinding](https://kubedb.com/docs/v2024.9.30/guides/mysql/concepts/appbinding/)
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

> **Note:** YAML files used in this tutorial are stored in [docs/addons/mysql/logical/examples](/docs/addons/mysql/logical/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Backup MySQL

KubeStash supports backups for `MySQL` instances across different configurations, including Standalone, Group Replication, and InnoDB Cluster setups.

In this demonstration, we’ll focus on a DigitalOcean-managed `MySQL` database configured in Standalone mode. The backup and restore process is similar for Group Replication and InnoDB Cluster configurations as well.

### Create a Sample MySQL Database

Let's create a sample `MySQL` database in DigitalOcean and insert some data into it.

<figure align="center">
  <img alt="Sample database in DigitalOcean" src="/docs/addons/mysql/logical/images/sample-mysql-database.png">
  <figcaption align="center">Fig: Sample database in DigitalOcean</figcaption>
</figure>

Here’s what we’ve done so far:
- Created a sample `MySQL` database named `kubestash-test`.
- The image also displays the necessary connection details for this database.

**Create Secret:**

Now, create a `Secret` that contains the authentication username and password.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-auth-secret
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
  name: mysql-appbinding
  namespace: demo
spec:
  clientConfig:
    url: mysql://kubestash-test-do-user-165729-0.m.db.ondigitalocean.com:25060/defaultdb?ssl-mode=REQUIRED
  secret:
    name: mysql-credentials-secret
  type: mysql
  version: "8.0.3"
```

Here,
- `.spec.clientConfig.url` Specifies the connection URL for the target database. You can construct the URL as follows:
    - `mysql://<host>:<port>/<primary_database>?ssl-mode=<sslmode_value>`
- `.spec.secret` Specifies the name of the secret containing the authentication credentials. In this case, we’ll use the secret we created earlier.
- `.spec.version` Specifies the version of targeted database.

**Insert Sample Data:**

Now, connect to the database using the `mysql` client. Once connected, create a new database and table, then insert some sample data into it.

```bash
$ mysql -u doadmin -pAVNS_Z575naNC0QfA1usflEg -h kubestash-test-do-user-165729-0.m.db.ondigitalocean.com -P 25060 -D defaultdb
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 242
Server version: 8.0.30 Source distribution

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE DATABASE playground;
Query OK, 1 row affected (0.30 sec)

mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| defaultdb          |
| information_schema |
| mysql              |
| performance_schema |
| playground         |
| sys                |
+--------------------+
6 rows in set (0.22 sec)

mysql> CREATE TABLE playground.equipment ( id INT NOT NULL AUTO_INCREMENT, type VARCHAR(50), quant INT, color VARCHAR(25), PRIMARY KEY(id));
Query OK, 0 rows affected (0.01 sec)

mysql> SHOW TABLES IN playground;
+----------------------+
| Tables_in_playground |
+----------------------+
| equipment            |
+----------------------+
1 row in set (0.01 sec)

mysql> INSERT INTO playground.equipment (type, quant, color) VALUES ('slide', 2, 'blue');
Query OK, 1 row affected (0.01 sec)

mysql> SELECT * FROM playground.equipment;
+----+-------+-------+-------+
| id | type  | quant | color |
+----+-------+-------+-------+
|  1 | slide |     2 | blue  |
+----+-------+-------+-------+
1 row in set (0.00 sec)

mysql> exit
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/mysql/logical/examples/backupstorage.yaml
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/mysql/logical/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

### Backup

We have to create a `BackupConfiguration` targeting the respective `mysql-appbinding` AppBinding custom resource. This AppBinding resource contains all necessary connection information for the target MySQL database. Then, KubeStash will create a `CronJob` for each session to take periodic backup of that database.

At first, we need to create a secret with a Restic password for backup data encryption.

Let's create a secret called `encrypt-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD
secret "encrypt-secret" created
```

**Create BackupConfiguration:**

Below is the YAML for `BackupConfiguration` CR to backup the `kubestash-test` externally managed MySQL database that we have created earlier,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-mysql-backup
  namespace: demo
spec:
  target:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: mysql-appbinding
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
        - name: gcs-mysql-repo
          backend: gcs-backend
          directory: /mysql
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: mysql-addon
        tasks:
          - name: logical-backup
            params:
              databases: playground
```

- `.spec.sessions[*].schedule` specifies that we want to backup the database at `5 minutes` interval.
- `.spec.target` refers to the `mysql-appbinding` AppBinding custom resource, Which contains all necessary connection information for the target MySQL database.
- `.spec.sessions[].addon.tasks[].params.databases` refers the targeted backup database list.

Let's create the `BackupConfiguration` CR that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/mysql/logical/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-mysql-backup created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the BackupConfiguration,

```bash
$ kubectl get backupconfiguration -n demo
NAME                  PHASE   PAUSED   AGE
sample-mysql-backup   Ready            2m50s
```

Additionally, we can verify that the `Repository` specified in the `BackupConfiguration` has been created using the following command,

```bash
$ kubectl get repo -n demo
NAME               INTEGRITY   SNAPSHOT-COUNT   SIZE     PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-mysql-repo                 0                0 B      Ready                            3m
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the `Repository` YAML stored in the `demo/mysql` directory.

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` CR.

Verify that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                          SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-sample-mysql-backup-frequent-backup   */5 * * * *             0        2m45s           3m25s
```

**Verify BackupSession:**

KubeStash triggers an instant backup as soon as the `BackupConfiguration` is ready. After that, backups are scheduled according to the specified schedule.

```bash
$ kubectl get backupsession -n demo -w

NAME                                             INVOKER-TYPE          INVOKER-NAME           PHASE       DURATION   AGE
sample-mysql-backup-frequent-backup-1731490567   BackupConfiguration   sample-mysql-backup    Succeeded              7m22s
```

We can see from the above output that the backup session has succeeded. Now, we are going to verify whether the backed up data has been stored in the backend.

**Verify Backup:**

Once a backup is complete, KubeStash will update the respective `Repository` CR to reflect the backup. Check that the repository `sample-mysql-backup` has been updated by the following command,

```bash
$ kubectl get repository -n demo gcs-mysql-repo
NAME                    INTEGRITY   SNAPSHOT-COUNT   SIZE    PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-mysql-repo          true        1                806 B   Ready   8m27s                    9m18s
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-mysql-repo
NAME                                                            REPOSITORY            SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-mysql-repo-sample-mysql-backup-frequent-backup-1731490567   gcs-mysql-repo        frequent-backup   2024-01-23T13:10:54Z   Delete            Succeeded   16h
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
$ kubectl get snapshots -n demo gcs-mysql-repo-sample-mysql-backup-frequent-backup-1731490567 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  creationTimestamp: "2024-11-13T09:37:47Z"
  finalizers:
    - kubestash.com/cleanup
  generation: 1
  labels:
    kubestash.com/app-ref-kind: AppBinding
    kubestash.com/app-ref-name: mysql-appbinding
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-mysql-repo
  name: gcs-mysql-repo-sample-mysql-backup-frequent-backup-1731490567
  namespace: demo
  ownerReferences:
    - apiVersion: storage.kubestash.com/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: Repository
      name: gcs-mysql-repo
      uid: c786be90-9a18-4c74-a29c-97e85d063a2e
  resourceVersion: "24024"
  uid: 9dd96c4b-2414-4e67-a692-6cd7f435062c
spec:
  appRef:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: mysql-appbinding
    namespace: demo
  backupSession: sample-mysql-backup-frequent-backup-1731490567
  deletionPolicy: Delete
  repository: gcs-mysql-repo
  session: frequent-backup
  snapshotID: 01JCJE5QCVVXZDKEKH8TC7Z8DV
  type: FullBackup
  version: v1
status:
  components:
    dump:
      driver: Restic
      duration: 9.532174884s
      integrity: true
      path: repository/v1/frequent-backup/dump
      phase: Succeeded
      resticStats:
        - hostPath: dumpfile.sql
          id: da60acb7a735312c0ace978d26565975a44ec90f5058eb73434880624dad3151
          size: 2.229 KiB
          uploaded: 2.521 KiB
      size: 7.003 KiB
  conditions:
    - lastTransitionTime: "2024-11-13T09:37:47Z"
      message: Recent snapshot list updated successfully
      reason: SuccessfullyUpdatedRecentSnapshotList
      status: "True"
      type: RecentSnapshotListUpdated
    - lastTransitionTime: "2024-11-13T09:38:20Z"
      message: Metadata uploaded to backend successfully
      reason: SuccessfullyUploadedSnapshotMetadata
      status: "True"
      type: SnapshotMetadataUploaded
  integrity: true
  phase: Succeeded
  size: 7.003 KiB
  snapshotTime: "2024-11-13T09:37:47Z"
  totalComponents: 1
```

> KubeStash uses the `mysqldump` command to take backups of target MySQL databases. Therefore, the component name for logical backups is set as `dump`.

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/mysql/repository/v1/frequent-backup/dump` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/dep/snapshots` directory.

> Note: KubeStash stores all dumped data encrypted in the backup directory, meaning it remains unreadable until decrypted.

## Restore

In this section, we are going to restore the database from the backup we have taken in the previous section. We are going to delete the backed-up database and initialize it from the backup.

**Delete Backed-up Database:**

Now, we have to delete the previously backed-up 'playground' database by connecting with the 'kubestash-test' MySQL database using the `mysql` client.

```bash
$ mysql -u doadmin -pAVNS_Z575naNC0QfA1usflEg -h kubestash-test-do-user-165729-0.m.db.ondigitalocean.com -P 25060 -D defaultdb
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5227
Server version: 8.0.30 Source distribution

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| defaultdb          |
| information_schema |
| mysql              |
| performance_schema |
| playground         |
| sys                |
+--------------------+
6 rows in set (0.28 sec)

mysql> DROP DATABASE playground;
Query OK, 1 row affected (0.34 sec)

mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| defaultdb          |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.27 sec)

mysql> exit
Bye

```

Above shows that 'playground' database are deleted successfully.

#### Create RestoreSession:

Now, we need to create a `RestoreSession` CR pointing to targeted `MySQL` database.

Below, is the contents of YAML file of the `RestoreSession` object that we are going to create to restore backed up data into the newly created database provisioned by MySQL object named `restored-mysql`.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: restore-sample-mysql
  namespace: demo
spec:
  target:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: mysql-appbinding
    namespace: demo
  dataSource:
    repository: gcs-mysql-repo
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret
      namespace: demo
  addon:
    name: mysql-addon
    tasks:
      - name: logical-backup-restore
```

Here,

- `.spec.target` refers to the `mysql-appbinding` AppBinding custom resource, Which contains all necessary connection information for the target MySQL database.
- `.spec.dataSource.repository` specifies the Repository object that holds the backed up data.
- `.spec.dataSource.snapshot` specifies to restore from latest `Snapshot`.

Let's create the RestoreSession CRD object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/addons/mysql/logical/examples/restoresession.yaml
restoresession.core.kubestash.com/sample-mysql-restore created
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

In this section, we are going to verify whether the desired data has been restored successfully. We are going to connect to the database server and check whether the database and the table we backed-up earlier are successfully restored or not.

Now, connect to the database using the `mysql` client. Once connected, check the database, table, and sample data existence.

```bash
$ mysql -u doadmin -pAVNS_Z575naNC0QfA1usflEg -h kubestash-test-do-user-165729-0.m.db.ondigitalocean.com -P 25060 -D defaultdb
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5828
Server version: 8.0.30 Source distribution

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| defaultdb          |
| information_schema |
| mysql              |
| performance_schema |
| playground         |
| sys                |
+--------------------+
6 rows in set (0.23 sec)

mysql> SHOW TABLES IN playground;
+----------------------+
| Tables_in_playground |
+----------------------+
| equipment            |
+----------------------+
1 row in set (0.34 sec)

mysql> SELECT * FROM playground.equipment;
+----+-------+-------+-------+
| id | type  | quant | color |
+----+-------+-------+-------+
|  1 | slide |     2 | blue  |
+----+-------+-------+-------+
1 row in set (0.26 sec)

mysql> exit
Bye
```

So, from the above output, we can see that the `playground` database and the `equipment` table we have created earlier, they are restored successfully.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete backupconfigurations.core.kubestash.com  -n demo sample-mysql-backup
kubectl delete restoresessions.core.kubestash.com -n demo restore-sample-mysql
kubectl delete retentionpolicies.storage.kubestash.com -n demo demo-retention
kubectl delete backupstorage -n demo gcs-storage
kubectl delete secret -n demo gcs-secret
kubectl delete secret -n demo encrypt-secret
```