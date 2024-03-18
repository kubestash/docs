---
title: RetentionPolicy Overview
menu:
  docs_{{ .version }}:
    identifier: retentionpolicy-overview
    name: RetentionPolicy
    parent: crds
    weight: 45
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# RetentionPolicy

## What is RetentionPolicy

A `RetentionPolicy` is a Kubernetes `CustomResourceDefinition`(CRD) which specifies how the old Snapshots should be cleaned up. 

You have to create at least one `RetentionPolicy` object and refer the `RetentionPolicy` in the `BackupConfiguration`.

## RetentionPolicy CRD Specification
Like any official Kubernetes resource, a `RetentionPolicy` has `TypeMeta`, `ObjectMeta` and `Spec` sections. However, unlike other Kubernetes resources, it does not have a `Status` section.

A sample `RetentionPolicy` object is shown below,
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
      from: Same
```
Here, we are going to describe the various sections of the `RetentionPolicy` crd.

## RetentionPolicy `Spec`
A `RetentionPolicy` object has the following fields in the `spec` section:

#### spec.maxRetentionPeriod
MaxRetentionPeriod specifies a duration up to which the old Snapshots should be kept. KubeStash will remove all the Snapshots that are older than the MaxRetentionPeriod. For example, MaxRetentionPeriod of `30d` will keep only the Snapshots of last 30 days.

Sample duration format:
- years: 	`2y`
- months: 	`6mo`
- days: 	`30d`
- hours: 	`12h`
- minutes: 	`30m`

You can also combine the above durations. For example: `30d12h30m`

#### spec.usagePolicy
UsagePolicy lets you control which namespaces are allowed to use the `RetentionPolicy` and which are not. If you refer to a `RetentionPolicy` from a restricted namespace, KubeStash will reject creating the respective `BackupConfiguration` from validating webhook. You can use the `usagePolicy` to allow only the same namespace, a subset of namespaces, or all the namespaces to refer to the `RetentionPolicy`. If you don’t specify any `usagePolicy`, KubeStash will allow referencing the `RetentionPolicy` only from the namespace where it was created.

Here is an example of `spec.usagePolicy` that limits referencing the `RetentionPolicy` only from the same namespace,
```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      from: Same
```
Here is an example of `spec.usagePolicy` that allows referencing the `RetentionPolicy` from only `prod` and `staging` namespaces,
```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      from: Selector
      selector:
        matchExpressions:
          - key: "kubernetes.io/metadata.name"
            operator: In
            values: ["prod","staging"]
```
Here is an example of `spec.usagePolicy` that allows referencing the `RetentionPolicy` from all namespaces,
```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      from: All
```

#### spec.successfulSnapshots
SuccessfulSnapshots specifies how many successful Snapshots should be kept. It consists of the following fields:
- **last :** specifies how many last Snapshots should be kept.
- **hourly :** specifies how many hourly Snapshots should be kept.
- **daily :** specifies how many daily Snapshots should be kept.
- **weekly :** specifies how many weekly Snapshots should be kept.
- **monthly :** specifies how many monthly Snapshots should be kept.
- **yearly :** specifies how many yearly Snapshots should be kept.

#### spec.failedSnapshots
FailedSnapshots specifies how many failed Snapshots should be kept. It consists of only one field **last** which specifies how many last failed Snapshots should be kept. By default, KubeStash will keep only the last 1 failed Snapshot.

#### spec.default
Default specifies whether to use this `RetentionPolicy` as a default `RetentionPolicy` for the current namespace as well as the permitted namespaces. One namespace can have at most one default `RetentionPolicy` configured.