# [6-9] 컷오버 전 E2E 사전 검증 - 최종 결과

날짜: 2026-04-01 14:53 KST
환경: WSL2 (OpenVPN 연결, /etc/hosts: EC2 EIP 매핑)

## ✅ 성공 항목

### 1. HTTPS 전체 경로 정상
- 메인 페이지: 200 OK
  - Next.js 캐싱 동작 확인 (x-nextjs-cache: HIT)
  - Auth 쿠키 정상 설정
- /health: 200 OK (uptime: 340700초 = 94.6시간)
  - readinessProbe 정상 동작 ✅
- Jenkins: 301 → /jenkins/ (정상 리다이렉트)
- MinIO: /s3/portfolio-web 200 OK

### 2. 인프라 경로 검증 완료
- EC2 iptables: 80/443 DNAT 정상 (패킷 처리 확인)
- WireGuard 터널: EC2 ↔ Bastion 정상 통신
- Bastion Nginx: 프록시 설정 정상
  - 소스 IP: <wg-ec2-ip> (EC2 WireGuard)
  - IP allow 규칙 통과 확인

## ⚠️ 예상된 동작 (문제 아님)

### 1. HTTP(80) `/` → 444 Empty reply
- 원인: `location / { return 444; }`
- 목적: certbot HTTP-01 챌린지 전용
- 동작: 응답 없이 연결 종료 (의도된 설정)
- 해결: [6-10] 컷오버 시 정상 사용 예정

### 2. S3 서브도메인 403 Forbidden
- 원인: Self-signed 인증서
- 해결: [6-10] Let's Encrypt 발급 후 정상화 예정

## 🎯 컷오버 준비 상태
- ✅ 전체 HTTPS 경로 검증 완료
- ✅ WireGuard 터널 정상 동작 확인
- ✅ Bastion Nginx 프록시 정상
- ✅ /health 엔드포인트 정상 (readinessProbe)
