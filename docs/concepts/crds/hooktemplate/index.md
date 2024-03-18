---
title: HookTemplate Overview
menu:
  docs_{{ .version }}:
    identifier: hooktemplate-overview
    name: HookTemplate
    parent: crds
    weight: 50
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

> New to KubeStash? Please start [here](/docs/concepts/README.md).

# HookTemplate

## What is HookTemplate

A `HookTemplate` is a Kubernetes `CustomResourceDefinition`(CRD) which basically specifies a template for an action that will be executed before or/and after backup/restore process. For example, there could be a `HookTemplate` that pause an application before backup and another `HookTemplate` that resume the application after backup.

## HookTemplate CRD Specification

Like any official Kubernetes resource, a `HookTemplate` has `TypeMeta`, `ObjectMeta` and `Spec` sections. However, unlike other Kubernetes resources, it does not have a `Status` section.

A sample `HookTemplate` object is shown below,
```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: HookTemplate
metadata:
  name: sample-hook
  namespace: demo
spec:
  usagePolicy:
    allowedNamespaces:
      from: All
  params:
  - name: TEST
    usage: This is a test param
    required: false
  action:
    exec:
      command:
      - /bin/sh
      - -c
      - echo data_test > /source/data/data.txt
  executor:
    type: Pod
    pod:
      selector: name=test-app, test=hook
```

Here, we are going to describe the various sections of a `HookTemplate` crd.

### HookTemplate `Spec`

A `HookTemplate` object has the following fields in the `spec` section:

#### spec.usagePolicy
`spec.usagePolicy` lets you control which namespaces are allowed to use the `HookTemplate` and which are not. If you refer to a `HookTemplate` from a restricted namespace, KubeStash will reject creating the respective `BackupConfiguration` from validating webhook. You can use the `usagePolicy` to allow only the same namespace, a subset of namespaces, or all the namespaces to refer to the `HookTemplate`. If you don’t specify any `usagePolicy`, KubeStash will allow referencing the `HookTemplate` only from the namespace where it was created.

Here is an example of `spec.usagePolicy` that limits referencing the `HookTemplate` only from the same namespace,
```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      from: Same
```
Here is an example of `spec.usagePolicy` that allows referencing the `HookTemplate` from only `prod` and `staging` namespaces,
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
Here is an example of `spec.usagePolicy` that allows referencing the `HookTemplate` from all namespaces,
```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      from: All
```

#### spec.params
`spec.params` defines a list of parameters that is used by the `HookTemplate` to execute its logic. Each param consists of the following fields:
- **name :** specifies the name of the parameter.
- **usage :** specifies the usage of this parameter.
- **required :** specify whether this parameter is required or not.
- **default :** specifies a default value for the parameter.

#### spec.action
`spec.action` specifies the operation that is performed by this `HookTemplate`. Valid values are:
- **exec :** Execute command in a shell
- **httpGet :** Do an HTTP GET request
- **httpPost :** Do an HTTP POST request
- **tcpSocket :** Check if a TCP socket open or not

For more details on how hook's action work in KubeStash and how to configure different types of hook, please visit [here](/docs/guides/hooks/overview/index.md).

#### spec.executor
`spec.executor` specifies the entity specification which is responsible for executing the hook. It consists of the following fields:
- **type :** indicate the types of entity that will execute the hook. Valid values are:
  - **Function :** KubeStash will create a job with the provided information in `function` section. The job will execute the hook.
  - **Pod :** KubeStash will select the pod that matches the selector provided in `pod` section. This pod(s) will execute the hook.
  - **Operator :** KubeStash operator itself will execute the hook.
- **function :** specifies the function information which will be used to create the hook executor job. It consists of the following fields:
  - **name :** indicate the name of the `Function` that contains the container definition for executing the hook logic.
  - **env :** specifies a list of environment variables that will be passed to the executor container.
  - **volumeMounts :** specifies the volumes mounts for the executor container.
  - **volumes :** specifies the volumes that will be mounted in the executor container.
- **pod :** specifies the criteria to use to select the hook executor pods.
  - **selector :** specifies list of key value pair that will be used as label selector to select the desired pods. You can use comma to separate multiple labels (i.e. "app=my-app,env=prod").
  - **owner :** specifies a template for owner reference that will be used to filter the selected pods.
  - **strategy :** specifies what should be the behavior when multiple pods are selected. Valid values are:
    - **ExecuteOnOne :** Execute hook on only one of the selected pods. This is default behavior.
    - **ExecuteOnAll :** Execute hook on all the selected pods.

## Next Steps
- Learn how to configure `HookTemplate` for different use cases from [here](/docs/guides/hooks/configuring-hooks/index.md).