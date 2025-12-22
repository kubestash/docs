---
title: "Backup Filtering Options | KubeStash"
description: "Available filtering options for KubeStash backups"
menu_name: docs_{{ .version }}
section_menu_id: guides
product_name: KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-cluster-resources-backup-filtering-options
    name: "Backup Filtering Options"
    parent: kubestash-cluster-resources
    weight: 30
---


### Flags in `manifest-backup` task in KubeDump 

To target specific backup resources, we’ve introduced a set of flags that can be configured under the `spec.sessions.addon.tasks.params` section of the `BackupConfiguration`. 

#### ANDedLabelSelectors:
This flag filters resources based on their labels. You can specify either key-value pairs or just keys. Format with key-value pairs: `key1:value1,key2:value2` or keys only: `key1,key2`.
```yaml 
  Usage: A set of labels, all of which need to be matched to filter the resources.
  Default: ""
  Required: false
  Format : "key1:value1,key2:value2,key3,key4..."
  # or "key1=value1,key2=value2,key3,key4..."
  Example: "app:my-app,db:postgres,db" 
```     
---

#### ORedLabelSelectors:
This flag filters resources based on their labels. You can specify either key-value pairs or just keys. Format with key-value pairs: `key1:value1,key2:value2` or keys only: `key1,key2`.
```yaml 
  Usage: A set of labels, at least one of which need to be matched to filter the resources. 
  Default: ""
  Required: false
  Format : "key1:value1,key2:value2,key3,key4..."
  # or "key1=value1,key2=value2,key3,key4..."
  Example: "app:nginx,app:redis,app"
```
---

#### IncludeClusterResources:
For backing up cluster-scoped resources this flag has to be `true`. Even if resources pass all the other flags, they will still be filtered out if this flag is set to `false`.
```yaml
  Usage: Specify whether to backup cluster scoped resources.
  Default: "false"
  Required: false
  Example: "true" 
``` 

---

#### IncludeNamespaces:
A namespace-scoped resource will be included in the backup **only if** its namespace is listed in this flag, or if the flag is set to the default value `*`.
```yaml
  Usage: Namespaces to include in backup.
  Default: "*"
  Required: false
  Example: "demo,kubedb,kubestash"
```

---

#### ExcludeNamespaces:
A namespace-scoped resource will be excluded from the backup if its namespace is listed in this flag.
```yaml
  Usage: Namespaces to exclude from backup.
  Default: ""
  Required: false
  Example: "default,kube-system"
```

---

#### IncludeResources:
A resource will be included in the backup only if its `resource` or `groupResource` name (in **plural form**) is listed in this flag, or if the flag is set to the default value `*`. 
```yaml
  Usage: Resource types and group resources to include in backup.
  Default: "*"
  Required: false
  Example: "secrets,configmaps,deployments,statefulsets.apps"
```

---

#### ExcludeResources:
A resource will be excluded from the backup if its `resource` or `groupResource` name (in **plural form**) is listed in this flag.
```yaml 
  Usage: Resource types and group resources to exclude from backup.
  Default: ""
  Required: false
  Example: "persistentvolumeclaims,persistentvolumes,pods.metrics.k8s.io,nodes.metrics.k8s.io"
``` 

--- 

### How does Filtering work?

These flags are **independent**, but they are **evaluated together** during backup. A resource will only be included if it satisfies all the applicable filters.

#### For example: 

Consider a deployment named as `my-deployment` in `demo-a` namespace having label `app=my-app`. It will pass the 
filter if the flags are set as followed: 
1. `IncludeResources` contain `deployments` in the list or set to default value `*`.    
2. `ExcludeResources` do not contain `deployments` in the list or set to default value `""`.
3. `IncludeNamespaces` contain `demo-a` in the list or set to default value `*`.
4. `ExcludeNamespaces` do not contain `demo-a` in the list or set to default value `""`.
5. `ANDedLabelSelectors` contain only `app:my-app` in the list or set to default value `""`.
6. `ORedLabelSelectors` contain `app:my-app` in the list or set to default value `""`.
7. `IncludeClusterResources` flag doesn't matter here as `deployments` are not cluster scoped resources. 

#### Conventions of the parameters: 
1. Resource types have to be in `plural` form for `IncludeResources` or `ExcludeResources` flag. 
2. Asterisk `*` indicates `all` and `""` indicates `empty`. 

---

**Example of a `BackupConfiguration` using those flags in the params section:**

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: cluster-resources-backup
  namespace: demo
spec:
...
      addon:
        name: kubedump-addon
        tasks:
          - name: manifest-backup
            params:
              IncludeClusterResources: "true"
              IncludeNamespaces: "demo-a,demo-b"
              ExcludeNamespaces: "kube-system,longhorn-system"
              IncludeResources: "*"
              ORedLabelSelectors: "app:my-app,app:my-sts"
        jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader-writer
```

Here,
- `spec.sessions[*].addon.name` specifies the name of the `Addon`.
- `spec.sessions[*].addon.tasks[*].name` specifies the name of the backup task.
- `spec.sessions[*].addon.jobTemplate.spec.serviceAccountName`specifies the ServiceAccount name that we have created earlier with cluster-wide resource reading permission.