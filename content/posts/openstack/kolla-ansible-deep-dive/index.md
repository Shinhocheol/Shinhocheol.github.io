---
title: "Kolla-Ansible 파헤치기 — 탄생 배경부터 merge_configs 코드까지"
date: 2026-04-11T01:20:00+09:00
draft: false
tags: ["openstack", "kolla-ansible", "ansible", "deployment", "internals", "nova", "ceph", "slurp"]
categories: ["openstack"]
series: ["kolla-ansible-internals"]
author: "신호철 (shingoon)"
description: "Kolla 프로젝트의 탄생 배경과 코알라 마스코트의 유래, SLURP 릴리즈 정책, Kolla-Ansible이 배포하는 컨테이너 이미지, 그리고 merge_configs 액션 플러그인 코드 분석까지 1차 출처만으로 정리했습니다."
cover:
  image: "images/kolla-mascot.png"
  alt: "OpenStack Kolla project mascot (Koala)"
  caption: "Source: openstack.org/project-mascots (Kolla-ansible)"
  relative: true
youtube: ""
toc: true
ShowToc: true
TocOpen: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowCodeCopyButtons: true
---

## 들어가며 — 이 글이 답하고 싶은 질문들

Kolla-Ansible로 OpenStack을 한 번이라도 올려본 분이라면 거의 틀림없이 이런 상황을 겪어 봤을 겁니다.

- `/etc/kolla/config/nova.conf`에 한 줄 넣고 `kolla-ansible reconfigure` 돌렸는데, 컴퓨트 노드 안의 `nova.conf`에는 반영이 안 된 것 같다.
- `global.conf`랑 `nova.conf`에 같은 옵션을 썼을 때 누가 이기는지 자신이 없다.
- External Ceph 붙이면서 `ceph config generate-minimal-conf`로 뽑은 `ceph.conf`를 그대로 넣었더니 배포가 깨진다.
- 그리고 가장 흔한 질문: **"Kolla-Ansible은 결국 어떤 컨테이너를, 왜 쓰는 건가?"**

이 글은 그런 질문들에 전부 답해 보려는 글입니다. 그런데 코드만 파기 전에, 이 프로젝트가 **누가 왜 만들었고 로고는 왜 코알라인지**, 그리고 2026년 지금 **어떤 버전을 골라 써야 하는지**부터 한 번 정리하고 갑니다. Kolla-Ansible을 써 본 분도, 이름만 들어본 분도 같이 읽을 수 있도록 쓰려고 노력했습니다.

**대상 독자**: OpenStack을 운영 중이거나 앞으로 올려 볼 분, 그리고 "Ansible이 도대체 무슨 파일을 가지고 `nova.conf`를 만드는지" 코드로 납득하고 싶은 분.

**이 글의 원칙**: 수치·버전·구조 설명은 전부 1차 출처(공식 governance, 공식 docs, opendev raw 파일)에서 직접 확인한 것만 씁니다. 확인되지 않은 주장은 "확인 필요"로 명시합니다.

---

## Kolla-Ansible의 탄생 배경 — 누가, 언제, 왜

먼저 공식 기록으로 확인되는 사실부터.

### 창시자와 시작 시점

Kolla 프로젝트는 **2014년 9월 Steven Dake**가 Launchpad에 등록하면서 시작됐습니다. OpenStack 릴리즈 사이클로는 **Kilo 사이클 직전**에 형태를 갖추기 시작했고, 이후 **Liberty 사이클에서 Big Tent**(공식 프로젝트로 편입)에 들어갑니다. 원래 목표는 이런 문장이었습니다.

> "Proof of concept to show that containers could be used as building blocks for deploying OpenStack — and was to become part of TripleO."
>
> *"컨테이너를 OpenStack 배포의 빌딩 블록으로 쓸 수 있는가"를 증명하는 PoC*
>
> — [OpenStack Wiki — Kolla](https://wiki.openstack.org/wiki/Kolla)

처음부터 "우리가 새 배포 도구를 만들자"로 출발한 게 아니라, **TripleO에 통합될 컨테이너 PoC**로 시작했다는 점이 재미있습니다. 그러다 점점 Kolla 단독 프로젝트로 성장하고, 결국 공식 TC(Technical Committee) 프로젝트가 됩니다.

### 공식 Mission Statement

현재 [OpenStack Technical Committee의 Kolla 프로젝트 페이지](https://governance.openstack.org/tc/reference/projects/kolla.html)에 적힌 공식 mission은 딱 한 줄입니다.

> "To provide tools to create production-ready containers and to provide deployment tools for operating OpenStack clouds."
>
> *프로덕션 품질의 컨테이너를 만드는 도구와, OpenStack 클라우드를 운영하기 위한 배포 도구를 제공한다.*

지금의 Kolla 프로젝트는 이 mission 아래에 **4개의 공식 deliverable**을 두고 있습니다.

1. **kolla** — 컨테이너 이미지 빌드 도구
2. **kolla-ansible** — Ansible 기반 배포 도구 (이 글의 주제)
3. **ansible-collection-kolla** — 공유 Ansible 리소스 컬렉션
4. **kayobe** (+ kayobe-config, kayobe-config-dev) — 베어메탈 컨트롤 플레인용 확장

**현 PTL**은 Michal Nasiadka입니다. (출처: 위 governance 페이지)

### Kolla와 Kolla-Ansible의 분리 — Newton 사이클

한 가지 헷갈리기 쉬운 점: 오늘날 `openstack/kolla`와 `openstack/kolla-ansible`은 별개 저장소지만, 원래는 한 저장소였습니다. `kolla-ansible` 저장소의 README가 직접 이렇게 말합니다.

> "Up to stable/newton, Kolla was a single project that lived in the git repository."
>
> — [kolla-ansible README.rst](https://opendev.org/openstack/kolla-ansible/raw/branch/master/README.rst)

**Newton 사이클(2016년)** 까지는 이미지 빌드와 Ansible 배포가 `openstack/kolla` 한 곳에 공존했습니다. Newton 이후 Ansible 관련 디렉터리만 `openstack/kolla-ansible`로 분리되었고, 오늘날의 역할 분담이 자리를 잡습니다.

- **Kolla** (`openstack/kolla`): 컨테이너 이미지 빌드 (`kolla-build` 명령)
- **Kolla-Ansible** (`openstack/kolla-ansible`): 빌드된 이미지를 Ansible playbook으로 실제 호스트에 배포

### 한 줄 요약

> 2014년 9월 Steven Dake가 "컨테이너를 OpenStack의 빌딩 블록으로 쓸 수 있을까?"라는 PoC 아이디어에서 출발 → Kilo/Liberty 사이클에서 성장 → Newton 시점에 이미지 빌드(kolla)와 배포 도구(kolla-ansible)가 별개 저장소로 분리 → 오늘날 TC 공식 프로젝트.

---

## 왜 마스코트가 코알라인가

이 글 상단에 박힌 귀여운 코알라 그림, 어디서 왔을까요? 생각보다 재미있는 이야기라 한 섹션 할애합니다.

### OpenStack Project Mascot 프로그램

2016년 7월, OpenStack Foundation의 Senior Marketing Manager였던 **Heidi Joy Tretheway**가 "각 프로젝트에 개성 있는 마스코트를 붙이자"는 프로그램을 시작합니다. Creative Director **Todd Morey**가 총괄을 맡고, **5명의 전문 일러스트레이터**가 붙어서 약 **50개 프로젝트 × 3개 변형 = 150개 일러스트**를 **2016년 Barcelona Summit** 전까지 만들어 냅니다. 그리고 2017년 2월 Atlanta PTG 직전에 거의 60개 팀에 로고가 배포됩니다.

선정 방식이 재미있는데, **각 프로젝트 팀이 직접 마스코트를 고르면** 일러스트레이터가 그걸 통일된 스타일의 카툰으로 그려 주는 방식이었습니다. 가이드라인은 "프로젝트의 특성을 반영하라"는 정도였고, 예를 들어 Keystone은 거북이인데 이유가 *"safe, smart"* 여서라고 합니다.

> 전체 히스토리는 Heidi Joy Tretheway가 직접 쓴 Superuser 아티클 [*"Creature or feature? Here's the story behind your favorite OpenStack project mascot"*](https://superuser.openinfra.org/articles/openstack-project-mascots/) 에 자세합니다.

### Kolla의 코알라 — 공식 선정 이유는 "두운"

그럼 Kolla 팀은 왜 코알라를 골랐을까요? 위 Superuser 아티클에 명시적으로 등장하는 공식 근거는 이렇습니다.

> *"Kolla's koala"* — 아티클에서 Kolla의 코알라는 **"두운(alliteration)으로 마스코트를 선택한 팀"** 의 사례로 직접 언급됩니다.

즉 공식적으로 기록된 이유는 **"Kolla"와 "Koala"의 발음 유사성(두운)** 입니다. 발음 놀이예요. Kolla → Koala, 얼마나 귀여운가요.

### 그리스어 κόλλα("glue") 어원설은?

커뮤니티에는 "Kolla는 그리스어 κόλλα, 즉 '아교·접착제'에서 왔고, OpenStack 컴포넌트들을 하나로 붙이는 도구라서 그렇게 이름 붙였다"는 설명이 돌아다닙니다. 이야기로는 그럴듯한데, **공식 기록에서는 확인이 되지 않습니다**.

- `openstack/kolla` README.rst, `wiki.openstack.org/wiki/Kolla`, governance project 페이지, 공식 docs 그 어디에도 어원 관련 서술 없음
- 따라서 Greek "glue" 어원은 **커뮤니티에서 회자되는 비공식적 해석**으로만 다룹니다

그러니 블로그 글이나 발표 자료에서 "Kolla는 그리스어로 접착제입니다"라고 단정짓는 건 조심하는 게 좋습니다. 공식 기록으로 확인되는 건 "두운"뿐이에요. (혹시 공식 기록을 찾으시면 알려 주세요. 이 글에 반영하겠습니다.)

---

## SLURP 릴리즈 정책 — 어떤 버전을 골라야 하나

본격적으로 Kolla-Ansible 구조로 들어가기 전에, "그래서 2026년 현재 나는 어떤 버전을 써야 하는가?"를 정리합니다. 여기서 SLURP라는 개념이 등장합니다.

### SLURP란 무엇인가 (공식 도입 결의)

**SLURP = Skip Level Upgrade Release Process**. OpenStack Technical Committee가 [2022년 2월 10일 결의문](https://governance.openstack.org/tc/resolutions/20220210-release-cadence-adjustment.html)으로 정식 도입한 정책입니다. 핵심 문장은 한 줄입니다.

> "Every other release will be considered to be a 'SLURP' release."
>
> *6개월 주기 릴리즈 중 **격월 교차**로 하나씩 SLURP로 지정한다.*

왜 이런 정책을 만들었냐 하면, **매번 6개월마다 업그레이드하기 부담스러운 운영자**를 위해 "SLURP 릴리즈 간에는 **스킵 레벨(12개월 주기) 업그레이드가 공식 지원됨**"을 보장하기 위해서입니다. 즉:

- 6개월마다 부지런히 올리는 팀 → 모든 릴리즈 계속 업그레이드 (기존대로)
- 1년마다 올리는 팀 → **SLURP → 다음 SLURP** 로 점프 (공식 지원)

첫 SLURP는 2023.1 **Antelope**부터 시작됐습니다.

### SLURP-only 매트릭스 (이 글이 권장하는 버전 뷰)

운영자 관점에서는 복잡한 전체 릴리즈 표 대신 **SLURP만 모아 놓고 보는 편이 훨씬 낫습니다**. 스킵 업그레이드 루트가 한눈에 들어오거든요.

| OpenStack SLURP | 코드명 | Release Date | Kolla-Ansible Major | 최신 버전 | 상태 (2026-04 기준) |
|---|---|---|---|---|---|
| **2023.1** | Antelope (첫 SLURP) | 2023-03-22 | 16.x | 16.7.0 | Unmaintained |
| **2024.1** | Caracal | 2024-04-03 | 18.x | 18.8.0 | Unmaintained |
| **2025.1** | Epoxy | 2025-04-02 | 20.x | 20.3.0 | **Maintained** |
| **2026.1** | Gazpacho | 2026-04-01 | 22.x (예상) | 확인 필요 | **Maintained (방금 릴리즈)** |

**출처**: [releases.openstack.org](https://releases.openstack.org/)의 각 릴리즈 index와 [`opendev.org/openstack/releases`](https://opendev.org/openstack/releases/src/branch/master/deliverables)의 deliverables YAML을 교차 확인했습니다.

{{< mermaid >}}
flowchart LR
    A["2023.1 Antelope<br/>Kolla-Ansible 16.x<br/>(첫 SLURP)"] -->|"skip upgrade"| B["2024.1 Caracal<br/>Kolla-Ansible 18.x"]
    B -->|"skip upgrade"| C["2025.1 Epoxy<br/>Kolla-Ansible 20.x"]
    C -->|"skip upgrade"| D["2026.1 Gazpacho<br/>Kolla-Ansible 22.x"]

    style C fill:#d9f0ff,stroke:#3b82f6,stroke-width:2px
    style D fill:#d9ffd9,stroke:#22c55e,stroke-width:2px
{{< /mermaid >}}

### 지금 고르라면?

- **새로 배포를 시작**: **2025.1 Epoxy (Kolla-Ansible 20.x)** 가 현재 가장 안정적인 선택입니다. Maintained 상태이고, 다음 SLURP인 Gazpacho(22.x)로 스킵 업그레이드 루트가 열려 있습니다.
- **이미 Antelope/Caracal 중**: 업스트림이 Unmaintained(과거의 Extended Maintenance)로 내려갔으므로, **Caracal → Epoxy** 또는 **Antelope → Caracal → Epoxy** 로 SLURP 루트를 타는 것이 권장됩니다.
- **최신을 따라가고 싶다**: 2026.1 Gazpacho가 방금 릴리즈되었습니다. 다만 Kolla-Ansible major 버전의 첫 패치는 안정화에 몇 주가 필요하다는 점을 염두에 두세요. (2026-04-10 시점에서 Gazpacho deliverables YAML 내용을 직접 확인하지 못해 *"22.x 예상/확인 필요"* 로 표기합니다.)

**한 가지 짚고 갈 것**: Kolla-Ansible 자체에는 SLURP 전용 LTS 브랜치가 따로 없습니다. Kolla 팀은 **모든 OpenStack 릴리즈마다** `stable/YYYY.N` 브랜치를 만들고 major 버전을 올리며, SLURP 지정은 OpenStack upstream의 것을 그대로 따릅니다. ([출처: 각 릴리즈 deliverables YAML](https://opendev.org/openstack/releases/src/branch/master/deliverables))

---

## Kolla-Ansible이 배포하는 컨테이너 이미지

"Kolla-Ansible은 결국 어떤 컨테이너를 쓰는가?" — 가장 자주 받는 질문 중 하나입니다. 정답은 **공식 기본값**을 그대로 확인하는 것이 가장 확실합니다.

### 기본 레지스트리와 네임스페이스 (코드 기반 확정값)

Kolla-Ansible의 모든 이미지 경로 기본값은 [`ansible/group_vars/all/common.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/group_vars/all/common.yml) 에 정의되어 있습니다. master 브랜치(2026-04 기준)에서 실제로 추출한 값은 이렇습니다.

```yaml
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/group_vars/all/common.yml

docker_registry: "quay.io"
docker_namespace: "openstack.kolla"
docker_image_name_prefix: ""
docker_image_url: "{{ docker_registry ~ '/' if docker_registry else '' }}{{ docker_namespace }}/{{ docker_image_name_prefix }}"

kolla_base_distro: "rocky"
kolla_container_engine: "docker"

openstack_release: "master"
openstack_tag: "{{ openstack_release }}-{{ kolla_base_distro }}-{{ kolla_base_distro_version }}{{ openstack_tag_suffix }}"
```

여기서 뽑히는 결론이 네 개 있는데, 이 중 두세 개는 **예전 블로그 글과 다를 수 있어서 주의**가 필요합니다.

1. **기본 레지스트리는 `quay.io`** — 예전 `hub.docker.com`(Docker Hub) 기반 안내 문서는 더 이상 기본값과 맞지 않습니다.
2. **기본 네임스페이스는 `openstack.kolla`** — `kolla`가 아닙니다. 예전 이미지 태그 형식(`kolla/ubuntu-source-nova-api:zed` 같은)은 네임스페이스가 마이그레이션되기 전 기준이라 현재 기본값과 맞지 않습니다.
3. **기본 베이스 디스트로는 `rocky`** — Rocky Linux 기반 이미지가 기본입니다.
4. **기본 컨테이너 엔진은 `docker`** — Podman도 지원되지만(뒤에서 다룸), 기본값은 여전히 Docker입니다.

### 이미지 이름 규칙 — 템플릿과 실제 예시

위 변수들을 합치면 이미지 이름이 조립됩니다. 규칙은 간단합니다.

```text
{docker_registry}/{docker_namespace}/{docker_image_name_prefix}<service>:{openstack_tag}

여기서 openstack_tag = {openstack_release}-{kolla_base_distro}-{kolla_base_distro_version}[-{openstack_tag_suffix}]
```

**Epoxy(2025.1) + Rocky Linux 9** 기준으로 실제 예시를 뽑으면 이렇게 됩니다.

```text
# 출처: ansible/group_vars/all/common.yml + Kolla-Ansible Quickstart
# https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html

# OpenStack 코어 서비스
quay.io/openstack.kolla/keystone:2025.1-rocky-9
quay.io/openstack.kolla/nova-api:2025.1-rocky-9
quay.io/openstack.kolla/nova-compute:2025.1-rocky-9
quay.io/openstack.kolla/neutron-server:2025.1-rocky-9
quay.io/openstack.kolla/ovn-controller:2025.1-rocky-9
quay.io/openstack.kolla/glance-api:2025.1-rocky-9
quay.io/openstack.kolla/cinder-volume:2025.1-rocky-9
quay.io/openstack.kolla/placement-api:2025.1-rocky-9

# 인프라 서비스
quay.io/openstack.kolla/mariadb-server:2025.1-rocky-9
quay.io/openstack.kolla/rabbitmq:2025.1-rocky-9
quay.io/openstack.kolla/haproxy:2025.1-rocky-9
quay.io/openstack.kolla/keepalived:2025.1-rocky-9

# Ubuntu 기반 변형
quay.io/openstack.kolla/nova-api:2025.1-ubuntu-noble

# ARM64 변형 (openstack_tag_suffix: "-aarch64")
quay.io/openstack.kolla/nova-api:2025.1-rocky-9-aarch64
```

> **주의**: 위 태그들은 템플릿 규칙에서 도출된 예시입니다. 특정 태그 문자열이 실제로 `quay.io/openstack.kolla`에 푸시되어 있는지는 배포 시점에 레지스트리에서 직접 확인하시는 게 좋습니다. 네임스페이스와 태그 형식 자체는 [Kolla-Ansible Quickstart 공식 문서](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html) 에 동일하게 명시되어 있습니다.

### `kolla_base_distro` 허용 값 (master 기준)

[공식 Support Matrix](https://docs.openstack.org/kolla-ansible/latest/user/support-matrix.html) 에 따르면 현재 master에서 허용되는 베이스 디스트로는 네 가지입니다.

| 값 | 베이스 OS |
|---|---|
| `centos` | CentOS Stream 10 |
| `debian` | Debian Trixie (13) |
| `rocky` | Rocky Linux 10 (master 기본) |
| `ubuntu` | Ubuntu Noble (24.04) |

> 공식 권장: *"For newcomers, we recommend to use Rocky Linux 10 or Ubuntu 24.04."*

### 몇 개의 서비스를 컨테이너로 다루는가

Kolla-Ansible이 "컨테이너로 배포할 수 있는 서비스"의 전체 커버리지는 `ansible/group_vars/all/` 디렉터리의 per-service variable 파일을 세어 보면 가늠할 수 있습니다. master 기준으로 대략 다음과 같습니다.

- **OpenStack 서비스 (약 26개)**: aodh, barbican, bifrost, blazar, ceilometer, cinder, cloudkitty, cyborg, designate, glance, gnocchi, heat, horizon, ironic, keystone, kuryr, magnum, manila, masakari, mistral, nova, octavia, placement, skyline, tacker, trove, watcher, zun
- **인프라 (약 6개)**: database, mariadb, proxysql, etcd, letsencrypt, common
- **스토리지·캐시 (약 6개)**: ceph, ceph-rgw, iscsi, memcached, valkey, s3
- **네트워킹·LB (약 7개)**: haproxy, hacluster, keepalived, loadbalancer, neutron, openvswitch, ovn
- **모니터링·로깅 (약 5개)**: collectd, fluentd, grafana, prometheus, opensearch
- **기타**: multipathd, rabbitmq

**총합 ≈ 52개 per-service 설정 파일**. 실제로 배포되는 건 `enable_*` 플래그로 켠 것들이라 보통은 이보다 적지만, "Kolla-Ansible이 컨테이너로 다룰 수 있는 서비스의 범위"는 이 정도라고 보면 됩니다.

> **최근 변경**: 2026-04-03 master에 `valkey.yml`이 추가됐습니다. Redis에서 Valkey로 마이그레이션이 진행 중입니다. 출처: [ansible/group_vars/all (master)](https://opendev.org/openstack/kolla-ansible/src/branch/master/ansible/group_vars/all)

---

## 왜 컨테이너로 배포하는가

정직하게 먼저 말해 두면, 공식 문서에서 "Kolla가 컨테이너를 쓰는 이유"를 명시적으로 서술한 대목은 의외로 짧습니다. 이 섹션은 "공식 근거로 확인된 것"과 "일반적으로 알려진 컨테이너 배포의 이점"을 분명히 분리해서 쓰겠습니다.

### 공식 근거 (인용 가능한 것)

**Kolla 공식 mission statement** ([governance.openstack.org/tc/reference/projects/kolla.html](https://governance.openstack.org/tc/reference/projects/kolla.html)):

> "To provide tools to create production-ready containers and to provide deployment tools for operating OpenStack clouds."

**Kolla-Ansible Deployment Philosophy** ([docs.openstack.org/kolla-ansible/latest/admin/deployment-philosophy.html](https://docs.openstack.org/kolla-ansible/latest/admin/deployment-philosophy.html)):

> "Replace the inflexible, painful, resource-intensive deployment process of OpenStack with a flexible, painless, inexpensive deployment process."
>
> *경직되고, 고통스럽고, 자원 집약적인 OpenStack 배포 과정을 유연하고, 덜 고통스럽고, 저렴한 과정으로 바꾼다.*

공식 문서가 명시적으로 말하는 건 여기까지입니다. "OS 패키지 충돌을 피하려고"라든가 "롤링 업그레이드를 위해서"라든가 하는 구체적 이유는 공식 문장으로는 확인되지 않습니다.

### 일반적으로 알려진 컨테이너 배포의 이점 (공식 근거 아님)

아래는 OpenStack 운영자 커뮤니티에서 널리 공유되는 이유들입니다. 공식 문서 근거는 아니라는 점을 분명히 하고 나열합니다.

- **OS 패키지 의존성 충돌 회피**: OpenStack 서비스마다 요구하는 파이썬 라이브러리 버전이 미묘하게 다를 때, 컨테이너 단위 격리가 이 충돌을 흡수합니다.
- **롤링 업그레이드 용이성**: 이미지 태그만 바꿔서 컨테이너 재시작으로 업그레이드를 수행할 수 있습니다.
- **재현성**: "이 버전의 `nova-api`를 올린다"가 이미지 태그 한 줄로 기술됩니다.
- **노드 간 환경 균질성**: 호스트 OS의 미세한 차이(패치 레벨, 로케일, 라이브러리)로 인한 "특정 노드에서만 실패" 이슈가 줄어듭니다.

이 네 가지 중 어느 하나라도 "Kolla 공식 문서가 이래서 컨테이너를 쓴다고 말한다"고 인용하면 부정확합니다. 운영자 관점에서 체감되는 장점과 공식 근거는 분리해서 생각하는 편이 좋습니다.

### Docker vs Podman

`kolla_container_engine` 변수로 Docker와 Podman 중 선택할 수 있습니다. 기본값은 `docker`이고, **Podman 지원은 Kolla-Ansible 17.0.0 (2023.2 Bobcat)** 에서 정식 도입됐습니다. 릴리즈 노트 원문:

> "Implements support for Podman deployment as an alternative to Docker. To perform deployment using Podman, set the variable `kolla_container_engine` to value `podman` inside of the `globals.yml` file."
>
> — [kolla-ansible 2023.2 Bobcat release notes](https://docs.openstack.org/releasenotes/kolla-ansible/2023.2.html)

같은 릴리즈에서 `kolla_docker` Ansible 모듈이 `kolla_container`로 이름도 바뀌었습니다. 즉 Podman 지원은 단순히 드라이버 추가가 아니라 내부 모듈까지 일관되게 리네임할 정도로 정식 퍼스트클래스 백엔드입니다.

---

## 동작 원리 한눈에 보기

이제 "배포할 때 도대체 무슨 일이 벌어지는가"를 그림 두 개로 요약합니다. 뒤의 코드 분석 섹션을 읽기 전에 전체 지도를 한 번 보고 가시라고 넣었습니다.

### 전체 배포 플로우

{{< mermaid >}}
flowchart TB
    subgraph ops ["배포 호스트 (deployer)"]
        cli["kolla-ansible deploy<br/>(console_scripts)"]
        stev["stevedore가<br/>Deploy 클래스 로드"]
        site["ansible/site.yml<br/>(1051 lines)"]
        groupby["group_by로<br/>disabled 서비스 제외"]
        roles["서비스별 role<br/>import_playbook"]
        merge["merge_configs<br/>액션 플러그인"]
        cfgdir["/etc/kolla/config/<br/>오버라이드 파일"]
        template["role 내장 Jinja2 템플릿<br/>nova.conf.j2 등"]
        cli --> stev --> site --> groupby --> roles --> merge
        template --> merge
        cfgdir --> merge
    end

    subgraph target ["타겟 호스트 (controller / compute / ...)"]
        pull["이미지 pull<br/>quay.io/openstack.kolla/*"]
        run["컨테이너 실행<br/>(docker / podman)"]
        confvol["/etc/kolla/&lt;service&gt;/<br/>최종 config 파일"]
    end

    merge -->|"merged nova.conf<br/>을 copy 액션으로 전송"| confvol
    confvol --> run
    pull --> run

    style merge fill:#fff3c4,stroke:#eab308,stroke-width:2px
    style cli fill:#d9f0ff,stroke:#3b82f6,stroke-width:2px
{{< /mermaid >}}

그림의 노란 박스(`merge_configs`)가 이 글의 본론입니다. **여러 소스의 INI 파일을 섹션·키 단위로 병합**해서 타겟에 뿌리는 바로 그 액션 플러그인입니다.

### `merge_configs` 내부의 단계별 흐름

{{< mermaid >}}
flowchart LR
    src1["role 내장 템플릿<br/>nova.conf.j2"]
    src2["global.conf"]
    src3["nova.conf"]
    src4["nova/&lt;file&gt;.conf"]
    src5["nova/&lt;hostname&gt;/nova.conf"]

    jinja["Jinja2 렌더링<br/>(read_config)"]
    parser["OverrideConfigParser<br/>parse() 호출"]
    sections["_sections<br/>(누적 병합 결과)"]
    output["임시 파일 작성"]
    copyact["내장 copy 액션<br/>재귀 호출"]
    dest["타겟 호스트<br/>/etc/kolla/nova-api/nova.conf"]

    src1 --> jinja
    src2 --> jinja
    src3 --> jinja
    src4 --> jinja
    src5 --> jinja
    jinja --> parser --> sections --> output --> copyact --> dest

    style sections fill:#fff3c4,stroke:#eab308,stroke-width:2px
{{< /mermaid >}}

이 두 그림이 머릿속에 있으면, 아래 코드 분석이 "지도 위의 한 지점을 확대해서 보는" 느낌으로 읽힙니다.

---

## 저장소 구조 한눈에 보기

[`opendev.org/openstack/kolla-ansible`](https://opendev.org/openstack/kolla-ansible) 저장소를 체크아웃하면 최상위가 이렇게 생겼습니다.

```text
kolla-ansible/
├── ansible/              # Ansible playbook · role · action plugin
│   ├── action_plugins/   # merge_configs.py 등 커스텀 액션
│   ├── group_vars/all/   # common.yml — node_config_directory 등 전역 변수
│   ├── roles/            # 52+ 서비스별 role (nova, neutron, ceph-rgw, ...)
│   └── site.yml          # 전체 배포 진입점 playbook (1051 lines)
├── etc/kolla/            # 운영자가 손대는 설정의 레퍼런스 템플릿
│   ├── globals.yml       # 가장 중요한 운영자 설정 파일 템플릿
│   └── passwords.yml     # kolla-genpwd로 채우는 시크릿
├── kolla_ansible/        # Python 패키지 (CLI 진입점 코드)
│   ├── cli/commands.py   # stevedore command 클래스들
│   └── cmd/              # kolla-ansible, kolla-genpwd 등 console_scripts 엔트리
├── releasenotes/         # reno 기반 릴리즈 노트
├── setup.cfg             # Python 패키지 메타 + entry_points
└── tests/
```

### Role의 표준 구조

각 role은 [`nova` 디렉터리](https://opendev.org/openstack/kolla-ansible/src/branch/master/ansible/roles/nova)를 예로 들면 이렇게 생겼습니다.

```text
ansible/roles/nova/
├── defaults/   # 기본 변수
├── handlers/   # restart 핸들러
├── tasks/
│   ├── bootstrap.yml
│   ├── config.yml                 # ★ merge_configs 호출 위치
│   ├── deploy.yml                 # kolla-ansible deploy 진입점
│   ├── reconfigure.yml            # kolla-ansible reconfigure 진입점
│   ├── precheck.yml               # kolla-ansible prechecks 진입점
│   ├── upgrade.yml
│   └── ...
├── templates/  # nova.conf.j2 등 Jinja2 내장 템플릿
└── vars/       # role 고정값
```

`kolla-ansible <subcommand>` 각각이 role 안의 `tasks/<subcommand>.yml`에 의미적으로 대응됩니다. 초보자가 자주 놓치는 포인트 하나: **Nova는 `nova`와 `nova-cell` 두 role로 쪼개져 있습니다.** `nova` role이 컨트롤 플레인(api/conductor/scheduler)을, `nova-cell` role이 컴퓨트 셀(libvirt/compute)을 담당합니다. "CPU pinning 설정을 어디에 넣어야 하지?" 같은 질문에서 가끔 `nova-cell/tasks/...`를 봐야 할 때가 있는 이유입니다.

---

## CLI 진입점 — `kolla-ansible deploy`는 무슨 일을 하는가

운영자가 입력하는 `kolla-ansible` 명령은 Python 콘솔 스크립트입니다. 시작점은 [`setup.cfg` 42~75번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/setup.cfg)에 선언된 entry_points입니다.

```ini
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/setup.cfg
# setup.cfg 42~75번 줄 (발췌)

console_scripts =
    kolla-genpwd = kolla_ansible.cmd.genpwd:main
    kolla-mergepwd = kolla_ansible.cmd.mergepwd:main
    kolla-writepwd = kolla_ansible.cmd.writepwd:main
    kolla-readpwd = kolla_ansible.cmd.readpwd:main
    kolla-ansible = kolla_ansible.cmd.kolla_ansible:main

kolla_ansible.cli =
    gather-facts = kolla_ansible.cli.commands:GatherFacts
    install-deps = kolla_ansible.cli.commands:InstallDeps
    prechecks = kolla_ansible.cli.commands:Prechecks
    genconfig = kolla_ansible.cli.commands:GenConfig
    reconfigure = kolla_ansible.cli.commands:Reconfigure
    validate-config = kolla_ansible.cli.commands:ValidateConfig
    bootstrap-servers = kolla_ansible.cli.commands:BootstrapServers
    pull = kolla_ansible.cli.commands:Pull
    certificates = kolla_ansible.cli.commands:Certificates
    deploy = kolla_ansible.cli.commands:Deploy
    deploy-containers = kolla_ansible.cli.commands:DeployContainers
    post-deploy = kolla_ansible.cli.commands:Postdeploy
    upgrade = kolla_ansible.cli.commands:Upgrade
    stop = kolla_ansible.cli.commands:Stop
    destroy = kolla_ansible.cli.commands:Destroy
    migrate-container-engine = kolla_ansible.cli.commands:MigrateContainerEngine
```

진입점이 두 층입니다.

1. `console_scripts`의 `kolla-ansible = kolla_ansible.cmd.kolla_ansible:main` — 쉘에서 `kolla-ansible`을 치면 실행되는 Python 함수.
2. `kolla_ansible.cli` 블록 — stevedore가 동적으로 로드하는 서브커맨드 클래스들. `deploy`, `prechecks`, `reconfigure`, `bootstrap-servers`, `post-deploy`, `upgrade` 등이 모두 `kolla_ansible/cli/commands.py` 내 클래스로 매핑됩니다.

즉 **`kolla-ansible deploy`를 입력하면 stevedore가 `Deploy` 클래스를 로드하고, 그 클래스가 site.yml 기반 playbook을 실행**합니다. 그리고 [`ansible/site.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/site.yml)(1051 lines)의 첫 부분은 이렇게 시작합니다.

```yaml
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/site.yml
# site.yml 1~10번 줄
---
- name: Gather facts
  import_playbook: gather-facts.yml

# NOTE(mgoddard): In large environments, even tasks that are skipped can take a
# significant amount of time. This is an optimisation to prevent any tasks
# running in the subsequent plays for services that are disabled.
- name: Group hosts based on configuration
  hosts: all
  gather_facts: false
```

흐름을 말로 풀면:

1. 모든 호스트에서 facts 수집.
2. `kolla_action`과 `enable_*` 변수 기준으로 `group_by`를 수행해서 **disabled 서비스 play는 호스트 그룹 자체에서 제외**. 대규모 환경에서는 skipped task 비용조차 무시 못 한다는 점이 주석에 박혀 있습니다.
3. 서비스별(`aodh`, `barbican`, `cinder`, `glance`, `nova`, `neutron`, ...) play를 `import_playbook`으로 순서대로 실행.

실전 배포 플로우는 공식 문서 [Operating Kolla](https://docs.openstack.org/kolla-ansible/latest/user/operating-kolla.html)와 위 entry_points를 합쳐 보면 이렇게 정리됩니다.

```bash
# 출처: https://docs.openstack.org/kolla-ansible/latest/user/operating-kolla.html
# + setup.cfg의 kolla_ansible.cli entry_points
[deployer@ops ~]$ kolla-ansible bootstrap-servers -i /etc/kolla/multinode
[deployer@ops ~]$ kolla-ansible prechecks         -i /etc/kolla/multinode
[deployer@ops ~]$ kolla-ansible deploy            -i /etc/kolla/multinode
[deployer@ops ~]$ kolla-ansible post-deploy       -i /etc/kolla/multinode
# 설정 수정 후 재배포
[deployer@ops ~]$ kolla-ansible reconfigure       -i /etc/kolla/multinode
```

---

## 핵심: `merge_configs` 액션 플러그인 분석

여기서부터가 이 글의 본론입니다. Kolla-Ansible의 커스텀 config 오버라이드 동작을 실제로 구현하는 것은 [`ansible/action_plugins/merge_configs.py`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/action_plugins/merge_configs.py) 한 파일입니다. 2026-04-10 시점 master 기준 **234 lines**.

### 왜 `template` 모듈이 아니라 커스텀 action plugin인가

OpenStack의 설정 파일은 거의 예외 없이 INI 포맷이고, Nova만 해도 `[DEFAULT]`, `[api]`, `[libvirt]`, `[neutron]`, `[cinder]` 같은 수많은 섹션을 가집니다. 여기에 요구가 하나 붙습니다: **여러 소스에서 온 값을 섹션·키 단위로 병합**해야 한다.

단순히 뒤 파일이 앞 파일을 통째로 교체해 버리면 안 되고, `[DEFAULT]`는 그대로 두되 `[libvirt] cpu_mode`만 덮어쓰는 식의 세밀한 병합이 필요합니다. Ansible 내장 `template` 모듈은 단일 Jinja2 템플릿을 렌더링해서 그대로 전송하는 1:1 구조라 이 요구를 만족하지 못합니다. 그래서 Kolla-Ansible 팀은 `oslo_config.iniparser.BaseParser`를 상속한 커스텀 파서를 action plugin으로 작성한 것입니다.

### `OverrideConfigParser` — 섹션·키 단위 병합

핵심 클래스는 `OverrideConfigParser`입니다.

```python
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/action_plugins/merge_configs.py
# ansible/action_plugins/merge_configs.py 83~112번 줄
class OverrideConfigParser(iniparser.BaseParser):

    def __init__(self, whitespace=True):
        self._cur_sections = collections.OrderedDict()
        self._sections = collections.OrderedDict()
        self._cur_section = None
        self._whitespace = ' ' if whitespace else ''

    def assignment(self, key, value):
        if self._cur_section is None:
            self.new_section(_ORPHAN_SECTION)
        cur_value = self._cur_section.get(key)
        if len(value) == 1 and value[0] == '':
            value = []
        if not cur_value:
            self._cur_section[key] = [value]
        else:
            self._cur_section[key].append(value)

    def parse(self, lineiter):
        self._cur_sections = collections.OrderedDict()
        self._cur_section = None
        super(OverrideConfigParser, self).parse(lineiter)

        # merge _cur_sections into _sections
        for section, values in self._cur_sections.items():
            if section not in self._sections:
                self._sections[section] = collections.OrderedDict()
            for key, value in values.items():
                self._sections[section][key] = value
```

포인트 세 가지입니다.

1. **상태 두 벌 유지**: `_cur_sections`는 "지금 파싱 중인 한 파일"의 결과, `_sections`는 "여러 소스를 거치며 누적된" 결과입니다.
2. **`parse()` 호출마다 `_cur_sections`가 초기화**되지만 `_sections`는 유지됩니다. 그래서 같은 파서 인스턴스에 여러 파일을 순서대로 먹이면 자연스럽게 누적 병합이 됩니다.
3. **병합 시 단순 대입**: `self._sections[section][key] = value`. 동일 섹션·동일 키가 이미 있으면 **그냥 덮어씁니다**. "나중에 parse된 값이 이긴다"는 precedence가 이 한 줄에서 나옵니다.

추가로 `collections.OrderedDict`를 쓰는 이유는 **직렬화 시 섹션·키 순서를 보장**하기 위함입니다. 결과물로 나오는 `nova.conf`의 섹션 순서가 원본 템플릿과 일치해서 diff 검사가 수월해집니다.

### Jinja2 템플릿 렌더링 — source 파일에도 변수가 먹는 이유

각 소스 파일은 파싱 전에 Jinja2로 먼저 렌더링됩니다. 이 부분이 `read_config()` 메서드입니다.

```python
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/action_plugins/merge_configs.py
# ansible/action_plugins/merge_configs.py 160~177번 줄
def read_config(self, source, config):
    # Only use config if present
    if os.access(source, os.R_OK):
        with open(source, 'r') as f:
            template_data = trust_as_template(f.read())

        # set search path to mimic 'template' module behavior
        searchpath = [
            self._loader._basedir,
            os.path.join(self._loader._basedir, 'templates'),
            os.path.dirname(source),
        ]
        self._templar.environment.loader.searchpath = searchpath

        result = self._templar.template(template_data)
        fakefile = StringIO(result)
        config.parse(fakefile)
        fakefile.close()
```

여기서 실무적으로 중요한 사실 세 가지.

1. **`os.access(source, os.R_OK)` 체크** — 존재하지 않거나 읽을 수 없는 파일은 **조용히 무시됩니다**. 에러도, 경고도 없습니다. "내 오버라이드가 먹는 것 같지 않다"의 70%가 이 체크 때문에 발생합니다. (오타, hostname 불일치 등)
2. **`self._templar.template(template_data)`** — 렌더링은 일반 Ansible templar가 수행합니다. 따라서 운영자가 `/etc/kolla/config/nova.conf`에 `{{ inventory_hostname }}`, `{{ kolla_internal_vip_address }}`, `{{ enable_ceph | bool }}` 같은 Ansible 변수를 자유롭게 쓸 수 있습니다. 이건 "custom config"이지만 사실상 Jinja2 템플릿입니다.
3. **`trust_as_template` shim** — Ansible 12+의 템플릿 신뢰 모델 변경에 대비한 하위 호환 wrapper입니다. [merge_configs.py 23~31번 줄 주석](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/action_plugins/merge_configs.py)에 `"This can be removed in the G cycle."`라고 명시되어 있습니다.

### 최종 출력 — `copy` 액션에 위임

병합이 끝난 결과는 어떻게 원격 호스트로 갈까요? 재미있게도 merge_configs는 SSH 전송을 **직접 하지 않습니다**. 로컬 임시 파일에 결과를 쓰고, Ansible 내장 `copy` 액션을 재귀 호출합니다.

```python
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/action_plugins/merge_configs.py
# ansible/action_plugins/merge_configs.py 202~231번 줄
local_tempdir = tempfile.mkdtemp(dir=constants.DEFAULT_LOCAL_TMP)

try:
    result_file = os.path.join(local_tempdir, 'source')
    with open(result_file, 'w') as f:
        f.write(full_source)

    new_task = self._task.copy()
    new_task.args.pop('sources', None)
    new_task.args.pop('whitespace', None)

    new_task.args.update(
        dict(
            src=result_file
        )
    )

    copy_action = self._shared_loader_obj.action_loader.get(
        'copy',
        task=new_task,
        connection=self._connection,
        play_context=self._play_context,
        loader=self._loader,
        templar=self._templar,
        shared_loader_obj=self._shared_loader_obj)
    copy_result = copy_action.run(task_vars=task_vars)
```

이건 Ansible action plugin 개발의 **composition 패턴 교과서 예제**입니다. 새 task를 복제해 `sources`·`whitespace` 같은 merge_configs 전용 인자를 제거하고, 그 자리에 임시 파일 경로(`src=result_file`)를 꽂은 다음, `action_loader.get('copy', ...)`로 내장 copy 액션을 로드해 호출합니다.

왜 이렇게 만들었을까요? 답은 단순합니다. **`mode`, `become`, `owner`, `group`, 권한 비트, 파일 해시 비교 같은 기능을 재구현하고 싶지 않아서**입니다. 내장 copy 액션은 이미 그 모든 걸 신뢰성 있게 해주고 있고, action plugin 레이어에서 재귀 호출하면 공짜로 얻을 수 있습니다. 그래서 role의 `config.yml`에 `mode: "0660"`, `become: true`를 써 두면 그 값들이 그대로 먹습니다.

---

## 병합 순서 — `nova.conf`가 만들어지는 정확한 과정

이제 실제 role 코드가 어떻게 `merge_configs`를 호출하는지 봅시다. Nova의 경우 [`ansible/roles/nova/tasks/config.yml` 54~70번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/nova/tasks/config.yml)이 핵심입니다.

```yaml
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/nova/tasks/config.yml
# ansible/roles/nova/tasks/config.yml 54~70번 줄
- name: Copying over nova.conf
  become: true
  vars:
    service_name: "{{ item.key }}"
  merge_configs:
    sources:
      - "{{ role_path }}/templates/nova.conf.j2"
      - "{{ node_custom_config }}/global.conf"
      - "{{ node_custom_config }}/nova.conf"
      - "{{ node_custom_config }}/nova/{{ item.key }}.conf"
      - "{{ node_custom_config }}/nova/{{ inventory_hostname }}/nova.conf"
    dest: "{{ node_config_directory }}/{{ item.key }}/nova.conf"
    mode: "0660"
  with_dict: "{{ nova_services | select_services_enabled_and_mapped_to_host }}"
```

`with_dict`로 반복하는 `nova_services`는 `nova-api`, `nova-scheduler`, `nova-conductor`, `nova-super-conductor` 같은 nova 컨트롤 플레인 서비스 딕셔너리입니다. 각 서비스마다 `sources` 리스트가 순서대로 병합되어 `dest`에 복사됩니다.

`sources` 리스트의 순서는 **precedence의 낮음 → 높음 순**입니다. merge_configs가 "뒤에 parse된 값이 이긴다"는 걸 떠올리면서 다음 그림을 보세요.

{{< mermaid >}}
flowchart TD
    s1["① role 내장 템플릿<br/>nova.conf.j2<br/>(가장 낮은 우선순위)"]
    s2["② /etc/kolla/config/global.conf<br/>모든 서비스 공통 오버라이드"]
    s3["③ /etc/kolla/config/nova.conf<br/>Nova 전체 오버라이드"]
    s4["④ /etc/kolla/config/nova/&lt;service&gt;.conf<br/>컨테이너별 오버라이드<br/>(nova-scheduler.conf 등)"]
    s5["⑤ /etc/kolla/config/nova/&lt;hostname&gt;/nova.conf<br/>호스트별 오버라이드<br/>(가장 높은 우선순위)"]

    s1 -->|"병합 →"| s2
    s2 -->|"병합 →"| s3
    s3 -->|"병합 →"| s4
    s4 -->|"병합 →"| s5
    s5 --> out["최종 nova.conf<br/>/etc/kolla/&lt;service&gt;/nova.conf"]

    style s5 fill:#ffe4e4,stroke:#ef4444,stroke-width:2px
    style s1 fill:#e4f1ff,stroke:#3b82f6,stroke-width:1px
    style out fill:#e4ffe4,stroke:#22c55e,stroke-width:2px
{{< /mermaid >}}

### 다른 role도 같은 패턴인가?

네. 교차 검증으로 [`ansible/roles/glance/tasks/config.yml` 70~77번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/glance/tasks/config.yml)을 보면 동일한 5단계 precedence를 씁니다.

```yaml
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/glance/tasks/config.yml
  merge_configs:
    sources:
      - "{{ role_path }}/templates/glance-api.conf.j2"
      - "{{ node_custom_config }}/global.conf"
      - "{{ node_custom_config }}/glance.conf"
      - "{{ node_custom_config }}/glance/glance-api.conf"
      - "{{ node_custom_config }}/glance/{{ inventory_hostname }}/glance-api.conf"
    dest: "{{ node_config_directory }}/glance-api/glance-api.conf"
    mode: "0660"
```

즉 **"role 내장 템플릿 → `global.conf` → `<service>.conf` → `<service>/<file>.conf` → `<service>/<hostname>/<file>.conf`"** 이 다섯 단계 precedence는 Kolla-Ansible 전체의 관용 규약입니다. 흥미로운 점: 이 규약이 어떤 변수나 매크로로 추상화되지 않고 각 role의 config.yml에 **그대로 복사**되어 있다는 것. 새 role을 추가할 때도 기존 role의 5줄을 그대로 따라 붙이면 오버라이드 동작이 자동으로 따라옵니다.

### `node_custom_config`와 `node_config_directory`

위 코드의 `{{ node_custom_config }}`와 `{{ node_config_directory }}`의 정체는 [`ansible/group_vars/all/common.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/group_vars/all/common.yml) 163·166번 줄에 있습니다.

```yaml
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/group_vars/all/common.yml
# 163, 166번 줄
node_custom_config: "{{ node_config }}/config"
node_config_directory: "/etc/kolla"
```

`node_config`는 배포 호스트 측 kolla 설정 루트이므로 실질적으로 `node_custom_config = /etc/kolla/config`가 됩니다. 공식 문서 [Advanced Configuration](https://docs.openstack.org/kolla-ansible/latest/admin/advanced-configuration.html)에도 이 경로가 그대로 나옵니다.

### 실전 예제: compute01에서만 CPU pinning 켜기

퍼즐이 맞춰졌으니 "compute01 한 노드에서만 `host-passthrough`를 쓰고 싶다" 같은 요구가 파일 한 개로 해결됩니다.

```ini
# /etc/kolla/config/nova/compute01.example.com/nova.conf
# 이 파일은 오직 inventory_hostname == "compute01.example.com"인 노드의
# nova-compute 컨테이너에만 적용된다 (precedence 5단계, 가장 높음)

[libvirt]
cpu_mode = host-passthrough
```

**`inventory_hostname`이 FQDN인지 short name인지가 핵심**입니다. Kolla-Ansible은 디렉터리명을 문자열 그대로 비교하므로, 인벤토리 파일에 `compute01.example.com`으로 써놓고 디렉터리는 `compute01`로 만들면 **조용히 스킵**됩니다. (함정 섹션에서 다시 다룹니다)

### `global.conf`와 `nova.conf`에 같은 키가 있으면?

답은 명확합니다. 동일 섹션·동일 키라면 뒤에 오는 `nova.conf`가 이깁니다. **서로 다른 키**면 둘 다 살아남습니다. 즉 `global.conf`를 "전역 공통 설정", `nova.conf`를 "nova 전용 설정"으로 역할 분리해서 두면 아무 문제 없이 공존합니다.

---

## External Ceph 연동 — `ceph.conf`와 keyring은 어디에 두는가

Nova/Cinder/Glance를 External Ceph(Kolla 외부의 Ceph 클러스터)에 붙이는 경우, `ceph.conf`와 cephx keyring 파일을 배포 호스트에 두면 Kolla-Ansible이 이를 각 컨테이너로 실어 나릅니다. 공식 문서 [External Ceph Guide](https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html)에 규약이 정리되어 있습니다.

### 파일 배치 규약

| 서비스 | `ceph.conf` 경로 | keyring 경로 |
|---|---|---|
| Glance | `/etc/kolla/config/glance/ceph.conf` | `/etc/kolla/config/glance/ceph.client.glance.keyring` |
| Cinder Volume | `/etc/kolla/config/cinder/ceph.conf` | `/etc/kolla/config/cinder/cinder-volume/ceph.client.cinder.keyring` |
| Cinder Backup | `/etc/kolla/config/cinder/ceph.conf` | `/etc/kolla/config/cinder/cinder-backup/ceph.client.cinder.keyring` + `ceph.client.cinder-backup.keyring` |
| Nova | `/etc/kolla/config/nova/ceph.conf` | `/etc/kolla/config/nova/ceph.client.cinder.keyring` |
| Gnocchi | `/etc/kolla/config/gnocchi/ceph.conf` | `/etc/kolla/config/gnocchi/ceph.client.gnocchi.keyring` |
| Manila | `/etc/kolla/config/manila/ceph.conf` | `/etc/kolla/config/manila/ceph.client.manila.keyring` |

실제 디렉터리를 `tree`로 찍어보면 아래와 같습니다.

```bash
# 출처: https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html
[deployer@ops ~]$ tree /etc/kolla/config
/etc/kolla/config
├── glance
│   ├── ceph.conf
│   └── ceph.client.glance.keyring
├── cinder
│   ├── ceph.conf
│   ├── cinder-volume
│   │   └── ceph.client.cinder.keyring
│   └── cinder-backup
│       ├── ceph.client.cinder.keyring
│       └── ceph.client.cinder-backup.keyring
└── nova
    ├── ceph.conf
    └── ceph.client.cinder.keyring
```

`globals.yml`에서 각 서비스의 Ceph 백엔드를 켜는 옵션은 별도로 존재합니다 ([`etc/kolla/globals.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/etc/kolla/globals.yml) 526·571·609번 줄 주석).

```yaml
# /etc/kolla/globals.yml
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
```

### keyring 탐색도 "호스트별 → 서비스별"로 일관된다

재미있는 점은 keyring도 config 병합과 동일한 철학을 따른다는 것입니다. [`ansible/roles/nova-cell/tasks/external_ceph.yml` 1~15번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/nova-cell/tasks/external_ceph.yml):

```yaml
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/nova-cell/tasks/external_ceph.yml
# ansible/roles/nova-cell/tasks/external_ceph.yml 1~15번 줄
- name: Check nova keyring file
  vars:
    keyring: "{{ nova_cell_ceph_backend['cluster'] }}.client.{{ nova_cell_ceph_backend['vms']['user'] }}.keyring"
    paths:
      - "{{ node_custom_config }}/nova/{{ inventory_hostname }}/{{ keyring }}"
      - "{{ node_custom_config }}/nova/{{ keyring }}"
  ansible.builtin.stat:
    path: "{{ lookup('first_found', paths) }}"
  delegate_to: localhost
  register: nova_cephx_keyring_file
  failed_when: not nova_cephx_keyring_file.stat.exists
```

`first_found` lookup을 씁니다. `paths` 리스트의 첫 번째 항목이 **호스트별** 경로, 두 번째가 **서비스별** 경로입니다. `first_found`는 앞에서부터 존재하는 파일을 찾아 반환하므로, 호스트별 파일이 있으면 그걸 쓰고 없으면 서비스별 파일로 폴백합니다.

merge_configs(뒤가 이긴다)와 `first_found`(앞에 있으면 쓴다)는 문법적으로 반대 방향이지만, **의미적으로는 "호스트별이 서비스별보다 우선"이라는 동일한 규칙**을 구현합니다. 설정 파일이든 시크릿이든 Kolla-Ansible에서 precedence를 헷갈릴 일이 없습니다.

---

## 실무에서 밟는 함정 4가지

책상에서 코드만 읽는 것과 실제로 배포하는 것 사이에는 언제나 함정이 있습니다. Kolla-Ansible 커스텀 config를 만지면서 자주 밟는 대표적 네 가지.

### 함정 1. `ceph config generate-minimal-conf`의 leading tab

External Ceph 연동 시 가장 자주 겪는 실수입니다. [External Ceph Guide](https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html)가 직접 경고합니다.

> "Commands like `ceph config generate-minimal-conf` generate configuration files that have leading tabs. These tabs break Kolla Ansible's ini parser. Be sure to remove the leading tabs from your `ceph.conf` files when copying them."

원인은 명확합니다. `merge_configs.py`가 쓰는 `oslo_config.iniparser.BaseParser`는 엄격한 INI 문법을 요구하는데, key 앞에 탭이 있으면 그 라인을 **continuation 라인**(앞 라인의 value 연장)으로 해석해 버립니다. 결과적으로 `mon_host = ...` 같은 필수 키가 파싱 결과에 누락됩니다.

회피 방법은 단순합니다.

```bash
# Ceph 클러스터에서 뽑은 ceph.conf의 leading tab 제거 후 Kolla 경로로 복사
[deployer@ops ~]$ ssh cephmon01 'ceph config generate-minimal-conf' \
    | sed 's/^\t//g' > /etc/kolla/config/nova/ceph.conf
[deployer@ops ~]$ head -5 /etc/kolla/config/nova/ceph.conf
# minimal ceph.conf for ...
[global]
fsid = a1b2c3d4-...
mon_host = [v2:10.0.0.11:3300/0,v1:10.0.0.11:6789/0], ...
```

### 함정 2. "덮어쓰기"는 기존 키가 있어야 의미가 있다

`OverrideConfigParser.parse()`의 병합 로직을 다시 읽어보면 `self._sections[section][key] = value`, 단순 대입입니다. 즉 **새 키는 새로 추가**됩니다. 오버라이드의 본래 목적인 "덮어쓰기"가 의미를 가지려면 role 내장 템플릿(`nova.conf.j2`)에 이미 그 키가 있어야 합니다.

- 같은 키 → 내 오버라이드가 이김 (의도대로)
- **내 오버라이드에만 있는 새 키** → 그 키가 단순히 추가됨 (부작용 없음)
- 내 오버라이드 여러 파일에 같은 키 → precedence 순서대로 가장 뒤에 온 파일이 이김

반대로 "기존에 있던 어떤 키를 **완전히 지우고 싶다**"는 요구는 merge_configs로 풀 수 없습니다. INI 레벨에 key 삭제 문법이 없기 때문입니다. 그런 경우는 role의 Jinja2 템플릿을 커스터마이즈하는 다른 접근이 필요한데, 이 글에서는 다루지 않습니다.

### 함정 3. `trust_as_template` shim은 한시적 코드

`merge_configs.py` 23~31번 줄 주석:

> "TODO(dougszu): From Ansible 12 onwards we must explicitly trust templates. Since this feature is not supported in previous releases, we define a noop method here for backwards compatibility. This can be removed in the G cycle."

Ansible 12에서 템플릿 신뢰 모델이 바뀌면서 추가된 하위 호환 shim입니다. **G 사이클에서 제거 예정**이라는 점이 명시되어 있습니다. 중요한 건 이 한 줄이 말하는 바: **Kolla-Ansible은 Ansible core 버전 호환성을 세밀하게 관리**합니다. Ansible을 아무 버전이나 써도 되는 게 아니므로, 운영 환경에서는 Kolla-Ansible 릴리즈 노트에서 권장 Ansible 버전을 확인하고 그대로 맞추는 것이 안전합니다.

### 함정 4. 존재하지 않는 오버라이드 파일은 조용히 스킵된다

세 번째이자 가장 흔한 "내 설정이 안 먹는다" 증상의 원인입니다. `merge_configs.py` 160~177번 줄의 `read_config()`가 `if os.access(source, os.R_OK):`로 감싸여 있습니다. 파일이 없거나 권한이 없으면 에러도, 로그도 없이 그냥 넘어갑니다.

대표 시나리오 두 가지:

1. **hostname 불일치**: inventory에는 `compute01.example.com`으로 써 두고, `/etc/kolla/config/nova/compute01/nova.conf`(FQDN이 아니라 short name)로 디렉터리를 만든 경우. 디렉터리명이 `{{ inventory_hostname }}`과 글자 단위로 정확히 일치해야 합니다.
2. **오타**: `nova-scheduler.conf`를 만든다는 게 `nova_scheduler.conf`(언더스코어)로 써버리거나, 서비스 디렉터리명을 `Nova/`로 (대문자) 만든 경우. 대소문자까지 정확해야 합니다.

디버깅 루틴은 이렇게 굳혔습니다.

```bash
# 1) inventory_hostname 확인 — ansible이 보는 값과 디렉터리명이 일치해야 한다
[deployer@ops ~]$ ansible -i /etc/kolla/multinode compute01.example.com \
    -m debug -a "var=inventory_hostname"

# 2) 배포 후 컨테이너 내부 최종 nova.conf 확인
[deployer@compute01 ~]$ docker exec nova_compute cat /etc/nova/nova.conf | grep -A2 '^\[libvirt\]'
[libvirt]
cpu_mode = host-passthrough

# 3) "설정이 있어야 할 자리에 없다"면 /etc/kolla/config 디렉터리 트리를 tree로 찍고
#    inventory_hostname과 문자열 비교
[deployer@ops ~]$ tree /etc/kolla/config/nova
```

특히 2번(컨테이너 내부 최종 파일 확인)을 강력히 추천합니다. 배포 시스템이 "병합해 준 결과물"을 직접 보는 것이 가장 빠른 진단입니다.

---

## 정리 & 다음 글 예고

긴 글이었습니다. 핵심만 다시 모으면 이렇습니다.

1. **Kolla는 2014년 9월 Steven Dake의 PoC에서 출발**했습니다. Newton 시점에 이미지 빌드(`kolla`)와 배포(`kolla-ansible`)가 별개 저장소로 분리됐고, 코알라 마스코트는 *"Kolla → Koala"의 두운* 이 공식 선정 이유입니다. 그리스어 "glue" 어원설은 공식 기록에 없습니다.
2. **2026년 현재는 SLURP 정책 하에서 Antelope / Caracal / Epoxy / Gazpacho** 가 지원 경로이며, 신규 배포는 **2025.1 Epoxy (Kolla-Ansible 20.x)** 가 가장 안전한 선택입니다. 기본 이미지는 `quay.io/openstack.kolla/*:2025.1-rocky-9` 형식, 네임스페이스가 예전 `kolla`가 아니라 **`openstack.kolla`** 라는 점을 꼭 기억하세요.
3. **Kolla-Ansible의 커스텀 config는 "덮어쓰기"가 아니라 "섹션·키 단위 병합"** 입니다. 이를 구현하는 게 `merge_configs.py`의 `OverrideConfigParser` 클래스이며, `parse()`가 호출될 때마다 `_cur_sections`를 초기화한 뒤 누적 `_sections`에 단순 대입으로 덮어씁니다. 그래서 **"뒤에 parse된 source가 이긴다"**.
4. **precedence는 role별로 하드코딩된 5단계** 입니다: `role 내장 템플릿 → global.conf → <service>.conf → <service>/<file>.conf → <service>/<hostname>/<file>.conf`. Nova·Glance 모두 동일. External Ceph keyring 탐색(`first_found`)도 "호스트별 → 서비스별" 방향으로 같은 규칙.
5. **가장 흔한 버그는 "내 파일이 파싱되지 않은 것"** 입니다. `merge_configs`는 `os.access(source, os.R_OK)`로 감싸고 있어 존재하지 않는 파일을 조용히 스킵합니다. 경로 오타, hostname 불일치(FQDN vs short name), leading tab 파싱 에러 — 이 세 가지만 점검하면 90%의 증상이 풀립니다.

### 다음 글 예고

이 시리즈 `kolla-ansible-internals`의 다음 편에서 다룰 후보들:

- `kolla-ansible reconfigure`는 내부적으로 정확히 어떤 playbook과 task를 실행하는가? 각 role의 `tasks/reconfigure.yml`이 `deploy.yml`과 무엇이 다른가?
- `kolla_container_engine: podman` 전환기 — Docker → Podman 마이그레이션의 실제 범위와, [`migrate-container-engine`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/setup.cfg) 서브커맨드가 하는 일.
- Kolla 이미지 빌드(`kolla-build`) 파이프라인 자체 파고들기 — Dockerfile 조립과 `kolla-build.conf` 커스터마이즈.

읽어 주셔서 고맙습니다. 틀린 부분이나 보강했으면 하는 지점이 있으면 주저 말고 댓글이나 이슈로 알려 주세요.

---

## 참고 자료

### 1차 출처 — Governance / Releases

- [OpenStack TC — Kolla project reference](https://governance.openstack.org/tc/reference/projects/kolla.html) — mission, PTL, deliverables
- [OpenStack TC — SLURP 도입 결의문 (2022-02-10)](https://governance.openstack.org/tc/resolutions/20220210-release-cadence-adjustment.html)
- [releases.openstack.org](https://releases.openstack.org/) — 전체 OpenStack 릴리즈 인덱스
- [releases.openstack.org/teams/kolla.html](https://releases.openstack.org/teams/kolla.html) — Kolla 팀 릴리즈 타임라인
- [opendev.org/openstack/releases — deliverables/](https://opendev.org/openstack/releases/src/branch/master/deliverables) — SLURP 매트릭스의 1차 데이터

### 1차 출처 — Kolla-Ansible 소스 (master, 2026-04-10 기준)

- [`ansible/action_plugins/merge_configs.py`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/action_plugins/merge_configs.py) — 본 글의 핵심 분석 대상
- [`ansible/roles/nova/tasks/config.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/nova/tasks/config.yml)
- [`ansible/roles/nova-cell/tasks/external_ceph.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/nova-cell/tasks/external_ceph.yml)
- [`ansible/roles/glance/tasks/config.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/glance/tasks/config.yml)
- [`ansible/site.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/site.yml)
- [`ansible/group_vars/all/common.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/group_vars/all/common.yml) — 레지스트리·네임스페이스·베이스 디스트로 기본값
- [`etc/kolla/globals.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/etc/kolla/globals.yml)
- [`setup.cfg` (entry_points)](https://opendev.org/openstack/kolla-ansible/raw/branch/master/setup.cfg)
- [`ansible/group_vars/all/` 디렉터리 목록](https://opendev.org/openstack/kolla-ansible/src/branch/master/ansible/group_vars/all) — per-service 파일 52개
- [`ansible/roles/` 디렉터리 목록](https://opendev.org/openstack/kolla-ansible/src/branch/master/ansible/roles)
- [kolla-ansible README.rst](https://opendev.org/openstack/kolla-ansible/raw/branch/master/README.rst) — Newton 분리 서술

### 1차 출처 — 공식 문서

- [Kolla-Ansible 메인 문서](https://docs.openstack.org/kolla-ansible/latest/)
- [Quickstart](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html) — `quay.io/openstack.kolla` 네임스페이스 공식 언급
- [Support Matrix](https://docs.openstack.org/kolla-ansible/latest/user/support-matrix.html) — `kolla_base_distro` 허용 값
- [Advanced Configuration](https://docs.openstack.org/kolla-ansible/latest/admin/advanced-configuration.html) — 커스텀 config 병합 규칙
- [Operating Kolla](https://docs.openstack.org/kolla-ansible/latest/user/operating-kolla.html) — prechecks/deploy/post-deploy/reconfigure 플로우
- [External Ceph Guide](https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html)
- [Deployment Philosophy](https://docs.openstack.org/kolla-ansible/latest/admin/deployment-philosophy.html) — "왜 컨테이너인가"에 대한 공식 서술
- [kolla-ansible 2023.2 Bobcat release notes](https://docs.openstack.org/releasenotes/kolla-ansible/2023.2.html) — Podman 정식 지원 도입

### 히스토리·마스코트

- [wiki.openstack.org/wiki/Kolla](https://wiki.openstack.org/wiki/Kolla) — Steven Dake, 2014-09-11 등록 기록
- [openstack.org/project-mascots/](https://www.openstack.org/project-mascots/) — 전체 프로젝트 마스코트 리스트
- [Superuser — "Creature or feature? Here's the story behind your favorite OpenStack project mascot"](https://superuser.openinfra.org/articles/openstack-project-mascots/) — Heidi Joy Tretheway 저, "Kolla's koala" 두운 근거
