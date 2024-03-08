---
title: Backup Customization | KubeStash
description: Customizing Backup Process with KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-kubedump-customization
    name: Customizing Backup Process
    parent: kubestash-kubedump
    weight: 50
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Customizing Backup Process

KubeStash provides rich customization supports for the backup and restores process to meet the requirements of various cluster configurations. This guide will show you some examples of these customizations.

In this section, we are going to show you how to customize the backup process. Here, we are going to show some examples of filtering resources using a label selector, running the backup process as a specific user, etc.

> Note: YAML files used in this tutorial are stored [here](https://github.com/kubestash/docs/tree/{{< param "info.version" >}}/docs/addons/kubedump/customization/examples).

### Filtering resources

You can use a label selector to backup the YAML for the resources that have particular labels. You just have to pass the `labelSelector` parameter under `task.params` section with your desired label selector.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: kube-system-backup
  namespace: demo
spec:
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
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: gcs-repository
          backend: gcs-backend
          directory: /kube-system-manifests
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
          deletionPolicy: WipeOut
      addon:
        name: kubedump-addon
        tasks:
          - name: manifest-backup
            params:
              labelSelector: "k8s-app=kube-dns"
        jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader
```

The above backup process will backup only the resources that has `k8s-app: kube-dns` label. Here, is a sample of the resources backed up by the above BackupConfiguration.

```bash
$ tree /home/anisur/Downloads/kubestash/label-selector
/home/anisur/Downloads/kubestash/label-selector
└── gcs-repository-kube-system-backup-frequent-backup-1708926900
    └── manifest
        └── tmp
            └── manifest
                └── namespaces
                    └── kube-system
                        ├── Deployment
                        │   └── coredns.yaml
                        ├── Endpoints
                        │   └── kube-dns.yaml
                        ├── EndpointSlice
                        │   └── kube-dns-nv9px.yaml
                        ├── Pod
                        │   ├── coredns-565d847f94-78r86.yaml
                        │   └── coredns-565d847f94-zdtcs.yaml
                        ├── ReplicaSet
                        │   └── coredns-565d847f94.yaml
                        └── Service
                            └── kube-dns.yaml

12 directories, 7 files
```

### Passing arguments

You can pass arguments to the backup process using `task.params` section.

The following example shows how passes `sanitize` argument value to `false` which tells the backup process not to remove decorators (i.e. `status`, `managedFields` etc.) from the YAML files.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: application-manifest-backup
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: Deployment
    name:  kubestash-kubestash-operator
    namespace: kubestash
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
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: gcs-repository
          backend: gcs-backend
          directory: /deployment-manifests
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
          deletionPolicy: WipeOut
      addon:
        name: kubedump-addon
        tasks:
          - name: manifest-backup
            params:
              sanitize: "false"
        jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader
```

### Running backup job as a specific user

If your cluster requires running the backup job as a specific user, you can provide `securityContext` under `runtimeSettings.pod` section. The below example shows how you can run the backup job as the root user.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: kube-system-backup
  namespace: demo
spec:
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
        schedule: "*/2 * * * *"
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: gcs-repository
          backend: gcs-backend
          directory: /kube-system-manifests
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
          deletionPolicy: WipeOut
      addon:
        name: kubedump-addon
        tasks:
          - name: manifest-backup
        jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader
            securityContext:
              runAsUser: 0
              runAsGroup: 0
```

### Specifying Memory/CPU limit/request for the backup job

If you want to specify the Memory/CPU limit/request for your backup job, you can specify `resources` field under `runtimeSettings.container` section.

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: kube-system-backup
  namespace: demo
spec:
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
        schedule: "*/2 * * * *"
        jobTemplate:
          backoffLimit: 1
      repositories:
        - name: gcs-repository
          backend: gcs-backend
          directory: /kube-system-manifests
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
          deletionPolicy: WipeOut
      addon:
        name: kubedump-addon
        tasks:
          - name: manifest-backup
            params:
              labelSelector: "k8s-app=kube-dns"
        jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader
            resources:
              requests:
                cpu: "200m"
                memory: "1Gi"
              limits:
                cpu: "200m"
                memory: "1Gi"
```