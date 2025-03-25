---
title: Backup/Restore Customization | KubeStash
description: A guide on how to customize backup/restore process in KubeStash.
menu:
  docs_{{ .version }}:
    identifier: use-cases-customize-backup-restore
    name: Customize Backup/Restore
    parent: use-cases
    weight: 70
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Customizing the Backup and Restore Process

This guide will show you how you can customize backup and restore processes in Stash according to your needs.

> Note: YAML files used in this tutorial are stored [here](https://github.com/kubestash/docs/tree/{{< param "info.version" >}}/docs/guides/use-cases/customize-backup-restore/examples).

## Customizing Backup Process

In this section, we are going to show you how to customize the backup process. Here, we are going to show some examples of providing arguments to the backup process, running the backup process as a specific user, using a volume for restic cache, etc.

### Passing include/exclude paths to the backup process

We can send a list of backup paths and a list of pattern for the files that should be ignored during backup.

### Using multiple backends

You can configure multiple backends within a single backupConfiguration. To back up the same data to different backends, such as `S3` and `GCS`, declare each backend in the `.spe.backends` section. Then, reference these backends in the `.spec.sessions[*].repositories` section.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: multiple-backend
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name: kubestash-demo
    namespace: demo
  backends:
    - name: gcs-backend
      storageRef:
        name: gcs-storage
        namespace: demo
      retentionPolicy:
        name: demo-retention
        namespace: demo
    - name: s3-backend
      storageRef:
        namespace: demo
        name: s3-storage
      retentionPolicy:
        name: demo-retention
        namespace: demo
  sessions:
    - name: demo-session
      scheduler:
        schedule: "*/5 * * * *"
      repositories:
        - name: gcs-demo-repo
          backend: gcs-backend
          directory: /dep-s3
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
        - name: s3-demo-repo
          backend: s3-backend
          directory: /dep-gcs
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: workload-addon
        tasks:
          - name: logical-backup
```

### Passing a `PVC` name or `volumeClaimTemplate` as restic cache volume

You can specify a Restic cache volume using either an existing PVC name or a volumeClaimTemplate:

- Use `addon.tasks[*].addonVolumes.source.PersistentVolumeClaim.claimName` to provide an existing PVC name.
- Use `addon.tasks[*].addonVolumes.volumeClaimTemplate` to define a volumeClaimTemplate.

Here is an example of passing `volumeClaimTemplate` during in `BackupConfiguration`:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: passing-params
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name: kubestash-demo
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
    - name: demo-session
      scheduler:
        schedule: "*/5 * * * *"
      repositories:
        - name: gcs-demo-repo
          backend: gcs-backend
          directory: /dep
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: workload-addon
        tasks:
          - name: logical-backup
            addonVolumes:
              - name: ${RESTIC_CACHE_VOLUME}
                source:
                  volumeClaimTemplate:
                     spec:
                       accessModes:
                       - ReadWriteOnce
                       resources:
                         requests:
                           storage: 1Gi
```
Here,
- `addon.tasks[*].addonVolumes[*].source.volumeClaimTemplate` dynamically creates a volume for use as the Restic cache during backup.

> Note: The placeholder `${RESTIC_CACHE_VOLUME}` is automatically resolved at runtime, ensuring the correct volume is created.


### Passing Include/Exclude Paths for Backup

We can define a list of paths to include via `spec.sessions[*].addon.tasks[*}.params.paths` in the backup and specify patterns for files that should be excluded via `spec.sessions[*].addon.tasks[*}.params.exclude` during the process.

Here is an example of passing `paths` and `exclude` parameters in `BackupConfiguration`:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: passing-params
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name: kubestash-demo
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
    - name: demo-session
      scheduler:
        schedule: "*/5 * * * *"
      repositories:
        - name: gcs-demo-repo
          backend: gcs-backend
          directory: /dep
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: workload-addon
        tasks:
          - name: logical-backup
            params:
              paths: /source/data,/source/config
              exclude: /source/data/lost+found,/source/config/lost+found
```
Here,
- `addon.tasks[*].params.paths` specifies the targeted paths during the backup process.
- `addon.tasks[*].params.exclude` specifies the list of  paths that should be ignored during backup.

### Running backup Container as a specific user

If your cluster requires running the backup job as a specific user, you can provide `securityContext` under `runtimeSettings.container` section. 

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: specific-user
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name: kubestash-demo
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
    - name: demo-session
      scheduler:
        schedule: "*/5 * * * *"
      repositories:
        - name: gcs-demo-repo
          backend: gcs-backend
          directory: /dep
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: workload-addon
        jobTemplate:
          spec:
            securityContext:
              runAsUser: 0
              runAsGroup: 0
        tasks:
          - name: logical-backup
```


### Specifying Memory/CPU limit/request for the backup job

If you want to specify the `Memory/CPU` limit/request for your backup job, you can specify `resources` field under `addon.jobTemplate.spec` section.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: resources-request-limit
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name: kubestash-demo
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
    - name: demo-session
      scheduler:
        schedule: "*/5 * * * *"
      repositories:
        - name: gcs-demo-repo
          backend: gcs-backend
          directory: /dep
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
      addon:
        name: workload-addon
        jobTemplate:
          spec:
            resources:
              requests:
                cpu: "200m"
                memory: "1Gi"
              limits:
                cpu: "200m"
                memory: "1Gi"
        tasks:
          - name: logical-backup
```

## Customizing Restore Process

In this section, we are going to show you how to customize the restore process. Here, we are going to show some examples of providing arguments to the restore process, running the restore process as a specific user, using a volume for restic cache, etc.

### Passing a `PVC` name or `volumeClaimTemplate` as restic cache volume

You can specify a Restic cache volume using either an existing PVC name or a volumeClaimTemplate:

- Use `addon.tasks[*].addonVolumes.source.PersistentVolumeClaim.claimName` to provide an existing PVC name.
- Use `addon.tasks[*].addonVolumes.volumeClaimTemplate` to define a volumeClaimTemplate.

Here is an example of passing `volumeClaimTemplate` during in `RestoreSession`:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: restic-cache-volume
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: StatefulSet
    namespace: demo
    name: kubestash-restore-statefulset
  dataSource:
    repository: statefulset-demo-gcs
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret # some addon may not support encryption
      namespace: demo
  addon:
    name: workload-addon
    tasks:
      - name: logical-backup-restore
        addonVolumes:
          - name: ${RESTIC_CACHE_VOLUME}
            source:
              volumeClaimTemplate:
                spec:
                  accessModes:
                    - ReadWriteOnce
                  resources:
                    requests:
                      storage: 1Gi
```
Here,
- `addon.tasks[*].addonVolumes[*].source.volumeClaimTemplate` dynamically creates a volume for use as the Restic cache during restore.

> Note: The placeholder `${RESTIC_CACHE_VOLUME}` is automatically resolved at runtime, ensuring the correct volume is created.


### Passing Include/Exclude Parameters for Restore

We can define a list of paths to include via `spec.addon.tasks[*}.params.include` in the restore and specify patterns for files that should be excluded via `spec.addon.tasks[*}.params.exclude` during the restore process.

Here is an example of passing `include` and `exclude` parameters in `RestoreSession`:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: passing-params
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: StatefulSet
    namespace: demo
    name: kubestash-restore-statefulset
  dataSource:
    repository: statefulset-demo-gcs
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret # some addon may not support encryption
      namespace: demo
  addon:
    name: workload-addon
    tasks:
    - name: logical-backup-restore
      params:
        exclude: /source/config
        include: /source/data
```

Here,
- `addon.tasks[*].params.include` specifies the list of patterns for the files that should be restored.
- `addon.tasks[*].params.exclude` specifies the list of patterns for the files that should be ignored during restore.


### Restore specific snapshot

You can also restore a specific snapshot. At first, list the available snapshot as bellow,

```bash
```

The below example shows how you can pass a specific snapshot name in `.dataSource` section.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: specific-snapshot
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: StatefulSet
    namespace: demo
    name: kubestash-restore-statefulset
  dataSource:
    repository: statefulset-demo-gcs
    snapshot: 
    encryptionSecret:
      name: encrypt-secret # some addon may not support encryption
      namespace: demo
  addon:
    name: workload-addon
    tasks:
      - name: logical-backup-restore
```


### Running restore job as a specific user

Similar to the backup process under the `addon.jobTemplate.spec` you can provide `securityContext` to run the restore job as a specific user.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: specific-user
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: StatefulSet
    namespace: demo
    name: kubestash-restore-statefulset
  dataSource:
    repository: statefulset-demo-gcs
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret # some addon may not support encryption
      namespace: demo
  addon:
    name: workload-addon
    jobTemplate:
      spec:
        securityContext:
          runAsUser: 0
          runAsGroup: 0
    tasks:
      - name: logical-backup-restore
```

### Specifying `Memory/CPU` limit/request for the restore job

Similar to the backup process, you can also provide resources field under the `addon.jobTemplate.spec.resources` section to limit the `Memory/CPU` for your restore job.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: resources-request-limit
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: StatefulSet
    namespace: demo
    name: kubestash-restore-statefulset
  dataSource:
    repository: statefulset-demo-gcs
    snapshot: latest
    encryptionSecret:
      name: encrypt-secret # some addon may not support encryption
      namespace: demo
  addon:
    name: workload-addon
    jobTemplate:
      spec:
        resources:
          requests:
            cpu: "200m"
            memory: "1Gi"
          limits:
            cpu: "200m"
            memory: "1Gi"
    tasks:
      - name: logical-backup-restore
```

> Note: You can configure additional runtime settings for restore jobs within the `addon.jobTemplate.spec` sections. For further details, please refer to the [reference](https://kubestash.com/docs/v2025.3.24/concepts/crds/restoresession/#podtemplate-spec).