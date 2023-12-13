---
title: Backup & Restore Hooks | KubeStash
menu:
  docs_{{ .version }}:
    identifier: backup-and-restore-hooks
    name: Backup & Restore Hooks
    parent: hooks
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Backup & Restore Hooks

KubeStash hooks let you perform some actions before and after the backup or restore process. This is particularly helpful when you want to prepare your application before backup or restore.

Here, we are going to demonstrate how you can perform different actions before and after backup and restore a MySQL database. Some of the examples might not reflect the real-world use cases, but it serves the sole purpose of demonstrating what is possible.

> Note that, this is an advanced concept. If you haven't tried the normal backup and restore processes yet, we will recommend to try them first.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
- Install KubeStash in your cluster following the steps [here](/docs/setup/install/kubestash/index.md).
- Install [KubeDB](https://kubedb.com) in your cluster following the steps [here](https://kubedb.com/docs/latest/setup/). This step is optional. You can deploy your database using any method you want. We are using KubeDB because KubeDB simplifies many of the difficult or tedious management tasks of running production-grade databases on private and public clouds.
- If you are not familiar with how KubeStash backup and restore MySQL databases, please check the following guide [here](/docs/addons/mysql/overview/index.md).
- Also, if you haven't read about how hooks work in KubeStash, please check it from [here](/docs/guides/hooks/overview/index.md).

You should be familiar with the following `KubeStash` concepts:

- [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
- [BackupSession](/docs/concepts/crds/backupsession/index.md)
- [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
- [Function](/docs/concepts/crds/function/index.md)
- [HookTemplate](/docs/concepts/crds/hooktemplate/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

## Prepare Database

At first, let's deploy a MySQL database. Here, we are going to deploy MySQL `8.0.32` using KubeDB. We are going to insert some sample data into the database so that we can verify that the backup and restore process is working properly.

**Deploy Database:**

Below is the `MySQL` CR(Custom Resource) that we are going to create,

```yaml
apiVersion: kubedb.com/v1alpha2
kind: MySQL
metadata:
  name: sample-mysql
  namespace: demo
spec:
  version: 8.0.35
  replicas: 1
  storageType: Durable
  storage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  terminationPolicy: WipeOut
```

Let's create the above `MySQL` CR,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/sample-mysql.yaml
mysql.kubedb.com/sample-mysql created
```

KubeDB will deploy a MySQL database according to the above specification. It will also create the necessary Secrets and Services to access the database.

Wait for the database to go into `Running` state,

```bash
$ kubectl get mysql -n demo -w
NAME           VERSION   STATUS    AGE
sample-mysql   8.0.35    Creating  5s
sample-mysql   8.0.35    Running   2m7s
```

**Verify Database Secret:**

Verify that KubeDB has created a Secret for the database.

```bash
$ kubectl get secret -n demo -l=app.kubernetes.io/instance=sample-mysql
NAME                TYPE                       DATA   AGE
sample-mysql-auth   kubernetes.io/basic-auth   2      75s
```

**Insert Sample Data:**

Now, let's insert some sample data into the above database. Here, we are going to `exec` into the database pod and create a database named `companyRecord`. Then, we are going to create a table named `employee` which will store employee's id, name and salary information. Then, we are going to insert a sample row in the table.

At first, let's export the database credentials as environment variables in our current shell so that we can use those variables to access the database instead of typing username and password every time.

```bash
# export username from the database secret
$ export MYSQL_USER=$(kubectl get secret -n demo  sample-mysql-auth -o jsonpath='{.data.username}'| base64 -d)

# verify that the username has been exported properly
$ echo $MYSQL_USER
root

# export the password from the database secret
$ export MYSQL_PASSWORD=$(kubectl get secret -n demo  sample-mysql-auth -o jsonpath='{.data.password}'| base64 -d)

# verify that the password has been exported properly
$ echo $MYSQL_PASSWORD
CWg2hru8b0Yu7dzS
```

Now, let's identify the database pod,

```bash
$ kubectl get pods -n demo --selector="app.kubernetes.io/instance=sample-mysql"
NAME             READY   STATUS    RESTARTS   AGE
sample-mysql-0   1/1     Running   0          6m50s
```

Let's `exec` into the database pod and insert sample data,

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD

mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 115
Server version: 8.0.35 MySQL Community Server - GPL

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

# create database named "companyRecord"
mysql> CREATE DATABASE companyRecord;
Query OK, 1 row affected (0.01 sec)

# verify that the database has been created
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| companyRecord      |
| information_schema |
| kubedb_system      |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
6 rows in set (0.00 sec)

# create a table called "employee" in "companyRecord" database
mysql> CREATE TABLE companyRecord.employee (id INT, name VARCHAR(50), salary INT, PRIMARY KEY(id));
Query OK, 0 rows affected (0.05 sec)

# insert a demo data into the table
mysql> INSERT INTO companyRecord.employee (id, name, salary) VALUES (1, "John Doe", 5000);
Query OK, 1 row affected (0.01 sec)

# verify that the data has been inserted
mysql> SELECT * FROM companyRecord.employee;
+----+----------+--------+
| id | name     | salary |
+----+----------+--------+
|  1 | John Doe |   5000 |
+----+----------+--------+
1 row in set (0.00 sec)

mysql> exit
Bye
```

### Prepare Backend

We are going to store our backed up data into a GCS bucket. We have to create a Secret with necessary credentials and a BackupStorage crd to use this backend. If you want to use a different backend, please read the respective backend configuration doc from [here](/docs/guides/backends/overview/index.md).

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

Now, create a `BackupStorage` using this secret. Below is the YAML of `BackupStorage` crd we are going to create,

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
  deletionPolicy: WipeOut
```

Let's create the BackupStorage we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-storage created
```

Now, we are ready to backup our sample data into this backend.

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
Notice the `spec.usagePolicy` that allows referencing the `RetentionPolicy` from all namespaces. To allow specific namespaces, we can configure it accordingly by following [RetentionPolicy usage policy](/docs/concepts/crds/retentionpolicy/index.md#retentionpolicy-spec).

Let’s create the above `RetentionPolicy`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

## Backup

In this section, we are going to demonstrate `preBackup` hook and `postBackup` hook. We are going to make MySQL database read-only in  `preBackup` hook so that no writes operation happens in the database during backup. Then, we are going to make the database writable in `postBackup` hook so that the application can write again into the database.

### PreBackup Hook

At first, we are going to set `super_read_only` flag `ON` in `preBackup` hook which will make the database read-only. However, we won't set this flag `OFF` in `postBackup` so that we can verify that the hook has been executed.

**Create HookTemplate**

Below is the YAML of the `HookTemplate` CR to make the database read-only,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: HookTemplate
metadata:
  name: readonly-hook
  namespace: demo
spec:
  usagePolicy:
    allowedNamespaces:
      from: All
  action:
    exec:
      command:
        - /bin/sh
        - -c
        - mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "SET GLOBAL super_read_only = ON;"
    containerName: mysql # KubeDB uses "mysql" name for MySQL database container. If you haven't used KubeDB, change this according to your setup.
  executor:
    type: Pod
    pod:
      selector: app.kubernetes.io/instance=sample-mysql, app.kubernetes.io/managed-by=kubedb.com, app.kubernetes.io/name=mysqls.kubedb.com
      strategy: ExecuteOnAll
```

Let’s create the above `HookTemplate`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/hooktemplate.yaml
hooktemplate.core.kubestash.com/readonly-hook created
```

Now, we need to create a secret with a Restic password for backup data encryption.

**Create Secret:**

Let's create a secret called `encry-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encry-secret \
    --from-file=./RESTIC_PASSWORD \
secret "encry-secret" created
```

**Create BackupConfiguration:**

Below is the YAML of the `BackupConfiguration` CR with `preBackup` hook configured to make the database read-only before backup,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-backup
  namespace: demo
spec:
  target:
    apiGroup: kubedb.com
    kind: MySQL
    name: sample-mysql
    namespace: demo
  backends:
    - name: gcs-backend
      storageRef:
        name: gcs-storage
        namespace: demo
      retentionPolicy:
        name: demo-retention
        namespace: demo
  sessions:
    - name: frequent-backup
      sessionHistoryLimit: 5
      scheduler:
        schedule: "*/5 * * * *"
        jobTemplate:
          backoffLimit: 1
      hooks:
        preBackup:
          - name: pre-hook
            hookTemplate:
              name: readonly-hook
              namespace: demo
            maxRetry: 3
            timeout: 30s
      repositories:
        - name: demo-repo
          backend: gcs-backend
          directory: /demo/hook
          encryptionSecret:
            name: encry-secret # some addon may not support encryption
            namespace: demo
      addon:
        name: mysql-addon
        tasks:
          - name: logical-backup
```

Let's create the above `BackupConfiguration`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/pre_backup_hook_demo.yaml
backupconfiguration.core.kubestash.com/sample-backup created
```

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.sessions[*].scheduler.schedule` field of `BackupConfiguration` crd.

Verify that the `CronJob` has been created using the following command,

```bash
$ kubectl get cronjob -n demo
NAME                                    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
trigger-sample-backup-frequent-backup   */5 * * * *   False     0        107m            108m
```

**Wait for BackupSession:**

Wait for the next schedule for backup. Run the following command to watch `BackupSession` crd,

```bash
$ kubectl get backupsession -n demo -w

NAME                                       INVOKER-TYPE          INVOKER-NAME    PHASE       DURATION   AGE
sample-backup-frequent-backup-1708327500   BackupConfiguration   sample-backup   Succeeded              109m
```

Here, the phase `Succeeded` means that the backup process has been completed successfully.

**Verify Backup:**

Once a backup is complete, KubeStash will update the respective `Repository` crd to reflect the backup. Check that the repository `demo-repo` has been updated by the following command,

```bash
$ kubectl get repository -n demo demo-repo

NAME        INTEGRITY   SNAPSHOT-COUNT   SIZE          PHASE   LAST-SUCCESSFUL-BACKUP   AGE
demo-repo   true        1                664.373 KiB   Ready   141m                     147m
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run to a particular `Repository`.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=gcs-demo-repo

NAME                                                 REPOSITORY   SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
demo-repo-sample-backup-frequent-backup-1708327500   demo-repo    frequent-backup   2024-02-19T07:25:01Z   Delete            Succeeded   168m
```

> When a backup is triggered according to schedule, KubeStash will create a `Snapshot` with the following labels  `kubestash.com/app-ref-kind: <workload-kind>`, `kubestash.com/app-ref-name: <workload-name>`, `kubestash.com/app-ref-namespace: <workload-namespace>` and `kubestash.com/repo-name: <repository-name>`. We can use these labels to watch only the `Snapshot` of our desired Workload or `Repository`.

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the MySQL database.

```bash
$ kubectl get snapshots -n demo demo-repo-sample-backup-frequent-backup-1708327500 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: MySQL
    kubestash.com/app-ref-name: sample-mysql
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: demo-repo
  name: demo-repo-sample-backup-frequent-backup-1708327500
  namespace: demo
spec:
  ...
status:
  components:
    dump:
      driver: Restic
      duration: 2.133976452s
      integrity: true
      path: repository/v1/frequent-backup/dump
      phase: Succeeded
      resticStats:
      - hostPath: dumpfile.sql
        id: 0c7b8887c09bfe3d31d6af3a8e461f7773b52cd350a50deb3e8bbf94b52de01b
        size: 3.736 MiB
        uploaded: 3.736 MiB
      size: 664.374 KiB
  ...
```

Now, if we navigate to `demo/hook/repository/v1/frequent-backup/` directory of our gcs bucket, we are going to see that the component of the MySQL database has been stored there.

> KubeStash keeps all backup data encrypted. So, the files in the bucket will not contain any meaningful data until they are decrypted.

**Verify PreBackup Hook Executed:**

If the `preBackup` hook executes successfully, the database will be marked as read-only. In this situation, if we try to make a write operation into the database, it should reject the operation. However, the database should serve the read operations without any problem.

Let's verify that the database is read-only by trying to execute a write operation,

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "CREATE DATABASE readOnlyTest;"

mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1290 (HY000) at line 1: The MySQL server is running with the --super-read-only option so it cannot execute this statement
command terminated with exit code 1
```

Here, the error message clearly states the database is now read-only. Let's try to execute a read operation.

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "SELECT * FROM companyRecord.employee;"

mysql: [Warning] Using a password on the command line interface can be insecure.
+----+----------+--------+
| id | name     | salary |
+----+----------+--------+
|  1 | John Doe |   5000 |
+----+----------+--------+
```

So, we can see that the database can serve read-only queries without any problem.

### PostBackup Hook

Now, let's update the `BackupConfiguration` CR and add a `postBackup` hook that set `super_read_only` flag to `OFF`. So, the database should be writable again from the next backup.

**Create HookTemplate**

Below is the YAML of the `HookTemplate` CR to make the database writable,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: HookTemplate
metadata:
  name: readonly-off-hook
  namespace: demo
spec:
  usagePolicy:
    allowedNamespaces:
      from: All
  action:
    exec:
      command:
        - /bin/sh
        - -c
        - mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "SET GLOBAL super_read_only = OFF;"
    containerName: mysql # KubeDB uses "mysql" name for MySQL database container. If you haven't used KubeDB, change this according to your setup.
  executor:
    type: Pod
    pod:
      selector: app.kubernetes.io/instance=sample-mysql, app.kubernetes.io/managed-by=kubedb.com, app.kubernetes.io/name=mysqls.kubedb.com
      strategy: ExecuteOnAll
```

Let’s create the above `HookTemplate`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/hooktemplate-post.yaml
hooktemplate.core.kubestash.com/readonly-off-hook created
```

**Update BackupConfiguration:**

Below is the YAML for the updated `BackupConfiguration` CR with `postBackup` hook.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-backup
  namespace: demo
spec:
  target:
    apiGroup: kubedb.com
    kind: MySQL
    name: sample-mysql
    namespace: demo
  backends:
    - name: gcs-backend
      storageRef:
        name: gcs-storage
        namespace: demo
      retentionPolicy:
        name: demo-retention
        namespace: demo
  sessions:
    - name: frequent-backup
      sessionHistoryLimit: 5
      scheduler:
        schedule: "*/5 * * * *"
        jobTemplate:
          backoffLimit: 1
      hooks:
        preBackup:
          - name: pre-hook
            hookTemplate:
              name: readonly-hook
              namespace: demo
            maxRetry: 3
            timeout: 30s
        postBackup:
          - name: post-hook
            hookTemplate:
              name: readonly-off-hook
              namespace: demo
            maxRetry: 3
            timeout: 30s
      repositories:
        - name: demo-repo
          backend: gcs-backend
          directory: /demo/hook
          encryptionSecret:
            name: encry-secret # some addon may not support encryption
            namespace: demo
      addon:
        name: mysql-addon
        tasks:
          - name: logical-backup
```

Let's apply the update,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/post_backup_hook_demo.yaml
backupconfiguration.core.kubestash.com/sample-backup configured
```

**Wait for Next BackupSession:**

Now, wait for the next backup slot,

```bash
$ kubectl get backupsession -n demo -w

NAME                                       INVOKER-TYPE          INVOKER-NAME    PHASE       DURATION   AGE
sample-backup-frequent-backup-1708340400   BackupConfiguration   sample-backup   Succeeded              109m
```

**Verify PostBackup Hook Executed:**

If the `postBackup` hook has been executed successfully, the database should be writable again. Let's try to execute a write operation to verify that the database writable,

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "CREATE DATABASE postBackupHookTest;"
mysql: [Warning] Using a password on the command line interface can be insecure.
```

Verify the test database has been created successfully,

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "SHOW DATABASES;"

mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| companyRecord      |
| information_schema |
| kubedb_system      |
| mysql              |
| performance_schema |
| postBackupHookTest |
| sys                |
+--------------------+
```

So, we can see the database is writable again after the backup.

## Restore

In this section, we are going to demonstrate `preRestore` and `postRestore` hooks. Here, we are going to delete corrupted data in `preRestore` hook and apply some migration on the database in `postRestore` hook.

We have used same name for `RestoreSession`. Make sure you have deleted the old `RestoreSession` before applying new one. You can use the command `kubectl delete restoresession sample-restore -n demo`.

**Simulate Disaster Scenario:**

Now, let's simulate a disaster scenario. Here, we are going to delete the `companyRecord` database before restoring so that we can verify that the data has been restored from backup.

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "DROP DATABASE companyRecord;"
mysql: [Warning] Using a password on the command line interface can be insecure.
```

Verify that the database has been deleted,

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "SHOW DATABASES;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| kubedb_system      |
| mysql              |
| performance_schema |
| postBackupHookTest |
| sys                |
+--------------------+
```

So, we can see from the above output that the database `companyRecord` has been deleted from the MySQL server.

### PreRestore Hook

Here, we are going to configure `preRestore` hook to delete the corrupted database. KubeStash will remove the corrupted database first, then it will restore the database from the backup.

**Create HookTemplate**

Below is the YAML of the `HookTemplate` CR to make the database writable,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: HookTemplate
metadata:
  name: drop-db-hook
  namespace: demo
spec:
  usagePolicy:
    allowedNamespaces:
      from: All
  action:
    exec:
      command:
        - /bin/sh
        - -c
        - mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "DROP DATABASE companyRecord;"
    containerName: mysql # KubeDB uses "mysql" name for MySQL database container. If you haven't used KubeDB, change this according to your setup.
  executor:
    type: Pod
    pod:
      selector: app.kubernetes.io/instance=sample-mysql, app.kubernetes.io/managed-by=kubedb.com, app.kubernetes.io/name=mysqls.kubedb.com
      strategy: ExecuteOnAll
```

Let’s create the above `HookTemplate`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/hooktemplate-post.yaml
hooktemplate.core.kubestash.com/drop-db-hook created
```

**Create RestoreSession:**

Below is the YAML for `RestoreSession` with `preRestore` hook configured to drop the `companyRecord` database before restoring from backup.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: sample-restore
  namespace: demo
spec:
  target:
    apiGroup: kubedb.com
    kind: MySQL
    name: sample-mysql
    namespace: demo
  dataSource:
    repository: demo-repo
    snapshot: latest
    encryptionSecret:
      name: encry-secret
      namespace: demo
  addon:
    name: mysql-addon
    tasks:
      - name: logical-backup-restore
  hooks:
    postRestore:
      - name: pre-hook
        hookTemplate:
          name: drop-db-hook
          namespace: demo
        maxRetry: 3
        timeout: 30s
```

Let's create the above `RestoreSession`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/pre_restore_hook_demo.yaml
restoresession.core.kubestash.com/sample-restore created
```

**Wait for Restore to Complete:**

Now, wait for the restore process to complete,

```bash
$ kubectl get restoresession -n demo -w

NAME             REPOSITORY   FAILURE-POLICY   PHASE     DURATION   AGE
sample-restore   demo-repo                     Succeeded            74m
```

Here, `RestoreSession` phase `Succeeded` means the restore process has been completed successfully.

**Verify Restored Data:**

Verify that the data has been restored successfully,

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "SELECT * FROM companyRecord.employee;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+----------+--------+
| id | name     | salary |
+----+----------+--------+
|  1 | John Doe |   5000 |
+----+----------+--------+
```

So, we can see that the data we had deleted from the `employee` table has been restored.

### PostRestore Hook

Now, let's consider that you want to perform some migration on the database during the restore process. You want to rename the `employee` table into `salaryRecord` as it holds the employee's salary information. You can configure a `postRestore` hook to perform the task automatically.

**Drop Old Database:**

Let's delete the old database `companyRecord` before restoring so that we can verify that the data has been restored from backup.

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "DROP DATABASE companyRecord;"
mysql: [Warning] Using a password on the command line interface can be insecure.
```

Verify that the database has been deleted,

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "SHOW DATABASES;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| kubedb_system      |
| mysql              |
| performance_schema |
| postBackupHookTest |
| sys                |
+--------------------+
```

**Create HookTemplate**

Below is the YAML of the `HookTemplate` CR to perform some migration on the database,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: HookTemplate
metadata:
  name: migration-hook
  namespace: demo
spec:
  usagePolicy:
    allowedNamespaces:
      from: All
  action:
    exec:
      command:
        - /bin/sh
        - -c
        - mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "RENAME TABLE companyRecord.employee TO companyRecord.salaryRecord;"
    containerName: mysql # KubeDB uses "mysql" name for MySQL database container. If you haven't used KubeDB, change this according to your setup.
  executor:
    type: Pod
    pod:
      selector: app.kubernetes.io/instance=sample-mysql, app.kubernetes.io/managed-by=kubedb.com, app.kubernetes.io/name=mysqls.kubedb.com
      strategy: ExecuteOnAll
```

Let’s create the above `HookTemplate`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/hooktemplate-post-rs.yaml
hooktemplate.core.kubestash.com/migration-hook created
```

**Create RestoreSession:**

Below is the YAML of the `RestoreSession` with `postRestore` hook configured to rename the `employee` table into `salaryRecord`.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: sample-restore
  namespace: demo
spec:
  target:
    apiGroup: kubedb.com
    kind: MySQL
    name: sample-mysql
    namespace: demo
  dataSource:
    repository: demo-repo
    snapshot: latest
    encryptionSecret:
      name: encry-secret
      namespace: demo
  addon:
    name: mysql-addon
    tasks:
      - name: logical-backup-restore
  hooks:
    postRestore:
      - name: pre-hook
        hookTemplate:
          name: migration-hook
          namespace: demo
        maxRetry: 3
        timeout: 30s
```

Let's create the above `RestoreSession`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/hooks/backup-and-restore-hooks/examples/post_restore_hook_demo.yaml
restoresession.core.kubestash.com/sample-restore created
```

**Wait for Restore process to Complete:**

Now, wait for the restore process to complete,

```bash
$ kubectl get restoresession -n demo sample-restore -w

NAME             REPOSITORY   FAILURE-POLICY   PHASE     DURATION   AGE
sample-restore   demo-repo                     Succeeded            74m
```

**Verify Restored Data:**

Verify that the `companyRecord` database has been restored and the `employee` table has been renamed to `salaryRecord`.

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "SHOW TABLES IN companyRecord;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+-------------------------+
| Tables_in_companyRecord |
+-------------------------+
| salaryRecord            |
+-------------------------+
```

Let's check `salaryRecord` table contains the original data of the `employee` table,

```bash
$ kubectl exec -it -n demo sample-mysql-0 -- mysql --user=$MYSQL_USER --password=$MYSQL_PASSWORD -e "SELECT * FROM companyRecord.salaryRecord;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+----------+--------+
| id | name     | salary |
+----+----------+--------+
|  1 | John Doe |   5000 |
+----+----------+--------+
```

So, we can see that the `postRestore` hook successfully performed migration on the restored database.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo restoresession sample-restore
kubectl delete -n demo backupconfiguration sample-backup
kubectl delete -n demo backupstorage gcs-storage
kubectl delete -n demo secret gcs-secret
kubectl delete -n demo secret encry-secret
kubectl delete -n demo mysql sample-mysql
kubectl delete -n demo hooktemplate readonly-hook readonly-off-hook migration-hook drop-db-hook
kubectl delete -n demo retentionpolicy demo-retention
```
