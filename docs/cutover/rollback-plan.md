# 서비스 경로 전환(컷오버) 롤백 계획

날짜: 2026-04-01
대상: [6-10] 서비스 경로 전환(컷오버) (DDNS → EC2 EIP + SSL 발급)
전략: Blue-Green DNS 방식 전환

## 🎯 롤백 트리거 (실패 시나리오)

다음 상황 발생 시 즉시 롤백:

### 1. DNS 전파 지연 (5분 이상)

# 증상

dig <ddns-domain> +short @8.8.8.8

# → <ec2-eip> 아님 (5분 경과 후)

# 조치: 즉시 롤백

### 2. 새 경로 서비스 불가 (5분 이상)

# 증상

curl -I https://<ddns-domain> -H "Host: <ddns-domain>"

# → 500/503 에러 또는 응답 없음

# 조치: 즉시 롤백

### 3. WireGuard 터널 불안정

# 증상

ssh <ec2-user>@<ec2-eip> "sudo wg show" | grep handshake

# → 5분 이상 handshake 없음

# 조치: 즉시 롤백

### 4. Pod 상태 비정상

# 증상

kubectl get pods -n portfolio-web

# → Running 아님 또는 CrashLoopBackOff

# 조치: 즉시 롤백

---

## ⚠️ 롤백 불필요 상황 (정상 진행)

다음 상황은 롤백하지 않고 대기:

### 1. SSL 인증서 발급 실패

- Self-signed 인증서로 서비스 계속 운영
- DNS 전파 완료 후 certbot 재시도
- 서비스 영향 없음, 롤백 불필요

### 2. s3 서브도메인 접속 실패

- /s3/ path 방식으로 이미지 정상 렌더링 중
- s3 서브도메인은 Phase 2 작업 (컷오버 후)
- 서비스 영향 없음, 롤백 불필요

### 3. DNS 전파 중 일부 사용자 접속 불가

- DNS TTL 대기 중 정상 현상
- 대부분 사용자는 DNS 캐시로 접속 유지
- 5~30분 대기 후 자동 해결
- 롤백 불필요

---

## 🔄 롤백 절차

### 1단계: DDNS 원복 (1분, 다운타임 ~1분)

# 1. 컷오버 전 IP 확인 (사전 백업)

cat ~/cutover-validation/ddns-before.txt

# → 예: <bastion-ip> 또는 기존 공인 IP

# 2. No-IP 대시보드 접속

DDNS 제공사 관리 페이지

# 3. <ddns-domain> 설정

Current IP: <ec2-eip> (EC2)
→ New IP: [컷오버 전 IP] (위에서 확인한 값)

# 4. 저장 (즉시 반영)

# 5. 즉시 확인

dig <ddns-domain> +short

# → 기존 IP 나오면 성공

- 예상 반영 시간:

* No-IP 변경: 즉시~1분
* DNS 전파: 5~30분 (ISP 캐시)

- 다운타임: ~1분 (DNS 변경 순간만 순단)

---

### 2단계: 로컬 DNS 캐시 클리어 (1분)

# WSL2 / Linux

sudo systemd-resolve --flush-caches

# Windows (관리자 CMD)

ipconfig /flushdns

# macOS

sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder

# /etc/hosts 테스트 라인 제거 (있었다면)

sudo vim /etc/hosts

# <ec2-eip> <ddns-domain> 삭제

---

### 3단계: 기존 경로 서비스 확인 (2분)

# 1. HTTPS 접속 확인

curl -I https://<ddns-domain>

# 2. /health 엔드포인트

curl https://<ddns-domain>/health | jq

# 3. 이미지 렌더링 (/s3/ path)

curl -I https://<ddns-domain>/s3/portfolio-web/profile.jpg

# 4. Pod 상태

kubectl get pods -n portfolio-web -o wide

# 5. ArgoCD 상태

kubectl get applications -n argocd | grep portfolio-web

- 예상 결과:

* ✅ HTTPS 200 OK
* ✅ /health 정상 응답
* ✅ 이미지 정상 로딩
* ✅ Pod 2개 Running
* ✅ ArgoCD Synced & Healthy

---

### 4단계: Before 지표 비교 (1분)

# Before 지표와 비교

cat ~/cutover-validation/before-metrics.md

# 현재 상태 확인

curl -s https://<ddns-domain>/health | jq
kubectl get pods -n portfolio-web
kubectl get applications -n argocd | grep portfolio-web

- 비교 항목:

* Pod 개수: 2개 ✅
* ArgoCD 상태: Synced & Healthy ✅
* /health uptime: 연속성 확인
* 이미지 URL: /s3/ path 정상 ✅

---

## ⏱️ 예상 롤백 소요 시간

| 단계               | 소요 시간  | 다운타임 | 비고              |
| ------------------ | ---------- | -------- | ----------------- |
| 1. DDNS 원복       | 1분        | **~1분** | 변경 순간만       |
| 2. DNS 캐시 클리어 | 1분        | **0분**  | 로컬 작업         |
| 3. 서비스 확인     | 2분        | **0분**  | 검증 단계         |
| 4. Before 비교     | 1분        | **0분**  | 검증 단계         |
| **DNS 전파 대기**  | **5~30분** | **0분**  | 기존 경로 운영 중 |
| **총 소요 시간**   | **35분**   | **~1분** | DNS 변경 순간만   |

**핵심:** 기존 경로는 계속 운영 중이므로 DNS 전파 중에도 서비스 정상!

---

## 🔔 롤백 모니터링

### 1. DNS 전파 모니터링 (30초마다)

watch -n 30 'echo "=== DNS 전파 확인 ==="; \
 dig <ddns-domain> +short; \
 dig <ddns-domain> +short @8.8.8.8; \
 dig <ddns-domain> +short @1.1.1.1'

### 2. Pod 상태 모니터링 (5초마다)

watch -n 5 'kubectl get pods -n portfolio-web -o wide'

### 3. /health 엔드포인트 (30초마다)

watch -n 30 'curl -s https://<ddns-domain>/health | jq'

### 4. Nginx 액세스 로그 (실시간)

ssh <admin>@<bastion-ip> 'sudo tail -f /var/log/nginx/access.log | grep -E "(GET|POST)"'

### 5. WireGuard 터널 (1분마다, 참고용)

watch -n 60 'ssh <ec2-user>@<ec2-eip> "sudo wg show wg0" | grep -E "peer|handshake"'

---

## 📞 에스컬레이션 (롤백 후에도 문제 지속 시)

### 1. DNS 해결되지 않을 때

# Cloudflare DNS 강제 사용

sudo vim /etc/resolv.conf

# nameserver 1.1.1.1

# nameserver 8.8.8.8

# 또는 브라우저에서 직접 IP 접속 (임시)

# https://[기존-IP]

### 2. Pod 문제

# Pod 재시작

kubectl rollout restart deployment/portfolio-web -n portfolio-web

# 로그 확인

kubectl logs -n portfolio-web -l app=portfolio-web --tail=100

# ArgoCD 강제 sync

kubectl patch application portfolio-web -n argocd \
 -p '{"operation":{"sync":{"prune":false}}}' --type=merge

### 3. Bastion Nginx 문제

# Nginx 재시작

ssh -p <ssh-port> <admin>@<bastion-ip> "sudo systemctl restart nginx"

# 설정 확인

ssh -p <ssh-port> <admin>@<bastion-ip> "sudo nginx -t"

# 로그 확인

ssh -p <ssh-port> <admin>@<bastion-ip> "sudo tail -100 /var/log/nginx/error.log"

---

## 🎯 롤백 성공 기준

✅ 체크리스트:
[ ] DDNS가 컷오버 전 IP를 가리킴 (dig로 확인)
[ ] HTTPS 접속 정상 (Self-signed 인증서 OK)
[ ] /health 엔드포인트 200 OK
[ ] 이미지 /s3/ path 정상 렌더링
[ ] Pod 2개 Running, Ready 2/2
[ ] ArgoCD: Synced & Healthy
[ ] Jenkins 접근 가능 (<ddns-domain>/jenkins)
[ ] Before 지표와 일치 확인

---

## ⚠️ 롤백 중 절대 금지 사항

### \*\* 하지 말아야 할 것:

- ❌ EC2 EIP 해제 → WireGuard 터널 끊김, 더 큰 장애
- ❌ EC2/Bastion 재시작 → 불필요한 다운타임
- ❌ iptables 규칙 수정 → 이미 검증됨, 건드리지 말 것
- ❌ ArgoCD 이미지 롤백 → 컷오버는 인프라 변경, 앱 버전 변경 없음
- ❌ K8s Pod 삭제\*\* → 자동 재시작으로 더 복잡해짐

### \* 롤백 시 변경하는 것:

- ✅ DDNS IP만 원복 → 나머지는 그대로 유지

---

## 📊 양쪽 경로 동시 운영 (DNS 전파 중)

롤백 중에도 서비스는 계속 운영됩니다:
DNS 캐시 O 사용자 → 기존 IP → 기존 경로 ✅ (정상)
DNS 캐시 X 사용자 → EC2 EIP → 새 경로 ✅ (정상, 잠깐 전환됐다가 원복)

5~30분 DNS 전파 대기 후:
모든 사용자 → 기존 IP → 기존 경로 ✅ (롤백 완료)

- 서비스 영향: 최소화 (~1분 DNS 변경 순간만)

---

## 📝 롤백 실행 기록

롤백 실행 시 아래 정보를 기록:

vim ~/cutover-validation/rollback-execution-log.md

# 롤백 실행 기록

## 기본 정보

- 롤백 시작 시간: [YYYY-MM-DD HH:MM:SS]
- 롤백 사유: [트리거 상황]
- 실행자: [이름]

## 단계별 기록

- [ ] DDNS 원복 완료: [HH:MM:SS]
- [ ] DNS 캐시 클리어: [HH:MM:SS]
- [ ] 서비스 확인 완료: [HH:MM:SS]
- [ ] Before 지표 일치: [HH:MM:SS]
- [ ] DNS 전파 완료: [HH:MM:SS]

## 최종 상태

- HTTPS 접속: [정상/비정상]
- Pod 상태: [2/2 Running]
- ArgoCD: [Synced & Healthy]
- 총 다운타임: [~1분]

---

## 🚀 롤백 완료 후 조치

1. 근본 원인 분석 (RCA)
    - 컷오버 실패 원인 파악
    - 문제 해결 방안 수립

2. 재시도 계획 수립
    - 문제 해결 후 재컷오버 일정 협의
    - 점검 사항 보완

3. Before 지표 재측정
    - 롤백 후 정상 상태 확인
    - 다음 컷오버 준비
