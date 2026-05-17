# Portfolio Web - Helm Chart

## Rolling Update Configuration

- **Replicas**: 2
- **Strategy**: RollingUpdate
  - maxUnavailable: 0
  - maxSurge: 1
- **Health Probes**:
  - readinessProbe: GET /health (10s delay, 5s period)
  - livenessProbe: GET /health (30s delay, 10s period)

## Deployment

ArgoCD automatically syncs changes from this repository.

Last updated: 2026-03-27
