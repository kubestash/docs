---
title: Changelog | KubeStash
description: Changelog
menu:
  docs_{{.version}}:
    identifier: changelog-kubestash-v2026.2.16-rc.0
    name: Changelog-v2026.2.16-rc.0
    parent: welcome
    weight: 20260216
product_name: kubestash
menu_name: docs_{{.version}}
section_menu_id: welcome
url: /docs/{{.version}}/welcome/changelog-v2026.2.16-rc.0/
aliases:
  - /docs/{{.version}}/CHANGELOG-v2026.2.16-rc.0/
---

# KubeStash v2026.2.16-rc.0 (2026-02-18)


## [kubestash/apimachinery](https://github.com/kubestash/apimachinery)

### [v0.24.0-rc.0](https://github.com/kubestash/apimachinery/releases/tag/v0.24.0-rc.0)

- [f43decb5](https://github.com/kubestash/apimachinery/commit/f43decb5) Update deps (#203)
- [e6004c6a](https://github.com/kubestash/apimachinery/commit/e6004c6a) Introduce `LogRetentionStatus` in the snapshot status (#182)
- [ee2e89dd](https://github.com/kubestash/apimachinery/commit/ee2e89dd) Test against k8s 1.35 (#202)
- [9bd2f734](https://github.com/kubestash/apimachinery/commit/9bd2f734) Make restic package as standalone & improve design (#198)
- [0f9b8460](https://github.com/kubestash/apimachinery/commit/0f9b8460) Fix Checksum error issue (#201)
- [958f80fe](https://github.com/kubestash/apimachinery/commit/958f80fe) Add type and reason for pod pending state (#200)
- [d9bfd689](https://github.com/kubestash/apimachinery/commit/d9bfd689) Test against k8s 1.35 (#197)
- [da2f78e7](https://github.com/kubestash/apimachinery/commit/da2f78e7) Generate annotation for s3 or gcs buckets (#196)
- [98384101](https://github.com/kubestash/apimachinery/commit/98384101) Add cloud annotations (#195)
- [b15af1d8](https://github.com/kubestash/apimachinery/commit/b15af1d8) Use image to refer to full docker image (#193)



## [kubestash/cli](https://github.com/kubestash/cli)

### [v0.23.0-rc.0](https://github.com/kubestash/cli/releases/tag/v0.23.0-rc.0)

- [507c9ba8](https://github.com/kubestash/cli/commit/507c9ba8) Prepare for release v0.23.0-rc.0 (#78)
- [019b6bab](https://github.com/kubestash/cli/commit/019b6bab) Incorporate changes for restic standalone pkg (#77)
- [2ab275f0](https://github.com/kubestash/cli/commit/2ab275f0) Prepare for release v0.22.0 (#75)



## [kubestash/installer](https://github.com/kubestash/installer)

### [v2026.2.16-rc.0](https://github.com/kubestash/installer/releases/tag/v2026.2.16-rc.0)

- [184bb50](https://github.com/kubestash/installer/commit/184bb50) Prepare for release v2026.2.16-rc.0 (#292)
- [0cb552b](https://github.com/kubestash/installer/commit/0cb552b) Update cve report (#291)
- [6e0e180](https://github.com/kubestash/installer/commit/6e0e180) Update cve report (#290)
- [f4f22fc](https://github.com/kubestash/installer/commit/f4f22fc) Update cve report (#289)



## [kubestash/kubedump](https://github.com/kubestash/kubedump)

### [v0.23.0-rc.0](https://github.com/kubestash/kubedump/releases/tag/v0.23.0-rc.0)

- [90556c58](https://github.com/kubestash/kubedump/commit/90556c58) Prepare for release v0.23.0-rc.0 (#77)
- [fa249783](https://github.com/kubestash/kubedump/commit/fa249783) Incorporate changes for restic standalone pkg (#76)
- [39b135f5](https://github.com/kubestash/kubedump/commit/39b135f5) Prepare for release v0.22.0 (#74)



## [kubestash/kubestash](https://github.com/kubestash/kubestash)

### [v0.24.0-rc.0](https://github.com/kubestash/kubestash/releases/tag/v0.24.0-rc.0)

- [83aed086](https://github.com/kubestash/kubestash/commit/83aed086) Prepare for release v0.24.0-rc.0 (#335)
- [b8d8eef2](https://github.com/kubestash/kubestash/commit/b8d8eef2) Add job get perimission (#331)
- [f54a784a](https://github.com/kubestash/kubestash/commit/f54a784a) Incorporate changes for restic standalone pkg (#334)
- [33fce7b2](https://github.com/kubestash/kubestash/commit/33fce7b2) Fix: retentionPolicy pod stuck in pending state (#332)
- [19a812cb](https://github.com/kubestash/kubestash/commit/19a812cb) Propagate cloud and bucket annotations  (#330)
- [3bd7fd28](https://github.com/kubestash/kubestash/commit/3bd7fd28) Prepare for release v0.23.0 (#329)



## [kubestash/manifest](https://github.com/kubestash/manifest)

### [v0.16.0-rc.0](https://github.com/kubestash/manifest/releases/tag/v0.16.0-rc.0)

- [d1b44811](https://github.com/kubestash/manifest/commit/d1b44811) Prepare for release v0.16.0-rc.0 (#56)
- [eac26f25](https://github.com/kubestash/manifest/commit/eac26f25) Incorporate changes for restic standalone pkg (#55)
- [2a6ef12b](https://github.com/kubestash/manifest/commit/2a6ef12b) Prepare for release v0.15.0 (#53)



## [kubestash/pvc](https://github.com/kubestash/pvc)

### [v0.23.0-rc.0](https://github.com/kubestash/pvc/releases/tag/v0.23.0-rc.0)

- [1b2d13fb](https://github.com/kubestash/pvc/commit/1b2d13fb) Prepare for release v0.23.0-rc.0 (#77)
- [3cbdd9c3](https://github.com/kubestash/pvc/commit/3cbdd9c3) Incorporate changes for restic standalone pkg (#75)
- [fcc82269](https://github.com/kubestash/pvc/commit/fcc82269) Prepare for release v0.22.0 (#74)



## [kubestash/volume-snapshotter](https://github.com/kubestash/volume-snapshotter)

### [v0.23.0-rc.0](https://github.com/kubestash/volume-snapshotter/releases/tag/v0.23.0-rc.0)

- [6d1de725](https://github.com/kubestash/volume-snapshotter/commit/6d1de725) Prepare for release v0.23.0-rc.0 (#67)
- [8992419d](https://github.com/kubestash/volume-snapshotter/commit/8992419d) Prepare for release v0.22.0 (#65)



## [kubestash/workload](https://github.com/kubestash/workload)

### [v0.23.0-rc.0](https://github.com/kubestash/workload/releases/tag/v0.23.0-rc.0)

- [b9d79bce](https://github.com/kubestash/workload/commit/b9d79bce) Prepare for release v0.23.0-rc.0 (#87)
- [01db48cf](https://github.com/kubestash/workload/commit/01db48cf) Incorporate changes for restic standalone pkg (#86)
- [a9800705](https://github.com/kubestash/workload/commit/a9800705) Prepare for release v0.22.0 (#84)




