# Immich GitOps Deployment

This repository contains a **complete Kubernetes GitOps setup** for [Immich](https://immich.app), including:
- Immich application
- Postgres (with SealedSecrets for credentials)
- Redis (standalone with authentication via SealedSecrets)
- NFS-based persistent storage
- GPU scheduling for ML pods
- Traefik ingress with self-signed TLS via cert-manager
- Staging & Production environments
- FluxCD and ArgoCD GitOps manifests
- Optional pure Kustomize deployment

---

## Prerequisites

### Install Argo CD
``` bash
kubectl apply -n argocd kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Install CRDs

Cert Manager
``` bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml        
```

## 📂 Repository Structure

```text
immich-gitops/
├── charts/immich-stack/        # All-in-one Helm chart
├── environments/
│   ├── staging/                # Staging namespace, values, certs, sealed secrets
│   └── production/             # Production namespace, values, certs, sealed secrets
├── flux/                       # FluxCD HelmRelease and source configs
├── argocd/                     # ArgoCD Application definitions
└── README.md                   # This file
```

---

## 🚀 Deployment Options

### **1. FluxCD**

1. Install FluxCD:
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux install
```

2.	Add this repo as a Flux Git source:
```bash
flux create source git immich-gitops \
  --url=https://github.com/YOUR_USER/immich-gitops \
  --branch=main
```

3.	Apply the environment:
```bash
kubectl apply -f environments/staging/kustomization.yaml
kubectl apply -f flux/
```

### **2. ArgoCD**

1.	Install ArgoCD:
```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2.	Apply the Application manifest:
```bash
kubectl apply -f argocd/app-staging.yaml
```

3.	Sync in the ArgoCD UI.

### **3. Pure Kustomize (kubectl only)**
For quick manual deploys without GitOps:
```bash
kubectl apply -k environments/staging
helm upgrade --install immich charts/immich-stack \
  --namespace immich-staging \
  -f environments/staging/values-staging.yaml
```

## 🔑 Secrets Management
All DB and Redis credentials are stored as SealedSecrets
Encrypted using your provided SealedSecrets controller public key

To rotate credentials:

```bash
kubectl create secret generic immich-postgres-user \
  --from-literal=username=newuser \
  --from-literal=password=newpass \
  --namespace immich-staging \
  --dry-run=client -o yaml \
| kubeseal --controller-namespace=sealed-secrets --controller-name=sealed-secrets -o yaml > environments/staging/sealed/db-sealed.yaml
```

## 📦 Storage
* NFS server: 192.168.18.96
* Export path: /mnt/cluster/nfs
* PVCs: immich-library-pvc, immich-mlcache-pvc

## 🎯 GPU Scheduling
ML pods run only on GPU nodes:
```yaml
nodeSelector:
  gpu: "true"
```

## 🔒 TLS
* Using self-signed TLS via cert-manager ClusterIssuer:
``` yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

## 🛠 Troubleshooting
* Check pods:
```bash
kubectl get pods -n immich-staging
```

* Describe ingress:
``` bash
kubectl describe ingress immich -n immich-staging
```
* Logs:
``` bash
kubectl logs deploy/immich -n immich-staging
```

---

✅ With this, your repo is **100% complete** — you can now create it locally by copying the files I’ve given into `immich-gitops/` and running:
```bash
zip -r immich-gitops.zip immich-gitops/