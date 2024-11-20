---
title: Backup Verification Examples | KubeStash
menu:
  docs_{{ .version }}:
    identifier: verification-configuring
    name: Configuring Verification
    parent: verification
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Configuring Different Types of Verification Strategies

In this guide, we are going to discuss how to configure different types of verification strategies in `BackupVerifier`. Here, we will give some examples of different configurations.

## `RestoreOnly` Verification

For `RestoreOnly` verification, KubeStash operator will initiate a `RestoreSession` using the addon information specified in `BackupVerifier`. The verification of the backup will rely on the status of the `RestoreSession` phase; if the restore completes successfully, the backup is considered verified.
Here is an example of `BackupVerifier` with `RestoreOnly` verification enabled:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupVerifier
metadata:
  name: mysql-verifier
  namespace: demo
spec:
  restoreOption:
    target:
      apiGroup: kubedb.com
      kind: MySQL
      name: sample-mysql
      namespace: verify
    addonInfo:
      name: mysql-addon
      tasks:
      - name: logical-backup-restore
  sessionHistoryLimit: 2
  schedule: "*/5 * * * *"
  type: RestoreOnly
```

## `Query` Verification

For `Query` verification, KubeStash operator will initiate a `RestoreSession` first and after successful restore, it will create a verifier job to run the queries provided in `BackupVerifier`.
Here is an example of `BackupVerifier` with `Query` verification enabled:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupVerifier
metadata:
  name: mysql-query-verifier
  namespace: demo
spec:
  function: query-verification
  restoreOption:
    target:
      apiGroup: kubedb.com
      kind: MySQL
      name: sample-mysql
      namespace: verify
    addonInfo:
      name: mysql-addon
      tasks:
      - name: logical-backup-restore
  sessionHistoryLimit: 2
  schedule: "*/5 * * * *"
  type: Query
  query:
    mySQL:
    - database: shop
      table: employee
      rowCount:
        operator: GreaterThanOrEqual
        value: 100
    - database: playground
      table: equipment
      rowCount:
        operator: GreaterThan
        value: 200
```

Here, the `spec.query` field in `BackupVerifier` enables you to define query checks for specific applications, such as KubeDB-managed `MySQL` databases. These checks verify the structure and data integrity of the restored database, ensuring that specific tables and row counts are as expected. For KubeDB-managed `MySQL`, all query checks are specified under the `mySQL` field. You can configure the following parameters for each query:
- **database :** The name of the database to verify. This parameter allows KubeStash to check if the specified database exists after the restore operation.
- **table :** The name of a table within the specified database. This enables verification of the table’s existence within the database, ensuring the correct database structure is restored.
- **rowCount :** The expected number of rows in the specified table. This parameter allows you to define an operator (Equal, LessThan, GreaterThan, GreaterThanOrEqual, etc.) and a value to verify if the restored table has the required number of rows. This check helps confirm data consistency within the restored table.

Here are the examples of query parameters for KubeDB-managed databases:

#### MySQL

```yaml
query:
  mySQL:
  - database: shop
    table: employee
    rowCount:
      operator: GreaterThanOrEqual
      value: 100
```

Each query may have the following parameters:
- **database :** The name of the database to verify. This parameter allows KubeStash to check if the specified database exists after the restore operation.
- **table :** The name of a table within the specified database. This enables verification of the table’s existence within the database, ensuring the correct database structure is restored.
- **rowCount :** The expected number of rows in the specified table. This parameter allows you to define an operator (Equal, LessThan, GreaterThan, GreaterThanOrEqual, etc.) and a value to verify if the restored table has the required number of rows. This check helps confirm data consistency within the restored table.

#### MariaDB

```yaml
query:
  mariaDB:
  - database: shop
    table: employee
    rowCount:
      operator: GreaterThanOrEqual
      value: 100
```

Each query may have the following parameters:
- **database :** The name of the database to verify. This parameter allows KubeStash to check if the specified database exists after the restore operation.
- **table :** The name of a table within the specified database. This enables verification of the table’s existence within the database, ensuring the correct database structure is restored.
- **rowCount :** The expected number of rows in the specified table. This parameter allows you to define an operator (Equal, LessThan, GreaterThan, GreaterThanOrEqual, etc.) and a value to verify if the restored table has the required number of rows. This check helps confirm data consistency within the restored table.

#### Postgres

```yaml
query:
  postgres:
  - database: shop
    schema: primary
    table: employee
    rowCount:
      operator: GreaterThanOrEqual
      value: 100
```

Each query may have the following parameters:
- **database :** The name of the database to verify. This parameter allows KubeStash to check if the specified database exists after the restore operation.
- **schema :** The schema name that will be checked for existence in specified Database.
- **table :** The name of a table within the specified database. This enables verification of the table’s existence within the database, ensuring the correct database structure is restored.
- **rowCount :** The expected number of rows in the specified table. This parameter allows you to define an operator (Equal, LessThan, GreaterThan, GreaterThanOrEqual, etc.) and a value to verify if the restored table has the required number of rows. This check helps confirm data consistency within the restored table.

#### MongoDB

```yaml
query:
  mongoDB:
  - database: shop
    collection: employee
    documentCount:
      operator: GreaterThanOrEqual
      value: 100
```

Each query may have the following parameters:
- **database :** The name of the database to verify. This parameter allows KubeStash to check if the specified database exists after the restore operation.
- **collection :** The name of a collection within the specified database. This enables verification of the collection’s existence within the database, ensuring the correct database structure is restored.
- **documentCount :** The expected number of documents in the specified collection. This parameter allows you to define an operator (Equal, LessThan, GreaterThan, GreaterThanOrEqual, etc.) and a value to verify if the restored collection has the required number of documents. This check helps confirm data consistency within the restored collection.

#### Elasticsearch

```yaml
query:
  elasticsearch:
  - index: shop
```

Each query may have the following parameters:
- **index :** The name of the index to verify. This parameter allows KubeStash to check if the specified index exists after the restore operation.

#### Redis

```yaml
query:
  redis:
  - index: 0
    dbSize: 100
```

Each query may have the following parameters:
- **index :** Index refers to the database index being checked for existence after the restore operation.
- **dbSize :** DbSize specifies the number of keys in the specified Database.

#### Singlestore

```yaml
query:
  singlestore:
  - database: shop
    table: employee
    rowCount:
      operator: GreaterThanOrEqual
      value: 100
```

Each query may have the following parameters:
- **database :** The name of the database to verify. This parameter allows KubeStash to check if the specified database exists after the restore operation.
- **table :** The name of a table within the specified database. This enables verification of the table’s existence within the database, ensuring the correct database structure is restored.
- **rowCount :** The expected number of rows in the specified table. This parameter allows you to define an operator (Equal, LessThan, GreaterThan, GreaterThanOrEqual, etc.) and a value to verify if the restored table has the required number of rows. This check helps confirm data consistency within the restored table.

#### MSSQLServer

```yaml
query:
  mssqlServer:
  - database: shop
    schema: dob
    table: employee
    rowCount:
      operator: GreaterThanOrEqual
      value: 100
```

Each query may have the following parameters:
- **database :** The name of the database to verify. This parameter allows KubeStash to check if the specified database exists after the restore operation.
- **schema :** The schema name that will be checked for existence in specified Database.
- **table :** The name of a table within the specified database. This enables verification of the table’s existence within the database, ensuring the correct database structure is restored.
- **rowCount :** The expected number of rows in the specified table. This parameter allows you to define an operator (Equal, LessThan, GreaterThan, GreaterThanOrEqual, etc.) and a value to verify if the restored table has the required number of rows. This check helps confirm data consistency within the restored table.

## `Script` Verification

For `Script` verification, KubeStash operator will initiate a `RestoreSession` first and after successful restore, it will create a verifier job to run the script provided in `BackupVerifier`.
We need to provide the script in a `ConfigMap` and mount this configmap in the verifier job. Here is an example of `ConfigMap` and `BackupVerifier` with `Script` verification enabled:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
  namespace: verify
data:
  check.sh: |
    # Execute SQL query to get table name
    export MYSQL_PWD=$MYSQL_ROOT_PASSWORD
    table_name=$(mysql -uroot -e "SELECT table_name FROM information_schema.tables WHERE table_schema = 'shop'AND table_name = 'employee';")
    # Check if the table name exists
    if [[ -z "$table_name" ]]; then
        echo "Table does not exist in the shop database."
        exit 1
    else
       echo "Table name in the shop database:"
       echo "$table_name"
    fi
    
---

apiVersion: core.kubestash.com/v1alpha1
kind: BackupVerifier
metadata:
  name: mysql-script-verifier
  namespace: demo
spec:
  function: script-verification
  restoreOption:
    target:
      apiGroup: kubedb.com
      kind: MySQL
      name: sample-mysql
      namespace: verify
    addonInfo:
      name: mysql-addon
      tasks:
      - name: logical-backup-restore
  sessionHistoryLimit: 2
  schedule: "*/5 * * * *"
  volumes:
  - name: config-vol
    configMap:
      name: config
  volumeMounts:
  - name: config-vol
    mountPath: /tmp/config
  type: Script
  script:
    location: /tmp/config/check.sh
```

