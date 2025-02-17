---
title: Elasticsearch Addon Overview | KubeStash
description: Elasticsearch Addon Overview | KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-elasticsearch-readme
    name: Readme
    parent: kubestash-elasticsearch
    weight: -1
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
url: /docs/{{ .version }}/addons/elasticsearch/
aliases:
  - /docs/{{ .version }}/addons/elasticsearch/README/
---

# KubeStash Elasticsearch Addon

KubeStash v2024.9.30+ supports extending its functionality through addons. KubeStash Elasticsearch addon enables KubeStash to backup and restore Elasticsearch databases.

This guide will give you an overview of which Elasticsearch versions are supported and how the docs are organized.

## Supported Elasticsearch Versions
6.x.x
7.x.x
8.x.x

## Supported Opensearch Versions
1.x.x
2.x.x

## Documentation Overview

KubeStash Elasticsearch documentations are organized as below:

- [How does it work?](/docs/addons/elasticsearch/overview/index.md) gives an overview of how backup and restore process for Elasticsearch database works in Stash.
- [Standalone Elasticsearch Database](/docs/addons/elasticsearch/logical/index.md) shows how to backup and restore an externally managed Elasticsearch database.