---
title: "Kolla-Ansible 구조와 동작 원리 — 코드로 읽는 커스텀 config 병합 메커니즘"
date: 2026-04-11T00:40:00+09:00
draft: false
tags: ["openstack", "kolla-ansible", "ansible", "deployment", "internals", "nova", "ceph"]
categories: ["openstack"]
series: ["kolla-ansible-internals"]
author: "신호철 (shingoon)"
description: "Kolla-Ansible의 저장소 구조부터 merge_configs 액션 플러그인 내부까지 실제 코드 라인을 따라 분석하고, nova.conf·ceph.conf 등 커스텀 옵션이 어떻게 병합되는지 1차 출처 기준으로 정리합니다."
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

## 들어가며 — 왜 이 글을 쓰는가

Kolla-Ansible로 OpenStack을 한 번이라도 올려본 분이라면 거의 틀림없이 이런 상황을 마주쳤을 겁니다.

- `/etc/kolla/config/nova.conf`에 `[DEFAULT] debug = True` 한 줄 넣고 `kolla-ansible reconfigure` 돌렸는데, 막상 컴퓨트 노드의 `nova.conf`에는 반영이 안 된 것처럼 보인다.
- 같은 옵션을 `global.conf`와 `nova.conf` 양쪽에 쓰면 누가 이기는지 자신이 없어 결국 둘 다 지우고 한쪽으로 몰아둔다.
- External Ceph 붙이면서 `ceph config generate-minimal-conf`로 뽑은 `ceph.conf`를 그대로 `/etc/kolla/config/nova/ceph.conf`에 넣었더니 배포가 깨진다.
- 특정 컴퓨트 노드에서만 CPU pinning을 쓰고 싶은데, 호스트별 오버라이드 경로가 FQDN이어야 하는지 short name이어야 하는지 헷갈린다.

이 증상들은 전부 Kolla-Ansible이 **어떻게 여러 config 소스를 병합하는지**를 코드 레벨에서 이해하면 한 번에 풀립니다. 이 글에서는 저장소 구조부터 CLI 진입점, `merge_configs` 액션 플러그인의 Python 코드, 그리고 실전에서 밟게 되는 함정까지 1차 출처(opendev 코드 + 공식 문서) 기준으로 하나씩 따라가 봅니다.

**다루는 버전**은 [`opendev master` 브랜치 기준 2026-04-10 체크아웃](https://opendev.org/openstack/kolla-ansible)과 릴리즈된 `21.0.0` (2025.2 Flamingo stable), `20.3.0` (2025.1 Epoxy 최신 패치), `19.7.0` (2024.2 Dalmatian 최신 패치)입니다. 코드 라인 번호는 master 기준이며, 안정 릴리즈에서도 핵심 로직은 사실상 동일합니다.

**대상 독자**: Kolla-Ansible 배포를 이미 해본 적이 있고, "왜 `/etc/kolla/config/nova.conf`에 값을 써도 기존 기본값이 사라지지 않는가?"를 코드 레벨에서 납득하고 싶은 OpenStack 운영자 / DevOps 엔지니어.

---

## Kolla-Ansible이란 — 역할과 지원 매트릭스

우선 용어부터 정리하고 가죠. OpenStack 생태계에는 이름이 비슷한 두 프로젝트가 있습니다.

- **Kolla**: OpenStack 서비스 컨테이너 이미지를 빌드하는 프로젝트. `kolla-build` 명령으로 Nova/Neutron/Cinder 등 수십 개 서비스의 컨테이너 이미지를 만들어 레지스트리에 밀어 넣습니다.
- **Kolla-Ansible**: 위에서 만든(혹은 공식 레지스트리에서 받은) 이미지를 **실제 호스트에 배포**하는 Ansible 기반 도구. 이 글의 주제입니다.

이 글에서는 Kolla-Ansible(배포 도구) 쪽만 다루고, 이미지 빌드 파이프라인(Kolla)은 범위에서 제외합니다.

### 릴리즈 매핑

OpenStack의 배포 사이클과 Kolla-Ansible의 major 버전은 1:1로 매핑됩니다. 본 글 시점의 매트릭스는 다음과 같습니다.

| OpenStack 릴리즈 | Kolla-Ansible 버전 | 출처 |
|---|---|---|
| 2025.2 Flamingo (stable) | **21.0.0** (commit `615b95ade9ebe8036b40ba737dd1390bb8a963ef`) | [flamingo/kolla-ansible.yaml](https://opendev.org/openstack/releases/raw/branch/master/deliverables/flamingo/kolla-ansible.yaml) |
| 2025.1 Epoxy | **20.3.0** (commit `78c1077e9508272275d9e633c2450ff0353f99c4`, 원 릴리즈 2025-04-02) | [epoxy/kolla-ansible.yaml](https://opendev.org/openstack/releases/raw/branch/master/deliverables/epoxy/kolla-ansible.yaml) |
| 2024.2 Dalmatian | **19.7.0** (commit `0c2f8168bfa190e8f558982f1d235d66ccf1c197`) | [dalmatian/kolla-ansible.yaml](https://opendev.org/openstack/releases/raw/branch/master/deliverables/dalmatian/kolla-ansible.yaml) |

### 지원 매트릭스 (master / 2025.2 기준)

- **지원 Host OS**: CentOS Stream 10, Debian Trixie(13), Rocky Linux 10, Ubuntu Noble(24.04) — [Support Matrix 문서](https://docs.openstack.org/kolla-ansible/latest/user/support-matrix.html)
- **`kolla_base_distro` 허용 값**: `['centos', 'debian', 'rocky', 'ubuntu']` — [`etc/kolla/globals.yml` 47번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/etc/kolla/globals.yml)
- **Python 요구 버전**: `>= 3.11` (Python 3.11, 3.12 classifier) — [`setup.cfg` 9번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/setup.cfg)
- **컨테이너 엔진**: `docker` (기본) 또는 `podman` — `kolla_container_engine` 변수로 선택 ([`globals.yml` 91~93번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/etc/kolla/globals.yml), [`common.yml` 221번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/group_vars/all/common.yml))
- **License**: Apache License 2.0

실무 팁: `kolla_base_distro`와 실제 host OS는 다를 수 있습니다. 예를 들어 Ubuntu 24.04 호스트에 `kolla_base_distro: "ubuntu"`를 박아 두면 컨테이너 이미지의 base도 Ubuntu가 됩니다. 이 값은 "컨테이너 이미지가 어떤 배포판 base로 빌드되었는지"를 지정하는 변수지 host OS와 무조건 일치해야 하는 건 아닙니다.

---

## 저장소 구조 한눈에 보기

[`opendev.org/openstack/kolla-ansible`](https://opendev.org/openstack/kolla-ansible) 저장소를 체크아웃하면 최상위에 다음과 같은 디렉터리가 보입니다.

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

### `ansible/roles/` — 서비스별 역할 52개 이상

`ansible/roles/` 디렉터리는 [master 기준 52개 이상의 역할](https://opendev.org/openstack/kolla-ansible/src/branch/master/ansible/roles)로 구성되어 있습니다. 주요 역할을 나열해 보면:

```text
aodh, barbican, bifrost, blazar, ceilometer, ceph-rgw, certificates,
cinder, cloudkitty, collectd, common, container-engine-migration, cron,
cyborg, designate, destroy, etcd, fluentd, glance, gnocchi, grafana,
hacluster, haproxy-config, heat, horizon, ironic, iscsi, keystone,
kolla-ansible, kuryr, letsencrypt, loadbalancer, loadbalancer-config,
magnum, manila, mariadb, masakari, memcached, mistral, module-load,
multipathd, neutron, nova, nova-cell, octavia, octavia-certificates,
opensearch, openvswitch, ovn-controller, ovn-db, ovs-dpdk, placement,
prechecks, prometheus, prometheus-node-exporters, proxysql-config,
prune-images, rabbitmq, service-*, skyline, sysctl, tacker, trove,
valkey, watcher, zun
```

여기서 초보자가 자주 놓치는 포인트 하나. **Nova는 `nova`와 `nova-cell` 두 role로 쪼개져 있습니다.** `nova` role이 컨트롤 플레인(api/conductor/scheduler/super-conductor)을, `nova-cell` role이 컴퓨트 셀(libvirt/compute/ssh 등)을 담당합니다. 그래서 "CPU pinning 설정을 어디에 넣어야 하지?" 같은 질문에서 의외로 `nova-cell/tasks/...`를 들여다봐야 할 때가 있습니다.

### Role의 표준 구조

각 role은 [`nova` 디렉터리](https://opendev.org/openstack/kolla-ansible/src/branch/master/ansible/roles/nova)를 예로 들면 다음 다섯 서브디렉터리로 구성됩니다. Ansible 표준 구조에 `meta/`가 빠져 있는 것이 특징입니다.

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

`tasks/` 내부에 `deploy.yml`, `reconfigure.yml`, `precheck.yml`, `upgrade.yml` 등이 개별 파일로 쪼개져 있다는 점을 기억해 두세요. `kolla-ansible <subcommand>` 각각이 이 파일들과 의미적으로 대응됩니다.

---

## CLI 진입점 — `kolla-ansible deploy`는 뭘 하는가

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

여기서 두 층의 진입점을 볼 수 있습니다.

1. `console_scripts` 블록의 `kolla-ansible = kolla_ansible.cmd.kolla_ansible:main` — 쉘에서 `kolla-ansible`을 치면 실행되는 Python 함수.
2. `kolla_ansible.cli` 블록 — stevedore가 로드하는 서브커맨드 클래스들. `deploy`, `prechecks`, `reconfigure`, `bootstrap-servers`, `post-deploy`, `upgrade` 등이 모두 `kolla_ansible/cli/commands.py` 내의 클래스로 매핑됩니다.

이 구조가 의미하는 바는 명확합니다. `kolla-ansible deploy`를 입력하면 stevedore가 `Deploy` 클래스를 로드하고, 그 클래스 내부에서 Ansible playbook이 호출되는 흐름입니다. 실제로 어떤 서브커맨드가 어떤 playbook 파일과 매핑되는지는 `kolla_ansible/cli/commands.py`를 직접 보지 않으면 단정하기 어렵습니다(본 글에서는 그 부분까지는 확인하지 않았으므로 "확인 필요"로 표기해 둡니다). 다만 전형적인 패턴은 다음과 같습니다.

- `deploy` 계열 → `ansible/site.yml`을 중심으로 서비스별 play import.
- `bootstrap-servers` → 별도 playbook(`bootstrap-servers.yml` 형태)이 존재할 가능성이 높으나, 이 역시 직접 소스를 보지 않은 이상 "확인 필요".
- `reconfigure` / `upgrade` → 각 role의 `tasks/reconfigure.yml`, `tasks/upgrade.yml`을 호출하는 site-level playbook.

그리고 배포의 전체 진입점이 되는 [`ansible/site.yml` (1051 lines)](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/site.yml)의 첫 부분은 이렇게 시작됩니다.

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

흐름은 단순합니다.

1. 모든 호스트에서 사실(facts) 수집.
2. `kolla_action`과 `enable_*` 변수 기준으로 `group_by`를 수행해서, **disabled 서비스에 대한 play는 아예 호스트 그룹에서 제외**. 대규모 환경에서 skipped task 비용조차 무시 못 하기 때문에 추가된 최적화라는 점을 site.yml 주석이 직접 설명하고 있습니다.
3. 이후 서비스별(`aodh`, `barbican`, `cinder`, `glance`, `nova`, `neutron`, ...) play를 각각 `import_playbook` 형태로 순서대로 실행.

한 줄 요약: **`kolla-ansible deploy`는 stevedore로 `Deploy` 클래스를 로드하고, 그 클래스가 site.yml 기반 playbook을 실행하는 구조**입니다. 그리고 site.yml은 "facts 수집 → 그룹화 → 서비스별 play 연속 import"라는 단순한 패턴을 따릅니다.

실전 배포 플로우는 공식 문서 [Operating Kolla](https://docs.openstack.org/kolla-ansible/latest/user/operating-kolla.html)와 위의 entry_points를 합쳐 보면 이렇게 정리됩니다.

```bash
# 출처: https://docs.openstack.org/kolla-ansible/latest/user/operating-kolla.html
# + setup.cfg kolla_ansible.cli entry_points
[deployer@ops ~]$ kolla-ansible bootstrap-servers -i /etc/kolla/multinode
[deployer@ops ~]$ kolla-ansible prechecks         -i /etc/kolla/multinode
[deployer@ops ~]$ kolla-ansible deploy            -i /etc/kolla/multinode
[deployer@ops ~]$ kolla-ansible post-deploy       -i /etc/kolla/multinode
# 설정 수정 후 재배포
[deployer@ops ~]$ kolla-ansible reconfigure       -i /etc/kolla/multinode
```

---

## 핵심: `merge_configs` 액션 플러그인 분석

이 글의 본론입니다. Kolla-Ansible의 커스텀 config 오버라이드 동작을 실제로 구현하는 것은 [`ansible/action_plugins/merge_configs.py`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/action_plugins/merge_configs.py)라는 한 파일입니다(2026-04-10 시점 master 브랜치 기준 **234 lines**).

### 왜 `template` 모듈이 아니라 커스텀 action plugin인가

OpenStack의 설정 파일은 거의 예외 없이 INI 포맷이고, Nova만 해도 `[DEFAULT]`, `[api]`, `[libvirt]`, `[neutron]`, `[cinder]` 같은 수많은 섹션이 있습니다. 여기에 요구 사항이 하나 추가됩니다. **"여러 소스에서 온 값을 섹션·키 단위로 병합"**해야 합니다. 단순히 뒤의 파일로 앞의 파일을 통째로 교체하면 안 되고, `[DEFAULT]`는 그대로 두되 `[libvirt] cpu_mode`만 덮어쓰는 식의 세밀한 병합이 필요합니다.

Ansible의 내장 `template` 모듈은 단일 Jinja2 템플릿을 렌더링해서 그대로 전송하는 1:1 구조라 이 요구를 만족하지 못합니다. 그래서 Kolla-Ansible 팀이 `oslo_config.iniparser.BaseParser`를 상속한 커스텀 파서를 action plugin 형태로 작성한 것입니다.

### `OverrideConfigParser` — 섹션·키 단위 병합

핵심 클래스는 `OverrideConfigParser`입니다. 아래가 파싱과 병합을 담당하는 원문입니다.

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

1. **상태 두 벌 유지**: `_cur_sections`는 "지금 이 순간 파싱 중인 한 파일"의 결과, `_sections`는 "여러 소스를 거치며 누적된" 결과입니다.
2. **`parse()` 호출마다 `_cur_sections`가 초기화**되지만 `_sections`는 유지됩니다. 그래서 같은 파서 인스턴스에 여러 파일을 순서대로 먹이면 자연스럽게 누적 병합이 됩니다.
3. **병합 시 단순 대입**: `self._sections[section][key] = value`. 동일 섹션·동일 키가 이미 있으면 **그냥 덮어씁니다**. 즉 "나중에 parse된 값이 이긴다"는 precedence가 이 한 줄에서 나옵니다.

추가로 알아둘 점은 `collections.OrderedDict`를 쓴다는 것입니다. Python 3.7+에서는 일반 dict도 삽입 순서를 보장하지만, 여기서 명시적으로 `OrderedDict`를 쓰는 이유는 **직렬화 시 섹션·키 순서를 보장**하기 위함입니다. 결과적으로 나오는 `nova.conf`의 섹션 순서가 원본 템플릿과 일치해서 diff 검사가 수월해집니다.

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

여기서 세 가지 실무적으로 중요한 사실이 나옵니다.

1. **`os.access(source, os.R_OK)` 체크** — 존재하지 않거나 읽을 수 없는 파일은 **조용히 무시됩니다**. 에러도, 경고도 남지 않습니다. "내 오버라이드가 먹는 것 같지 않다"의 70%가 이 체크 때문에 발생합니다(오타 난 경로, hostname 불일치 등).
2. **`self._templar.template(template_data)`** — 렌더링은 일반 Ansible templar가 수행합니다. 따라서 운영자가 `/etc/kolla/config/nova.conf`에 `{{ inventory_hostname }}`, `{{ kolla_internal_vip_address }}`, `{{ enable_ceph | bool }}` 같은 Ansible 변수를 자유롭게 쓸 수 있습니다. 이건 "custom config"지만 사실상 Jinja2 템플릿입니다.
3. **`trust_as_template` shim** — Ansible 12+에서는 템플릿 신뢰를 명시적으로 선언해야 하는 변경이 예정되어 있어, 그에 대비한 하위 호환 wrapper입니다. [merge_configs.py 23~31번 줄 주석](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/action_plugins/merge_configs.py)에 `"TODO(dougszu): ... This can be removed in the G cycle."`라고 명시되어 있어 향후 G 릴리즈에서 제거될 예정임을 알 수 있습니다.

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

이 설계는 Ansible action plugin에서 권장되는 **composition 패턴**의 좋은 교과서 예제입니다. 새 task를 복제해 `sources` / `whitespace` 같은 merge_configs 전용 인자를 제거하고, 그 자리에 임시 파일 경로(`src=result_file`)를 꽂은 다음, `action_loader.get('copy', ...)`로 내장 copy 액션을 로드해 `.run(task_vars=...)`로 호출합니다.

왜 이렇게 만들었을까요? 이유는 단순합니다. **`mode`, `become`, `owner`, `group`, 권한 비트, 파일 해시 비교 같은 기능을 재구현하고 싶지 않아서**입니다. 내장 copy 액션은 이미 그 모든 것을 신뢰성 있게 수행하고 있고, action plugin 레이어에서 재귀 호출하면 이걸 공짜로 얻을 수 있습니다. 그래서 우리가 role의 `config.yml`에 `mode: "0660"`, `become: true`를 쓰면 그 값들이 그대로 먹는 것입니다.

---

## 병합 순서 — nova.conf가 만들어지는 정확한 과정

이제 실제 role 코드가 어떻게 `merge_configs`를 호출하는지 봅시다. Nova의 경우 [`ansible/roles/nova/tasks/config.yml` 54~70번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/nova/tasks/config.yml)에 핵심 태스크가 있습니다.

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

여기서 `with_dict`로 반복하는 `nova_services`는 `nova-api`, `nova-scheduler`, `nova-conductor`, `nova-super-conductor` 같은 nova 컨트롤 플레인 서비스 딕셔너리입니다. 각 서비스마다 `sources` 리스트가 순서대로 병합되어 `dest`에 복사됩니다.

`sources` 리스트의 순서는 **precedence의 낮음 → 높음 순**입니다. merge_configs의 동작 원리(뒤에 parse된 값이 이긴다)를 상기하면서 다음 다이어그램을 보세요.

```text
nova.conf 병합 순서 (precedence: 낮음 → 높음)

┌─────────────────────────────────────────────────────────────┐
│ 1. {{ role_path }}/templates/nova.conf.j2                   │  ← 가장 낮음
│    Kolla-Ansible이 제공하는 내장 템플릿                     │
│    (여기에 없는 키는 오버라이드로도 "추가"할 순 있지만       │
│     의미 있는 "덮어쓰기"는 이미 키가 있어야 한다)           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. {{ node_custom_config }}/global.conf                     │
│    ≡ /etc/kolla/config/global.conf                          │
│    모든 OpenStack 서비스 공통 오버라이드                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. {{ node_custom_config }}/nova.conf                       │
│    ≡ /etc/kolla/config/nova.conf                            │
│    Nova 전체에 적용되는 오버라이드                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. {{ node_custom_config }}/nova/{{ item.key }}.conf        │
│    ≡ /etc/kolla/config/nova/nova-scheduler.conf 등          │
│    서비스별(=컨테이너별) 오버라이드                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. {{ node_custom_config }}/nova/                           │
│      {{ inventory_hostname }}/nova.conf                     │
│    ≡ /etc/kolla/config/nova/compute-01/nova.conf 등         │
│    호스트별 오버라이드 (가장 높음)                          │
└─────────────────────────────────────────────────────────────┘
```

### 다른 role도 같은 패턴인가?

네. 교차 검증으로 [`ansible/roles/glance/tasks/config.yml` 70~77번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/glance/tasks/config.yml)을 보면 동일한 5단계 precedence를 쓰고 있습니다.

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

즉 "role 내장 템플릿 → `global.conf` → `<service>.conf` → `<service>/<file>.conf` → `<service>/<hostname>/<file>.conf`" 이 다섯 단계 precedence는 **Kolla-Ansible 전체의 관용 규약**입니다. 흥미로운 점은 이 규약이 어떤 변수나 매크로로 추상화되지 않고, 각 role의 config.yml에 **그대로 복사되어** 있다는 것입니다. 그래서 새 역할을 추가할 때도 기존 role의 5줄을 그대로 따라 붙이면 오버라이드 동작이 자동으로 따라옵니다.

### `node_custom_config`와 `node_config_directory`

위 코드에서 `{{ node_custom_config }}`와 `{{ node_config_directory }}`의 정체는 무엇일까요? [`ansible/group_vars/all/common.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/group_vars/all/common.yml) 163번 줄과 166번 줄에서 기본값을 찾을 수 있습니다.

```yaml
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/group_vars/all/common.yml
# 163번, 166번 줄
node_custom_config: "{{ node_config }}/config"
node_config_directory: "/etc/kolla"
```

`node_config`는 배포 호스트 측의 kolla 설정 루트이므로 실질적으로 `node_custom_config = /etc/kolla/config`가 됩니다. 공식 문서 [Advanced Configuration](https://docs.openstack.org/kolla-ansible/latest/admin/advanced-configuration.html)에도 이 경로가 그대로 나옵니다. 필요하면 `/etc/kolla/globals.yml`에서 이렇게 덮어쓸 수 있습니다.

```yaml
# /etc/kolla/globals.yml
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/etc/kolla/globals.yml
# globals.yml 58~59번 줄 주석에 기본값 명시

node_custom_config: "/etc/kolla/config"
```

### 실전 예제: compute01에서만 CPU pinning 켜기

이제 모든 퍼즐이 맞춰졌으니 "compute01 한 노드에서만 `host-passthrough`를 쓰고 싶다" 같은 요구를 파일 한 개로 해결할 수 있습니다.

```ini
# /etc/kolla/config/nova/compute01.example.com/nova.conf
# 이 파일은 오직 inventory_hostname == "compute01.example.com"인 노드의
# nova-compute 컨테이너에만 적용된다 (precedence 5단계)

[libvirt]
cpu_mode = host-passthrough
```

`inventory_hostname`이 FQDN인지 short name인지가 핵심입니다. Kolla-Ansible은 디렉터리명을 문자열 그대로 비교하므로, 인벤토리 파일에 `compute01.example.com`으로 써놓고 디렉터리는 `compute01`로 만들면 **조용히 스킵**됩니다. 오타 디버깅 섹션에서 다시 다루겠습니다.

### 주의: `global.conf`와 `nova.conf`에 같은 키가 있으면?

`OverrideConfigParser.parse()`의 병합 로직을 떠올리면 답이 명확합니다. 동일 섹션·동일 키라면 뒤에 온 `nova.conf`가 이깁니다. 반대로 **서로 다른 키**면 둘 다 살아남습니다. 즉 `global.conf`를 "전역 공통 설정", `nova.conf`를 "nova 전용 설정"으로 역할 분리해서 두면 아무 문제 없이 공존합니다.

---

## External Ceph 연동 — ceph.conf와 keyring은 어디에 두는가

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

실제 디렉터리를 `tree`로 찍어보면 아래와 같은 모습이 됩니다.

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

`globals.yml`에서 각 서비스의 Ceph 백엔드를 켜는 옵션은 별도로 존재합니다 ([`etc/kolla/globals.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/etc/kolla/globals.yml) 526·571·609번 줄).

```yaml
# /etc/kolla/globals.yml
# 출처: https://opendev.org/openstack/kolla-ansible/raw/branch/master/etc/kolla/globals.yml

glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
```

### keyring 탐색 순서도 "호스트별 → 서비스별"로 일관된다

재미있는 점은 keyring도 config 병합과 동일한 철학을 따른다는 것입니다. [`ansible/roles/nova-cell/tasks/external_ceph.yml` 1~15번 줄](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/nova-cell/tasks/external_ceph.yml)을 보면 이렇게 되어 있습니다.

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

merge_configs(뒤가 이긴다)와 `first_found`(앞에 있으면 쓴다)는 문법적으로는 반대 방향이지만, **의미적으로는 "호스트별이 서비스별보다 우선"이라는 동일한 규칙**을 구현합니다. 설정 파일이든 시크릿 파일이든 Kolla-Ansible에서 precedence를 헷갈릴 일은 없습니다.

---

## 실무에서 밟는 함정 4가지

책상에서 코드를 읽는 것과 실제로 배포하는 것 사이에는 언제나 함정이 있습니다. Kolla-Ansible 커스텀 config를 만지면서 제가 직접 밟았고, 공식 문서·코드에 명시된 대표적 4가지를 정리합니다.

### 함정 1. `ceph config generate-minimal-conf`의 leading tab

External Ceph 연동 시 가장 자주 겪는 실수입니다. [External Ceph Guide](https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html)가 직접 경고합니다.

> "Commands like `ceph config generate-minimal-conf` generate configuration files that have leading tabs. These tabs break Kolla Ansible's ini parser. Be sure to remove the leading tabs from your `ceph.conf` files when copying them."

원인은 명확합니다. `merge_configs.py`가 쓰는 `oslo_config.iniparser.BaseParser`는 엄격한 INI 문법을 요구하는데, key 앞에 탭이 있으면 그 라인을 **continuation 라인**(앞 라인의 value 연장)으로 해석해 버립니다. 결과적으로 `mon_host = ...` 같은 필수 키가 파싱 결과에 누락됩니다.

회피 방법은 단순합니다. `ceph.conf`를 Kolla 쪽으로 복사하기 전에 탭을 스페이스로 치환하거나 아예 제거해 주세요.

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

`merge_configs`의 `OverrideConfigParser.parse()`를 다시 읽어보면, 병합은 단순히 `self._sections[section][key] = value`입니다. 즉 **새 키는 새로 추가**됩니다. 오버라이드의 본래 목적인 "덮어쓰기"가 의미를 가지려면 role 내장 템플릿(`nova.conf.j2`)에 이미 그 키가 있어야 합니다.

- 같은 키 → 내 오버라이드가 이김 (의도대로)
- **내 오버라이드에만 있는 새 키** → 그 키가 단순히 추가됨 (부작용 없음, 오히려 편리)
- 내 오버라이드 여러 파일에 같은 키 → precedence 순서대로 가장 뒤에 온 파일이 이김

반대로 "기존에 있던 어떤 키를 **완전히 지우고 싶다**"는 요구는 merge_configs로 풀 수 없습니다. INI 레벨에서 key 삭제 문법이 없기 때문입니다. 그런 경우는 role의 Jinja2 템플릿(`nova.conf.j2`)을 커스터마이즈하는 다른 방법을 고민해야 합니다(본 글에서는 다루지 않습니다).

### 함정 3. `trust_as_template` shim은 한시적 코드

`merge_configs.py` 23~31번 줄 주석은 이렇게 말합니다.

> "TODO(dougszu): From Ansible 12 onwards we must explicitly trust templates. Since this feature is not supported in previous releases, we define a noop method here for backwards compatibility. This can be removed in the G cycle."

Ansible 12에서 템플릿 신뢰 모델이 바뀌면서 추가된 하위 호환 shim입니다. G 사이클에서 제거 예정이라는 점이 명시되어 있습니다. 무엇이 중요한가 하면, **Kolla-Ansible은 Ansible core 버전 호환성을 세밀하게 관리**한다는 것입니다. Ansible을 아무 버전이나 써도 되는 것이 아니므로, 운영 환경에서는 Kolla-Ansible 릴리즈 노트에서 권장 Ansible 버전을 확인하고 그대로 맞추는 것이 안전합니다.

(참고로 "Kolla-Ansible이 요구하는 정확한 Ansible 최소 버전"은 `setup.cfg`와 quickstart/support-matrix 문서에 명시적으로 나와 있지 않고, `requirements-core.yml` 내부에 pin이 있을 가능성이 높습니다. 본 글에서는 이 부분까지는 확인하지 않았으므로 "확인 필요"로 표기합니다.)

### 함정 4. 존재하지 않는 오버라이드 파일은 조용히 스킵된다

세 번째 함정이자 가장 흔한 "내 설정이 안 먹는다" 증상의 원인입니다. `merge_configs.py` 160~177번 줄의 `read_config()`가 `if os.access(source, os.R_OK):`로 감싸여 있음을 다시 보세요. 파일이 없거나 권한이 없으면 에러도, 로그도 없이 그냥 넘어갑니다.

실전에서 이 함정을 밟는 대표 시나리오는 두 가지입니다.

1. **hostname 불일치**: inventory 파일에는 `compute01.example.com`으로 써 두고, `/etc/kolla/config/nova/compute01/nova.conf`(FQDN이 아니라 short name)로 디렉터리를 만든 경우. 디렉터리명이 `{{ inventory_hostname }}`과 글자 단위로 정확히 일치해야 합니다.
2. **오타**: `nova-scheduler.conf`를 만든다는 게 `nova_scheduler.conf`(언더스코어)로 써버리거나, 서비스 디렉터리명을 `nova/` 대신 `Nova/`로 만든 경우. 대소문자까지 정확해야 합니다.

디버깅 루틴은 다음과 같이 정착시켰습니다.

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

저는 특히 2번(컨테이너 내부 최종 파일 확인)을 강력히 추천합니다. 배포 시스템이 "병합해 준 결과물"을 직접 보는 것이 가장 빠른 진단입니다.

---

## 정리 & 다음 글 예고

핵심 3줄 요약입니다.

1. **Kolla-Ansible의 커스텀 config는 "덮어쓰기"가 아니라 "섹션·키 단위 병합"이다.** 이를 구현하는 것이 `ansible/action_plugins/merge_configs.py`의 `OverrideConfigParser` 클래스이며, `parse()`가 호출될 때마다 `_cur_sections`를 초기화한 뒤 누적 `_sections`에 단순 대입으로 덮어쓴다. 그래서 "뒤에 parse된 source가 이긴다".
2. **precedence는 role별로 하드코딩된 5단계다**: `role 내장 템플릿 → global.conf → <service>.conf → <service>/<file>.conf → <service>/<hostname>/<file>.conf`. Nova, Glance 모두 동일한 패턴을 그대로 복사해서 쓴다. External Ceph의 keyring 탐색(`first_found`)도 "호스트별 → 서비스별" 방향으로 동일한 규칙을 따른다.
3. **가장 흔한 버그는 "내 파일이 파싱되지 않은 것"이다.** `merge_configs`는 `os.access(source, os.R_OK)`로 감싸고 있어 존재하지 않는 파일을 조용히 스킵한다. 경로 오타, hostname 불일치(FQDN vs short name), leading tab 파싱 에러 — 이 세 가지만 점검하면 90%의 증상이 풀린다.

### 다음 글 예고

이 시리즈 `kolla-ansible-internals`의 다음 편에서는 다음 중 하나를 다룰 예정입니다.

- `kolla-ansible reconfigure`는 내부적으로 정확히 어떤 playbook과 task를 실행하는가? 각 role의 `tasks/reconfigure.yml`이 `deploy.yml`과 무엇이 다른가?
- `kolla_container_engine: podman` 전환기 — Docker → Podman 마이그레이션이 실제로 어디까지 지원되며, `migrate-container-engine` 서브커맨드는 무엇을 하는가? (참고로 [`setup.cfg`의 `migrate-container-engine = kolla_ansible.cli.commands:MigrateContainerEngine`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/setup.cfg) entry에서 이 서브커맨드가 공식 지원됨을 확인할 수 있습니다.)

---

## 참고 자료

### 1차 출처 — opendev 소스 코드 (master 브랜치, 2026-04-10 기준)

- [`ansible/action_plugins/merge_configs.py`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/action_plugins/merge_configs.py) — 본 글의 핵심 분석 대상
- [`ansible/roles/nova/tasks/config.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/nova/tasks/config.yml)
- [`ansible/roles/nova-cell/tasks/external_ceph.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/nova-cell/tasks/external_ceph.yml)
- [`ansible/roles/glance/tasks/config.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/roles/glance/tasks/config.yml)
- [`ansible/site.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/site.yml)
- [`etc/kolla/globals.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/etc/kolla/globals.yml)
- [`ansible/group_vars/all/common.yml`](https://opendev.org/openstack/kolla-ansible/raw/branch/master/ansible/group_vars/all/common.yml)
- [`setup.cfg` (entry_points)](https://opendev.org/openstack/kolla-ansible/raw/branch/master/setup.cfg)
- [`ansible/roles/` 디렉터리 목록](https://opendev.org/openstack/kolla-ansible/src/branch/master/ansible/roles)
- [`ansible/roles/nova/` 디렉터리](https://opendev.org/openstack/kolla-ansible/src/branch/master/ansible/roles/nova)

### 1차 출처 — 공식 문서

- [Kolla-Ansible 메인 문서](https://docs.openstack.org/kolla-ansible/latest/)
- [Advanced Configuration (custom config 병합 규칙)](https://docs.openstack.org/kolla-ansible/latest/admin/advanced-configuration.html)
- [Operating Kolla (prechecks/deploy/post-deploy/reconfigure)](https://docs.openstack.org/kolla-ansible/latest/user/operating-kolla.html)
- [External Ceph Guide](https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html)
- [Bootstrap Servers](https://docs.openstack.org/kolla-ansible/latest/reference/deployment-and-bootstrapping/bootstrap-servers.html)
- [Support Matrix](https://docs.openstack.org/kolla-ansible/latest/user/support-matrix.html)
- [Quickstart](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html)

### 1차 출처 — 릴리즈 메타

- [2025.2 Flamingo kolla-ansible deliverable](https://opendev.org/openstack/releases/raw/branch/master/deliverables/flamingo/kolla-ansible.yaml)
- [2025.1 Epoxy kolla-ansible deliverable](https://opendev.org/openstack/releases/raw/branch/master/deliverables/epoxy/kolla-ansible.yaml)
- [2024.2 Dalmatian kolla-ansible deliverable](https://opendev.org/openstack/releases/raw/branch/master/deliverables/dalmatian/kolla-ansible.yaml)
- [OpenStack Epoxy 릴리즈 페이지 (2025-04-02)](https://releases.openstack.org/epoxy/index.html)

### 프로젝트 마스코트

- [OpenStack Project Mascots — Kolla-ansible](https://www.openstack.org/project-mascots/)
