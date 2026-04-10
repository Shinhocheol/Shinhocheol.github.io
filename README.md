# shingoon-lab

> Open Infrastructure 기술 블로그 — 1차 출처 기반의 팩트만 기록합니다.

**Live**: https://shinhocheol.github.io

## 기술 도메인

| 카테고리 | 주제 |
|---------|-----|
| Cloud Platform | OpenStack (2024.2 Dalmatian+), Kolla-Ansible, Heat, Horizon |
| Storage | Ceph (Reef/Squid+), RBD, CephFS, RGW, BlueStore |
| Container | Kubernetes, Helm, Operator, CRI-O/containerd |
| Networking | OVN, OVS, Neutron, Calico, Cilium, MetalLB |
| Observability | Prometheus, Grafana, Alertmanager, Loki, Thanos |
| CI/CD | GitLab CI, GitHub Actions, ArgoCD, Tekton, Jenkins |
| MLOps | Kubeflow, MLflow, Seldon Core, KServe, Ray |
| IaC | Ansible, Terraform, Packer |

## Stack

- [Hugo](https://gohugo.io/) (extended) — static site generator
- [PaperMod](https://github.com/adityatelange/hugo-PaperMod) — theme (git submodule)
- GitHub Actions → GitHub Pages 배포

## Local Development

```bash
# 저장소 클론 (submodule 포함)
git clone --recurse-submodules https://github.com/Shinhocheol/Shinhocheol.github.io.git
cd Shinhocheol.github.io

# 로컬 서버 실행
hugo server -D

# 새 글 작성
hugo new content posts/<category>/<slug>/index.md
```

## 디렉터리 규칙

- 블로그 글: `content/posts/{category}/{slug}/index.md`
- 이미지: `content/posts/{category}/{slug}/images/`
- 글로벌 이미지: `static/images/`
- slug 형식: 영문 소문자 + 하이픈 (예: `ceph-rbd-benchmarking`)

## License

글: CC BY-NC-SA 4.0 / 코드 예제: MIT
