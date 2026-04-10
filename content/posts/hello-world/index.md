---
title: "Hello, shingoon-lab"
date: 2026-04-11T00:10:00+09:00
draft: false
tags: ["announcement"]
categories: ["notice"]
series: []
author: "신호철 (shingoon)"
description: "shingoon-lab 오픈 인프라스트럭처 기술 블로그 오픈 공지"
cover:
  image: ""
  alt: ""
youtube: ""
toc: true
ShowToc: true
TocOpen: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowCodeCopyButtons: true
---

## 블로그를 시작하며

`shingoon-lab`은 **Open Infrastructure** 영역의 기술을 1차 출처 기반으로 기록하는 기술 블로그입니다.

## 다룰 주제

- **OpenStack**: 2024.2 Dalmatian 이후 릴리스, Kolla-Ansible 운영, Neutron/OVN
- **Ceph**: Reef/Squid 이후 릴리스, RBD·CephFS·RGW 벤치마킹, BlueStore 튜닝
- **Kubernetes**: Operator 패턴, CRI-O/containerd, Cilium/Calico
- **Observability**: Prometheus, Grafana, Loki, Thanos 운영 경험
- **MLOps**: Kubeflow, KServe, Ray 기반 인프라

## 팩트 기반 원칙

모든 글은 아래 원칙을 따릅니다.

1. **1차 출처 우선** — 공식 문서 > 공식 GitHub > RFC > 벤더 문서
2. **버전 고정** — 모든 기술은 구체적 버전 + Release Date 명시
3. **추측 금지** — 확인되지 않은 내용은 포함하지 않음

## 예시 코드블록

OpenStack 2024.2 컨트롤러 노드에서 버전 확인:

```bash
[root@controller ~]# openstack --version
openstack 7.1.0
```

Ceph Squid 19.x 클러스터 상태 확인:

```bash
[root@ceph-mon01 ~]# ceph -s
  cluster:
    id:     00000000-0000-0000-0000-000000000000
    health: HEALTH_OK
```

---

앞으로 꾸준히 업데이트 하겠습니다.
