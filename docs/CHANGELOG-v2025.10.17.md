---
title: Changelog | KubeStash
description: Changelog
menu:
  docs_{{.version}}:
    identifier: changelog-kubestash-v2025.10.17
    name: Changelog-v2025.10.17
    parent: welcome
    weight: 20251017
product_name: kubestash
menu_name: docs_{{.version}}
section_menu_id: welcome
url: /docs/{{.version}}/welcome/changelog-v2025.10.17/
aliases:
  - /docs/{{.version}}/CHANGELOG-v2025.10.17/
---

# KubeStash v2025.10.17 (2025-10-17)


## [kubestash/apimachinery](https://github.com/kubestash/apimachinery)

### [v0.21.0](https://github.com/kubestash/apimachinery/releases/tag/v0.21.0)

- [25288256](https://github.com/kubestash/apimachinery/commit/25288256) fix: respect --kubeconfig flag by creating client from RESTClientGetter (#180)
- [60ef100e](https://github.com/kubestash/apimachinery/commit/60ef100e) Excluded non-createable resources from backup and restore (#178)
- [130e59c7](https://github.com/kubestash/apimachinery/commit/130e59c7) Register necessary api schema to the UncachedClient  generation
- [cdc847b1](https://github.com/kubestash/apimachinery/commit/cdc847b1) Fix vsClass name nil pointer
- [13f5072d](https://github.com/kubestash/apimachinery/commit/13f5072d) Add `NewVolumeSnapshot` to Handle `VS` Version Compatibility (#174)
- [32a92de1](https://github.com/kubestash/apimachinery/commit/32a92de1) Merge pull request #172 from kubestash/gha-up
- [b0ab537b](https://github.com/kubestash/apimachinery/commit/b0ab537b) Use Go 1.25
- [e3c1ba32](https://github.com/kubestash/apimachinery/commit/e3c1ba32) Added file operations and priority to resourceops (#171)



## [kubestash/cli](https://github.com/kubestash/cli)

### [v0.20.0](https://github.com/kubestash/cli/releases/tag/v0.20.0)

- [4f3938eb](https://github.com/kubestash/cli/commit/4f3938eb) Prepare for release v0.20.0 (#65)
- [994ea7c2](https://github.com/kubestash/cli/commit/994ea7c2) fix: respect --kubeconfig flag by creating client from RESTClientGetter
- [4c3a61b3](https://github.com/kubestash/cli/commit/4c3a61b3) Use Go 1.25 (#63)
- [b8818555](https://github.com/kubestash/cli/commit/b8818555) Added manifest restore, view cli support. (#60)
- [01495ddb](https://github.com/kubestash/cli/commit/01495ddb) Bump golang.org/x/net from 0.33.0 to 0.36.0 (#51)



## [kubestash/installer](https://github.com/kubestash/installer)

### [v2025.10.17](https://github.com/kubestash/installer/releases/tag/v2025.10.17)

- [2e125f4](https://github.com/kubestash/installer/commit/2e125f4) Prepare for release v2025.10.17 (#221)
- [019fb6c](https://github.com/kubestash/installer/commit/019fb6c) Update cve report (#220)
- [a9860cb](https://github.com/kubestash/installer/commit/a9860cb) Update cve report (#219)
- [bb162b4](https://github.com/kubestash/installer/commit/bb162b4) Update cve report (#218)
- [298f600](https://github.com/kubestash/installer/commit/298f600) Update cve report (#217)
- [07ab8d4](https://github.com/kubestash/installer/commit/07ab8d4) Update cve report (#216)
- [3eb758f](https://github.com/kubestash/installer/commit/3eb758f) Update cve report (#215)
- [2d56667](https://github.com/kubestash/installer/commit/2d56667) Update cve report (#214)
- [3875246](https://github.com/kubestash/installer/commit/3875246) Remove runAs* for openshift deployment (#213)
- [23dc214](https://github.com/kubestash/installer/commit/23dc214) Test against k8s 1.34 (#212)
- [9b5460e](https://github.com/kubestash/installer/commit/9b5460e) Use Go 1.25 (#211)
- [b493268](https://github.com/kubestash/installer/commit/b493268) Test against k8s 1.33.2 (#209)
- [28ec303](https://github.com/kubestash/installer/commit/28ec303) Update cve report (#208)
- [8734a0b](https://github.com/kubestash/installer/commit/8734a0b) Update cve report (#207)
- [9eee097](https://github.com/kubestash/installer/commit/9eee097) Update cve report (#206)



## [kubestash/kubedump](https://github.com/kubestash/kubedump)

### [v0.20.0](https://github.com/kubestash/kubedump/releases/tag/v0.20.0)

- [25cc454](https://github.com/kubestash/kubedump/commit/25cc454) Prepare for release v0.20.0 (#61)
- [b548963](https://github.com/kubestash/kubedump/commit/b548963) Excluded non createable resources from backup and restore (#60)
- [f45fba0](https://github.com/kubestash/kubedump/commit/f45fba0) Use Go 1.25 (#57)
- [5c0df64](https://github.com/kubestash/kubedump/commit/5c0df64) Test against k8s 1.33.2 (#56)



## [kubestash/kubestash](https://github.com/kubestash/kubestash)

### [v0.21.0](https://github.com/kubestash/kubestash/releases/tag/v0.21.0)

- [e75b7dd2](https://github.com/kubestash/kubestash/commit/e75b7dd2) Prepare for release v0.21.0 (#307)
- [cd50f94a](https://github.com/kubestash/kubestash/commit/cd50f94a) Fix CI by updating deps (#306)
- [b02fd0bb](https://github.com/kubestash/kubestash/commit/b02fd0bb) Fix session-name (#305)
- [fa126bdf](https://github.com/kubestash/kubestash/commit/fa126bdf) Add Pod Exec Perm to backup/restore & Exclude KubeSlice container
- [76ec8db3](https://github.com/kubestash/kubestash/commit/76ec8db3) Remove VSAPI dependency;
- [11a8f3b6](https://github.com/kubestash/kubestash/commit/11a8f3b6) Use Go 1.25 (#300)



## [kubestash/manifest](https://github.com/kubestash/manifest)

### [v0.13.0](https://github.com/kubestash/manifest/releases/tag/v0.13.0)

- [db84f12](https://github.com/kubestash/manifest/commit/db84f12) Prepare for release v0.13.0 (#40)
- [158fac4](https://github.com/kubestash/manifest/commit/158fac4) Use Go 1.25 (#39)
- [571603b](https://github.com/kubestash/manifest/commit/571603b) Test against k8s 1.33.2 (#38)



## [kubestash/pvc](https://github.com/kubestash/pvc)

### [v0.20.0](https://github.com/kubestash/pvc/releases/tag/v0.20.0)

- [32b98b3](https://github.com/kubestash/pvc/commit/32b98b3) Prepare for release v0.20.0 (#61)
- [8267128](https://github.com/kubestash/pvc/commit/8267128) Use Go 1.25 (#60)
- [bbae4e4](https://github.com/kubestash/pvc/commit/bbae4e4) Test against k8s 1.33.2 (#59)



## [kubestash/volume-snapshotter](https://github.com/kubestash/volume-snapshotter)

### [v0.20.0](https://github.com/kubestash/volume-snapshotter/releases/tag/v0.20.0)

- [cd535ceb](https://github.com/kubestash/volume-snapshotter/commit/cd535ceb) Prepare for release v0.20.0 (#52)
- [2201bc61](https://github.com/kubestash/volume-snapshotter/commit/2201bc61) Remove VSAPI dependency;
- [3f3c21f4](https://github.com/kubestash/volume-snapshotter/commit/3f3c21f4) Use Go 1.25 (#50)
- [80fa1226](https://github.com/kubestash/volume-snapshotter/commit/80fa1226) Test against k8s 1.33.2 (#49)



## [kubestash/workload](https://github.com/kubestash/workload)

### [v0.20.0](https://github.com/kubestash/workload/releases/tag/v0.20.0)

- [837e44c](https://github.com/kubestash/workload/commit/837e44c) Prepare for release v0.20.0 (#71)
- [09e1cbd](https://github.com/kubestash/workload/commit/09e1cbd) Use Go 1.25 (#70)




