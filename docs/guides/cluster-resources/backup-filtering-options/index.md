---
title: Backup Filtering Options | KubeStash
description: 
menu:
  docs_{{ .version }}:
    identifier: kubestash-cluster-backup-filtering-options
    name: Backup Filtering Options
    parent: kubestash-cluster-resources
    weight: 40
product_name: KubeStash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

### Flags in `manifest-backup` task in KubeDump 

We have introduced some flags for filtering resources while taking backup.  

#### ANDedLabelSelectors:
```yaml 
  Usage: A set of labels, all of which need to be matched
  to filter the resources.
  Default: ""
  Required: false
  Format : "key1:value1,key2:value2,key3,key4..." or `"key1=value1,key2=value2,key3,key4..."`
  Example: "app:my-app,db:postgres,db" 
```     
> If the filter is set to `"key1:value1,key2:value2,key3"` then to pass the filter resources labels has to be something like `"key1:value1,key2:value2,key3:vlaue3, ..."` or `"key1:value1,key2:value2,key3, ..."`. Order of the lables doesn't matter.

---

#### ORedLabelSelectors:
```yaml 
  Usage: A set of labels, at least one of which need to 
  be matched to filter the resources. 
  Default: ""
  Required: false
  Format : "key1:value1,key2:value2,key3,key4..." or `"key1=value1,key2=value2,key3,key4..."`
  Example: "app:nginx,app:redis,app"
```
> If the filter is set to `"key1:value1,key2:value2,key3"` then to pass the filter resources labels has to be something like `"key1:value1, ..."` or `"...,key2:value2, ..."` or `"...,key3:value3, ..."` or `"...,key3, ..."`. Order of the labels doesn't matter.

---

#### IncludeClusterResources:
```yaml
  Usage: Specify whether to backup
cluster scoped resources.
  Default: "false"
  Required: false
  Example: "true" 
``` 
> For backing up cluster scoped resources this flag has to be true. Even if resources pass all the other flags it will be filtered out if this flag is set to false.

---

#### IncludeNamespaces:
```yaml
  Usage: Namespaces to include in backup.
  Default: "*"
  Required: false
  Example: "demo,kubedb,kubestash"
```
> A namespace scoped resource will pass the filter if and only if this flag listed it's namespace or the flag set to default value `*`. That means, if the `IncludeNamespaces` flag contains a list of namespaces like `"namespace-a,namespace-b,namespace-c,..."` then for any namespace scoped resource if it's namespace is not listed in the `IncludeNamespaces` flag then it'll be removed from backup.

---

#### ExcludeNamespaces:
```yaml
  Usage: Namespaces to exclude from backup.
  Default: ""
  Required: false
  Example: "default,kube-system"
```
> If this flag is set to `"namespace1,namespace2,namespace3..."` any resources within those namespaces won't be included in backup.

---

#### IncludeResources:
```yaml
  Usage: Resource types and group resources to include in backup.
  Default: "*"
  Required: false
  Example: "secrets,configmaps,deployments,statefulsets.apps"
```
   
> A resource will pass the filter if and only if this flag listed it's `resource/groupResource` name or the flag set to default value `*`. That means, if the `IncludeResources` flag contains a list of resources like `"resource-a,resource-b,groupResource-a,groupResource-b..."` then for any resource if it's resource/groupResource name is not listed in the `IncludeResources` flag then it won't be included in backup.
---

#### ExcludeResources:
```yaml 
  Usage: Resource types and group resources to exclude from backup
  Default: ""
  Required: false
  Example: "persistentvolumeclaims,persistentvolumes,pods.metrics.k8s.io,metrics.k8s.io"
``` 
> If this flag is set to `"resource-a,resource-b,groupResource-a,groupResource-b"` then all these listed `resources/groupResources` won't be included in backup.
--- 

These flags are **independent**, but they are **evaluated together** during backup. A resource will only be included if it satisfies all the applicable filters.

For example: 

Consider a deployment named as `my-deployment` in `demo-a` namespace having label `app=my-app`. It will pass the 
filter if the flags are set as followed: 
1. `IncludeResources` contain `deployments` in the list or set to default value `*`.    
2. `ExcludeResources` do not contain `deployments` in the list or set to default value `""`.
3. `IncludeNamespaces` contain `demo-a` in the list or set to default value `*`.
4. `ExcludeNamespaces` do not contain `demo-a` in the list or set to default value `""`.
5. `ANDedLabelSelectors` contain only `app:my-app` in the list or set to default value `""`.
6. `ORedLabelSelectors` contain `app:my-app` in the list or set to default value `""`.
7. `IncludeClusterResources` flag doesn't matter here as `deployments` are not cluster scoped resources. 

Conventions that're followed in the parameters: 
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
---