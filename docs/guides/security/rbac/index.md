---
title: RBAC | KubeStash
description: Using KubeStash in RBAC enabled cluster
menu:
  docs_{{ .version }}:
    identifier: security-rbac
    name: RBAC
    parent: security
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# KubeStash with RBAC Enabled Cluster

KubeStash comes with built-in support for RBAC enabled cluster. KubeStash installer create a `ClusterRole` and `ClusterRoleBinding` giving necessary permission to the operator.

## Operator Permissions

KubeStash operator needs the following RBAC permissions,

| API Groups                   | Resources                                                      | Permissions                                     |
|------------------------------|----------------------------------------------------------------|-------------------------------------------------|
| apiextensions.k8s.io         | customresourcedefinitions                                      | get, create, patch, update                      |
| admissionregistration.k8s.io | mutatingwebhookconfigurations, validatingwebhookconfigurations | *                                               |
| core.kubestash.com           | *                                                              | *                                               |
| storage.kubestash.com        | *                                                              | *                                               |
| config.kubestash.com         | *                                                              | *                                               |
| addons.kubestash.com         | *                                                              | *                                               |
| kubedb.com                   | *                                                              | *                                               |
| catalog.kubedb.com           | elasticsearchs                                                 | get, list, watch                                |
| elasticsearch.kubedb.com     | elasticsearchdashboards                                        | list                                            |
| appcatalog.appscode.com      | *                                                              | get, list, watch                                |
| apps                         | daemonsets, replicasets, statefulsets                          | get, list, watch                                |
| apps                         | deployments                                                    | get, list, watch, create, patch, update         |
| batch                        | jobs, cronjobs                                                 | get, list, watch, create, patch, update, delete |
| ""                           | events                                                         | create                                          |
| ""                           | persistentvolumeclaims, persistentvolumes                      | get, list, watch, create, patch, delete, update |
| ""                           | services, endpoints, pods                                      | get, list, watch                                |
| ""                           | secrets                                                        | get, list, create, patch, watch, delete         |
| ""                           | nodes, namespaces                                              | get, list, watch                                |
| ""                           | pods/exec                                                      | create                                          |
| ""                           | serviceaccounts                                                | get, list, watch, create, delete, patch, update |
| rbac.authorization.k8s.io    | clusterroles, roles, rolebindings, clusterrolebindings         | get, list, watch, create, delete, patch, update |
| apps.openshift.io            | deploymentconfigs                                              | get, list, watch, patch                         |
| policy                       | podsecuritypolicies                                            | use                                             |
| snapshot.storage.k8s.io      | *                                                              | *                                               |
| storage.k8s.io               | storageclasses                                                 | get, list, watch                                |

Here,

- `""` in API Group column means `core` API groups.
- `*` in Resources colum means all resources.
- `*` in Permission colum means all permissions.

## User facing ClusterRoles

KubeStash introduces custom resources, such as, `BackupConfiguration`, `BackupSession`,  `BackupStorage`, `RestoreSession`, `Function`, and `Addon` etc. KubeStash installer will create 2 user facing cluster roles:

| ClusterRole                                | Aggregates To | Description                           |
|--------------------------------------------|---------------|---------------------------------------|
| appscode:kubestash-kubestash-operator:edit | admin, edit   | Allows edit access to KubeStash CRDs. |
| appscode:kubestash-kubestash-operator:view | view          | Allows read-only access to Stash CRDs |

These user facing roles supports [ClusterRole Aggregation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles) feature in Kubernetes 1.9 or later clusters.
