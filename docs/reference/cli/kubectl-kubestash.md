---
title: Kubectl-Kubestash
menu:
  docs_{{ .version }}:
    identifier: kubectl-kubestash
    name: Kubectl-Kubestash
    parent: reference-cli
    weight: 0

menu_name: docs_{{ .version }}
section_menu_id: reference
url: /docs/{{ .version }}/reference/cli/
aliases:
- /docs/{{ .version }}/reference/cli/kubectl-kubestash/
---
## kubectl-kubestash

kubectl plugin for KubeStash

### Synopsis

kubectl plugin for KubeStash. For more information, visit here: https://kubestash.com

### Options

```
      --as string                      Username to impersonate for the operation. User could be a regular user or a service account in a namespace.
      --as-group stringArray           Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --as-uid string                  UID to impersonate for the operation.
      --cache-dir string               Default cache directory (default "/home/runner/.kube/cache")
      --certificate-authority string   Path to a cert file for the certificate authority
      --client-certificate string      Path to a client certificate file for TLS
      --client-key string              Path to a client key file for TLS
      --cluster string                 The name of the kubeconfig cluster to use
      --context string                 The name of the kubeconfig context to use
      --disable-compression            If true, opt-out of response compression for all requests to the server
  -h, --help                           help for kubectl-kubestash
      --insecure-skip-tls-verify       If true, the server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kubeconfig string              Path to the kubeconfig file to use for CLI requests.
      --match-server-version           Require server version to match client version
  -n, --namespace string               If present, the namespace scope for this CLI request
      --request-timeout string         The length of time to wait before giving up on a single server request. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests. (default "0")
  -s, --server string                  The address and port of the Kubernetes API server
      --tls-server-name string         Server name to use for server certificate validation. If it is not provided, the hostname used to contact the server is used
      --token string                   Bearer token for authentication to the API server
      --user string                    The name of the kubeconfig user to use
```

### SEE ALSO

* [kubectl-kubestash clone](/docs/reference/cli/kubectl-kubestash_clone.md)	 - Clone Kubernetes resources
* [kubectl-kubestash completion](/docs/reference/cli/kubectl-kubestash_completion.md)	 - Generate completion script
* [kubectl-kubestash copy](/docs/reference/cli/kubectl-kubestash_copy.md)	 - Copy kubestash resources from one namespace to another namespace
* [kubectl-kubestash debug](/docs/reference/cli/kubectl-kubestash_debug.md)	 - Debug common KubeStash issues
* [kubectl-kubestash download](/docs/reference/cli/kubectl-kubestash_download.md)	 - Download components of a snapshot
* [kubectl-kubestash password](/docs/reference/cli/kubectl-kubestash_password.md)	 - Manage restic keys (passwords) for accessing the repository
* [kubectl-kubestash pause](/docs/reference/cli/kubectl-kubestash_pause.md)	 - Pause KubeStash backup temporarily
* [kubectl-kubestash resume](/docs/reference/cli/kubectl-kubestash_resume.md)	 - Resume KubeStash backup
* [kubectl-kubestash trigger](/docs/reference/cli/kubectl-kubestash_trigger.md)	 - Trigger a backup
* [kubectl-kubestash unlock](/docs/reference/cli/kubectl-kubestash_unlock.md)	 - Unlock Restic Repositories
* [kubectl-kubestash version](/docs/reference/cli/kubectl-kubestash_version.md)	 - Prints binary version number.

