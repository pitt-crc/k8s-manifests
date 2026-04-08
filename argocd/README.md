# ArgoCD Bootstrap

## Prerequisites

- `kubectl` configured and pointing at the target cluster
- `helm`



## Install

Verify your context before proceeding:

```bash
kubectl config current-context
```

Install Argo CD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 9.4.17 \
  --create-namespace \
  --set configs.cm."kustomize\.buildOptions"="--enable-helm" \
  --wait
```

> `--enable-helm` is required for Kustomize to render Helm charts and must be
> set at install time. It is also declared in `argocd/values.yaml` so GitOps
> keeps it in place after the first sync.

Get the Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Access the UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open https://localhost:8080 and log in as `admin`.

Apply the App-of-Apps

```bash
kubectl apply -f apps/app-of-apps.yaml
kubectl apply -f apps/applicationset.yaml
```

ArgoCD will sync itself and all child applications from Git. Manual `kubectl`
edits will be reverted automatically (`selfHeal: true`).


---

## Appendix

### HA Install (alternative to Helm)

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

### Update LDAP Bind Password

```bash
kubectl -n argocd patch secret argocd-secret \
  --type='merge' \
  -p '{"data":{"dex.ldap.bindPW":"'$(echo -n "yourpassword" | base64)'"}}'
```