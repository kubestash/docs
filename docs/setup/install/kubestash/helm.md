---
title: Install KubeStash using Helm 3
description: Install KubeStash using Helm 3
menu:
  docs_{{ .version }}:
    identifier: install-kubestash-helm
    name: Helm 3
    parent: install-kubestash-enterprise
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Using Helm 3

KubeStash can be installed via [Helm](https://helm.sh/) using the [chart](https://github.com/kubestash/installer/tree/master/charts/kubestash) from [AppsCode Charts Repository](https://github.com/appscode/charts). To install the chart with the release name `kubestash`:

```bash
$ helm install kubestash oci://ghcr.io/appscode-charts/kubestash \
        --version {{< param "info.version" >}} \
        --namespace stash --create-namespace \
        --set-file global.license=/path/to/the/license.txt \
        --wait --burst-limit=10000 --debug
```

## Offline (Air-gapped) Installation

In an air-gapped or offline cluster, the nodes cannot reach `ghcr.io`, which is the only public registry KubeStash pulls from. To install KubeStash in such an environment, mirror the Helm chart and the required images into a registry your cluster can reach, then point the operator at that registry using `global.registryFQDN` (operator and helper images) and `kubestash-catalog.proxies.ghcr` (backup addon images).

The examples below use `registry.example.com` as the private registry FQDN. Replace it with your own (add `:port` if your registry runs on a non-standard port).

### 1. Mirror the required images

KubeStash pulls every image from two `ghcr.io` namespaces. Mirror them into your private registry, or configure your registry as a pull-through cache for them:

- `ghcr.io/appscode` (the `kubectl-nonroot` helper image)
- `ghcr.io/kubestash` (the operator and the backup or restore addon functions)

The Helm chart itself is published at `ghcr.io/appscode-charts/kubestash`.

The installer repository ships helper scripts under [`catalog`](https://github.com/kubestash/installer/tree/master/catalog) that copy every required image into your registry. They download [`crane`](https://github.com/google/go-containerregistry) automatically and copy each image to `${IMAGE_REGISTRY}/<original-repository-path>`, preserving the upstream path. Choose the workflow that matches your network.

If the machine running the script can reach both `ghcr.io` and your private registry, copy everything directly:

```bash
wget https://github.com/kubestash/installer/raw/master/catalog/copy-images.sh
chmod +x copy-images.sh
export IMAGE_REGISTRY=registry.example.com
./copy-images.sh
```

If the cluster is fully air-gapped, run the export script on an internet-connected machine to produce a single `images.tar.gz`, carry it across (for example on removable media), then import it on the inside:

```bash
# On an internet-connected machine
wget https://github.com/kubestash/installer/raw/master/catalog/export-images.sh
chmod +x export-images.sh
./export-images.sh   # produces images.tar.gz

# Inside the air-gapped network, after copying images.tar.gz across
wget https://github.com/kubestash/installer/raw/master/catalog/import-images.sh
chmod +x import-images.sh
export IMAGE_REGISTRY=registry.example.com
./import-images.sh images.tar.gz
```

> **Note:** On a k3s cluster with no private registry, run `import-into-k3s.sh images.tar.gz` instead. It loads the image tarballs straight into each node's containerd under their original names, so you skip the registry entirely and do not set `global.registryFQDN` or the `proxies.ghcr` value.

Because the scripts copy every image under your registry host with its original repository path, set `global.registryFQDN` and `kubestash-catalog.proxies.ghcr` to that same `IMAGE_REGISTRY` host, as shown in steps 2 and 3.

### 2. Verify the image paths are rewritten

Before installing, render the chart and confirm that every `image:` points at your private registry:

```bash
$ helm template kubestash oci://registry.example.com/appscode-charts/kubestash \
        --version {{< param "info.version" >}} \
        --namespace stash --create-namespace \
        --set global.registryFQDN=registry.example.com \
        --set kubestash-catalog.proxies.ghcr=registry.example.com \
        | grep 'image:' | sort | uniq
```

Every image in the output should be prefixed with your registry FQDN. This de-duplicated list is also the precise set of images you need to make available in your registry for the chart version you selected.

### 3. Install KubeStash from the private registry

Once the chart and images are mirrored, install the operator. The `global.registryFQDN` value rewrites the operator and helper image paths, and `kubestash-catalog.proxies.ghcr` rewrites the backup addon image paths:

```bash
$ helm upgrade -i kubestash oci://registry.example.com/appscode-charts/kubestash \
        --version {{< param "info.version" >}} \
        --namespace stash --create-namespace \
        --set-file global.license=/path/to/the/license.txt \
        --set global.registryFQDN=registry.example.com \
        --set kubestash-catalog.proxies.ghcr=registry.example.com \
        --wait --burst-limit=10000 --debug
```

### 4. Image list for pre-seeded registries

If your registry cannot mirror an entire namespace (for example, because of storage limits), mirror only the images you need. The [installer repository](https://github.com/kubestash/installer/tree/master/catalog) publishes the full list at `catalog/imagelist.yaml`, generated from the same chart. KubeStash is compact: the complete set is the operator, the `kubectl-nonroot` helper, and the backup or restore addon functions, all under `ghcr.io`:

```
ghcr.io/appscode/kubectl-nonroot
ghcr.io/kubestash/kubestash
ghcr.io/kubestash/kubedump
ghcr.io/kubestash/manifest
ghcr.io/kubestash/pvc
ghcr.io/kubestash/vault
ghcr.io/kubestash/volume-snapshotter
ghcr.io/kubestash/workload
```

Run the verification command in step 2 (or read `catalog/imagelist.yaml`) to capture the exact image tags for your chart version.

To see the detailed configuration options, visit [here](https://github.com/kubestash/installer/tree/master/charts/kubestash).

Next: [verify the installation](/docs/setup/install/kubestash/configuration.md).
