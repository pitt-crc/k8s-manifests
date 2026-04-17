# Bootstrapping ArgoCD

This document outlines procedures for bootstrapping ArgoCD on a Kubernetes cluster.
The steps outlined below only need to be performed once.
Once finished, all further deployments to the cluster should be managed through ArgoCD itself.

## Prerequisites

- `kubectl` configured for the target cluster
- `helm` installed locally

## Bootstrap Procedure

ArgoCD is first installed manually via Helm with a minimal configuration.
Once running, manifest files are applied which configure ArgoCD to take over managing its own installation.
After this handoff, future updates/deployments are deployed exclusively through ArgoCD.

### 1. Verify your cluster context

Ensure `kubectl` is configured to administrate the correct Kubernetes cluster.

```bash
kubectl config current-context
```

If necessary, update the current context to the desired cluster.

```bash
kubectl config use-context <context-name>
```

### 2. Install Argo CD

Install ArgoCD using `helm`.

The `--enable-helm` option must be set at install time so ArgoCD can leverage Helm to reinstall itself later on.
Enabling a load balancer (ideally with a known external IP) is also useful to ensure you can connect to and
validate the installed application.

Careful attention should be paid to the destination namespace.
Migrating ArgoCD to a new namespace after installation can break the deployment.
The chosen value should be treated as permanent.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 9.4.17 \
  --create-namespace \
  --set configs.cm."kustomize\.buildOptions"="--enable-helm" \
  --set server.service.loadBalancerIP=10.7.84.16 \
  --set server.service.type=LoadBalancer \
  --wait
```

### 3. Migrate to Self Management

Applying the manifests below hands control of the cluster over to ArgoCD.
Once properly configured, argoCD will detect every subdirectory under the `apps/` directory.
From this point on, direct `kubectl` edits to managed resources will be reverted by ArgoCD automatically.

```bash
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/application.yaml
```

### 4. Retrieve Initial Credentials

ArgoCD will automatically create an `admin` user at install time.
The default password for this user can be fetched using the command below.

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
