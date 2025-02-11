---
title: Backup & Restore  | KubeStash
description: Backup an externally managed MariaDB database using KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-mariadb-logical-backup
    name: MariaDB Database Backup
    parent: kubestash-mariadb
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
---

{{< notice type="warning" message="If you are using a KubeDB-managed MariaDB database, please refer to the following [guide](https://kubedb.com/docs/latest/guides/mariadb/backup/kubestash/overview/). This guide covers backup and restore procedures for externally managed MariaDB databases." >}}

# Backup and Restore MariaDB database using KubeStash

KubeStash allows you to backup and restore `MariaDB` databases. It supports backups for `MariaDB` instances running in Standalone, Group Replication, and InnoDB cluster configurations. KubeStash makes managing your `MariaDB` backups and restorations more straightforward and efficient.

This guide will give you how you can take backup and restore your externally managed `MariaDB` databases using `Kubestash`.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using `Minikube` or `Kind`.
- Install `kubedb-kubestash-catalog` in your cluster following the steps [here](https://github.com/kubedb/installer/tree/master/charts/kubedb-kubestash-catalog).
- Install `KubeStash` in your cluster following the steps [here](https://kubestash.com/docs/latest/setup/install/kubestash).
- Install KubeStash `kubectl` plugin following the steps [here](https://kubestash.com/docs/latest/setup/install/kubectl-plugin/).
- If you are not familiar with how KubeStash backup and restore MariaDB databases, please check the following guide [here](/docs/addons/mariadb/overview/index.md).

You should be familiar with the following `KubeStash` concepts:

- [BackupStorage](https://kubestash.com/docs/latest/concepts/crds/backupstorage/)
- [BackupConfiguration](https://kubestash.com/docs/latest/concepts/crds/backupconfiguration/)
- [AppBinding](https://kubedb.com/docs/v2024.9.30/guides/mariadb/concepts/appbinding/)
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

> **Note:** YAML files used in this tutorial are stored in [docs/addons/mariadb/logical/examples](/docs/addons/mariadb/logical/examples) directory of [kubestash/docs](https://github.com/kubestash/docs) repository.

## Backup MariaDB

KubeStash supports backups for `MariaDB` instances across different configurations, including Standalone, Group Replication, and InnoDB Cluster setups.

In this demonstration, we will focus on a [Bitnami-managed](https://github.com/bitnami/charts/tree/main/bitnami/mariadb) MariaDB database configured in Standalone mode. The backup and restore process is similar for configurations using Group Replication and InnoDB Cluster as well.


### Create a Sample MariaDB Database

Let's create a sample `MariaDB` database in DigitalOcean and insert some data into it.

<figure align="center">
  <img alt="Sample database in DigitalOcean" src="/docs/addons/mysql/logical/images/sample-mysql-database.png">
  <figcaption align="center">Fig: Sample database in DigitalOcean</figcaption>
</figure>

Here’s what we’ve done so far:
- Created a sample `MariaDB` database named `kubestash-test`.
- The image also displays the necessary connection details for this database.

**Create Secret:**

Now, create a `Secret` that contains the authentication username and password.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-auth-secret
  namespace: demo
type: Opaque
stringData:
  username: "doadmin" # replace with your authentication username
  password: "" # replace with your authentication password
```

**Create AppBinding:**

Next, we need to manually create an `AppBinding` custom resource (CR) in the same namespace as the database secret. This `AppBinding` will contain the necessary connection details for the database we created earlier.

```yaml
apiVersion: appcatalog.appscode.com/v1alpha1
kind: AppBinding
metadata:
  name: mariadb-appbinding
  namespace: demo
spec:
  clientConfig:
    url: mariadb://mariadb.mariadb.svc.cluster.local:3306/mydb
  secret:
    name: mariadb-auth-secret
  type: mariadb
  version: "11.4.4"
```

Here,
- `.spec.clientConfig.url` Specifies the connection URL for the target database. You can construct the URL as follows:
    - `mariadb://<host>:<port>/<primary_database>?ssl-mode=<sslmode_value>`
- `.spec.secret` Specifies the name of the secret containing the authentication credentials. In this case, we’ll use the secret we created earlier.
- `.spec.version` Specifies the version of targeted database.

**Insert Sample Data:**

Now, connect to the database using the `mariadb` client. Once connected, create a new database and table, then insert some sample data into it.

```bash
$ kubectl exec -it -n mariadb mariadb-0 -- bash
Defaulted container "mariadb" out of: mariadb, preserve-logs-symlinks (init)
I have no name!@mariadb-0:/$ mariadb -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 8193
Server version: 11.4.4-MariaDB Source distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE playground;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> SHOW DATABASES; 
+--------------------+
| Database           |
+--------------------+
| information_schema |
| my_database        |
| mysql              |
| performance_schema |
| playground         |
| sys                |
| test               |
+--------------------+
7 rows in set (0.000 sec)

MariaDB [(none)]> USE playground;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [playground]> CREATE TABLE playground.equipment ( id INT NOT NULL AUTO_INCREMENT, type VARCHAR(50), quant INT, color VARCHAR(25), PRIMARY KEY(id));

Query OK, 0 rows affected (0.022 sec)

MariaDB [playground]> SHOW TABLES IN playground;
+----------------------+
| Tables_in_playground |
+----------------------+
| equipment            |
+----------------------+
1 row in set (0.000 sec)


MariaDB [playground]> INSERT INTO playground.equipment (type, quant, color) VALUES ('slide', 2, 'blue');
Query OK, 1 row affected (0.006 sec)

MariaDB [playground]> SELECT * FROM playground.equipment;
+----+-------+-------+-------+
| id | type  | quant | color |
+----+-------+-------+-------+
|  1 | slide |     2 | blue  |
+----+-------+-------+-------+
1 row in set (0.000 sec)

MariaDB [playground]> exit
Bye
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/mariadb/logical/examples/backupstorage.yaml
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
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/mariadb/logical/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

### Backup

We have to create a `BackupConfiguration` targeting the respective `mariadb-appbinding` AppBinding custom resource. This AppBinding resource contains all necessary connection information for the target `MariaDB` database. Then, KubeStash will create a `CronJob` for each session to take periodic backup of that database.

At first, we need to create a secret with a Restic password for backup data encryption.

Let's create a secret called `encrypt-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD
secret "encrypt-secret" created
```

**Create BackupConfiguration:**

Below is the YAML for `BackupConfiguration` CR to backup the `kubestash-test` externally managed MariaDB database that we have created earlier,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-mariadb-backup
  namespace: demo
spec:
  target:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: mariadb-appbinding
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
        - name: gcs-mariadb-repo
          backend: gcs-backend
          directory: /mariadb
          encryptionSecret:
           name: encrypt-secret
           namespace: demo
      addon:
        name: mariadb-addon
        tasks:
          - name: logical-backup
            params:
              args: --databases playground
```

- `.spec.sessions[*].schedule` specifies that we want to backup the database at `5 minutes` interval.
- `.spec.target` refers to the `mariadb-appbinding` AppBinding custom resource, Which contains all necessary connection information for the target MariaDB database.
- `.spec.sessions[].addon.tasks[].params.databases` refers the targeted backup database list.

Let's create the `BackupConfiguration` CR that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/addons/mariadb/logical/examples/backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-mariadb-backup created
```

**Verify Backup Setup Successful**

If everything goes well, the phase of the `BackupConfiguration` should be `Ready`. The `Ready` phase indicates that the backup setup is successful. Let's verify the `Phase` of the `BackupConfiguration`,

```bash
$ kubectl get backupconfigurations.core.kubestash.com -n demo
NAME                     PHASE   PAUSED   AGE
sample-mariadb-backup    Ready            50s

```

Additionally, we can verify that the `Repository` specified in the `BackupConfiguration` has been created using the following command,

```bash
$ kubectl get repo -n demo
NAME                     INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-mariadb-repo         true        5                6.744 KiB   Ready   99s                      14s
```

KubeStash keeps the backup for `Repository` YAMLs. If we navigate to the GCS bucket, we will see the `Repository` YAML stored in the `demo/mariadb` directory.

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` CR.

Verify that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo

NAME                                             SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-sample-mariadb-backup-frequent-backup    */5 * * * *   <none>     False     0        2m40s           5m16s

```

**Verify BackupSession:**

KubeStash triggers an instant backup as soon as the `BackupConfiguration` is ready. After that, backups are scheduled according to the specified schedule.

```bash
$ kubectl get backupsession -n demo

NAME                                                     INVOKER-TYPE          INVOKER-NAME                  PHASE       DURATION   AGE
sample-mariadb-backup-frequent-backup-1738680600         BackupConfiguration   sample-mariadb-backup         Succeeded   32s        3m34s
```

We can see from the above output that the backup session has succeeded. Now, we are going to verify whether the backed up data has been stored in the backend.

**Verify Backup:**

Once a backup is complete, KubeStash will update the respective `Repository` CR to reflect the backup. Check that the repository `sample-mariadb-backup` has been updated by the following command,

```bash
$ kubectl get repository -n demo gcs-mariadb-repo

NAME               INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
gcs-mariadb-repo   true        5                6.744 KiB   Ready   34s                      4m9s
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots.storage.kubestash.com -n demo -l=kubestash.com/repo-name=gcs-mariadb-repo

NAME                                                              REPOSITORY         SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
gcs-mariadb-repo-sample-mariadb-ckup-frequent-backup-1738680444   gcs-mariadb-repo   frequent-backup   2025-02-04T14:47:24Z   Delete            Succeeded   5m18s
gcs-mariadb-repo-sample-mariadb-ckup-frequent-backup-1738680600   gcs-mariadb-repo   frequent-backup   2025-02-04T14:50:00Z   Delete            Succeeded   5m18s
gcs-mariadb-repo-sample-mariadb-ckup-frequent-backup-1738680900   gcs-mariadb-repo   frequent-backup   2025-02-04T14:55:00Z   Delete            Succeeded   106s
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
$ kubectl get snapshots.storage.kubestash.com -n demo gcs-mariadb-repo-sample-mariadb-backup-frequent-backup-1738680900 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  creationTimestamp: "2025-02-04T14:55:00Z"
  finalizers:
  - kubestash.com/cleanup
  generation: 1
  labels:
    kubestash.com/app-ref-kind: AppBinding
    kubestash.com/app-ref-name: mariadb-appbinding
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: gcs-mariadb-repo
  name: gcs-mariadb-repo-sample-mariadb-backup-frequent-backup-1738680900
  namespace: demo
  ownerReferences:
  - apiVersion: storage.kubestash.com/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: Repository
    name: gcs-mariadb-repo
    uid: bdb6288f-f3e7-4543-b26b-85477f735b6e
  resourceVersion: "2756460"
  uid: 0fdeea00-4236-4cd7-882b-e717f10bd10a
spec:
  appRef:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: mariadb-appbinding
    namespace: demo
  backupSession: sample-mariadb-backup-frequent-backup-1738680900
  deletionPolicy: Delete
  repository: gcs-mariadb-repo
  session: frequent-backup
  snapshotID: 01JK8QA74KM5VQ6S2TTQSKNBV0
  type: FullBackup
  version: v1
status:
  components:
    dump:
      driver: Restic
      duration: 6.300290013s
      integrity: true
      path: repository/v1/frequent-backup/dump
      phase: Succeeded
      resticStats:
      - hostPath: dumpfile.sql
        id: 9ab28fd7a447db63c929326344b7c85a3a6585eaeafcbc39e722088c3403ac2b
        size: 2.198 KiB
        uploaded: 2.490 KiB
      size: 6.746 KiB
  conditions:
  - lastTransitionTime: "2025-02-04T14:55:00Z"
    message: Recent snapshot list updated successfully
    reason: SuccessfullyUpdatedRecentSnapshotList
    status: "True"
    type: RecentSnapshotListUpdated
  - lastTransitionTime: "2025-02-04T14:55:20Z"
    message: Metadata uploaded to backend successfully
    reason: SuccessfullyUploadedSnapshotMetadata
    status: "True"
    type: SnapshotMetadataUploaded
  integrity: true
  phase: Succeeded
  size: 6.745 KiB
  snapshotTime: "2025-02-04T14:55:00Z"
  totalComponents: 1
```

> KubeStash uses the `mariadbdump` command to take backups of target MariaDB databases. Therefore, the component name for logical backups is set as `dump`.

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/mariadb/repository/v1/frequent-backup/dump` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/dep/snapshots` directory.

> Note: KubeStash stores all dumped data encrypted in the backup directory, meaning it remains unreadable until decrypted.

## Restore

In this section, we are going to restore the database from the backup we have taken in the previous section. We are going to delete the backed-up database and initialize it from the backup.

**Delete Backed-up Database:**

Now, we have to delete the previously backed-up 'playground' database by connecting with the 'kubestash-test' MariaDB database using the `mariadb` client.

```bash
$ kubectl exec -it -n mariadb mariadb-0 -- bash
Defaulted container "mariadb" out of: mariadb, preserve-logs-symlinks (init)
I have no name!@mariadb-0:/$ mariadb -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 8568
Server version: 11.4.4-MariaDB Source distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| my_database        |
| mysql              |
| performance_schema |
| playground         |
| sys                |
| test               |
+--------------------+
7 rows in set (0.000 sec)

MariaDB [(none)]> DROP DATABASE playground;
Query OK, 1 row affected (0.017 sec)

MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| my_database        |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
6 rows in set (0.000 sec)

MariaDB [(none)]> exit
Bye
```

Above shows that 'playground' database are deleted successfully.

#### Create RestoreSession:

Now, we need to create a `RestoreSession` CR pointing to targeted `MariaDB` database.

Below, is the contents of YAML file of the `RestoreSession` object that we are going to create to restore backed up data into the newly created database provisioned by MariaDB object named `restored-mariadb`.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: restore-mariadb
  namespace: demo
spec:
  target:
    apiGroup: appcatalog.appscode.com
    kind: AppBinding
    name: mariadb-appbinding
    namespace: demo
  dataSource:
    repository: gcs-mariadb-repo
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret
      namespace: demo
  addon:
    name: mariadb-addon
    tasks:
      - name: logical-backup-restore
```

Here,

- `.spec.target` refers to the `mariadb-appbinding` AppBinding custom resource, Which contains all necessary connection information for the target MariaDB database.
- `.spec.dataSource.repository` specifies the Repository object that holds the backed up data.
- `.spec.dataSource.snapshot` specifies to restore from latest `Snapshot`.

Let's create the RestoreSession CRD object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/addons/mariadb/logical/examples/restoresession.yaml
restoresession.core.kubestash.com/sample-mariadb-restore created
```

Once, you have created the `RestoreSession` object, KubeStash will create restore Job. Run the following command to watch the phase of the `RestoreSession` object,

```bash
$ watch kubectl get restoresession -n demo

Every 2.0s: kubectl get restoresession -n demo                                                                                          appscodepc-H510M-H: Tue Feb  4 21:09:50 2025

NAME                              REPOSITORY         PHASE       DURATION   AGE
sample-mariadb-restore            gcs-mariadb-repo   Succeeded   8s         70s

```

The `Succeeded` phase means that the restore process has been completed successfully.

#### Verify Restored Data:

In this section, we are going to verify whether the desired data has been restored successfully. We are going to connect to the database server and check whether the database and the table we backed-up earlier are successfully restored or not.

Now, connect to the database using the `mariadb` client. Once connected, check the database, table, and sample data existence.

```bash
$ kubectl exec -it -n mariadb mariadb-0 -- bash
Defaulted container "mariadb" out of: mariadb, preserve-logs-symlinks (init)
I have no name!@mariadb-0:/$ mariadb -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 8659
Server version: 11.4.4-MariaDB Source distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.


MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| my_database        |
| mysql              |
| performance_schema |
| playground         |
| sys                |
| test               |
+--------------------+
7 rows in set (0.000 sec)

MariaDB [(none)]> USE playground;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [playground]> SHOW TABLES IN playground; 
+----------------------+
| Tables_in_playground |
+----------------------+
| equipment            |
+----------------------+
1 row in set (0.000 sec)

MariaDB [playground]> SELECT * FROM playground.equipment;  
+----+-------+-------+-------+
| id | type  | quant | color |
+----+-------+-------+-------+
|  1 | slide |     2 | blue  |
+----+-------+-------+-------+
1 row in set (0.000 sec)

MariaDB [playground]> exit 
Bye
```

So, from the above output, we can see that the `playground` database and the `equipment` table we have created earlier, they are restored successfully.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete backupconfigurations.core.kubestash.com  -n demo sample-mariadb-backup
kubectl delete restoresessions.core.kubestash.com -n demo sample-mariadb-restore 
kubectl delete retentionpolicies.storage.kubestash.com -n demo demo-retention
kubectl delete backupstorage -n demo gcs-storage
kubectl delete secret -n demo gcs-secret
kubectl delete secret -n demo encrypt-secret
```