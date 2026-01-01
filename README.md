# ArgoCD Getting Started Guide

Local development setup using KinD, Traefik, and Helm.

## Prerequisites

- Docker
- kubectl
- Helm
- KinD

## 1. Create KinD Cluster

Create `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: argocd-dev
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

```bash
kind create cluster --config kind-config.yaml
```

## 2. Install Traefik

Create `traefik-values.yaml`:

```yaml
hostNetwork: true

ports:
  web:
    port: 80
  websecure:
    port: 443

nodeSelector:
  ingress-ready: "true"

tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Equal
    effect: NoSchedule
```

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik -n traefik --create-namespace -f traefik-values.yaml
```

## 3. Install ArgoCD

Create `argocd-values.yaml`:

```yaml
dex:
  enabled: false

notifications:
  enabled: false

applicationSet:
  enabled: false

configs:
  params:
    server.insecure: true
```

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd -f argocd-values.yaml
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
```

## 4. Create IngressRoute

Create `argocd-ingress.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`argocd.localhost`)
      services:
        - name: argocd-server
          port: 80
  tls: {}
```

```bash
kubectl apply -f argocd-ingress.yaml
```

## 5. Access ArgoCD

Get admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Access UI at https://argocd.localhost (accept self-signed cert warning).

CLI login:

```bash
argocd login argocd.localhost --grpc-web --insecure
```

## Cleanup

```bash
kind delete cluster --name argocd-dev
```
