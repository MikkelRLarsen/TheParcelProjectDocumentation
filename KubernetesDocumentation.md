# Kubernetes Opsætning – Staging og Production

## 1. Cluster Layout

| Environment | Control Plane VM | Worker VM | Notes |
|-------------|-----------------|-----------|-------|
| Staging     | Staging-CP      | Staging-Worker | Testmiljø, integration og E2E tests |
| Production  | Prod-CP         | Prod-Worker    | Live-miljø, kun godkendte artefakter |

- Separate kubeconfigs per cluster  
- Kubernetes version ensartet  
- Miljøer isolerede fysisk og netværksmæssigt  

---

## 2. Self-Hosted Runners

- Staging Runner: på Staging control plane  
- Production Runner: på Production control plane  
- Kører som system service, har lokal kubeconfig  
- GitHub Actions workflow bruger labels og environment:

```yaml
jobs:
  deploy-staging:
    runs-on: [self-hosted, staging]
    environment: staging
    needs: build
    steps:
      - uses: actions/checkout@v6
      - run: kubectl apply -f k8s/ {subject to change}

  deploy-prod:
    runs-on: [self-hosted, production]
    environment: production
    needs: deploy-staging
    steps:
      - uses: actions/checkout@v6
      - run: kubectl apply -f k8s/ {subject to change}
