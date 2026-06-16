---
title: Install KubeStash
description: Installation guide for KubeStash
menu:
  docs_{{ .version }}:
    identifier: install-kubestash-enterprise
    name: KubeStash
    parent: installation-guide
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: setup
---

# Install KubeStash

## Get a Free License

Download a FREE license from [AppsCode License Server](https://appscode.com/issue-license?p=stash).

> KubeStash licensing process has been designed to work with CI/CD workflow. You can automatically obtain a license from your CI/CD pipeline by following the guide from [here](https://github.com/appscode/offline-license-server#offline-license-server).

## Choose an Installation Method

KubeStash can be installed in several ways. Pick the one that fits your workflow:

- [Helm 3](/docs/setup/install/kubestash/helm.md) — recommended for most users.
- [YAML](/docs/setup/install/kubestash/yaml.md) — render manifests and apply with `kubectl`.
- [ArgoCD](/docs/setup/install/kubestash/argocd.md) — GitOps via ArgoCD `Application` resources.
- [FluxCD](/docs/setup/install/kubestash/fluxcd.md) — GitOps via the Flux Helm Controller.
- [OpenShift](/docs/setup/install/kubestash/openshift.md) — standard chart or Red Hat certified chart.

After installing, see [Common Configuration](/docs/setup/install/kubestash/configuration.md) to verify the installation.

## Purchase KubeStash License

If you are interested in purchasing KubeStash license, please contact us via sales@appscode.com for further discussion. You can also set up a meeting via our [calendly link](https://calendly.com/appscode/intro).

If you are willing to purchase KubeStash license but need more time to test in your dev cluster, feel free to contact sales@appscode.com. We will be happy to extend your trial period.
