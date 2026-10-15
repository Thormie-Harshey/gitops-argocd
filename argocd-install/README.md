# ArgoCD Installation

This installation was tested on a local kind cluster with ArgoCD v3.5.5

## Prerequisites
- A running Kubernetes cluster and `kubectl` configured for it.

## 1. Create the namespace
```bash
kubectl apply -f argocd-install/namespace.yaml
```

## 2. Install ArgoCD
```bash
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=900s
```

## 3. Access the UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
In a second terminal, get the initial admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
Open https://localhost:8080, accept the self-signed certificate warning, and log in as `admin`.

After first login, change the password and delete the `argocd-initial-admin-secret`.

## 4. Deploy the applications
```bash
kubectl apply -f argocd-install/argocd-app.yaml
kubectl get applications -n argocd
```
