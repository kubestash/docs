---
title: MySQL Addon Overview | KubeStash
description: MySQL Addon Overview | KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-mysql-readme
    name: Readme
    parent: kubestash-mysql
    weight: -1
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
url: /docs/{{ .version }}/addons/mysql/
aliases:
  - /docs/{{ .version }}/addons/mysql/README/
---

# KubeStash MySQL Addon

KubeStash v2024.9.30+ supports extending its functionality through addons. KubeStash MySQL addon enables KubeStash to backup and restore MySQL databases.

This guide will give you an overview of which MySQL versions are supported and how the docs are organized.

## Supported MySQL Versions

KubeStash has the following addon versions for MySQL:

{{< versionlist "mysql">}}

Here, the addon follows `M.M.P` versioning scheme where `M.M.P` (Major.Minor.Patch) represents the respective database version.

## Addon Version Compatibility

Any addon with matching major version with the database version should be able to take backup of that database. For example, MySQL addon with version `8.x.x` should be able take backup of any MySQL of `8.x.x` series. However, this might not be true for some versions. In that case, we will have separate addon for that version.

## Documentation Overview

KubeStash MySQL documentations are organized as below:

- [How does it work?](/docs/addons/mysql/overview/index.md) gives an overview of how backup and restore process for MySQL database works in Stash.
- [Standalone MySQL Database](/docs/addons/mysql/logical/index.md) shows how to backup and restore an externally managed MySQL database.