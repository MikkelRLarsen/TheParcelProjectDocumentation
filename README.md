Husk at tilføje architecture decision record (ADR) til eksamenprojekt. Gerne flere.

Husk at tilføje C4 model til eksamensprojekt


```
on:
  push:
    branches: [main]

jobs:
  deploy-minikube:
    runs-on: self-hosted

    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-kubectl@v4

      - uses: azure/k8s-set-context@v4
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBECONFIG_MINIKUBE }}

      - uses: azure/k8s-deploy@v5
        with:
          namespace: default
          manifests: |
            k8s/deployment.yaml
          images: |
            ghcr.io/myuser/myapp:${{ github.sha }}
```
