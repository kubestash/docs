---
title: MariaDB Addon Overview | KubeStash
description: MariaDB Addon Overview | KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-mariadb-readme
    name: Readme
    parent: kubestash-mariadb
    weight: -1
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
url: /docs/{{ .version }}/addons/mariadb/
aliases:
  - /docs/{{ .version }}/addons/mariadb/README/
---

# KubeStash MariaDB Addon

KubeStash v2024.9.30+ supports extending its functionality through addons. KubeStash MariaDB addon enables KubeStash to backup and restore MariaDB databases.

This guide will give you an overview of which MariaDB versions are supported and how the docs are organized.

## Supported MariaDB Versions
- `10.x.x` 
- `11.x.x`

## Documentation Overview

KubeStash MariaDB documentations are organized as below:

- [How does it work?](/docs/addons/mariadb/overview/index.md) gives an overview of how backup and restore process for MariaDB database works in KubeStash.
- [Standalone MariaDB Database](/docs/addons/mariadb/logical/index.md) shows how to backup and restore an externally managed MariaDB database.