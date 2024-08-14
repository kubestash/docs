---
title: MongoDB Addon Overview | KubeStash
description: MongoDB Addon Overview | KubeStash
menu:
  docs_{{ .version }}:
    identifier: kubestash-mongodb-readme
    name: Readme
    parent: kubestash-mongodb
    weight: -1
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: kubestash-addons
url: /docs/{{ .version }}/addons/mongodb/
aliases:
  - /docs/{{ .version }}/addons/mongodb/README/
---

# KubeStash MongoDB Addon

KubeStash MongoDB addon allows you to backup and restore MongoDB databases. It supports backups for MongoDB instances running in Standalone, Replicaset, and Sharded cluster configurations. KubeStash makes managing your MongoDB backups and restorations more straightforward and efficient.

This guide will give you an overview of which MongoDB versions are supported and how the docs are organized.

## Supported MongoDB Versions

KubeStash has the following addon versions for MongoDB:

{{< versionlist "mongodb">}}

Here, the addon follows `M.M.P` versioning scheme where `M.M.P` (Major.Minor.Patch) represents the respective database version.

## Addon Version Compatibility

Any addon with matching major version with the database version should be able to take backup of that database. For example, MongoDB addon with version `4.x.x` should be able take backup of any MongoDB of `4.x.x` series. However, this might not be true for some versions. In that case, we will have separate addon for that version.

## Documentation Overview


KubeStash MongoDB documentations are organized as below:

- [How does it work?](/docs/addons/mongodb/overview/index.md) gives an overview of how backup and restore process for MongoDB database works in KubeStash.
- [Standalone MongoDB](/docs/addons/mongodb/standalone/index.md) shows how to backup and restore a standalone MongoDB database.
- [MongoDB ReplicaSet](/docs/addons/mongodb/replicaset/index.md) shows how to backup & restore  a MongoDB ReplicaSet.
- [Sharded MongoDB Cluster](/docs/addons/mongodb/sharding/index.md) shows how to backup & restore a sharded MongoDB cluster.
