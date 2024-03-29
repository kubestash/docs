---
title: Volume Populator Overview | KubeStash
description: An overview of how Volume Populator works in KubeStash
menu:
  docs_{{ .version }}:
    identifier: volume-populator-overview
    name: How Volume Populator works?
    parent: volume-populator
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Populate Volume Using KubeStash

This guide will give you an overview of how volume populator process works in KubeStash.

## How Volume Populator Process Works?
The following diagram shows how KubeStash populates a Persistent Volume Claim with data sourced from a KubeStash Snapshot.

<figure align="center">
  <img alt="KubeStash Volume Populator Process Flow" src="/docs/guides/volume-populator/overview/images/volume-populator-overview.svg">
<figcaption align="center">Fig: Volume Populating Process in KubeStash</figcaption>
</figure>

The `Volume Populator` process consists of the following steps:

1. KubeStash operator watches for `PVCs`. 

2. When the operator finds a `PVC` with `spec.dataSourceRef` set and refers to a KubeStash `Snapshot` object, it creates a volume populator `Job` that mounts a temporary `PVC`.

3. Then the populator Job restores the referred `Snapshot` into the temporary PVC.

4. Once the restoration process is complete, the KubeStash operator deletes this populator `Job` and temporary `PVC`, and binds the `PV` containing the restored data with the desired `PVC`.

## Next Steps
1. See a step by step guide to populate the volume of a stand-alone PVC [here](/docs/guides/volume-populator/pvc/index.md).
2. See a step by step guide to populate the volumes of a Deployment [here](/docs/guides/volume-populator/deployment/index.md).
3. See a step by step guide to populate the volumes of a StatefulSet [here](/docs/guides/volume-populator/statefulset/index.md).
