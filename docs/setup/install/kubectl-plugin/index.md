---
title: Install KubeStash kubectl Plugin
description: Installation guide for KubeStash kubectl Plugin
menu:
  docs_{{ .version }}:
    identifier: install-kubestash-kubectl-plugin
    name: KubeStash kubectl Plugin
    parent: installation-guide
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Install KubeStash kubectl Plugin

KubeStash provides a `kubectl` plugin to interact with KubeStash resources.

## Install using Krew

KubeStash `kubectl` plugin can be installed using Krew. [Krew](https://krew.sigs.k8s.io/) is the plugin manager for kubectl command-line tool. To install follow the steps below:

- Install `krew` following the steps [here](https://krew.sigs.k8s.io/docs/user-guide/setup/install/).

- If you have already installed `krew`, please upgrade `krew` to version v0.4.0 or later so that you can use [custom plugin indexes](https://krew.sigs.k8s.io/docs/user-guide/custom-indexes/).

```bash
kubectl krew upgrade
kubectl krew version
```

- Add [AppsCode's kubectl plugin index](https://github.com/appscode/krew-index). If you have already added the index, update the index.

```bash
kubectl krew index add appscode https://github.com/appscode/krew-index.git
kubectl krew index list
kubectl krew update
```

- Install Kubestash `kubectl` plugin following the commands below:

```bash
kubectl krew install appscode/kubestash
kubectl kubestash version
```

- If KubeStash `kubectl` plugin is already installed, run the following command to upgrade the plugin:

```bash
kubectl krew upgrade
kubectl kubestash version
```

## Install using pre-built binary

You can download the pre-build binaries from [kubestash/cli](https://github.com/kubestash/cli/releases) releases and put it into one of your installation directory denoted by `$PATH` variable.

Here is a simple Linux command to install the latest 64-bit Linux binary directly into your `/usr/local/bin` directory:

```bash
# Linux amd 64-bit
curl -o kubectl-kubestash.tar.gz -fsSL https://github.com/kubestash/cli/releases/download/{{< param "info.cli" >}}/kubectl-kubestash-linux-amd64.tar.gz \
  && tar zxvf kubectl-kubestash.tar.gz \
  && chmod +x kubectl-kubestash-linux-amd64 \
  && sudo mv kubectl-kubestash-linux-amd64 /usr/local/bin/kubectl-kubestash \
  && rm kubectl-kubestash.tar.gz LICENSE.md

# Mac OSX 64-bit
curl -o kubectl-kubestash.tar.gz -fsSL https://github.com/kubestash/cli/releases/download/{{< param "info.cli" >}}/kubectl-kubestash-darwin-amd64.tar.gz \
  && tar zxvf kubectl-kubestash.tar.gz \
  && chmod +x kubectl-kubestash-darwin-amd64 \
  && sudo mv kubectl-kubestash-darwin-amd64 /usr/local/bin/kubectl-kubestash \
  && rm kubectl-kubestash.tar.gz LICENSE.md
```

If you prefer to install kubectl KubeStash cli from source code, make sure that your go development environment has been setup properly. Then, just run:

```bash
go get github.com/kubestash/cli/...
```

> Please note that this will install KubeStash cli from master branch which might include breaking and/or undocumented changes.
