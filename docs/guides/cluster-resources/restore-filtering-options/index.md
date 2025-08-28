---
title: "Restore Filtering Options | KubeStash"
description: "Available filtering options for KubeStash restores"
menu_name: docs_{{ .version }}
section_menu_id: guides
product_name: KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-cluster-restore-filtering-options
    name: "Restore Filtering Options"
    parent: kubestash-cluster-resources
    weight: 25
---

### Flags in `manifest-restore` task in KubeDump 

To target specific resources in the restore process, we’ve introduced a set of flags that can be configured under the `spec.sessions.addon.tasks.params` section of the `RestoreSession`.

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
For restoring up cluster-scoped resources this flag has to be `true`. Even if resources pass all the other flags, they will still be filtered out if this flag is set to `false`.
```yaml
  Usage: Specify whether to restore cluster scoped resources.
  Default: "false"
  Required: false
  Example: "true" 
``` 

---

#### IncludeNamespaces:
A namespace-scoped resource will be restored **only if** its namespace is listed in this flag, or if the flag is set to the default value `*`.
```yaml
  Usage: Namespaces to include in the restore process.
  Default: "*"
  Required: false
  Example: "demo,kubedb,kubestash"
```

---

#### ExcludeNamespaces:
A namespace-scoped resource will be excluded from the restore process if its namespace is listed in this flag.
```yaml
  Usage: Namespaces to exclude from the restore process.
  Default: ""
  Required: false
  Example: "default,kube-system"
```

---

#### IncludeResources:
A resource will be included in the restore process only if its `resource` or `groupResource` name (in **plural form**) is listed in this flag, or if the flag is set to the default value `*`. 
```yaml
  Usage: Resource types and group resources to include in the restore process.
  Default: "*"
  Required: false
  Example: "secrets,configmaps,deployments,statefulsets.apps"
```
   
---

#### ExcludeResources:
A resource will be excluded from the restore process if its `resource` or `groupResource` name (in **plural form**) is listed in this flag.
```yaml 
  Usage: Resource types and group resources to exclude from the restore process.
  Default: ""
  Required: false
  Example: "persistentvolumeclaims,persistentvolumes,pods.metrics.k8s.io,nodes.metrics.k8s.io"
``` 

--- 

#### OverrideResources:
This flag controls whether existing resources in the cluster should be **overwritten** during the restore process.
```yaml
Usage: Specify whether to override existing resources while restoring.
Default: "false"
Required: false
Example: "true"
```
---

#### RestorePVs:
```yaml 
  Usage: Specify whether to restore PersistentVolumes.
  Default: "false"
  Required: false
  Example: "true"
``` 

--- 

#### StorageClassMappings:
This flag is used to remap PersistentVolume `StorageClasses` during restore. For example, if a PV in the backup used the `gp2` StorageClass but the restore cluster does not have `gp2`, you can provide a mapping like `gp2=ebs-sc`. This ensures the PVs are restored with a valid StorageClass in the target cluster. Multiple mappings can be defined as a comma-separated list.
```yaml 
  Usage: Mapping of old to new storage classes.
  Default: ""
  Required: false
  Format: "oldStorageClass1=newStorageClass1,oldStorageClass2=newStorageClass2,..."
  Example: "gp2=ebs-sc,standard=fast-storage"
``` 

---

### How does Filtering work?

These flags are **independent**, but they are **evaluated together** during restore process. A resource will only be included if it satisfies all the applicable filters.

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