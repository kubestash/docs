---
title: Restore Filtering Options | KubeStash
description: 
menu:
  docs_{{ .version }}:
    identifier: kubestash-cluster-restore-filtering-options
    name: Restore Filtering Options
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
  Format : "key1:value1,key2:value2,key3,key4..." or "key1=value1,key2=value2,key3,key4..."
  Example: "app:my-app,db:postgres,db" 
```     
> If the filter is set to `"key1:value1,key2:value2,key3"` then to pass the filter resources labels has to be something like `"key1:value1,key2:value2,key3:vlaue3,..."` or `"key1:value1,key2:value2,key3,..."`. Order of the lables doesn't matter.

---

#### ORedLabelSelectors:
```yaml 
  Usage: A set of labels, at least one of which need to 
  be matched to filter the resources. 
  Default: ""
  Required: false
  Format : "key1:value1,key2:value2,key3,key4..." or "key1=value1,key2=value2,key3,key4..."
  Example: "app:nginx,app:redis,app"
```
> If the filter is set to `"key1:value1,key2:value2,key3"` then to pass the filter resources labels has to be something like `"key1:value1, ..."` or `"...,key2:value2,..."` or `"...,key3:value3,..."` or `"...,key3,..."`. Order of the labels doesn't matter.

---

#### IncludeClusterResources:
```yaml
  Usage: Specify whether to backup
cluster scoped resources.
  Default: "false"
  Required: false
  Example: "true" 
``` 
> For cluster scoped resources this flag has to be true. Even if resources pass all the other flags it will be filtered out if this flag is set to false.

---

#### IncludeNamespaces:
```yaml
  Usage: Namespaces to include in backup.
  Default: "*"
  Required: false
  Example: "demo,kubedb,kubestash"
```
> A namespace scoped resource will pass the filter if and only if this flag listed it's namespace or the flag set to default value `*`. That means, if the `IncludeNamespaces` flag contains a list of namespaces like `"namespace-a,namespace-b,namespace-c,..."` then for any namespace scoped resource if it's namespace is not listed in the `IncludeNamespaces` flag then it'll be removed from restoration.

---

#### ExcludeNamespaces:
```yaml
  Usage: Namespaces to exclude from backup.
  Default: ""
  Required: false
  Example: "default,kube-system"
```
> If this flag is set to `"namespace1,namespace2,namespace3..."` any resources within those namespaces won't be included in restoration.

---

#### IncludeResources:
```yaml
  Usage: Resource types and group resources to include in backup.
  Default: "*"
  Required: false
  Example: "secrets,configmaps,deployments,statefulsets.apps"
```
   
> A resource will pass the filter if and only if this flag listed it's `resource/groupResource` name or the flag set to default value `*`. That means, if the `IncludeResources` flag contains a list of resources like `"resource-a,resource-b,groupResource-a,groupResource-b..."` then for any resource if it's `resource/groupResource` name is not listed in the `IncludeResources` flag then it won't be included in restoration.
---

#### ExcludeResources:
```yaml 
  Usage: Resource types and group resources to exclude from backup
  Default: ""
  Required: false
  Example: "persistentvolumeclaims,persistentvolumes,pods.metrics.k8s.io,metrics.k8s.io"
``` 
> If this flag is set to `"resource-a,resource-b,groupResource-a,groupResource-b"` then all these listed `resources/groupResources` won't be included in restoration.

--- 

#### RestorePVs:
```yaml 
  Usage: Specify whether to restore PersistentVolumes
  Default: "false"
  Required: false
  Example: "true"
``` 

--- 

#### StorageClassMappings:
```yaml 
  Usage: Mapping of old to new storage classes
  Default: ""
  Required: false
  Format: "oldStorageClass1=newStorageClass1,oldStorageClass2=newStorageClass2,..."
  Example: "gp2=ebs-sc,standard=fast-storage"
``` 
>This flag is used to remap PersistentVolume `StorageClasses` during restore. For example, if a PV in the backup used the `gp2` StorageClass but the restore cluster does not have `gp2`, you can provide a mapping like `gp2=ebs-sc`. This ensures the PVs are restored with a valid StorageClass in the target cluster. Multiple mappings can be defined as a comma-separated list.

---

These flags are **independent**, but they are **evaluated together** during restore. A resource will only be included if it satisfies all the applicable filters.

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

**Example of a `RestoreSession` using those flags in the params section:**

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: cluster-resources-restore
  namespace: demo
spec:
...
  addon:
    name: kubedump-addon
    tasks:
      - name: manifest-restore
        params:
          IncludeClusterResources: "true"
          ExcludeNamespaces: "demo-a"
          ExcludeResources: "nodes.metrics.k8s.io,nodes,pods.metrics.k8s.io,metrics.k8s.io,endpointslices.discovery.k8s.io"
    jobTemplate:
          spec:
            serviceAccountName: cluster-resource-reader-writer
```
---