# infra-repo
K8s GitOps 인프라 매니페스트 레포

## 구조
```
apps/
  portfolio-web/   # Next.js 포트폴리오 웹 서비스
  jenkins/         # CI 플랫폼
  grafana/         # 모니터링
```

## 주의
- *.secret.yaml, *-credentials.yaml 은 git에 커밋 금지
- Secret 관리는 Sealed Secrets 사용 예정
