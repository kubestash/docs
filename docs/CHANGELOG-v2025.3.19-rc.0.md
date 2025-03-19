---
title: Changelog | KubeStash
description: Changelog
menu:
  docs_{{.version}}:
    identifier: changelog-kubestash-v2025.3.19-rc.0
    name: Changelog-v2025.3.19-rc.0
    parent: welcome
    weight: 20250319
product_name: kubestash
menu_name: docs_{{.version}}
section_menu_id: welcome
url: /docs/{{.version}}/welcome/changelog-v2025.3.19-rc.0/
aliases:
  - /docs/{{.version}}/CHANGELOG-v2025.3.19-rc.0/
---

# KubeStash v2025.3.19-rc.0 (2025-03-19)


## [kubestash/apimachinery](https://github.com/kubestash/apimachinery)

### [v0.17.0-rc.0](https://github.com/kubestash/apimachinery/releases/tag/v0.17.0-rc.0)

- [0e805914](https://github.com/kubestash/apimachinery/commit/0e805914) Update deps
- [aefab6a6](https://github.com/kubestash/apimachinery/commit/aefab6a6) Update controller config
- [aa10be69](https://github.com/kubestash/apimachinery/commit/aa10be69) Add IRSA annotation utility (#161)
- [5fc5e168](https://github.com/kubestash/apimachinery/commit/5fc5e168) Skip extracting restic output lines if `message_type` is not summary
- [74531c6c](https://github.com/kubestash/apimachinery/commit/74531c6c) Merge pull request #159 from kubestash/v3
- [7a5e5c63](https://github.com/kubestash/apimachinery/commit/7a5e5c63) incorporate cahnges with one struct
- [d928d133](https://github.com/kubestash/apimachinery/commit/d928d133) set region for S3-IRSA
- [d11830db](https://github.com/kubestash/apimachinery/commit/d11830db) change webhooks directory
- [ff90d99c](https://github.com/kubestash/apimachinery/commit/ff90d99c) Fix v3 webhook signature for BackupStorage
- [2ecbedaa](https://github.com/kubestash/apimachinery/commit/2ecbedaa) Use k8s 1.32 client libs
- [1bebb656](https://github.com/kubestash/apimachinery/commit/1bebb656) Use Go 1.24 (#157)
- [57156d93](https://github.com/kubestash/apimachinery/commit/57156d93) Update deps
- [1a6fe49b](https://github.com/kubestash/apimachinery/commit/1a6fe49b) fix s3-compitbale storage endpoint resolver (#156)
- [f1e579e0](https://github.com/kubestash/apimachinery/commit/f1e579e0) Disable image caching in setup-qemu action (#155)



## [kubestash/cli](https://github.com/kubestash/cli)

### [v0.16.0-rc.0](https://github.com/kubestash/cli/releases/tag/v0.16.0-rc.0)

- [0d06dba1](https://github.com/kubestash/cli/commit/0d06dba1) Prepare for release v0.16.0-rc.0 (#53)
- [3359c429](https://github.com/kubestash/cli/commit/3359c429) Use restic 0.17.3 (#52)
- [7eca0331](https://github.com/kubestash/cli/commit/7eca0331) Use Go 1.24 (#50)
- [b641496f](https://github.com/kubestash/cli/commit/b641496f) Add restic latest tag (#49)
- [1122a79d](https://github.com/kubestash/cli/commit/1122a79d) Add cmd to convert stash resources yaml to kubestash (#45)
- [0bb3aa74](https://github.com/kubestash/cli/commit/0bb3aa74) Prepare for release v0.15.0 (#48)
- [0e2c3a44](https://github.com/kubestash/cli/commit/0e2c3a44) Disable image caching in setup-qemu action (#47)



## [kubestash/installer](https://github.com/kubestash/installer)

### [v2025.3.19-rc.0](https://github.com/kubestash/installer/releases/tag/v2025.3.19-rc.0)

- [186e7fe](https://github.com/kubestash/installer/commit/186e7fe) Prepare for release v2025.3.19-rc.0 (#165)
- [6550b73](https://github.com/kubestash/installer/commit/6550b73) Use k8s 1.32 client libs
- [82dbc25](https://github.com/kubestash/installer/commit/82dbc25) Mount tls certs in operator (#163)
- [608fb58](https://github.com/kubestash/installer/commit/608fb58) Add serviceAccount name as env for `credential-less` (#161)
- [0b1e9ca](https://github.com/kubestash/installer/commit/0b1e9ca) Remove rbac-proxy sidecar (#160)
- [19d09c4](https://github.com/kubestash/installer/commit/19d09c4) Update cve report (#159)
- [f580c8f](https://github.com/kubestash/installer/commit/f580c8f) Update cve report (#158)
- [b2f984c](https://github.com/kubestash/installer/commit/b2f984c) Update cve report (#156)
- [834a338](https://github.com/kubestash/installer/commit/834a338) Update cve report (#155)
- [bd7ed89](https://github.com/kubestash/installer/commit/bd7ed89) Update ace-user-roles
- [9e2dfa3](https://github.com/kubestash/installer/commit/9e2dfa3) Test against k8s 1.32 (#154)
- [ea15004](https://github.com/kubestash/installer/commit/ea15004) Test against k8s 1.32 (#153)
- [52834b2](https://github.com/kubestash/installer/commit/52834b2) Update cve report (#152)



## [kubestash/kubedump](https://github.com/kubestash/kubedump)

### [v0.16.0-rc.0](https://github.com/kubestash/kubedump/releases/tag/v0.16.0-rc.0)

- [0f07df1](https://github.com/kubestash/kubedump/commit/0f07df1) Prepare for release v0.16.0-rc.0 (#45)
- [20b8e91](https://github.com/kubestash/kubedump/commit/20b8e91) Use restic 0.17.3 (#44)
- [c851794](https://github.com/kubestash/kubedump/commit/c851794) Skip fake api-resources (#43)
- [02a4c7b](https://github.com/kubestash/kubedump/commit/02a4c7b) Use Go 1.24 (#42)
- [dd0d658](https://github.com/kubestash/kubedump/commit/dd0d658) Prepare for release v0.15.0 (#41)
- [9af244f](https://github.com/kubestash/kubedump/commit/9af244f) Disable image caching in setup-qemu action (#40)



## [kubestash/kubestash](https://github.com/kubestash/kubestash)

### [v0.17.0-rc.0](https://github.com/kubestash/kubestash/releases/tag/v0.17.0-rc.0)

- [4da9a4a5](https://github.com/kubestash/kubestash/commit/4da9a4a5) Prepare for release v0.17.0-rc.0 (#288)
- [e4da511f](https://github.com/kubestash/kubestash/commit/e4da511f) Add irsa role annotation to every executor SA (#285)
- [9dfa0ae3](https://github.com/kubestash/kubestash/commit/9dfa0ae3) Remove rabcproxy sidecar (#287)
- [499b0bec](https://github.com/kubestash/kubestash/commit/499b0bec) Use restic 0.17.3 (#286)
- [f3b84de4](https://github.com/kubestash/kubestash/commit/f3b84de4) Merge pull request #284 from kubestash/v3
- [177eeda8](https://github.com/kubestash/kubestash/commit/177eeda8) update deps
- [4e993db0](https://github.com/kubestash/kubestash/commit/4e993db0) update apimachinery Signed-off-by: Anisur Rahman <anisur@appscode.com>
- [e75220d8](https://github.com/kubestash/kubestash/commit/e75220d8) Re-import `SetupWebhookWithManager` from webhook cmd
- [8414822c](https://github.com/kubestash/kubestash/commit/8414822c) Use Go 1.24 (#282)
- [bf782c59](https://github.com/kubestash/kubestash/commit/bf782c59) Use Go 1.24 (#281)
- [0972e4cc](https://github.com/kubestash/kubestash/commit/0972e4cc) Fix restore executor nil reference check for `archiverRef` (#280)
- [f55d932d](https://github.com/kubestash/kubestash/commit/f55d932d) Prepare for release v0.16.0 (#278)



## [kubestash/manifest](https://github.com/kubestash/manifest)

### [v0.9.0-rc.0](https://github.com/kubestash/manifest/releases/tag/v0.9.0-rc.0)

- [839a765](https://github.com/kubestash/manifest/commit/839a765) Prepare for release v0.9.0-rc.0 (#30)
- [3cb17f6](https://github.com/kubestash/manifest/commit/3cb17f6) Use restic 0.17.3 (#29)
- [557b621](https://github.com/kubestash/manifest/commit/557b621) Use Go 1.24 (#26)
- [88fe74e](https://github.com/kubestash/manifest/commit/88fe74e) Prepare for release v0.8.0 (#25)
- [60fab00](https://github.com/kubestash/manifest/commit/60fab00) Disable image caching in setup-qemu action (#24)



## [kubestash/pvc](https://github.com/kubestash/pvc)

### [v0.16.0-rc.0](https://github.com/kubestash/pvc/releases/tag/v0.16.0-rc.0)

- [9984b83](https://github.com/kubestash/pvc/commit/9984b83) Prepare for release v0.16.0-rc.0 (#53)
- [d6b45e2](https://github.com/kubestash/pvc/commit/d6b45e2) Use restic 0.17.3 (#52)
- [14a4597](https://github.com/kubestash/pvc/commit/14a4597) Use Go 1.24 (#51)
- [044e79b](https://github.com/kubestash/pvc/commit/044e79b) Prepare for release v0.15.0 (#50)
- [946f9e2](https://github.com/kubestash/pvc/commit/946f9e2) Disable image caching in setup-qemu action (#49)



## [kubestash/volume-snapshotter](https://github.com/kubestash/volume-snapshotter)

### [v0.16.0-rc.0](https://github.com/kubestash/volume-snapshotter/releases/tag/v0.16.0-rc.0)

- [f1a49b49](https://github.com/kubestash/volume-snapshotter/commit/f1a49b49) Prepare for release v0.16.0-rc.0 (#44)
- [6f865414](https://github.com/kubestash/volume-snapshotter/commit/6f865414) Use Go 1.24 (#43)
- [58ed7586](https://github.com/kubestash/volume-snapshotter/commit/58ed7586) Prepare for release v0.15.0 (#42)
- [4b15908d](https://github.com/kubestash/volume-snapshotter/commit/4b15908d) Disable image caching in setup-qemu action (#41)



## [kubestash/workload](https://github.com/kubestash/workload)

### [v0.16.0-rc.0](https://github.com/kubestash/workload/releases/tag/v0.16.0-rc.0)

- [4a4e153](https://github.com/kubestash/workload/commit/4a4e153) Prepare for release v0.16.0-rc.0 (#64)
- [16349de](https://github.com/kubestash/workload/commit/16349de) Use restic 0.17.3 (#63)
- [c28e244](https://github.com/kubestash/workload/commit/c28e244) Use Go 1.24 (#62)
- [33c8889](https://github.com/kubestash/workload/commit/33c8889) Prepare for release v0.15.0 (#61)
- [d923af9](https://github.com/kubestash/workload/commit/d923af9) Update github actions workflows (#60)




