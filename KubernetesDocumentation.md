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
```
# Kubernetes Opsætning – Essentiel

## Cluster Layout

* Separate kubeconfigs per cluster
* Staging og Production isolerede

## Self-Hosted Runners

* Kører på control plane
* Brug kubeconfig med dedikeret ServiceAccount

## ServiceAccount & RBAC

* Én ServiceAccount per service (service1–service4)
* Role: kun `get`, `patch`, `update` på én deployment
* RoleBinding: binder ServiceAccount til Role
* Token bruges i kubeconfig for runneren

### Eksempel ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github-runner-deployer-service1
  namespace: production
```

### Eksempel Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: single-deployment-updater-service1
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  resourceNames: ["service1-deployment"]
  verbs: ["get", "patch", "update"]
```

### Eksempel RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-service1-deployer
  namespace: production
subjects:
- kind: ServiceAccount
  name: github-runner-deployer-service1
  namespace: production
roleRef:
  kind: Role
  name: single-deployment-updater-service1
  apiGroup: rbac.authorization.k8s.io
```

## Deployment Metadata

* `metadata.name` → refereres i Role `resourceNames`
* `namespace` → RBAC scope
* Labels → selector
* Annotations → dokumentation
