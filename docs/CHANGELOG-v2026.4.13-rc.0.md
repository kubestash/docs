---
title: Changelog | KubeStash
description: Changelog
menu:
  docs_{{.version}}:
    identifier: changelog-kubestash-v2026.4.13-rc.0
    name: Changelog-v2026.4.13-rc.0
    parent: welcome
    weight: 20260413
product_name: kubestash
menu_name: docs_{{.version}}
section_menu_id: welcome
url: /docs/{{.version}}/welcome/changelog-v2026.4.13-rc.0/
aliases:
  - /docs/{{.version}}/CHANGELOG-v2026.4.13-rc.0/
---

# KubeStash v2026.4.13-rc.0 (2026-04-15)


## [kubestash/apimachinery](https://github.com/kubestash/apimachinery)

### [v0.27.0-rc.0](https://github.com/kubestash/apimachinery/releases/tag/v0.27.0-rc.0)

- [edb0aa0a](https://github.com/kubestash/apimachinery/commit/edb0aa0a) Cleanup CVEs
- [471e1d5b](https://github.com/kubestash/apimachinery/commit/471e1d5b) Add Cloud Annotations for restore session (#226)
- [0532bb60](https://github.com/kubestash/apimachinery/commit/0532bb60) Fix vendor for restic plugins (#222)
- [ca5351d9](https://github.com/kubestash/apimachinery/commit/ca5351d9) Add credential manger and  cloud annotation methods (#220)
- [65ad1567](https://github.com/kubestash/apimachinery/commit/65ad1567) Replace azidentity master with beta version (#221)
- [0b73be0b](https://github.com/kubestash/apimachinery/commit/0b73be0b) Update blob & annotations for azure identity binding (#217)
- [d6aff5e5](https://github.com/kubestash/apimachinery/commit/d6aff5e5) Test against k8s 1.35 (#218)
- [d106d3eb](https://github.com/kubestash/apimachinery/commit/d106d3eb) Inject S3 credential Manually for IRSA
- [aa7a3c4f](https://github.com/kubestash/apimachinery/commit/aa7a3c4f) Add ClickHouse Stats
- [d1da8aec](https://github.com/kubestash/apimachinery/commit/d1da8aec) Update aws annotations for seed role (#214)
- [e1b88a14](https://github.com/kubestash/apimachinery/commit/e1b88a14) Update KubeDB and cert-manager (#211)



## [kubestash/cli](https://github.com/kubestash/cli)

### [v0.26.0-rc.0](https://github.com/kubestash/cli/releases/tag/v0.26.0-rc.0)

- [b23f01ca](https://github.com/kubestash/cli/commit/b23f01ca) Prepare for release v0.26.0-rc.0 (#89)
- [79d4abf0](https://github.com/kubestash/cli/commit/79d4abf0) Incorporate changes for the AWS credless feature (#85)



## [kubestash/installer](https://github.com/kubestash/installer)

### [v2026.4.13-rc.0](https://github.com/kubestash/installer/releases/tag/v2026.4.13-rc.0)

- [c0d134c](https://github.com/kubestash/installer/commit/c0d134c) Prepare for release v2026.4.13-rc.0 (#330)
- [736ee3a](https://github.com/kubestash/installer/commit/736ee3a) Update cve report (#329)
- [bd53f74](https://github.com/kubestash/installer/commit/bd53f74) Update cve report (#328)
- [3636538](https://github.com/kubestash/installer/commit/3636538) Update cve report (#327)
- [0426ac5](https://github.com/kubestash/installer/commit/0426ac5) Update cve report (#326)
- [b4e6591](https://github.com/kubestash/installer/commit/b4e6591) Update cve report (#325)
- [ff05930](https://github.com/kubestash/installer/commit/ff05930) Update cve report (#324)
- [bf7d377](https://github.com/kubestash/installer/commit/bf7d377) Update cve report (#323)
- [494bbe1](https://github.com/kubestash/installer/commit/494bbe1) Update cve report (#322)
- [9c65356](https://github.com/kubestash/installer/commit/9c65356) Update cve report (#321)
- [83bc968](https://github.com/kubestash/installer/commit/83bc968) Update cve report (#320)
- [1875a58](https://github.com/kubestash/installer/commit/1875a58) Update cve report (#319)
- [4d19c59](https://github.com/kubestash/installer/commit/4d19c59) Update cve report (#318)
- [0c54731](https://github.com/kubestash/installer/commit/0c54731) charts/kubestash-operator: rename token secret to metrics-token (#317)
- [7e6ed5d](https://github.com/kubestash/installer/commit/7e6ed5d) charts/kubestash-operator: use authorization block with SA token Secret (#316)
- [ecb17c9](https://github.com/kubestash/installer/commit/ecb17c9) Update cve report (#315)
- [4928b5b](https://github.com/kubestash/installer/commit/4928b5b) Update cve report (#314)
- [01565ae](https://github.com/kubestash/installer/commit/01565ae) Update cve report (#313)
- [a842676](https://github.com/kubestash/installer/commit/a842676) Update cve report (#312)
- [8dc1f4e](https://github.com/kubestash/installer/commit/8dc1f4e) Update cve report (#311)
- [7ea16e3](https://github.com/kubestash/installer/commit/7ea16e3) Update sub charts w/ cert-manager support (#310)
- [2825f41](https://github.com/kubestash/installer/commit/2825f41) Fix CVEs (#309)
- [365aa34](https://github.com/kubestash/installer/commit/365aa34) Unset runAsUser fields in securityContext (#308)
- [de5b698](https://github.com/kubestash/installer/commit/de5b698) Add cert-manager CRD import for kubestash-operator chart (#307)
- [ccc1d03](https://github.com/kubestash/installer/commit/ccc1d03) Add cert-manager option for kubestash-operator serving certs (#306)
- [6bbfb61](https://github.com/kubestash/installer/commit/6bbfb61) Add EndpointSlice rbac permission (#304)
- [cda1d77](https://github.com/kubestash/installer/commit/cda1d77) Update cve report (#302)
- [2a105a5](https://github.com/kubestash/installer/commit/2a105a5) Update cve report (#301)
- [610c0c1](https://github.com/kubestash/installer/commit/610c0c1) Update cve report (#300)
- [9a556ac](https://github.com/kubestash/installer/commit/9a556ac) Prepare for release v2026.2.26 (#299)



## [kubestash/kubedump](https://github.com/kubestash/kubedump)

### [v0.26.0-rc.0](https://github.com/kubestash/kubedump/releases/tag/v0.26.0-rc.0)

- [7d5d93d3](https://github.com/kubestash/kubedump/commit/7d5d93d3) Prepare for release v0.26.0-rc.0 (#84)
- [25d059a6](https://github.com/kubestash/kubedump/commit/25d059a6) Incorporate changes for the AWS credless feature (#83)



## [kubestash/kubestash](https://github.com/kubestash/kubestash)

### [v0.27.0-rc.0](https://github.com/kubestash/kubestash/releases/tag/v0.27.0-rc.0)

- [4dce7391](https://github.com/kubestash/kubestash/commit/4dce7391) Prepare for release v0.27.0-rc.0 (#351)
- [144181ed](https://github.com/kubestash/kubestash/commit/144181ed) Update apimachinery and unsuspend job for azure identity binding (#346)
- [4054ab20](https://github.com/kubestash/kubestash/commit/4054ab20) Update job controller to unsuspend job for AWS credless
- [084529b5](https://github.com/kubestash/kubestash/commit/084529b5) Give endpointslices permissions (#341)



## [kubestash/manifest](https://github.com/kubestash/manifest)

### [v0.19.0-rc.0](https://github.com/kubestash/manifest/releases/tag/v0.19.0-rc.0)

- [a2927d49](https://github.com/kubestash/manifest/commit/a2927d49) Prepare for release v0.19.0-rc.0 (#62)
- [42ac18cf](https://github.com/kubestash/manifest/commit/42ac18cf) Incorporate changes for the AWS credless feature (#61)



## [kubestash/pvc](https://github.com/kubestash/pvc)

### [v0.26.0-rc.0](https://github.com/kubestash/pvc/releases/tag/v0.26.0-rc.0)

- [de896841](https://github.com/kubestash/pvc/commit/de896841) Prepare for release v0.26.0-rc.0 (#83)
- [b4565d9e](https://github.com/kubestash/pvc/commit/b4565d9e) Incorporate changes for aws credless operation (#82)



## [kubestash/volume-snapshotter](https://github.com/kubestash/volume-snapshotter)

### [v0.26.0-rc.0](https://github.com/kubestash/volume-snapshotter/releases/tag/v0.26.0-rc.0)

- [d782a3b8](https://github.com/kubestash/volume-snapshotter/commit/d782a3b8) Prepare for release v0.26.0-rc.0 (#72)



## [kubestash/workload](https://github.com/kubestash/workload)

### [v0.26.0-rc.0](https://github.com/kubestash/workload/releases/tag/v0.26.0-rc.0)

- [a6863790](https://github.com/kubestash/workload/commit/a6863790) Prepare for release v0.26.0-rc.0 (#93)
- [fd2954c8](https://github.com/kubestash/workload/commit/fd2954c8) Incorporate changes for the AWS credless feature




