# ansible

홈랩(Hyper-V·Proxmox)의 Kubernetes HA 클러스터와 주변 서버(bastion, storage, DB)를 **단계별 플레이북**으로 구성하는 Ansible 코드입니다. 서버를 옮길 때 VM 이미지를 복사하지 않고 이 코드로 처음부터 다시 구축했습니다.

## 플레이북 (`playbooks/`)
| 번호 | 내용 |
|---|---|
| 00 | bastion 부트스트랩(NAT, sshd 포트 전환) |
| 01 | 공통 설정(common) |
| 02 | bastion: iptables, 포트 노킹(knockd), BIND9, nginx, chrony, OpenVPN, WireGuard |
| 03 | storage: LVM, MinIO, NFS, SMB, firewalld, SELinux |
| 04 | DB: MariaDB, 계정 권한, ufw |
| 05~10 | Kubernetes: 공통 설정 → control-plane HA(keepalived·HAProxy) → kubeadm init → control-plane·worker join → Cilium |
| 11 | Sealed Secrets |
| 12 | MetalLB |
| 13 | Longhorn(디스크, R2 백업) |
| 14 | Istio |
| 15 | CoreDNS |
| 16 | ArgoCD |
| 17 | kube-prometheus-stack |
| 18 | App of Apps 시작: `root-app.yaml`만 적용하면 ArgoCD가 `apps/`를 읽어 나머지 앱을 만듭니다 |
| 19 | ECR 자격 증명 갱신 CronJob |
| 20 | Promtail |
| `verify-phase.yml` | 인프라 검증(25개) |
| `verify-final.yml` | 최종 E2E 검증: 외부 접속 경로, 장애 시나리오(`--tags resilience`), 모니터링(`--tags monitoring`) |

각 단계의 역할은 `roles/`에 있습니다(argocd, bastion, common, coredns, db, ecr-credentials-refresh, istio, k8s, kube-prometheus-stack, longhorn, metallb, promtail, sealed-secrets, storage).

## 실행
```bash
ansible-playbook -i inventories/hyperv/hosts.ini playbooks/02-bastion.yml --ask-vault-pass
```
`ansible.cfg`는 inventory를 **일부러 비워 두었습니다**. `-i`를 빠뜨리면 오류가 나게 해서 Hyper-V와 Proxmox 환경을 혼동하는 사고를 막습니다.

## 시크릿과 공개용 정리
- 비밀번호·토큰·키는 `inventories/<환경>/group_vars/all.yml`의 `vault_*` 변수로 받고 ansible-vault로 암호화합니다.
- 이 저장소에는 **예시 파일만** 있습니다(`inventories/hyperv/*.example`). 실제 inventory, 변수 파일, 인증서·키(`roles/bastion/files/`)는 올리지 않았고 `.gitignore`로 막아 두었습니다.
- 공개하면서 IP(`10.99.x.x`), 도메인(`example.ddns.net`, `lab.internal`), 포트 노킹 순서(`10001,10002,10003`)는 예시 값으로 바꿨습니다. 그대로 실행하려면 실제 값으로 채워야 합니다.
