# infra-repo

홈랩 Kubernetes 클러스터의 GitOps 매니페스트와 구성 코드 저장소입니다. ArgoCD가 `apps/`를 보고 클러스터 상태를 맞추고, 클러스터와 서버 구성은 `ansible/`의 단계별 플레이북으로 만듭니다.

## 동작 방식
- `root-app.yaml`이 `apps/`를 가리키는 App of Apps입니다. `apps/*-app.yaml`을 추가하면 ArgoCD가 하위 앱을 만들고, 자동 동기화(prune·selfHeal)가 켜져 있습니다.
- 앱마다 Helm 차트(`apps/<이름>/`)로 구성하고, 이미지 태그는 `apps/portfolio-web/values.yaml`에서 관리합니다.
- 배포는 [jiong-minihome](https://github.com/wownae-ct/jiong-minihome)의 Jenkins 파이프라인이 이미지를 빌드해 이 저장소의 태그를 커밋하는 방식입니다. 헬스 체크가 실패하면 그 커밋을 `git revert`해 이전 이미지로 되돌립니다.

## 구조
```
root-app.yaml                         # App of Apps 진입점
apps/
  argocd, istio-gateway, kiali        # 진입·서비스 메시
  portfolio-web                       # jiong-minihome 배포 차트(replica 2, 롤링 업데이트)
  jenkins                             # CI(Kaniko 빌드)
  prometheus, loki, promtail, grafana # 메트릭·로그·대시보드, Alertmanager 규칙
  longhorn, minio                     # 스토리지
ansible/                              # 클러스터·서버 구성 플레이북(ansible/README.md 참고)
docs/                                 # 작업 기록
```

## 시크릿
Sealed Secrets로 암호화한 파일(`*sealed-secret.yaml`)만 커밋합니다. `*.secret.yaml`, `*-credentials.yaml`, `.env*`는 커밋하지 않습니다(.gitignore).

## 관련 저장소
- [homelab-terraform](https://github.com/wownae-ct/homelab-terraform): Proxmox·AWS·Cloudflare를 정의한 Terraform
- [jiong-minihome](https://github.com/wownae-ct/jiong-minihome): 배포 대상 앱과 Jenkins 파이프라인
- [k3s-repo](https://github.com/wownae-ct/k3s-repo): cortex-agent용 k3s 클러스터의 GitOps 매니페스트
