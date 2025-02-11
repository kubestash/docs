---
title: Changelog | KubeStash
description: Changelog
menu:
  docs_{{.version}}:
    identifier: changelog-kubestash-v2025.2.10
    name: Changelog-v2025.2.10
    parent: welcome
    weight: 20250210
product_name: kubestash
menu_name: docs_{{.version}}
section_menu_id: welcome
url: /docs/{{.version}}/welcome/changelog-v2025.2.10/
aliases:
  - /docs/{{.version}}/CHANGELOG-v2025.2.10/
---

# KubeStash v2025.2.10 (2025-02-11)


## [kubestash/apimachinery](https://github.com/kubestash/apimachinery)

### [v0.16.0](https://github.com/kubestash/apimachinery/releases/tag/v0.16.0)

- [57156d93](https://github.com/kubestash/apimachinery/commit/57156d93) Update deps
- [1a6fe49b](https://github.com/kubestash/apimachinery/commit/1a6fe49b) fix s3-compitbale storage endpoint resolver (#156)
- [f1e579e0](https://github.com/kubestash/apimachinery/commit/f1e579e0) Disable image caching in setup-qemu action (#155)
- [610cd73c](https://github.com/kubestash/apimachinery/commit/610cd73c) Update deps
- [125449b4](https://github.com/kubestash/apimachinery/commit/125449b4) add `GetCaPath` method to restic package (#153)
- [89b4aa59](https://github.com/kubestash/apimachinery/commit/89b4aa59) Fixed each backend envs separately for restic (#152)
- [add71a4c](https://github.com/kubestash/apimachinery/commit/add71a4c) Remove trailing slashes on backupStorage (#151)
- [4fcba883](https://github.com/kubestash/apimachinery/commit/4fcba883) Add seperate DeletionPolicy struct for backupConfig (#150)
- [20ea92f4](https://github.com/kubestash/apimachinery/commit/20ea92f4) Added archiver restore field into kubedbManifest options  (#143)
- [ba0f7c7d](https://github.com/kubestash/apimachinery/commit/ba0f7c7d) Add Stdin Backup Leaf Command support (#140)
- [9169b0b2](https://github.com/kubestash/apimachinery/commit/9169b0b2) Convert `aws-sdk-v1` to `aws-sdk-v2` (#148)
- [bb8c168f](https://github.com/kubestash/apimachinery/commit/bb8c168f) Add proxyEnvs functionfor internet-security (#149)
- [60e237cb](https://github.com/kubestash/apimachinery/commit/60e237cb) Hard code the function images that dont abide by rules  (#147)



## [kubestash/cli](https://github.com/kubestash/cli)

### [v0.15.0](https://github.com/kubestash/cli/releases/tag/v0.15.0)

- [0bb3aa7](https://github.com/kubestash/cli/commit/0bb3aa7) Prepare for release v0.15.0 (#48)
- [0e2c3a4](https://github.com/kubestash/cli/commit/0e2c3a4) Disable image caching in setup-qemu action (#47)
- [9538f75](https://github.com/kubestash/cli/commit/9538f75) Prepare for release v0.15.0-rc.0 (#46)
- [1b89868](https://github.com/kubestash/cli/commit/1b89868) Incorporate with `go-sh` leaf command execution (#44)



## [kubestash/installer](https://github.com/kubestash/installer)

### [v2025.2.10](https://github.com/kubestash/installer/releases/tag/v2025.2.10)

- [079138b](https://github.com/kubestash/installer/commit/079138b) Prepare for release v2025.2.10 (#144)
- [709ac6f](https://github.com/kubestash/installer/commit/709ac6f) Update cve report (#143)
- [79caa1a](https://github.com/kubestash/installer/commit/79caa1a) Update cve report (#142)
- [3d61832](https://github.com/kubestash/installer/commit/3d61832) Update cve report (#141)
- [a719119](https://github.com/kubestash/installer/commit/a719119) Update cve report (#140)
- [ba88087](https://github.com/kubestash/installer/commit/ba88087) Disable image caching in setup-qemu action (#139)
- [f0bb881](https://github.com/kubestash/installer/commit/f0bb881) Prepare for release v2025.2.6-rc.0 (#138)
- [9200c2d](https://github.com/kubestash/installer/commit/9200c2d) Update ace-user-roles chart schema
- [44395a0](https://github.com/kubestash/installer/commit/44395a0) Move user roles to ace-user-roles chart (#137)
- [22c5788](https://github.com/kubestash/installer/commit/22c5788) Get and create permission added for archiver (#136)
- [2b60ace](https://github.com/kubestash/installer/commit/2b60ace) Update cve report (#135)
- [340f1fb](https://github.com/kubestash/installer/commit/340f1fb) Update cve report (#132)
- [862ddc7](https://github.com/kubestash/installer/commit/862ddc7) Pass env variables to operator (#133)
- [d89ce25](https://github.com/kubestash/installer/commit/d89ce25) Update cve report (#131)



## [kubestash/kubedump](https://github.com/kubestash/kubedump)

### [v0.15.0](https://github.com/kubestash/kubedump/releases/tag/v0.15.0)

- [dd0d658](https://github.com/kubestash/kubedump/commit/dd0d658) Prepare for release v0.15.0 (#41)
- [9af244f](https://github.com/kubestash/kubedump/commit/9af244f) Disable image caching in setup-qemu action (#40)
- [627cfb7](https://github.com/kubestash/kubedump/commit/627cfb7) Prepare for release v0.15.0-rc.0 (#39)
- [06e15fb](https://github.com/kubestash/kubedump/commit/06e15fb) Incorporate with go-sh leaf command execution (#38)



## [kubestash/kubestash](https://github.com/kubestash/kubestash)

### [v0.16.0](https://github.com/kubestash/kubestash/releases/tag/v0.16.0)

- [f55d932d](https://github.com/kubestash/kubestash/commit/f55d932d) Prepare for release v0.16.0 (#278)
- [37bc93fb](https://github.com/kubestash/kubestash/commit/37bc93fb) Fix for s3-compatible storage
- [600af18a](https://github.com/kubestash/kubestash/commit/600af18a) Disable image caching in setup-qemu action (#277)
- [c62da9e8](https://github.com/kubestash/kubestash/commit/c62da9e8) Prepare for release v0.16.0-rc.0 (#276)
- [4e96fbbd](https://github.com/kubestash/kubestash/commit/4e96fbbd) Fix job controller for retention policy jobs (#275)
- [d436001b](https://github.com/kubestash/kubestash/commit/d436001b) Set rbac permissions for archiver backup & restore  (#265)
- [37e903e0](https://github.com/kubestash/kubestash/commit/37e903e0) Update deps with new restic package changes (#273)
- [beca6fbf](https://github.com/kubestash/kubestash/commit/beca6fbf) Delete repositories based on the backupConfig deletionPolicy (#274)
- [af6db7bd](https://github.com/kubestash/kubestash/commit/af6db7bd) Set proxy env variables to containers (#272)



## [kubestash/manifest](https://github.com/kubestash/manifest)

### [v0.8.0](https://github.com/kubestash/manifest/releases/tag/v0.8.0)

- [88fe74e](https://github.com/kubestash/manifest/commit/88fe74e) Prepare for release v0.8.0 (#25)
- [60fab00](https://github.com/kubestash/manifest/commit/60fab00) Disable image caching in setup-qemu action (#24)
- [f5591b5](https://github.com/kubestash/manifest/commit/f5591b5) Prepare for release v0.8.0-rc.0 (#23)
- [ce23273](https://github.com/kubestash/manifest/commit/ce23273) Incorporate with `go-sh` leaf command execution (#22)



## [kubestash/pvc](https://github.com/kubestash/pvc)

### [v0.15.0](https://github.com/kubestash/pvc/releases/tag/v0.15.0)

- [044e79b](https://github.com/kubestash/pvc/commit/044e79b) Prepare for release v0.15.0 (#50)
- [946f9e2](https://github.com/kubestash/pvc/commit/946f9e2) Disable image caching in setup-qemu action (#49)
- [91f8469](https://github.com/kubestash/pvc/commit/91f8469) Prepare for release v0.15.0-rc.0 (#48)
- [7804c01](https://github.com/kubestash/pvc/commit/7804c01) Incorporate with `go-sh` leaf command execution (#47)



## [kubestash/volume-snapshotter](https://github.com/kubestash/volume-snapshotter)

### [v0.15.0](https://github.com/kubestash/volume-snapshotter/releases/tag/v0.15.0)

- [58ed7586](https://github.com/kubestash/volume-snapshotter/commit/58ed7586) Prepare for release v0.15.0 (#42)
- [4b15908d](https://github.com/kubestash/volume-snapshotter/commit/4b15908d) Disable image caching in setup-qemu action (#41)
- [5807d4b9](https://github.com/kubestash/volume-snapshotter/commit/5807d4b9) Prepare for release v0.15.0-rc.0 (#40)



## [kubestash/workload](https://github.com/kubestash/workload)

### [v0.15.0](https://github.com/kubestash/workload/releases/tag/v0.15.0)

- [33c8889](https://github.com/kubestash/workload/commit/33c8889) Prepare for release v0.15.0 (#61)
- [d923af9](https://github.com/kubestash/workload/commit/d923af9) Update github actions workflows (#60)
- [80d5a2a](https://github.com/kubestash/workload/commit/80d5a2a) Prepare for release v0.15.0-rc.0 (#59)
- [8f5ef5b](https://github.com/kubestash/workload/commit/8f5ef5b) Add Stdin Backup Leaf Command support (#58)




