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

The following diagram shows how KubeStash restores backup data into a specific volume and attaching it to user-provided Persistent Volume Claims (PVCs) for usage of various workload applications.

<figure align="center">
  <img alt="KubeStash Volume Populator Process Flow" src="/docs/guides/volume-populator/overview/images/volume-populator-overview.svg">
<figcaption align="center">Fig: Volume Populating Process in KubeStash</figcaption>
</figure>

The `Volume Populator` process consists of the following steps:

1. At first, a user creates a `workload` application that may mount a single or multiple `PVCs`. 

2. KubeStash operator watches for `PVCs`. 

3. When the operator finds a `PVC` with `spec.dataSourceRef` set and refers our `snapshot` object, it starts populating this `PVC` volume by creating a volume populator `Job` that mounts a temporary `PVC`.

4. Then the populator `Job` reads restore data information from this `snapshot` object and restore data into this temporary `PVC` .

5. Once the restoration process is complete, the KubeStash operator deletes this populator `Job` and temporary `PVC`, and binds the `PV` containing the restored data with our desired `PVC`.

## Next Steps
1. See a step by step guide to populate the volume of a stand-alone PVC [here](/docs/guides/volume-populator/pvc/index.md).
2. See a step by step guide to populate the volumes of a Deployment [here](/docs/guides/volume-populator/deployment/index.md).
3. See a step by step guide to populate the volumes of a StatefulSet [here](/docs/guides/volume-populator/statefulset/index.md).
