---
title: Manifest-View
menu:
  docs_{{ .version }}:
    identifier: kubectl-kubestash-manifest-view
    name: Manifest-View
    parent: reference-cli
menu_name: docs_{{ .version }}
section_menu_id: reference
---
## kubectl-kubestash manifest-view

view components(manifest) of a snapshot

### Synopsis

view components(manifest) of a snapshot from restic repositories

```
kubectl-kubestash manifest-view [flags]
```

### Options

```
      --and-label-selectors strings   A set of labels, all of which need to be matched to filter the resources (comma-separated, e.g., 'key1:value1,key2:value2')
      --components strings            List of components to restore
      --enable-cache                  Specify whether to enable caching for restic
      --exclude strings               List of pattern for directory/file to ignore during restore
      --exclude-namespaces strings    Namespaces to exclude from backup (comma-separated, e.g., 'kube-public,temp')
      --exclude-resources strings     Resource types to exclude (comma-separated, e.g., 'secrets,configmaps')
  -h, --help                          help for manifest-view
      --include strings               List of pattern for directory/file to restore
      --include-cluster-resources     Specify whether to backup cluster scoped resources (default true)
      --include-namespaces strings    Namespaces to include in backup (comma-separated, e.g., 'default,kube-system')
      --include-resources strings     Resource types to include (comma-separated, e.g., 'pods,deployments')
      --namespace string              Namespace of the snapshot (default "default")
      --or-label-selectors strings    A set of labels, a subset of which need to be matched to filter the resources (comma-separated, e.g., 'key1:value1,key2:value2')
      --paths strings                 Gives a random list of paths
      --scratch-dir string            Temporary directory
      --snapshot string               Name of the snapshot
```

### Options inherited from parent commands

```
      --as string                             Username to impersonate for the operation. User could be a regular user or a service account in a namespace.
      --as-group stringArray                  Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --as-uid string                         UID to impersonate for the operation.
      --cache-dir string                      Default cache directory (default "/home/runner/.kube/cache")
      --certificate-authority string          Path to a cert file for the certificate authority
      --client-certificate string             Path to a client certificate file for TLS
      --client-key string                     Path to a client key file for TLS
      --cluster string                        The name of the kubeconfig cluster to use
      --context string                        The name of the kubeconfig context to use
      --default-seccomp-profile-type string   Default seccomp profile
      --disable-compression                   If true, opt-out of response compression for all requests to the server
      --insecure-skip-tls-verify              If true, the server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kubeconfig string                     Path to the kubeconfig file to use for CLI requests.
      --match-server-version                  Require server version to match client version
      --request-timeout string                The length of time to wait before giving up on a single server request. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests. (default "0")
  -s, --server string                         The address and port of the Kubernetes API server
      --tls-server-name string                Server name to use for server certificate validation. If it is not provided, the hostname used to contact the server is used
      --token string                          Bearer token for authentication to the API server
      --user string                           The name of the kubeconfig user to use
```

### SEE ALSO

* [kubectl-kubestash](/docs/reference/cli/kubectl-kubestash.md)	 - kubectl plugin for KubeStash

