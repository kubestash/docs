---
title: Changelog | KubeStash
description: Changelog
menu:
  docs_{{.version}}:
    identifier: changelog-kubestash-v2026.6.19
    name: Changelog-v2026.6.19
    parent: welcome
    weight: 20260619
product_name: kubestash
menu_name: docs_{{.version}}
section_menu_id: welcome
url: /docs/{{.version}}/welcome/changelog-v2026.6.19/
aliases:
  - /docs/{{.version}}/CHANGELOG-v2026.6.19/
---

# KubeStash v2026.6.19 (2026-06-19)


## [kubestash/apimachinery](https://github.com/kubestash/apimachinery)

### [v0.28.0](https://github.com/kubestash/apimachinery/releases/tag/v0.28.0)

- [62b1daff](https://github.com/kubestash/apimachinery/commit/62b1daff) Update Driverlist For Neo4j (#231)
- [d4ad6ef7](https://github.com/kubestash/apimachinery/commit/d4ad6ef7) Snapshot/Restoresession Failed if one component Failed (#243)
- [f0e134f9](https://github.com/kubestash/apimachinery/commit/f0e134f9) Add Progress structure in snapshot & restoresession status (#232)
- [f6141bc6](https://github.com/kubestash/apimachinery/commit/f6141bc6) Bump go.bytebuilders.dev/audit to v0.0.52 (#241)
- [6f7a74e0](https://github.com/kubestash/apimachinery/commit/6f7a74e0) Update go.bytebuilders.dev/audit to v0.0.51 (#240)
- [165cb08b](https://github.com/kubestash/apimachinery/commit/165cb08b) Add Stream Reader for Download & Upload (#239)
- [1d30072d](https://github.com/kubestash/apimachinery/commit/1d30072d) Add ClickHouse Manifest (#233)
- [d5787deb](https://github.com/kubestash/apimachinery/commit/d5787deb) Add CLAUDE.md pointing to AGENTS.md
- [54445516](https://github.com/kubestash/apimachinery/commit/54445516) Add AGENTS.md (#237)
- [41bc2438](https://github.com/kubestash/apimachinery/commit/41bc2438) Harden CI workflows (#235)
- [a8c23c8d](https://github.com/kubestash/apimachinery/commit/a8c23c8d) Add wal backup support for azure credless mode (#229)



## [kubestash/installer](https://github.com/kubestash/installer)

### [v2026.6.19](https://github.com/kubestash/installer/releases/tag/v2026.6.19)

- [a4435e6](https://github.com/kubestash/installer/commit/a4435e6) Prepare for release v2026.6.19 (#350)
- [b7a1bae](https://github.com/kubestash/installer/commit/b7a1bae) Introduce cilium network policy (#348)
- [4371fb6](https://github.com/kubestash/installer/commit/4371fb6) Prepare for release v2026.6.18-rc.2 (#349)
- [f2c3b1c](https://github.com/kubestash/installer/commit/f2c3b1c) Use ace-user-roles v2026.6.12 with audit cluster role (#347)
- [6453c9d](https://github.com/kubestash/installer/commit/6453c9d) Prepare for release v2026.5.18-rc.0 (#345)
- [179c8de](https://github.com/kubestash/installer/commit/179c8de) Add CLAUDE.md pointing to AGENTS.md



## [kubestash/kubedump](https://github.com/kubestash/kubedump)

### [v0.27.0](https://github.com/kubestash/kubedump/releases/tag/v0.27.0)

- [ac91852b](https://github.com/kubestash/kubedump/commit/ac91852b) Prepare for release v0.27.0 (#97)
- [a87ce1d7](https://github.com/kubestash/kubedump/commit/a87ce1d7) Prepare for release v0.27.0-rc.2 (#96)
- [3c5bec08](https://github.com/kubestash/kubedump/commit/3c5bec08) Add Backup & Restore Progress Streaming (#95)
- [512026be](https://github.com/kubestash/kubedump/commit/512026be) Harden release and release-tracker workflows (#92)
- [813646cd](https://github.com/kubestash/kubedump/commit/813646cd) Prepare for release v0.27.0-rc.0 (#91)
- [74c76a73](https://github.com/kubestash/kubedump/commit/74c76a73) Add CLAUDE.md pointing to AGENTS.md
- [c80250fd](https://github.com/kubestash/kubedump/commit/c80250fd) Add AGENTS.md (#90)
- [15603b29](https://github.com/kubestash/kubedump/commit/15603b29) Harden CI workflows (#89)



## [kubestash/kubestash](https://github.com/kubestash/kubestash)

### [v0.28.0](https://github.com/kubestash/kubestash/releases/tag/v0.28.0)

- [02990a07](https://github.com/kubestash/kubestash/commit/02990a07) Prepare for release v0.28.0 (#373)
- [4697f09d](https://github.com/kubestash/kubestash/commit/4697f09d) Skip BackupSession when another one is running (#372)
- [5ee35503](https://github.com/kubestash/kubestash/commit/5ee35503) Bump github.com/moby/spdystream to v0.5.1 (#371)
- [8b46cb0f](https://github.com/kubestash/kubestash/commit/8b46cb0f) Prepare for release v0.28.0-rc.2 (#370)
- [75e18ed2](https://github.com/kubestash/kubestash/commit/75e18ed2) Fix Restic Repository Integrity Issue Skylotec (#369)
- [7fe53c55](https://github.com/kubestash/kubestash/commit/7fe53c55) Incorporate with API changes (#368)
- [5f80e378](https://github.com/kubestash/kubestash/commit/5f80e378) Pin openshift-preflight version in release workflow (#365)
- [78e9dc7c](https://github.com/kubestash/kubestash/commit/78e9dc7c) Document why Dockerfile.dbg keeps root user (#367)
- [9be833f1](https://github.com/kubestash/kubestash/commit/9be833f1) Remove duplicate NewCmdDownload registration (#363)
- [d1502070](https://github.com/kubestash/kubestash/commit/d1502070) Harden release and release-tracker workflows
- [d4f0312f](https://github.com/kubestash/kubestash/commit/d4f0312f) Prepare for release v0.28.0-rc.0 (#361)
- [dc81c8ab](https://github.com/kubestash/kubestash/commit/dc81c8ab) Add CLAUDE.md pointing to AGENTS.md
- [2a4c7d2f](https://github.com/kubestash/kubestash/commit/2a4c7d2f) Add AGENTS.md (#360)
- [dcbb855f](https://github.com/kubestash/kubestash/commit/dcbb855f) Harden CI workflows (#359)
- [c6e91ff7](https://github.com/kubestash/kubestash/commit/c6e91ff7) Give RBAC Permission for Vault (#356)



## [kubestash/manifest](https://github.com/kubestash/manifest)

### [v0.20.0](https://github.com/kubestash/manifest/releases/tag/v0.20.0)

- [59ba0025](https://github.com/kubestash/manifest/commit/59ba0025) Prepare for release v0.20.0 (#73)
- [8e1922f1](https://github.com/kubestash/manifest/commit/8e1922f1) Prepare for release v0.20.0-rc.2 (#72)
- [6b66a2e4](https://github.com/kubestash/manifest/commit/6b66a2e4) Add Backup & Restore Progress Streaming (#71)
- [ef88695a](https://github.com/kubestash/manifest/commit/ef88695a) Harden release and release-tracker workflows
- [5a77d020](https://github.com/kubestash/manifest/commit/5a77d020) Prepare for release v0.20.0-rc.0 (#69)
- [6ab52d94](https://github.com/kubestash/manifest/commit/6ab52d94) Add CLAUDE.md pointing to AGENTS.md
- [6a59c1db](https://github.com/kubestash/manifest/commit/6a59c1db) Add AGENTS.md (#68)
- [624ce1be](https://github.com/kubestash/manifest/commit/624ce1be) Harden CI workflows (#67)



## [kubestash/pvc](https://github.com/kubestash/pvc)

### [v0.27.0](https://github.com/kubestash/pvc/releases/tag/v0.27.0)

- [0897ae03](https://github.com/kubestash/pvc/commit/0897ae03) Prepare for release v0.27.0 (#94)
- [5401cafc](https://github.com/kubestash/pvc/commit/5401cafc) Prepare for release v0.27.0-rc.2 (#93)
- [7d35b81d](https://github.com/kubestash/pvc/commit/7d35b81d) Add Backup & Restore Progress Streaming (#88)
- [ed757857](https://github.com/kubestash/pvc/commit/ed757857) Harden release and release-tracker workflows
- [f8085237](https://github.com/kubestash/pvc/commit/f8085237) Prepare for release v0.27.0-rc.0 (#92)
- [4a75a698](https://github.com/kubestash/pvc/commit/4a75a698) Add CLAUDE.md pointing to AGENTS.md
- [5a5ac911](https://github.com/kubestash/pvc/commit/5a5ac911) Add AGENTS.md (#91)
- [c653a576](https://github.com/kubestash/pvc/commit/c653a576) Harden CI workflows (#90)



## [kubestash/vault](https://github.com/kubestash/vault)

### [v0.2.0](https://github.com/kubestash/vault/releases/tag/v0.2.0)

- [703269d](https://github.com/kubestash/vault/commit/703269d) Prepare for release v0.2.0 (#11)
- [38fecc1](https://github.com/kubestash/vault/commit/38fecc1) Prepare for release v0.2.0-rc.2 (#10)
- [5a75c93](https://github.com/kubestash/vault/commit/5a75c93) Prepare for release v0.2.0-rc.0 (#9)
- [87ff5ea](https://github.com/kubestash/vault/commit/87ff5ea) Harden CI workflows (#8)
- [32c3c24](https://github.com/kubestash/vault/commit/32c3c24) Fix docker Entrypoint



## [kubestash/volume-snapshotter](https://github.com/kubestash/volume-snapshotter)

### [v0.27.0](https://github.com/kubestash/volume-snapshotter/releases/tag/v0.27.0)

- [de72c3b1](https://github.com/kubestash/volume-snapshotter/commit/de72c3b1) Prepare for release v0.27.0 (#79)
- [ab5a5de7](https://github.com/kubestash/volume-snapshotter/commit/ab5a5de7) Prepare for release v0.27.0-rc.2 (#78)
- [3a71c59e](https://github.com/kubestash/volume-snapshotter/commit/3a71c59e) Harden release and release-tracker workflows
- [d970b2d0](https://github.com/kubestash/volume-snapshotter/commit/d970b2d0) Prepare for release v0.27.0-rc.0 (#77)
- [abfaa512](https://github.com/kubestash/volume-snapshotter/commit/abfaa512) Add CLAUDE.md pointing to AGENTS.md
- [43682b88](https://github.com/kubestash/volume-snapshotter/commit/43682b88) Add AGENTS.md (#76)
- [2f5685aa](https://github.com/kubestash/volume-snapshotter/commit/2f5685aa) Harden CI workflows (#75)



## [kubestash/workload](https://github.com/kubestash/workload)

### [v0.27.0](https://github.com/kubestash/workload/releases/tag/v0.27.0)

- [dcfcac5e](https://github.com/kubestash/workload/commit/dcfcac5e) Prepare for release v0.27.0 (#105)
- [3d571a3d](https://github.com/kubestash/workload/commit/3d571a3d) Prepare for release v0.27.0-rc.2 (#104)
- [2488fbb7](https://github.com/kubestash/workload/commit/2488fbb7) Add Backup & Restore Progress Streaming (#103)
- [ea347e19](https://github.com/kubestash/workload/commit/ea347e19) Harden release and release-tracker workflows
- [e7622298](https://github.com/kubestash/workload/commit/e7622298) Prepare for release v0.27.0-rc.0 (#100)
- [6d56a144](https://github.com/kubestash/workload/commit/6d56a144) Add CLAUDE.md pointing to AGENTS.md
- [03cc19e7](https://github.com/kubestash/workload/commit/03cc19e7) Add AGENTS.md (#99)
- [1cd1314d](https://github.com/kubestash/workload/commit/1cd1314d) Harden CI workflows (#98)




