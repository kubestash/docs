---
title: Changelog | KubeStash
description: Changelog
menu:
  docs_{{.version}}:
    identifier: changelog-kubestash-v2024.9.30
    name: Changelog-v2024.9.30
    parent: welcome
    weight: 20240930
product_name: kubestash
menu_name: docs_{{.version}}
section_menu_id: welcome
url: /docs/{{.version}}/welcome/changelog-v2024.9.30/
aliases:
  - /docs/{{.version}}/CHANGELOG-v2024.9.30/
---

# KubeStash v2024.9.30 (2024-09-27)


## [kubestash/apimachinery](https://github.com/kubestash/apimachinery)

### [v0.13.0](https://github.com/kubestash/apimachinery/releases/tag/v0.13.0)

- [84569fe2](https://github.com/kubestash/apimachinery/commit/84569fe2) Update deps
- [b0787ab7](https://github.com/kubestash/apimachinery/commit/b0787ab7) Add support for NetworkPolicy (#136)
- [ef35d264](https://github.com/kubestash/apimachinery/commit/ef35d264) Add conditions for unexpected failures (#134)
- [86c61246](https://github.com/kubestash/apimachinery/commit/86c61246) Add supportive function for kubedb `restoring` phase (#133)
- [6f40843d](https://github.com/kubestash/apimachinery/commit/6f40843d) Update manifest restore APIs (#132)
- [f5d4345b](https://github.com/kubestash/apimachinery/commit/f5d4345b) Add support for timeout for restic backup and restore (#130)
- [6ec04e5b](https://github.com/kubestash/apimachinery/commit/6ec04e5b) Add druid manifest backup (#124)
- [e2698888](https://github.com/kubestash/apimachinery/commit/e2698888) fix build
- [414e3512](https://github.com/kubestash/apimachinery/commit/414e3512) Add the `Content-Type` header when uploading a file to the backend (#131)
- [662b18f0](https://github.com/kubestash/apimachinery/commit/662b18f0) Cleanup s3 connection opener (#129)
- [9fa19655](https://github.com/kubestash/apimachinery/commit/9fa19655) Add db version label (#128)



## [kubestash/cli](https://github.com/kubestash/cli)

### [v0.12.0](https://github.com/kubestash/cli/releases/tag/v0.12.0)

- [01e39a8](https://github.com/kubestash/cli/commit/01e39a8) Prepare for release v0.12.0 (#39)
- [d72b630](https://github.com/kubestash/cli/commit/d72b630) Add support for timeout for restic backup and restore (#37)



## [kubestash/installer](https://github.com/kubestash/installer)

### [v2024.9.30](https://github.com/kubestash/installer/releases/tag/v2024.9.30)

- [935dc37](https://github.com/kubestash/installer/commit/935dc37) Prepare for release v2024.9.30 (#86)
- [06f2300](https://github.com/kubestash/installer/commit/06f2300) Add support for NetworkPolicy (#85)
- [ecb6e3d](https://github.com/kubestash/installer/commit/ecb6e3d) Use seccompProfile RuntimeDefault
- [ec46913](https://github.com/kubestash/installer/commit/ec46913) Add flag to enable subcharts (#84)
- [e1119ac](https://github.com/kubestash/installer/commit/e1119ac) Update crd for kubedb doc ci pass  (#83)
- [d267e2c](https://github.com/kubestash/installer/commit/d267e2c) Add imagelist.yaml
- [f4438bd](https://github.com/kubestash/installer/commit/f4438bd) Add metrics config user role
- [345e9b6](https://github.com/kubestash/installer/commit/345e9b6) Revise appcatalog user roles
- [783e8dc](https://github.com/kubestash/installer/commit/783e8dc) Review user roles (#82)
- [f2d1228](https://github.com/kubestash/installer/commit/f2d1228) Use KIND v0.24.0 (#81)



## [kubestash/kubedump](https://github.com/kubestash/kubedump)

### [v0.12.0](https://github.com/kubestash/kubedump/releases/tag/v0.12.0)




## [kubestash/kubestash](https://github.com/kubestash/kubestash)

### [v0.13.0](https://github.com/kubestash/kubestash/releases/tag/v0.13.0)

- [698df5e0](https://github.com/kubestash/kubestash/commit/698df5e0) Prepare for release v0.13.0 (#256)
- [a103a91c](https://github.com/kubestash/kubestash/commit/a103a91c) Set 'managed-by' labels to backup pods (#255)
- [741858f3](https://github.com/kubestash/kubestash/commit/741858f3) Fix OOMkill issue (#253)
- [c53c63c3](https://github.com/kubestash/kubestash/commit/c53c63c3) Change `kubedbDBintegration` method for application-level restore (#252)
- [d1cde855](https://github.com/kubestash/kubestash/commit/d1cde855) Update rbac for manifest options (#254)
- [a4f2aeaa](https://github.com/kubestash/kubestash/commit/a4f2aeaa) Fix restoresession phase update (#249)



## [kubestash/manifest](https://github.com/kubestash/manifest)

### [v0.5.0](https://github.com/kubestash/manifest/releases/tag/v0.5.0)

- [91c3898](https://github.com/kubestash/manifest/commit/91c3898) Prepare for release v0.5.0 (#16)
- [4ae6ed4](https://github.com/kubestash/manifest/commit/4ae6ed4) Update manifest restore namespace (#15)
- [75e00a1](https://github.com/kubestash/manifest/commit/75e00a1) Add timeout for backup and restore (#14)
- [915df3a](https://github.com/kubestash/manifest/commit/915df3a) Use restic 0.17.1 (#13)



## [kubestash/pvc](https://github.com/kubestash/pvc)

### [v0.12.0](https://github.com/kubestash/pvc/releases/tag/v0.12.0)

- [41e1967](https://github.com/kubestash/pvc/commit/41e1967) Prepare for release v0.12.0 (#41)
- [2b2a89f](https://github.com/kubestash/pvc/commit/2b2a89f) Add support for timeout for restic backup and restore (#40)
- [ebc03a1](https://github.com/kubestash/pvc/commit/ebc03a1) Use restic 0.17.1 (#39)



## [kubestash/volume-snapshotter](https://github.com/kubestash/volume-snapshotter)

### [v0.12.0](https://github.com/kubestash/volume-snapshotter/releases/tag/v0.12.0)

- [96a0761a](https://github.com/kubestash/volume-snapshotter/commit/96a0761a) Prepare for release v0.12.0 (#34)



## [kubestash/workload](https://github.com/kubestash/workload)

### [v0.12.0](https://github.com/kubestash/workload/releases/tag/v0.12.0)

- [fb9febf](https://github.com/kubestash/workload/commit/fb9febf) Prepare for release v0.12.0 (#51)
- [5515c6d](https://github.com/kubestash/workload/commit/5515c6d) Add support for timeout for restic backup and restore (#50)




