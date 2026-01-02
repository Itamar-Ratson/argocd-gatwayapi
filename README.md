# ArgoCD Getting Started Guide (Gateway API)

Local development setup using KinD, Cilium (with WireGuard encryption), Traefik, Helm, cert-manager, Sealed Secrets, and Kubernetes Gateway API.

## Prerequisites

- Docker
- kubectl
- Helm
- KinD
- htpasswd (from apache2-utils)
- kubeseal

## 1. Create KinD Cluster

```bash
kind create cluster --config kind-config.yaml
```

The cluster won't be Ready until a CNI is installed (next step).

## 2. Add Helm Repos & Build Dependencies

```bash
helm repo add cilium https://helm.cilium.io
helm repo add jetstack https://charts.jetstack.io
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo add traefik https://traefik.github.io/charts
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

for chart in cilium cert-manager sealed-secrets traefik argocd; do
  helm dependency build ./helm/$chart
done
```

## 3. Install Components

```bash
# Cilium CNI
helm install cilium ./helm/cilium -n kube-system -f ./helm/cilium/values.yaml
kubectl wait --for=condition=Ready nodes --all --timeout=300s

# Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

# cert-manager
helm upgrade --install cert-manager ./helm/cert-manager -n cert-manager --create-namespace
kubectl wait --for=condition=Available deployment/cert-manager -n cert-manager --timeout=120s

# Sealed Secrets
helm upgrade --install sealed-secrets ./helm/sealed-secrets -n sealed-secrets --create-namespace
kubectl wait --for=condition=Available deployment/sealed-secrets -n sealed-secrets --timeout=120s

# Traefik
helm upgrade --install traefik ./helm/traefik -n traefik --create-namespace
kubectl wait --for=condition=Available deployment/traefik -n traefik --timeout=120s

# ArgoCD (after setting up sealed secret below)
helm upgrade --install argocd ./helm/argocd -n argocd --create-namespace
kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=300s
```

## 4. Create ArgoCD Admin Secret

```bash
HASH=$(htpasswd -nbBC 10 "" "admin" | tr -d ':\n' | sed 's/$2y/$2a/')
SECRET_KEY=$(openssl rand -base64 32)
sed -e "s|REPLACE_WITH_BCRYPT_HASH|$HASH|" \
    -e "s|REPLACE_WITH_SECRET_KEY|$SECRET_KEY|" \
    example.argocd-admin-secret.yaml > argocd-admin-secret.yaml

kubeseal --controller-name sealed-secrets --controller-namespace sealed-secrets \
  -o yaml < argocd-admin-secret.yaml > helm/argocd/templates/sealed-argocd-secret.yaml

rm argocd-admin-secret.yaml
```

## 5. Access ArgoCD

Access UI at <https://argocd.localhost> (accept the self-signed certificate warning).

Login: `admin` / `admin`

```bash
argocd login argocd.localhost --grpc-web
```

## Troubleshooting

```bash
# Verify Cilium encryption
kubectl -n kube-system exec ds/cilium -- cilium-dbg encrypt status

# Check Gateway/HTTPRoute/Certificate status
kubectl get gateway -n traefik
kubectl get httproute -n argocd
kubectl get certificate -n traefik

# Traefik logs
kubectl logs -n traefik -l app.kubernetes.io/name=traefik

# Reset password if needed
HASH=$(htpasswd -nbBC 10 "" "admin" | tr -d ':\n' | sed 's/$2y/$2a/')
kubectl patch secret argocd-secret -n argocd \
  -p "{\"stringData\": {\"admin.password\": \"$HASH\", \"admin.passwordMtime\": \"$(date -Iseconds)\"}}"
kubectl rollout restart deployment/argocd-server -n argocd
```

## Cleanup

```bash
kind delete cluster --name argocd-dev
```
