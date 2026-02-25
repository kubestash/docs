---
title: Changelog | KubeStash
description: Changelog
menu:
  docs_{{.version}}:
    identifier: changelog-kubestash-v2026.2.27
    name: Changelog-v2026.2.27
    parent: welcome
    weight: 20260227
product_name: kubestash
menu_name: docs_{{.version}}
section_menu_id: welcome
url: /docs/{{.version}}/welcome/changelog-v2026.2.27/
aliases:
  - /docs/{{.version}}/CHANGELOG-v2026.2.27/
---

# KubeStash v2026.2.27 (2026-02-25)


## [kubestash/apimachinery](https://github.com/kubestash/apimachinery)

### [v0.24.0](https://github.com/kubestash/apimachinery/releases/tag/v0.24.0)

- [b7f693f8](https://github.com/kubestash/apimachinery/commit/b7f693f8) Annotations for GCP IAM values (#209)
- [0c6bdb31](https://github.com/kubestash/apimachinery/commit/0c6bdb31) Report features via site info (#208)
- [aabc34e9](https://github.com/kubestash/apimachinery/commit/aabc34e9) Run go fix ./... (#207)
- [fb24af1c](https://github.com/kubestash/apimachinery/commit/fb24af1c) Fix CVE (#206)
- [d0b5437e](https://github.com/kubestash/apimachinery/commit/d0b5437e) go fix ./... (#204)
- [f43decb5](https://github.com/kubestash/apimachinery/commit/f43decb5) Update deps (#203)
- [e6004c6a](https://github.com/kubestash/apimachinery/commit/e6004c6a) Introduce `LogRetentionStatus` in the snapshot status (#182)
- [ee2e89dd](https://github.com/kubestash/apimachinery/commit/ee2e89dd) Test against k8s 1.35 (#202)
- [9bd2f734](https://github.com/kubestash/apimachinery/commit/9bd2f734) Make restic package as standalone & improve design (#198)
- [0f9b8460](https://github.com/kubestash/apimachinery/commit/0f9b8460) Fix Checksum error issue (#201)
- [958f80fe](https://github.com/kubestash/apimachinery/commit/958f80fe) Add type and reason for pod pending state (#200)
- [d9bfd689](https://github.com/kubestash/apimachinery/commit/d9bfd689) Test against k8s 1.35 (#197)
- [da2f78e7](https://github.com/kubestash/apimachinery/commit/da2f78e7) Generate annotation for s3 or gcs buckets (#196)
- [98384101](https://github.com/kubestash/apimachinery/commit/98384101) Add cloud annotations (#195)



## [kubestash/cli](https://github.com/kubestash/cli)

### [v0.23.0](https://github.com/kubestash/cli/releases/tag/v0.23.0)

- [118b5869](https://github.com/kubestash/cli/commit/118b5869) Prepare for release v0.23.0 (#80)
- [507c9ba8](https://github.com/kubestash/cli/commit/507c9ba8) Prepare for release v0.23.0-rc.0 (#78)
- [019b6bab](https://github.com/kubestash/cli/commit/019b6bab) Incorporate changes for restic standalone pkg (#77)



## [kubestash/installer](https://github.com/kubestash/installer)

### [v2026.2.27](https://github.com/kubestash/installer/releases/tag/v2026.2.27)

- [d94a613](https://github.com/kubestash/installer/commit/d94a613) Prepare for release v2026.2.27 (#297)
- [0fd942c](https://github.com/kubestash/installer/commit/0fd942c) Add permission to report features in site info (#296)
- [b4d6f04](https://github.com/kubestash/installer/commit/b4d6f04) Update cve report (#295)
- [e114ef7](https://github.com/kubestash/installer/commit/e114ef7) Update cve report (#294)
- [60e5c76](https://github.com/kubestash/installer/commit/60e5c76) Update cve report (#293)
- [184bb50](https://github.com/kubestash/installer/commit/184bb50) Prepare for release v2026.2.16-rc.0 (#292)
- [0cb552b](https://github.com/kubestash/installer/commit/0cb552b) Update cve report (#291)
- [6e0e180](https://github.com/kubestash/installer/commit/6e0e180) Update cve report (#290)
- [f4f22fc](https://github.com/kubestash/installer/commit/f4f22fc) Update cve report (#289)



## [kubestash/kubedump](https://github.com/kubestash/kubedump)

### [v0.23.0](https://github.com/kubestash/kubedump/releases/tag/v0.23.0)

- [9d5cb936](https://github.com/kubestash/kubedump/commit/9d5cb936) Prepare for release v0.23.0 (#80)
- [3d28e95b](https://github.com/kubestash/kubedump/commit/3d28e95b) Run go fix ./... (#79)
- [90556c58](https://github.com/kubestash/kubedump/commit/90556c58) Prepare for release v0.23.0-rc.0 (#77)
- [fa249783](https://github.com/kubestash/kubedump/commit/fa249783) Incorporate changes for restic standalone pkg (#76)



## [kubestash/kubestash](https://github.com/kubestash/kubestash)

### [v0.24.0](https://github.com/kubestash/kubestash/releases/tag/v0.24.0)

- [80892b28](https://github.com/kubestash/kubestash/commit/80892b28) Prepare for release v0.24.0 (#338)
- [76f06837](https://github.com/kubestash/kubestash/commit/76f06837) Run go fix ./... (#337)
- [83aed086](https://github.com/kubestash/kubestash/commit/83aed086) Prepare for release v0.24.0-rc.0 (#335)
- [b8d8eef2](https://github.com/kubestash/kubestash/commit/b8d8eef2) Add job get perimission (#331)
- [f54a784a](https://github.com/kubestash/kubestash/commit/f54a784a) Incorporate changes for restic standalone pkg (#334)
- [33fce7b2](https://github.com/kubestash/kubestash/commit/33fce7b2) Fix: retentionPolicy pod stuck in pending state (#332)
- [19a812cb](https://github.com/kubestash/kubestash/commit/19a812cb) Propagate cloud and bucket annotations  (#330)



## [kubestash/manifest](https://github.com/kubestash/manifest)

### [v0.16.0](https://github.com/kubestash/manifest/releases/tag/v0.16.0)

- [3cfbccae](https://github.com/kubestash/manifest/commit/3cfbccae) Prepare for release v0.16.0 (#58)
- [d1b44811](https://github.com/kubestash/manifest/commit/d1b44811) Prepare for release v0.16.0-rc.0 (#56)
- [eac26f25](https://github.com/kubestash/manifest/commit/eac26f25) Incorporate changes for restic standalone pkg (#55)



## [kubestash/pvc](https://github.com/kubestash/pvc)

### [v0.23.0](https://github.com/kubestash/pvc/releases/tag/v0.23.0)

- [b866d4c4](https://github.com/kubestash/pvc/commit/b866d4c4) Prepare for release v0.23.0 (#79)
- [1b2d13fb](https://github.com/kubestash/pvc/commit/1b2d13fb) Prepare for release v0.23.0-rc.0 (#77)
- [3cbdd9c3](https://github.com/kubestash/pvc/commit/3cbdd9c3) Incorporate changes for restic standalone pkg (#75)



## [kubestash/volume-snapshotter](https://github.com/kubestash/volume-snapshotter)

### [v0.23.0](https://github.com/kubestash/volume-snapshotter/releases/tag/v0.23.0)

- [a6d02e42](https://github.com/kubestash/volume-snapshotter/commit/a6d02e42) Prepare for release v0.23.0 (#69)
- [6d1de725](https://github.com/kubestash/volume-snapshotter/commit/6d1de725) Prepare for release v0.23.0-rc.0 (#67)



## [kubestash/workload](https://github.com/kubestash/workload)

### [v0.23.0](https://github.com/kubestash/workload/releases/tag/v0.23.0)

- [1cbaadbe](https://github.com/kubestash/workload/commit/1cbaadbe) Prepare for release v0.23.0 (#89)
- [b9d79bce](https://github.com/kubestash/workload/commit/b9d79bce) Prepare for release v0.23.0-rc.0 (#87)
- [01db48cf](https://github.com/kubestash/workload/commit/01db48cf) Incorporate changes for restic standalone pkg (#86)




